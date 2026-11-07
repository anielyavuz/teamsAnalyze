# About & Transparency — Design

RBAC module key: `about` (tab "About") · Package: `core` (always on) · Features: none of its own (reads the
permission check results and token roles recorded by the platform)

---

## 1 Purpose

One page that explains, to an admin or a customer's security reviewer, what the application does, which data it
pulls from Microsoft Graph and how much traffic that causes, which permissions it was granted versus which it
actually uses (least privilege), where every piece of data comes from, how long it is kept, and what it
deliberately does not read. It makes no new Graph request: everything is read from the database, the code's
feature map and recorded token roles.

## 2 Scope / Out of scope

In scope:
1. What it does + data flow diagram (Graph → ingest → PostgreSQL → web app).
2. Graph traffic calculator (interactive, client-side) with coefficient sources.
3. Permission transparency: per feature/package required permission and status; **granted vs used** diff
   (granted but unused permissions grouped by risk); recommendation to remove.
4. Data map: tab → data → Graph source → permission → API version → freshness → retention.
5. Privacy & retention: what is never read, where data lives, secrets handling, log scrubbing, data-protection
   note.
6. Technical details: app/tenant ids (masked to admins only), software version, derive version, latest
   migration, DB size, last run per source, biggest tables, known limitations.

Out of scope:
- Editing permissions or packages (Admin → Settings, `../../platform/permissions.md`).
- Logs tab (platform).
- The per-module "Which permissions does this need?" popover component itself — defined in
  `../../platform/permissions.md`; this page reuses it in aggregate form.

## 3 Dependencies

- `../../platform/permissions.md` — feature map (`key`, `needs` with `A|B` alternatives least-privileged first,
  `optional`, `why`, `tabs`), packages, enabled packages, `permission_checks` table, token roles snapshot
  (`settings.app_roles` from the `roles` claim at each check), `used_permissions(enabled packages)`,
  shared permission popover component.
- `../../platform/database.md` (migrations table, size functions), `../../platform/ingest-scheduler.md`
  (`ingest_runs` per source, cadence), `../../platform/logging.md` (scrub rules to describe),
  `../../platform/ui-shell.md`, `../../platform/i18n.md`, `../../platform/auth-rbac.md`,
  `../../platform/deployment.md` (version string), `../../platform/testing.md`.
- Module docs for the data map rows: `../call-ingest/DESIGN.md`, `../rooms-devices/DESIGN.md`,
  `../service-health/DESIGN.md`, `../messaging/DESIGN.md`, `../licensing/DESIGN.md`, `../copilot/DESIGN.md`,
  `../transcripts/DESIGN.md`, `../user-lookup/DESIGN.md`.

## 4 Microsoft Graph endpoints & permissions

No Graph calls. The page **describes** the endpoints of all enabled modules from their registered data-map
entries (each module registers: tab label, data description, endpoint(s), permission, version, freshness,
retention). Example rows (from the reference):

| Tab | Endpoint | Permission | Version | Freshness | Retention |
|---|---|---|---|---|---|
| Calls & analytics | `/communications/callRecords` (+ participants_v2, sessions/segments) | `CallRecords.Read.All` | v1.0 | ~15–30 min after call end | Graph 30 days; kept in DB |
| Messaging | `/reports/getTeamsUserActivityUserDetail` etc. | `Reports.Read.All` | v1.0 | 24–48 h delayed | Graph 28 days (`date=`); kept in DB |
| Rooms devices | `/teamwork/devices` (+ parts) | `TeamworkDevice.Read.All` | **beta (being retired, MC1183294)** | per run | current + snapshots |
| Service health | `/admin/serviceAnnouncement/issues` | `ServiceHealth.Read.All` | v1.0 | hourly | ~9–10 months from Microsoft; kept in DB |

Granted roles come from the access token's `roles` claim (decoded locally; no request). Delegated scopes of the
sign-in app (`openid profile`) are listed separately as "sign-in only, no Graph data access".

## 5 Data model

No new tables. Reads:

| Source | Use |
|---|---|
| `settings.app_roles` (+ captured_at) | granted application permissions snapshot |
| `permission_checks` | per feature: status (`ok`, `missing`, `limited`, `not_needed`), needed, detail, checked_at |
| `settings.enabled_packages` | which features count as "used" |
| `ingest_runs` | last ok/partial run per source, runs per day over last 7 days (calculator default) |
| `call_records` | count, average raw size of the newest 300 records, meetings/day, average participants |
| `directory_users`, `room_devices` | tenant-size defaults for the calculator |
| catalogue functions | DB size, biggest tables, latest migration |

Permission risk groups (static map in code, reviewed per release):
- **Can read content**: e.g. `Chat.Read.All`, `ChannelMessage.Read.All`, `Mail.Read`, `Files.Read.All`,
  `Sites.Read.All`, `OnlineMeetingTranscript.Read.All` (when transcript content reading is not used),
  `OnlineMeetingRecording.Read.All` (content).
- **Can write**: any `*.ReadWrite*`, `*.Send`, `*.Manage*`, `Directory.AccessAsUser.All`.
- **Other**: everything else.

## 6 Ingest / request flow

`GET /api/tab/about` → auth → module `about` → build (pure DB reads, ≤ 300 ms target; avoid full-table size
scans — sample newest 300 records for average size) → render → audit `view_tab` → log `report.tab_build`.
The traffic calculator runs entirely in the browser from embedded inputs and constants (no server round trip).

## 7 Derived logic & metric definitions

### 7.1 Granted vs used

- `granted` = token roles snapshot.
- `used` = union of all permission options named by features of **enabled** packages (`A|B` → both A and B count
  as "used options").
