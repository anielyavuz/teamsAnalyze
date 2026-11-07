# Graph permissions: features, packages, checks, locked UI

In most customer tenants some permissions will be missing. The system must say exactly what is missing,
for which function, and how to fix it — without the operator reading logs.

## 1. Model

```
Package (operator switches on/off) ──contains──▶ Modules (RBAC, screens)
        └────────────────────────────contains──▶ Features ──need──▶ Graph application permissions
```

**Feature** (declared by the platform or a module manifest):

| Field | Meaning |
|---|---|
| `key` | English snake_case, unique (`calls`, `directory`, `room_devices`) |
| `label`, `why` | Translatable: what it enables and why the permission is needed |
| `needs` | List of requirements. Each requirement is `"A|B|C"`: any one option satisfies it; the **first option is the least-privileged and recommended**. All requirements must be satisfied |
| `optional` | true = the module works without it (reduced); missing optional → INFO, yellow note, never a lock |
| `beta` | true = depends on a Graph beta endpoint (`graph-client.md` §6) |
| `probe` | Optional live check function (section 4) |
| `module` | Owning module key (or `core`) |
| `sections` | UI sections/tabs to lock when the feature is not usable |

**Package**: `key`, `label`, `why`, `modules` (list of module keys), `features` (list), `core` (bool).

- The **core** package is always enabled and cannot be switched off. In Step 0 it has no Graph features; in
  Step 3 it gains `directory` (`User.Read.All|Directory.Read.All`) used for scope and admin user picker.
- Enabled packages: setting `enabled_packages` (comma list). **Missing setting = core only.** Why: a new
  installation must not ask for every permission (least privilege). The reference defaulted to "all", which
  produced warnings for permissions the operator never intended to grant.
- A disabled package: its modules disappear for everyone (server-side), its ingest steps do not run
  (`ingest.module_off`), no probes run, its features show "not required" and never produce warnings.

## 2. Evaluation from the token

At the start of every collector invocation (and on demand from the admin screen, rate-limited to once per
minute):

1. Acquire the app token; decode `roles` (application permissions), `appid`, `app_displayname`, `tid`.
   No HTTP request beyond the token.
2. For each feature: for each requirement, the first option present in `roles` is "granted"; requirements
   with no present option are "needed". Status `missing` if anything is needed, else `ok`.
3. For `ok` features of enabled packages that define a probe: run the probe (section 4); a failing probe
   sets `limited` with a detail code.
4. Upsert into `permission_checks` and store app identity + full role list in `settings`.
5. Log `permissions.missing` (WARNING; INFO if optional) / `permissions.limited` (INFO) per problem feature
   of an enabled package, then `permissions.checked` (INFO) with ok/problem lists, role count and count of
   granted-but-unused roles.

Table `permission_checks`:

| Column | Meaning |
|---|---|
| `feature` PK | feature key |
| `status` | `ok` / `missing` / `limited` |
| `granted` | text[] options found |
| `needed` | text[] unsatisfied requirements (each `"A|B"`) |
| `detail` | language-independent code `code[:arg]` (e.g. `no_premium`, `device_api:404`, `probe_failed:403`, `tenant_switch_off`) — rendered to text at display time |
| `checked_at` | timestamp |

Why store, not compute per page: pages must not call Graph to decide what to lock; the last check is the
truth and its age is shown.

## 3. Gating

- **Ingest:** a step runs only if its package is enabled and all its non-optional features are `ok` or
  `limited`-but-allowed (the step decides for `limited`). Otherwise `ingest.skipped` (WARNING) with the
  feature and needed permissions — never a 403 storm.
- **Live requests (web):** the module checks the feature status before calling Graph; if the status is
  stale or a request still returns 403, it shows the locked card and logs `<module>.no_permission`.
- **UI:** sections/tabs listed in `sections` of a non-usable feature render the locked card instead of
  content. If data from earlier permitted runs exists, it may be shown with the note "from the last
  permitted collection, may be outdated", and contradictory status cards are hidden.

## 4. Live probes

Some permissions are not sufficient alone: a licence is required (e.g. sign-in activity needs Entra ID
P1/P2), a tenant switch must be on, an application access policy must exist, or a beta API may be retired.

Probe rules:

- Minimal request (`$top=1`, `$select` one field), through the shared throttle.
- The refusal is **expected**, so the request is made in quiet mode (no `graph.error`).
- Map the response to a detail code; never store the response body.
- One probe per feature per check; total probe budget per check ≤ 10 requests.
- Each probe is documented in the owning module doc §4.

## 5. Unused permissions

`unused = token roles − all options of features in enabled packages`. Shown on the permissions screen and
in About/transparency as "granted but not used by this installation — can be removed". Permissions used
only by disabled packages are listed separately.

## 6. UI: status chip, locked cards, instructions

- **Top bar chip:** "Permissions complete" or, in danger colour, "N permissions missing" (count of
  non-optional missing features in enabled packages), linking to the permissions screen. Visible to roles
  with the Logs/permissions screen; others see nothing.
