# Module: Licensing (seats and idle licenses)

> Module key: `licensing` · Report tab: **License analytics** · Data: scheduled ingest (directory + subscriptions + Microsoft 365 activity report) into PostgreSQL.
> **Decision: no prices, no cost.** Customers do not enter purchase prices; the module reports seats and people, never money.

---

## 1 Purpose

Answer two questions with evidence:

1. **Are purchased licenses assigned?** Seats bought vs. assigned per subscription → unassigned seats.
2. **Are assigned licenses used by the person?** For each licensed person: last real service use, and a state (using / not using / newly assigned / cannot be verified / account disabled) with a human-readable reason ("assigned on 1 May 2026, no use since then").

"Used" means **service usage** (Exchange, OneDrive, SharePoint, Teams, Viva Engage), not sign-ins: signing in without using anything is not a reason to keep a paid license.

## 2 Scope / Out of scope

**In scope**

- Subscriptions (`subscribedSkus`): bought (enabled prepaid units), suspended/warning units, assigned (consumed), capability status.
- Directory users with assigned licenses, account enabled, member/guest.
- Idle-license detection using `getOffice365ActiveUserDetail(period='D180')` as the **main method** (no Entra ID P1 needed).
- Optional extra precision from `signInActivity` **only if** the tenant has Entra ID P1/P2.
- Optional enrichment from other modules' data when those modules are enabled (calls, messaging, copilot, rooms).
- Per-subscription usage breakdown and a licensed-people table.

**Out of scope**

- Prices, monthly cost, wasted money, meeting cost (removed by owner decision; the reference had a price table and meeting-cost estimation — do **not** rebuild it).
- Assigning or removing licenses (read-only module).
- Per-person live license/plan/policy detail → that is the [user-lookup](../user-lookup/DESIGN.md) module.
- Unlicensed Copilot Chat usage (not in Graph).

## 3 Dependencies

| Platform doc | Use |
|---|---|
| [architecture](../../platform/architecture.md) | module/tab registration, package `licensing` |
| [graph-client](../../platform/graph-client.md) | paged GET, report download (302 → CSV, no Authorization header to the download URL), throttle/retry |
| [database](../../platform/database.md) | migrations (append-only), raw JSONB + derived columns, settings table, ingest runs |
| [ingest-scheduler](../../platform/ingest-scheduler.md) | scheduled directory/subscription sync and daily activity-report fetch; module-off gate |
| [permissions](../../platform/permissions.md) | features `licensing`, `o365`, `signin` (optional) |
| [auth-rbac](../../platform/auth-rbac.md) | module `licensing` view right |
| [scope-masking](../../platform/scope-masking.md) | RLS on person rows, masking of names, small-group suppression |
| [logging](../../platform/logging.md), [i18n](../../platform/i18n.md), [ui-shell](../../platform/ui-shell.md), [testing](../../platform/testing.md), [deployment](../../platform/deployment.md) | standard |

**Assumption:** the directory user sync (users + manager + department) is owned by the platform because scope (department/manager) needs it. If the platform already syncs `assignedLicenses` with the directory, this module reuses that table; otherwise it adds the license columns described in §5.

Module dependencies (all optional): calls (last call/meeting), messaging (last message day, Teams report last activity), copilot (Copilot SKU usage), rooms (room accounts are excluded from idle detection), user-lookup (admin drill-down link).

## 4 Microsoft Graph endpoints & permissions

| Method | Endpoint | Version | Permission (least privileged first) | Alternatives | Why |
|---|---|---|---|---|---|
| GET | `/subscribedSkus` | v1.0 | `LicenseAssignment.Read.All` | `Organization.Read.All`, `Directory.Read.All` | Bought/assigned seats, capability status, part number |
| GET | `/users?$select=id,displayName,userPrincipalName,accountEnabled,userType,createdDateTime,assignedLicenses,mail,department,officeLocation,city,country&$expand=manager($select=id)&$top=999` (paged) | v1.0 | `User.Read.All` | `Directory.Read.All` | Who holds which license; account state; scope attributes |
| GET | `/reports/getOffice365ActiveUserDetail(period='D180')` → 302 → CSV | v1.0 | `Reports.Read.All` | — | Per-service last activity **and license assignment dates** (main idle method) |
| GET | `/users?$select=id,signInActivity` (paged) | v1.0 | `AuditLog.Read.All` **+ Entra ID P1/P2 license** | — | Optional: last sign-in, extra precision only |

