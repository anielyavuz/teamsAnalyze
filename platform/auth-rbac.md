# Authentication, sessions, RBAC, admin, audit

## 1. Sign-in (Entra ID OpenID Connect)

- Flow: **authorization code + PKCE** using MSAL's confidential client (`initiate_auth_code_flow` /
  `acquire_token_by_auth_code_flow`). Scopes: `openid profile` only — **no Graph scopes**. Why: the web
  sign-in must not be able to read tenant data; data access is the collector app's job.
- **Separate app registration** for sign-in (`AUTH_CLIENT_ID`, redirect URI `<WEB_BASE_URL>/auth/callback`,
  platform "Web"), distinct from the collector app. Why: least privilege; the sign-in app has no
  application permissions. Single-tenant registration (authority `https://login.microsoftonline.com/<tenant-id>`).
- Recommended tenant-side hardening (documented for operators): enterprise app "Assignment required = Yes"
  with a group of allowed users; Conditional Access as the organisation requires.
- The auth flow state (MSAL flow dict incl. `state`, `nonce`, PKCE verifier) is kept server-side, keyed by
  a short-lived, HttpOnly, SameSite=Lax pre-auth cookie (10 min). Never in the URL, never logged.
- On callback: validate the ID token (MSAL does signature/issuer/audience/nonce), then:
  - `tid` must equal the configured tenant id, else `login_denied` (`reason=wrong_tenant`).
  - Identity is the `oid` claim (lower-case). UPN (`preferred_username`) and name are display data only.
  - Find `app_users` by `oid`. If none: find a pre-provisioned row with the same UPN (case-insensitive)
    and **no oid yet**, bind the oid (audit `user_bound`). Otherwise `login_denied` (`reason=not_registered`).
  - Status `disabled` → `login_denied` (`reason=disabled`).
  - Success: create session, update `last_login_at`, upn/name refresh, audit `login_ok`.
- Microsoft tokens are discarded after validation. Nothing from Microsoft is stored except oid, UPN, name.
- Logout: delete session row, clear cookie, audit `logout`; optionally redirect to the Entra logout endpoint.

## 2. Sessions

| Property | Value |
|---|---|
| Cookie | `ta_session`: random 256-bit token; HttpOnly; SameSite=Lax; `Secure` when `WEB_BASE_URL` is https (required in production); Path=/ |
| Storage | `app_sessions(token_hash PK = SHA-256(token), user_id, csrf, created_at, last_seen_at, expires_at, ip, user_agent)` — raw token never stored |
| Lifetime | 8 h absolute (`SESSION_ABSOLUTE_HOURS`), 30 min idle (`SESSION_IDLE_MINUTES`) |
| Per request | load session + user status + grants **from the DB every request** — role changes and disabling take effect immediately |
| last_seen | updated at most once per minute (avoid write per request) |
| Revocation | user disabled → delete all their sessions; admin "end sessions" button; directory sync (Step 3) ends sessions of accounts disabled in Entra |
| Cleanup | expired sessions deleted by the collector run |

Session fixation: a new token is issued at login; pre-auth cookie deleted.

## 3. CSRF and request hardening

- Every state-changing request is POST (never GET) and carries the session's CSRF token in a hidden form
  field or `X-CSRF-Token` header; compared in constant time. Failure → 403, audit `csrf_failed`.
- All admin POSTs go through one wrapper: CSRF check → permission check → lockout check → change in a
  transaction → audit with before/after → redirect. No admin handler bypasses it.
- Response headers: `Content-Security-Policy` (default-src 'self'; script-src 'self' with nonces for inline
  scripts; style-src 'self' 'nonce-…'; img-src 'self' data:; font-src 'self' data:; frame-ancestors 'self'),
  `X-Content-Type-Options: nosniff`, `Referrer-Policy: same-origin`, `Strict-Transport-Security` behind
  https, `Cache-Control: no-store` on all authenticated responses.
- Redirect targets after login/language switch: same-site relative paths only.
- Unauthenticated: HTML routes redirect to `/login`; API routes return 401 JSON `{"error": "unauthorized"}`.

## 4. Data model

