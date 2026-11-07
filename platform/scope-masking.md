# Scope and masking (security-critical)

Every role × module grant answers three questions: **may the user open the module**, **whose data**
(scope), and **how much of each person** (detail level). This document is normative. Any change to it, or
to code implementing it, passes the security review gate (`CONVENTIONS.md` §8).

Two independent layers enforce it, and both must hold:

1. **Database (scope):** RLS on the restricted web role returns only rows in scope.
2. **Presentation (detail):** person references are rendered according to the viewer's detail level for
   that person; person-level blocks are not built in aggregate mode; a leak guard verifies the output.

## 1. Directory: the source of scope (Step 3)

Core feature `directory` (`User.Read.All|Directory.Read.All`). Ingest step `directory` runs every
collector invocation (cheap; needed for timely revocation).

- Request: `GET /v1.0/users?$select=id,displayName,userPrincipalName,mail,department,officeLocation,city,country,jobTitle,accountEnabled,userType,assignedLicenses&$expand=manager($select=id)&$top=999`, following next links.
- Table `directory_users`: `key` PK (`user:<oid lower-case>`), `oid`, `upn`, `display_name`, `mail`,
  `department`, `office_location`, `city`, `account_enabled`, `user_type`, `manager_key`, `raw` jsonb,
  `first_seen_at`, `last_synced_run` (ingest run id), `last_synced_at`. RLS: `in_scope(key)`.
- **Current directory** = rows whose `last_synced_run` equals the latest **successful** (`ok`) directory
  run (stored in `settings.directory_current_run`). Why: a time-window heuristic breaks when a sync is
  partial or slow; a failed sync must not shrink anyone's scope.
- Accounts that became disabled or disappeared: their app sessions are ended (`auth.sessions_revoked`).
- Table `identity_names`: (`key`, `display_name`, `upn`, `first_seen_at`, `last_seen_at`) — every name/UPN
  ever observed for a key (directory sync and modules that store identities write here). Used only by the
  masking engine to catch **old names**. Owner-only: RLS on, no policy, `REVOKE ALL FROM teams_web`.
- SQL function `org_reports(root_key, all_levels)` → keys under a manager (direct or full hierarchy, depth
  cap 20, cycle-safe), over the current directory.
- Identity key formats (shared by all modules): `user:<oid>` (tenant or other-tenant users),
  `guest:<id-or-normalised-name>`, `phone:<E.164>`, `app:<app-id>`; other kinds documented by the module
  that creates them. External = not a `user:` of the home tenant (`settings.home_tenant`).

## 2. Scope types

| Module kind | Scope type | Resolution (with the owner connection, current directory) |
|---|---|---|
| people | `all` | everyone (`scope_all = on`) |
| people | `department_own` | users whose `department` equals the viewer's (exact match); empty if the viewer has none |
| people | `department` | users whose department is in `scope_value` (pipe-separated, exact match) |
| people | `team_direct` | viewer + direct reports |
| people | `team_all` | viewer + full reporting hierarchy |
| people | `self` | viewer only |
| rooms | `all` | all rooms |
| rooms | `location` | rooms whose effective location is in `scope_value` (exact match) |
| unscoped | — | all or nothing (admin, logs, about, live lookups until scoped) |

Department scopes do not add the viewer; the viewer's own data comes only through self-view (section 3).
The role editor warns when people have no department / rooms have no location (they are invisible to
those scopes) and says where to fix it.

## 3. Detail levels and combination

| Level | Shows |
|---|---|
| `full` | real display name and UPN/e-mail |
| `masked` | pseudonym "Employee #n"; no UPN, no oid; all non-identifying values (counts, durations) visible |
| `aggregate` | only totals, charts and breakdowns; every person-level block is absent |

Combination for one user and one module (all non-expired roles):

1. If any grant has `masked` or `full` → **rows mode**: use only those grants; aggregate grants of the same
   module are ignored. Why: an aggregate grant over a wider scope must not open person rows; showing the
   row-level scope's totals is the safe direction.