- **Locked rail entry:** a module whose primary feature is missing shows a lock glyph and danger colour; the
  module still opens to show the instruction card (no data).
- **Locked card** (section or module level), content:
  1. What does not work and why (feature `why`).
  2. App name and application (client) id of the collector app.
  3. The **exact permission name(s)** to add, least-privileged first, alternatives in smaller text.
  4. Steps: Entra admin center → App registrations → the app → API permissions → Add a permission →
     Microsoft Graph → **Application permissions** → select → Add → **Grant admin consent for <tenant>** →
     wait a few minutes → run "Check permissions again" (admin button) or the next scheduled collection.
  5. A direct link to the app's API permissions page in the Azure/Entra portal built from the app id.
  6. For `limited`: the detail text (licence requirement, tenant switch, retired beta API) instead of
     steps 3–5.
- **Optional feature missing:** yellow info note inside the module, no lock.
- **Permissions screen:** (1) unique list of permissions required by enabled packages, granted ones green,
  required vs optional separated; (2) granted-but-unused list; (3) table grouped by package: feature, needed
  permission, status, detail, last checked.

## 7. Adding a Graph feature (checklist)

Repeated from `CONVENTIONS.md` §2 for convenience: feature row with least-privileged-first `needs`; package
membership; ingest/live gating; probe if a role is insufficient; locked card in the right section; transparency popovers (§8) incl.
`not_needed` list and, for live screens, trace step declarations; test by
setting the feature `missing` and `limited` in a test DB and taking screenshots; document in module doc §4.

## 8. Permission transparency (shared platform component)

Every module screen that uses Graph (live or through ingest) tells the viewer which permissions it needs;
every live screen also tells what it actually called. Both are **generated** from the permission registry
and from a request trace — never hand-written text. Reference implementation:
[modules/user-lookup/DESIGN.md](../modules/user-lookup/DESIGN.md). Visual spec: `ui-shell.md` §6.

### 8.1 (A) "Which permissions does this need?" — per module screen

Generated from the module's feature rows plus the current token roles. Content, in order:

1. Table: **Application permission** | **Required / optional** | **Used for** (feature `why`). One row per
   requirement; the permission cell follows the rendering rule in 8.3.
2. How to grant: Azure Portal → Microsoft Entra ID → App registrations → *<collector app name>* → API
   permissions → Add a permission → Microsoft Graph → **Application permissions** → select → Add →
   **Grant admin consent**. Includes the app's client id and the direct portal link (§6).
3. Legend: ✓ held / ✗ missing. "Held" is read from the `roles` claim of the collector's current access token
   (no extra Graph request). A note states that a newly granted or removed permission becomes visible after
   the token is renewed (about 1 hour) or the service restarts.
4. "Not needed" list: permissions this screen deliberately does **not** need (e.g. all `*.ReadWrite.*`
   write permissions, content permissions such as chat/transcript content) — declared in the module
   manifest as `not_needed` with a short reason.
5. Who can use the screen: the module key and the roles that currently grant it (names only, from RBAC).

Data source for "held": the web process may read token claims from the platform token provider (decode
only); if the collector credentials are not configured in the web process, it uses the stored
`settings.app_roles` from the last permission check and says so ("as of <checked_at>").

### 8.2 (B) "What did this use?" — per section of live screens

Every live screen builds a **request trace** while serving the request. Each step is declared once in the
module (step registry): `step key → {method, endpoint template, permission requirement ("A|B"), purpose}`.
Every Graph call is made through `trace.call(step_key, fn, …)`, which records per step:

| Field | Meaning |
|---|---|
| `status` | `ok`, `denied` (403), `not_found` (404), `error` (other) |
| `ms` | total duration |
| `count` | number of calls of this step in the request (repeat count) |
| `results` | number of items returned (when meaningful) |

A section's popover lists the steps that section used: method + endpoint template (ids replaced by
`{id}`; never real ids, names, query text or secret parameters), permission cell (8.3), purpose, status,
duration, calls, results. Because the text comes from the same trace that executed the calls, what is
explained is always what was done. Traces are not persisted; summary counts may be logged
(`<module>.trace`, DEBUG).

### 8.3 Permission cell rendering rule

- Options in registry order: the **least-privileged option first**, then alternatives in muted text:
  "(alternative: X or Y)".
- Each option the app holds gets a green ✓ next to it.
- The row gets a leading ✓ if **any** option is held, else a red ✗.
- When only an alternative is held, the recommended (first) option is still shown, greyed, so the operator
  can move to least privilege.
- Status is never conveyed by colour alone (✓/✗ glyph + accessible text "held"/"missing").

## 9. Admin consent notes for operators (shown in About and deployment docs)

- Application permissions always require tenant admin consent.
- Usage reports conceal user names by default (Microsoft 365 admin center → Settings → Org settings →
  Services → Reports → "Display concealed user, group, and site names in all reports"). When concealed,
  report rows cannot be joined to people; modules relying on reports detect hashed identifiers and show a
  note.
- Some APIs need tenant-side configuration beyond consent (application access policies for online meeting
  artefacts, Teams admin settings). The owning module documents them and its probe detects them.