| Table | Columns (meaning) |
|---|---|
| `app_users` | `id`, `oid` (unique, nullable until first login), `upn`, `display_name`, `status` (`active`/`disabled`), `locale`, `created_at`, `created_by`, `last_login_at` |
| `roles` | `id`, `key` (stable English key for seeded roles, null for user-created), `name`, `description`, `is_system`, timestamps |
| `role_modules` | `role_id`, `module`, `can_view`, `scope_type` (default `all`), `scope_value` (pipe-separated values), `detail_level` (`full`/`masked`/`aggregate`); PK (role, module). **No row = no access** |
| `user_roles` | `user_id`, `role_id`, `valid_until` (null = no expiry), `granted_by`, `granted_at` |
| `app_sessions` | section 2 |
| `access_audit` | `id`, `ts`, `user_id`, `upn`, `action`, `module`, `detail` jsonb, `ip`, `request_id` — append-only |

Seeded roles (keys stable, names/descriptions translated at display): `system_admin` (all modules incl.
admin; is_system: cannot be deleted or have its modules changed). Other templates (`it_uc`, `management`,
`facilities`, `finance`) are created empty in Step 0 and receive module rows in the migration of the module
they concern, with conservative detail levels (`aggregate` for management, `masked` for IT/UC on people
modules).

## 5. Authorization model

```
user ──(active, not expired)── user_roles ──▶ roles ──▶ role_modules(module, scope, detail)
```

- A user's modules = union of `can_view` rows of their non-expired roles, intersected with modules of
  enabled packages. **Empty set means nothing** — no code path may treat "empty" as "all".
- Scope and detail per module are combined across roles as defined in `scope-masking.md` §3.
- Platform screens are modules too: `admin` (users, roles, settings, audit, preview) and `logs`
  (logs and permissions). They are unscoped (all or nothing).
- **Enforcement is server-side on every route**: the module gate is a dependency of each module route and
  API; a module's data is not computed for a user without the grant. Hiding with CSS/JS is never access
  control. Forbidden → 403, audit `forbidden` (module, route).
- The CLI is fully privileged (it runs as the owner role on the server) and is not exposed to users.

## 6. Lockout protection

Refuse (and audit `change_denied` with reason) any admin change after which:
- no active user holds a non-expired `system_admin` role; or
- the acting user would remove their own `admin` module access, their own `system_admin` role, or disable
  themselves.
Checks run inside the same transaction as the change; on violation the transaction rolls back.

## 7. Admin module screens

| Screen | Functions |
|---|---|
| Users | list (status, roles, last login), add (Step 0: by UPN; Step 3+: directory picker storing oid), assign roles with optional expiry, disable/enable, end sessions, preview |
| Roles | matrix overview (module × role, read-only, cell shows view/scope/detail), create, rename, delete (not system), edit |
| Role editor | per module: view checkbox, scope type + values, detail level; modules of disabled packages marked "(module off)"; warnings when chosen scope leaves people/rooms invisible (no department / no location) |
| Settings | enabled packages, min group size, self-view, default language; timezone shown read-only when set by environment |
| Access audit | filter by user/action/module/time, search; unauthorised attempts highlighted |
| Preview as user | renders shell and modules exactly as the selected user would see them (same scope, detail, masking) |
| Check permissions | button to re-run the permission evaluation (rate-limited) |

Preview rules: requires `admin`; computed with the **target user's** grants and view; any API calls made
by the preview page (lazy tabs, search) must carry the preview context and are evaluated with the target's
view, never the admin's; audited as `preview` / `preview_tab` with the target user id.

## 8. Audit

`access_audit` is append-only (UPDATE/DELETE blocked at DB level). Every access and change is recorded
with **structured, language-independent detail** (keys and codes, not rendered sentences), and mirrored to
`app_log` as `auth.<action>` (without person names).

| Action | Detail |
|---|---|
| `login_ok`, `login_failed`, `login_denied`, `logout`, `user_bound` | reason code, tid mismatch flag |
| `view_shell`, `view_tab` | module, view summary (`scope-masking.md` §8) |
| `preview`, `preview_tab` | target user id, module, view summary |
| `forbidden`, `csrf_failed`, `change_denied` | route/module, reason |
| `user_add`, `user_roles`, `user_status`, `sessions_revoked` | target user id, before/after |
| `role_create`, `role_update`, `role_delete` | role id, before/after cells `scope:values/detail` |
| `settings_update`, `packages_update` | key, before/after |
| module actions | declared in the module manifest, e.g. `lookup_user`, `search_calls`, `export_calls`, `view_call` |

Readers of old audit rows must tolerate unknown keys (rows are never migrated).

## 9. Bootstrap

CLI `admin grant-admin <upn>`: create or update the user, grant `system_admin`, audit `cli_grant_admin`.
Also `admin add <upn> --role <key|name>`, `admin disable <upn>`, `admin list`, `admin roles`. Emergency
access works without the web UI.