2. Otherwise → **aggregate mode** with the union of aggregate scopes.
3. Scope = union of resolved keys; `scope_all` if any used grant is `all`.
4. Per-person level = the **highest** level among grants whose scope contains that person; `scope_all`
   grants set the default level for everyone else.
5. **Self-view** (setting `self_view`, default on): for people modules, the viewer's own key is added to
   the scope with level `full`, in both modes. In aggregate mode it does not open other persons' rows.
6. Rooms modules: person names inside room data (organisers etc.) use the highest room grant level;
   room identity itself is not a person and is not masked.
7. **Small group:** in aggregate mode, if the number of people in scope (current directory count for
   `all`) is below `min_group_size` (default 5), the module shows only the note "small group, hidden for
   privacy"; no numbers, no source notes.
8. In aggregate mode, breakdowns **by a person attribute** (department, manager, location of people) hide
   or merge groups with fewer than `min_group_size` people into "Other".

The resolved **view** object passed to builders: `module`, `mode` (`rows`/`aggregate`), `scope_all`,
`keys`, `default_level`, `level_by_key`, `small`, `people_count`, `min_group`, `parts` (structured
description for audit).

## 4. RLS context (database layer)

Per module build (and per API call), on a web-role connection:

1. Begin a **read-only transaction**.
2. `set_config('app.scope_all', 'on'|'off', true)` — transaction-local.
3. Create temp table `scope_keys(key text primary key)` **ON COMMIT DROP** and bulk-insert the keys
   (COPY). Why a table, not a comma-separated setting: tens of thousands of keys, indexable, and it cannot
   leak to the next request on a pooled connection.
4. Run the module's queries. 5. Commit (drops the table and settings).

`in_scope(key)` = `app.scope_all = 'on'` OR key exists in `pg_temp.scope_keys`. If the setting or table is
absent the result is **false or an error — never true**. A test asserts that every RLS table returns 0
rows with no context.

Policy patterns (module docs pick one per table and name it in §5/§9):

| Pattern | Policy for `teams_web` |
|---|---|
| Person row | `in_scope(person_key)` |
| Record with participants | visible if **any** participant is in scope (`record_in_scope(id)`, SECURITY DEFINER EXISTS) |
| Participant rows of a record | own key in scope, **or** (identity is external **and** record in scope). Out-of-scope colleagues' rows are invisible |
| Identity rows | in scope or external |
| Org-wide aggregate (no person key) | `scope_all` only |
| Room rows | `scope_all` or room key in scope |
| Owner-only (names history) | RLS on, no policy, revoked |

Rows derived from a record (child tables, materialised facts) reuse the parent check.

## 5. Presentation layer (detail)

**Decision:** masking is applied **structurally in the view model**, not by searching rendered HTML.
Why: the reference replaced names in the final HTML; it worked but needed per-escaping variants, and a
display name that imitated a section marker bypassed it. Structural masking makes the safe path the only
path; a text-level guard remains as defence in depth.

Rules:

- Builders never put a person's name/UPN/oid into a view model as a plain string. They emit a
  **PersonRef** (`key`, `display_name`, `upn`, `kind`). Templates render it only through the `person`
  filter/macro, which asks the request's **Masker**:
  - level `full` → display name (+ UPN where the design shows it);
  - level `masked` → "Employee #n" (`n` assigned per response in order of first appearance; deliberately
    not stable across requests, so pages cannot be correlated);
  - in-scope record but person out of scope → "Out of scope";
  - external identity → name if the record is shown at `full`, else "External participant";
  - bots/apps → their type label (never a person).
- A PersonRef in JSON for client scripts carries the rendered label and an **opaque per-response id**,
  never the oid, unless the person is at `full`. Any attribute/selector carrying a person key contains only
  people at `full`.
- Aggregate mode: builders receive `view.mode == "aggregate"` and **do not compute** person-level blocks
  (people tables, record lists, name tooltips, per-person heatmaps, external participant lists). Templates
  additionally guard those blocks. Person-level API endpoints (lists, search, export, record detail) return
  403 in aggregate mode.