- `unused granted` = granted − used → grouped by risk; recommendation "remove in Azure Portal → App registrations
  → API permissions" (reference tenant: 111 of 121 granted were unused).
- `missing` = for each required need of an enabled feature, no option held → red.
- `over-privileged held option`: a need satisfied only by a non-first (broader) option → amber note "works, but
  the least-privileged option is X".
- Permissions only used by **disabled** packages → "not needed (module off) — can be removed".
- Snapshot age shown; if older than the last run, say so.

### 7.2 Traffic calculator

Inputs (defaults from this tenant): people, Teams devices/rooms, daily meetings + calls, average participants,
runs per day (last 7 days' actual), usage reports live (yes/no), rooms on every run (else once a day).
Per-run formulas (constants with source tag measured / docs / code / assumption, shown in a table):

| Step | Requests per run |
|---|---|
| Call list | ⌈meetings per run window / 60⌉ (list page 60, docs) |
| Call detail | 2 × new records × 1.05 (re-fetch ratio, assumption); unchanged records cost 0 |
| Usage reports | 10 when live / 56 while empty (+ same number of CSV downloads, not Graph) |
| Service health | 3 pages (~1.9 MB) when due |
| Directory | 1 + ⌈people / 999⌉ |
| Permission probes | 2 |
| Rooms | ⌈devices / 100⌉ + 4 × devices + ⌈rooms / 650⌉ + 1 |

Outputs: requests per run / day / month, daily data volume (list 1.6 KB/record; meeting raw ≈ 3 KB + 14 KB ×
participants; device 4.4 KB; directory 0.6 KB/person), estimated run time (requests / min(workers / latency,
max_rps)), DB growth (raw × 2.0). Validation in the reference: predicted 71 requests per run vs 72 measured.
Also show the Graph limits (1,500 req / 20 s per app × tenant for call records; 40 / 20 s per record and for
list) and whether the configured max rps stays below them.

### 7.3 Privacy statements (must be true; tests assert the code paths)

Never read: chat/channel message content, mail, files, meeting transcript text, recording media. Message data =
usage report counts only. Secrets only in environment/secret store; tokens never stored or logged; log scrubbing
of tokens, signed report download URLs, JWT-like strings. Person-level monitoring metrics default to aggregate
views. Data stays in the customer's database.

## 8 UI

Sections in order (§2). Components:
- Data flow: inline SVG diagram, theme-aware, with text alternative.
- Calculator: input grid, KPI outputs, per-step table, collapsible "Coefficients and their sources".
- Permissions: package-grouped table (feature, needs — least privileged first with "(alternative: X or Y)",
  status ✓/✗/limited/not needed, why) built from the **shared permission-transparency component** of
  `../../platform/permissions.md` (same rendering as each module's "Which permissions does this need?" popover,
  here expanded for all modules); then "Granted but unused" grouped by risk with counts; then "How to grant / remove"
  steps (Azure Portal → App registrations → the app → API permissions → Add / Remove → Grant admin consent) and a
  direct link to the app's API-permissions blade.
- Provenance per section ("What fed this?"): each data-map row shows ingest step, endpoint, permission and last
  successful run time (relative + absolute). There are no live Graph calls on this page, so no request-trace
  popover.
- Data map table; Privacy & retention list; Technical details (admins only for ids).

States: no permission snapshot yet → "Permissions are checked at the first import." Empty tenant → calculator
still works with manual inputs.

## 9 Access control

- Module `about`; no scope/masking (contains no person data). Tenant/app ids and DB internals visible only to
  users with module `admin`; others see the section without identifiers.
- Audit: `view_tab`.

## 10 Logging events

| Event | Level | Context |
|---|---|---|
| `report.tab_build` | INFO | module about, duration_ms, granted, used, unused, missing |
| `about.roles_stale` | WARNING | snapshot_age_s |

## 11 i18n notes

- Permission names, endpoints, versions are technical identifiers: not translated.
- Feature labels / "why" texts come from the permission map (translatable).
- Coefficient source tags translated with context ("coefficient source|measured").
- Calculator numbers via `Intl.NumberFormat(lang)`; byte units localised.

## 12 Testing & acceptance criteria

- [ ] Given a roles snapshot of 121 roles and the enabled packages, unused = granted − used exactly; risk grouping
      matches the static map.
- [ ] Missing / over-privileged / not-needed cases render ✗ / amber / grey correctly; alternatives shown as
      "(alternative: X or Y)" and ✓ per held option.
- [ ] Disabling a package moves its permissions to "not needed — can be removed".
- [ ] Calculator with reference-tenant inputs predicts within ±5% of a measured HTTP-counted run.
- [ ] Each data-map row shows last successful run time from `ingest_runs`.
- [ ] Non-admin sees no tenant/app id.
- [ ] Page build makes 0 Graph requests (fake Graph asserts none).
- [ ] Light/dark/390 px screenshots.

## 13 Implementation steps for an agent

1. Confirm platform exposes roles snapshot, permission checks, feature/package map, used-permission function,
   and the shared permission component.
2. Data-map registry: each module registers its rows (tab, data, endpoint, permission, version, freshness,
   retention, ingest source key).
3. Risk-group map for permissions.
4. Builder (DB reads) + renderer; calculator JS with constants embedded via the safe JSON-in-script helper.
5. Tests §12.

## 14 Open questions / future

- Export of the transparency page as a PDF for security reviews.
- Show the per-module request trace history (counts per endpoint over last 7 days) from logs.
- Keep the calculator constants re-measured per release on a test tenant.