Feature map entries: `licensing` (subscribedSkus + users; core-ish), `o365` (Reports.Read.All), `signin` (optional; AuditLog.Read.All, plus a live probe because the role alone is not sufficient).

### Quirks and real-API findings

- **`signInActivity` without P1/P2 → 403** with a message that the tenant "doesn't have premium license" (observed 2026-09-26) even when `AuditLog.Read.All` is granted. Permission check therefore needs a **live probe** (a single `$top=1` request issued directly, not via the error-logging path) and must show "restricted" (yellow), not "missing" (red).
- **The activity report CSV has no user ID.** Match to directory users by **lower-cased UPN**.
- **Concealed names:** if the tenant setting "Display concealed user, group, and site names in all reports" is on (Microsoft's default), UPNs arrive as hashes and nothing matches. Detect (UPN without `@`) and show a setup card with the path: Microsoft 365 admin center → Settings → Org settings → Services → Reports → untick the option.
- **Empty "last activity date" is ambiguous** (never vs. not within the period — undocumented). Mitigation: fetch the longest period **D180** and decide empty dates together with the **license assignment date** columns.
- **New tenants return header-only CSVs** for a while even with correct permissions; this is not a permission problem.
- **CSV may start with a UTF-8 BOM**; decode with BOM stripping. Header spelling/case/spacing is inconsistent across reports — normalize headers to snake_case and read by name, never by position.
- **Skype for Business** columns still exist but the service is retired; do not display them.
- **Free / self-service SKUs** (e.g. `TEAMS_FREE`, `FLOW_FREE`) appear in `assignedLicenses` but not in `subscribedSkus`. Count them as "not purchased" and exclude them from idle detection and seat math.
- Report data lags 24–48 hours; the snapshot date is the CSV's `Report Refresh Date`.
- Never log the 302 download URL (it serves data without authentication for several minutes).

## 5 Data model

**Subscriptions** (`license_skus`, replaced on each sync):

| Column | Meaning |
|---|---|
| sku_id (PK) | Microsoft SKU GUID |
| sku_part_number | e.g. `ENTERPRISEPACK` |
| prepaid_enabled | bought (active) units |
| prepaid_suspended, prepaid_warning, prepaid_locked_out | units in those states |
| consumed | assigned units |
| capability_status | Enabled / Warning / Suspended / LockedOut |
| applies_to | `User` / `Company` (only `User` SKUs are shown) |
| raw | Graph object as returned |
| updated_at | |

**Directory license columns** (on the platform's directory table, or a module table keyed by the same person key `user:<object id>`): `sku_ids` (array of assigned SKU IDs), `account_enabled`, `user_type`, `created_at`, `last_synced_at` (rows not seen in the latest sync are excluded from views), `raw`.

**Microsoft 365 activity snapshot** (`o365_user_activity`, one row per snapshot date × UPN; same snapshot date rewritten):

| Column | Meaning |
|---|---|
| snapshot_date, upn_lower (PK) | report refresh date, lower-cased UPN |
| display_name, is_deleted | from CSV |
| exchange_last, onedrive_last, sharepoint_last, yammer_last, teams_last | per-service last activity date |
| has_exchange … has_teams | license flags per service (text "True"/"False" → boolean; null when unknown) |
| first_assign | earliest service license assignment date |
| last_any, last_any_service | newest service activity and which service |
| assigned_products | text as returned |
| period, raw, run_id | |

**Optional sign-in snapshot** (only if P1): `user_key`, `last_sign_in`, `last_non_interactive_sign_in`, `fetched_at`.

**Settings:** `idle_days` (default **45**).

**Derived view (per licensed person):** key, name, UPN, user type, account enabled, SKU IDs, last call/meeting (if calls module), last message day and Teams report last activity (if messaging module), o365 last/last service/first assign/snapshot, overall last activity = newest of all sources, and the source of that value.

**Coverage function:** the earliest moment from which our own activity data is reliable (earliest successful call-ingest window start, earliest messaging report day with rows). Used for the fallback rule.

RLS: person rows visible only for people in the viewer's scope (via the directory key); subscriptions are tenant-wide (no person data) and visible to anyone with the module.

## 6 Ingest / request flow

1. **Directory + subscriptions** (every scheduled run or at least hourly; 2–3 requests): `subscribedSkus` → replace `license_skus`; `/users` paged → upsert directory license columns, set `last_synced_at`. Own ingest-run row, source `graph.directory`.
2. **Microsoft 365 activity report** (once per day is enough; data is daily): `getOffice365ActiveUserDetail(period='D180')` → 302 → CSV (no Authorization header on the download) → normalize → upsert snapshot by refresh date. Source `graph.o365`. Empty CSV → warning event, no failure.
3. **Sign-in (optional, P1 only)**: only if the `signin` feature probe says available.
4. Each step is gated by its feature (permission present + package enabled); a missing permission skips the step with a warning (`ingest.skipped`), never a 403 error storm. One step's failure does not stop the others.

## 7 Derived logic & metric definitions

**Seats (per user-assignable subscription):**
- bought = prepaid enabled units; assigned = consumed; **unassigned = max(bought − assigned, 0)**.
- Subscriptions with status ≠ Enabled are flagged.

**Person state** (evaluated in order; only people with ≥ 1 *purchased* SKU are candidates):

| Order | State | Rule | Counts as "not using" |
|---|---|---|---|
| 1 | `room` | person is a room account (rooms module) | no — measured on the Rooms tab |
| 2 | `disabled` | account disabled but holds a purchased license | yes |
| 3 | `free` | only free/self-service SKUs | excluded from all KPIs |
| 4 | `active` | newest activity from any source is younger than `idle_days` | no |
| 5 | `idle` (M365 report) | report has an activity date and it is older than `idle_days` | yes |
| 6 | `idle` (since assignment) | report row has no activity, license assigned ≥ `idle_days` ago | yes — reason "assigned on {date}, no use since then" |
| 7 | `new` | report row has no activity, license assigned < `idle_days` ago | no — reason "license assigned on {date} ({n} days)" |
| 8 | `idle` (own data) | no report row; our own coverage ≥ `idle_days` and no activity in it | yes — reason "our own data" |
| 9 | `unverified` | no report row; coverage shorter than `idle_days` | no |

- Last activity source label: call, message, Teams report, or "M365: {service}".
- **Per-subscription usage:** for people holding the SKU: using / not using (idle + disabled) / uncertain (new + unverified) / room; usage rate = using ÷ (using + not using + uncertain). For a Copilot SKU, "using" comes from the Copilot usage report (last `copilot_idle_days`), not general activity — only when the copilot module is enabled.
- **Service cell** per person: green = used within `idle_days`, amber = older, red = licensed for the service but never used, grey = not licensed.

## 8 UI

Tab **License analytics**:

- **Status line:** directory synced at …, M365 report snapshot date, coverage note ("our own data covers N days; idle detection without the M365 report needs {idle_days}").
- **KPI strip:** bought seats, unassigned seats, licensed people, using, not using (with "no activity in the last {n} days"), uncertain.
- **Subscriptions table:** part number (friendly name when known) + status, bought, assigned, unassigned, using / not using / uncertain, usage-rate bar.
- **Licensed people table** (person-level, scope + masking): name, UPN, licenses, service cells, last activity + source, days idle, state pill with reason. Sort: disabled, idle, unverified, new, active, room, free. Search box (visible text only), state/SKU filters, CSV export (respecting masking).
- Admin-only: a person row links to User lookup.

States:
- Permission missing (`licensing` feature) → locked tab card with the exact permission and grant steps.
- `o365` missing → yellow card: idle detection falls back to own data; lists `Reports.Read.All`.
- Report empty (new tenant) → one-line note "Microsoft has not produced activity reports yet".
- Concealed names detected → red setup card with the admin-center path.
- `signin` restricted (no P1) → small yellow note; nothing else changes.
- No purchased subscriptions → note-line.

## 9 Access control

- RBAC module key `licensing`; new module grants itself only to the system admin role by default.
- RLS: person-level rows (directory license columns, activity snapshots, sign-in snapshot) filtered by the viewer's scope; snapshot rows visible only if they match an in-scope directory person by UPN. Subscriptions unrestricted.
- Masking: person names/UPNs in the people table and tooltips go through the platform masking pipeline; aggregate views show counts only; small groups suppressed per platform threshold.
- Audit: tab view (`view_tab` with module), CSV export (`export` with module and row count).

## 10 Logging events

| Event | Level | Context |
|---|---|---|
| `directory.skus` | INFO timed | `skus` |
| `directory.users` | INFO timed | `users`, `pages` |
| `directory.done` | INFO | users, bought, assigned |
| `directory.failed` | ERROR (exc) | |
| `o365.fetch` | INFO timed | `rows`, `snapshot_date`, `period` |
| `o365.empty` | WARNING | report returned header only |
| `o365.concealed` | WARNING | `rows`, `concealed_rows` |
| `o365.unmatched` | INFO | count of report rows without a directory match |
| `o365.no_permission` / `o365.failed` | ERROR | |
| `signin.restricted` | WARNING | probe found no P1 |
| `licensing.build` | INFO timed | people, states histogram |
| `ingest.skipped` | WARNING | feature, reason |

Never log UPNs in bulk, the report download URL, or tokens.

## 11 i18n notes

- State names and reasons are translatable with parameters (dates via the platform date formatter, plural for days).
- Product and service names (Exchange, OneDrive, SharePoint, Teams, Viva Engage) and SKU part numbers are not translated.
- Stored data holds state codes and dates only; reason text is produced at render time.

## 12 Testing & acceptance criteria

- Fake CSV rows covering all nine states (including BOM, mixed header case, concealed UPN) → every state assigned as in §7.
- Seat math: bought 25 / assigned 2 → unassigned 23; never negative.
- Free SKU not counted in seats or idle detection.
- Missing `Reports.Read.All` → ingest skips with warning, tab shows fallback card, states use own data or `unverified`.
- `signInActivity` 403 premium → `signin` restricted, no ERROR rows.
- Download URL never appears in logs (leak test).
- RLS: a department-scoped viewer sees only in-scope people; KPIs computed over in-scope rows.
- Screenshots: light, dark, 390 px; locked and fallback states.

## 13 Implementation steps for an agent

1. Confirm the platform directory sync; add license columns or the module table (§5) via a new migration.
2. Add `license_skus` and `o365_user_activity` tables, `idle_days` setting, RLS policies, the per-person activity view (security invoker) and coverage function.
3. Register features `licensing`, `o365`, `signin` (optional, with live probe) and the `licensing` package.
4. Implement Graph calls (subscribedSkus, paged users, report download) through the graph client.
5. Implement CSV normalization + row derivation as pure functions (no I/O, no logging).
6. Wire ingest steps into the scheduler with feature gates; log events (§10).
7. Implement the state machine (§7) as a pure function with unit tests.
8. Build the tab (KPIs, subscriptions, people table, states) using ui-shell components; masking markers on person-level sections.
9. Translations, screenshots, leak/RLS tests.
10. Document real-API findings in module notes.

## 14 Open questions / future

- Should "uncertain" people be listed in a separate section to drive follow-up?
- Per-service idle (e.g. has Exchange use but never Teams → "Teams add-on unused") for add-on SKUs such as Teams Premium or Teams Phone.
- Recommended threshold per SKU (`idle_days` per SKU instead of one global value).
- Use Microsoft's "Product names and service plan identifiers" CSV (shared with user-lookup) for friendly names.
- Prices/cost stay removed unless the owner revisits that decision.