- Free text from Graph that may contain names (meeting subjects, team names, device/room names that
  embed people, error texts) must be classified in the module doc: `safe` (codes only), or
  `may_contain_names` → shown only when the whole module view is `full` for the relevant record, otherwise
  passed through the **text redactor** or replaced with a generic label.
- Exports (CSV) use the same Masker; cells starting with `= + - @` or tab/CR are prefixed to prevent
  formula injection; UTF-8 with BOM.
- **Name search side channel:** search by person name/UPN matches only people the viewer sees at `full`
  in that module. A masked or out-of-scope name yields exactly the same response as a non-existent name.
- JSON embedded into `<script>` only via the safe helper (escapes `<`, `>`, `&`, U+2028, U+2029). Text
  set in JS via `textContent`, never `innerHTML` with data.
- No rendered-HTML cache keyed by anything other than the exact user, view and language — in practice:
  no cache.

## 6. Text redactor and leak guard (defence in depth)

Name index (built per request from the owner connection, only for keys **not** at `full`): all display
names and UPNs from `identity_names` and `directory_users`, with variants: plain; HTML-escaped (with and
without quote escaping); JSON-escaped (ASCII-escaped and raw); JSON inside an HTML attribute; double
escaped. Matching: Unicode case-insensitive (UPNs arrive lower-cased from some APIs), whole-word, single
pass, **longest match wins** at the earliest position (so "Ann Lee 2" is caught even when "Ann Lee" is
visible). Non-Latin scripts are matched without word-boundary assumptions.

- **Redactor:** applied to `may_contain_names` free text before rendering (replaces with the masked label).
- **Leak guard:** applied to every masked/aggregate response body (HTML, JSON, CSV) before sending. Any
  hit → the response is replaced with a generic error (500), `security.mask_leak` is logged at CRITICAL
  with module, route and the **key** (never the name), and a test must be added. Fail closed. Setting
  `MASKING_LEAK_GUARD` exists only to allow measuring cost in tests; production default and only
  supported value is on.

## 7. Endpoint checklist (every people/rooms route)

1. Session valid → else 401/redirect.
2. Module granted and package enabled → else 403 + audit `forbidden`.
3. Resolve view (section 3) with owner connection.
4. Small group → note only.
5. Aggregate mode and person-level route → 403.
6. Web-role read-only transaction with scope context (section 4).
7. Build view model with PersonRefs; render through Masker.
8. Leak guard. 9. Audit with view summary. 10. Log `web.tab_build`/module event (no names).

Record-by-id endpoints: out-of-scope and non-existent ids return the **same** 404 body; malformed ids 400.

## 8. Audit view summary

Each `view_tab`/`preview_tab`/module action stores `detail.view = {mode, small, min_group, parts:[{scope,
values, count, detail, role}]}` — language-independent; rendered as text only on the audit screen.

## 9. Leak tests (required, automated)

Fixtures: a synthetic directory with hard names — names that are prefixes of others ("Ann Lee" / "Ann Lee
2"), Turkish İ/ı and other non-ASCII, a non-Latin script name, names containing `<`, `>`, `"`, `'`, `&`,
`</script>`, `<!--`, `-->`, JSON-breaking quotes, a UPN differing only by case, a person renamed (old name in
`identity_names`), a guest with the same first name as an employee.

For every module × detail level (full, masked, aggregate, small group) × scope type (all, department,
team, self) × endpoint (shell, tab, APIs, export, preview):
- no name/UPN/oid of a person not at `full` appears in the response (case-insensitive, all variants);
- aggregate responses contain no person-level blocks and person APIs return 403;
- RLS: web role sees exactly the expected row counts; no context → 0 rows;
- search by a masked/out-of-scope name returns the same as a random string;
- injection payloads in display names do not change page structure (DOM probe) or bypass masking.

Any new screen that shows people extends this suite before it ships.
