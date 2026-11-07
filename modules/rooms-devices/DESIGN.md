# Rooms & Devices (video conferencing) — Design

RBAC module key: `rooms` (tab "Rooms and devices") · Package: `rooms` ·
Features: `rooms` (required), `room_devices` (degradable), `room_presence` (optional), `room_places` (optional) ·
Admin page: Room locations (module `admin`)

---

## 1 Purpose

Give facilities and IT a fleet-first view of Microsoft Teams Rooms: which room accounts exist and how they are
licensed (Basic / Pro / other tiers), which meetings each room joined and how much of business hours the rooms
are actually used, what the audio quality is on the room side, and — while Microsoft still exposes it — the
health of each room device (offline, critical, pending updates, disconnected peripherals, sign-in state) with a
history of availability. It must stay useful for hundreds of rooms and keep working when the beta device API is
retired.

## 2 Scope / Out of scope

In scope:
- Room inventory from directory + licenses; room license tiers; Basic limit; extra person licenses on room accounts.
- Usage, utilization (business-hours), day × hour occupancy, room-side audio quality — from call records.
- Device inventory and health from beta `/teamwork/devices` (+ health, activity, configuration, operations).
- Device availability history from periodic snapshots (outages, flaps, night/weekend shutdown patterns).
- Peripherals and software freshness.
- Optional current presence of room accounts; optional Exchange room catalogue (building, floor, capacity).
- Attention list (rules with severity), fleet summary, searchable/paged room list, room detail.
- Admin room locations (override, bulk assignment), location-based RBAC scope.
- Graceful degradation of every device-dependent part.

Out of scope:
- Remote actions (restart, update, diagnostics) — the reference app has `ReadWrite` capability but deliberately
  does not act; the beta API is being retired (§14).
- Room booking / calendar (ghost meetings, booked vs used) — §14.
- Teams phones and displays beyond an inventory count ("other Teams devices").
- Money: license prices and meeting cost → `../licensing/DESIGN.md`.

## 3 Dependencies

- `../call-ingest/DESIGN.md` — `call_records`, `call_participants`, `derived.timeline.lanes` (room lane =
  lane whose `person_id` equals the room account object id), stream fields, classification, outcome.
- Directory sync (core platform): `directory_users` (key `user:<oid>` lower-case, display name, UPN,
  account_enabled, sku_ids, officeLocation, city, department, manager, last_synced_at) and `license_skus`
  (sku_id, sku_part_number, prepaid/consumed units) — `../../platform/ingest-scheduler.md`,
  `../licensing/DESIGN.md`.
- `../call-search/DESIGN.md` — drill-down with `room=<key>`.
- `../meeting-analytics/DESIGN.md` — capacity vs attendees (future).
- Platform: `../../platform/graph-client.md`, `../../platform/permissions.md` (live probe support),
  `../../platform/database.md`, `../../platform/auth-rbac.md`, `../../platform/scope-masking.md`
  (location scope), `../../platform/ui-shell.md`, `../../platform/i18n.md`, `../../platform/logging.md`,
  `../../platform/ingest-scheduler.md`, `../../platform/testing.md`.

## 4 Microsoft Graph endpoints & permissions

| Method | Endpoint | Version | Permission (least privileged first) | Alternatives | Why |
|---|---|---|---|---|---|
| GET | `/users?$select=…assignedLicenses,officeLocation,city,department,accountEnabled` | v1.0 | `User.Read.All` | `Directory.Read.All` | room accounts, tiers (core directory sync) |
| GET | `/subscribedSkus` | v1.0 | `LicenseAssignment.Read.All` | `Organization.Read.All`, `Directory.Read.All` | room license pool (core) |
| GET | call records | v1.0 | `CallRecords.Read.All` | — | usage, utilization, room-side quality (call-ingest) |
| GET | `/teamwork/devices` (paged) | **beta** | `TeamworkDevice.Read.All` | `TeamworkDevice.ReadWrite.All` | device inventory, health status, activity state, current user |
| GET | `/teamwork/devices/{id}/health` | beta | same | | connection, login status (Teams/Exchange), peripherals, hardware, software freshness |
| GET | `/teamwork/devices/{id}/activity` | beta | same | | active peripherals |
| GET | `/teamwork/devices/{id}/configuration` | beta | same | | configured peripherals, software versions |
| GET | `/teamwork/devices/{id}/operations` (paged) | beta | same | | remote operation history (read only) |
| POST | `/communications/getPresencesByUserId` body `{"ids": [≤650 ids]}` | v1.0 | `Presence.Read.All` | — | optional: room in a meeting / free / offline now |
| GET | `/places/microsoft.graph.room?$top=100` | v1.0 | `Place.Read.All` | — | optional: building, floor, capacity, booking type, tags |

Request cost per rooms run: `⌈devices/100⌉` list pages (page size undocumented; 100 assumed) + 4 × devices
(parallel, shared throttle) + `⌈room accounts/650⌉` presence + catalogue pages. 500 devices ≈ 2,000 requests ≈
40 s at 50 req/s.

### Real-API findings and lifecycle (dated)

| Date | Finding | Consequence |
|---|---|---|
| 2025-12-08 → | Microsoft is gradually retiring the Teams device management Graph APIs (message center **MC1183294**); recommended replacement is the Teams Rooms Pro Management portal, **not** an API. | Device health is a degradable feature. Design for "unavailable" from day one. |
| 2026-09-27 | Beta `/teamwork/devices` still returned 200 in the reference tenant. | Keep using while it works; probe every run. |
| 2026-09-27 | On Basic-licensed rooms, Teams/Exchange login status and peripherals report `unknown`. Pro includes peripheral health management, so this is the probable cause (state it as "probable" in the UI). | Do not raise peripheral alerts on `unknown`. |
| 2026-09-27 | Device `health.connection.lastModifiedDateTime` is the moment the connection state **last changed**, independent of when we polled. `0001-01-01…` means unknown. | Outage start/end accurate to the second even with sparse polling. |
| 2026-09-27 | Propagation delay: a device switched off at T still showed `connected` in a poll ~2 min later. | Treat state changes as accurate only after a few minutes; do not alert on a single fresh poll. |
| — | `Place.Read.All` was not available in the reference tenant: the catalogue path is **not live-tested**. | Treat as optional; fall back to directory officeLocation/city. |
| 2026-08-14 | Microsoft licensing docs: Teams Rooms Basic — free, max **25** per tenant, **one** room system per account, cannot license a Teams Panel. Pro — unlimited, Pro Management, peripheral health, device analytics, remote configuration. Room accounts should hold only Teams Rooms Basic/Pro (plus calling/security if needed), no other licenses. | Rules in §7.6. |

Error mapping for the device list: 403 → permission problem (ERROR `rooms.devices_forbidden`);
400 / 404 / 410 / 501 → API unavailable (WARNING `rooms.devices_unavailable`, setting `room_devices_api =
"unavailable <date>"`), skip only the device part. Any other error → re-raise (step fails, other steps continue).
Per-device part failure → keep the parts that worked; keep the previous stored value for failed parts
(`rooms.device_partial`).

Permission probe (each run, via the platform permission check): token roles first; then a live GET of the device
list with `$top=1` sent directly (not through the logging GET helper, so an expected rejection does not log as an
error). Non-200 → feature status `limited` with explanation "Teams device API not responding (beta, being
retired)".

## 5 Data model

| Table / object | Grain | Content |
|---|---|---|
| `room_devices` | device (current) | device_id, device_type, manufacturer, model, serial (serialNumber or uniqueId), account_key (`user:<oid>` lower-case from `currentUser.id`), account_name, health_status (`healthy`/`nonUrgent`/`critical`/`offline`/`unknown`), activity_state, connection_status, conn_changed_at, teams_login, exchange_login, updates_pending, enrolled_at (`createdDateTime`), modified_at, asset_tag, notes, raw_device, raw_health, raw_activity, raw_configuration, raw_operations (jsonb), last_seen_run, updated_at |
| `room_device_snapshots` | device × run | taken_at, health_status, connection_status, conn_changed_at, updates_pending, teams_login, exchange_login. Precision of state = polling cadence; precision of transitions = `conn_changed_at` |
| `room_presence` | room account × run | taken_at, availability, activity (raw Graph values) |
| `room_places` | catalogue room | place_id, email_lower, display_name, capacity, building, floor_number, floor_label, label, booking_type, tags[] |
| `room_location_overrides` | room account | room_key, location, updated_by, updated_at (admin-entered) |
| `v_rooms` | room account | key, user_id, display_name, upn, account_enabled, tier, room_skus[], other_skus[], devices, location, location_source (`override`/`places`/`office`/`city`), capacity, building, floor |
| `v_room_calls` | room × record | room_key, call_id, start_at, class, connected, duration_s, role, people, room_seconds (sum of room lane segments), audio (room lane sent audio), share_sent, share_recv, video_sent, platform |
| `room_usage(from, to)` | room | records, meetings, p2p, group_calls, unanswered, seconds, avg_people, share_used, video_used, audio_measured, audio_poor, avg jitter/loss/rtt, connection types, last_used |
| `room_hourly(from, to)` | room × local day × hour | room seconds, segments split at hour boundaries |
| `room_business_hours(from, to)` | range | business hours in window (per settings) |
| `v_device_events` | device transition | at, from_state, to_state, approx (bool), missed_flap (bool) |
| `device_intervals(from, to)` | device interval | state (`online`/`offline`/`unknown`), start_at, end_at (clipped), ongoing |
| `device_hourly(from, to)` | device × local dow × hour | online_s, offline_s |

Room definition: an account with a Teams Rooms license (tier not null) **or** an account signed in on a room-type
device (`teamsRoom`, `collaborationBar`, `surfaceHub`, `touchConsole`, `teamsPanel`). Unlicensed signed-in
accounts appear so the "no room license" rule can fire.

License tier from `sku_part_number` (case-insensitive patterns; Microsoft names change, keep patterns in one
function): contains `teams_rooms_pro` → `pro`; `teams_rooms_premium` → `premium`; `teams_rooms_basic` → `basic`;
starts with `meeting_room` or contains `teams_rooms_standard` → `standard`; starts with `mcocap` or contains
`shared_device` → `shared`. Multiple → highest rank pro > premium > standard > basic > shared.

Device columns are derived by pure functions from the raw device + parts (peripherals list: room camera, content
camera, microphone, speaker, communication speaker, each display; hardware: compute, HDMI ingest; software:
firmware, Teams client, admin agent, partner agent, OS, Company Portal with current/available/freshness).

Settings: `room_hours_start` 9, `room_hours_end` 18, `room_workdays` 1..5, `room_idle_days` 14,
`room_devices_api` (status string).

RLS: room tables are not person data, but `v_room_calls` exposes meeting participants through call-ingest
tables (already RLS). Location scope filters `v_rooms` by `location`.

## 6 Ingest / request flow

Step `rooms` runs after the directory sync (room accounts and licenses must be fresh) on every scheduled run
(or once a day if configured; snapshot cadence drives flap detection precision).

1. Gate: package `rooms` on and feature `rooms` granted; else skip (`ingest.module_off` / `ingest.skipped`).
2. `start_run(source = graph.rooms)`.
3. Catalogue (only if `room_places` status ok): list places → upsert `room_places`; failure → WARNING
   `rooms.places_failed`, continue.
4. Devices (if `room_devices` status ok or limited): list; on unavailable/forbidden see §4; else fetch the
   4 parts per device in parallel (shared throttle), upsert `room_devices` (failed parts keep previous values),
   insert one `room_device_snapshots` row per device with `conn_changed_at`; set `room_devices_api = ok`.
5. Presence (if `room_presence` ok and room accounts exist): batches of 650 → `room_presence`; failure → WARNING.
6. Finish run `ok` / `partial` (any part failed); log `rooms.done` (rooms, devices, health distribution, notes).

Usage, utilization and quality need no requests (call records).

## 7 Derived logic & metric definitions

Windows: 7 / 30 / 90 days (default 30) for usage and quality; attention rules use 30 days.

### 7.1 Usage

- Room records: records with the room account as participant. Split meetings (linked), p2p, group calls,
  unanswered (not connected).
- Room seconds: sum of the room lane's segments (time the room was actually connected), not the record duration.
- Avg people: mean participant count (room included).
- Content share used: records where the room lane sent screen-share packets; camera used: room sent video packets.
- Last used: latest connected record.

### 7.2 Utilization

- Business-hours utilization (room) = room seconds falling in business-hour cells (local workdays,
  `[start, end)` hours) / (business hours in the window × 3600).
- Fleet utilization = Σ room business seconds / (rooms × business hours × 3600).
- Day × hour occupancy cell = Σ room seconds in that (local dow, hour) / (rooms × number of occurrences of that
  cell in the window × 3600). Segments are split at hour boundaries. Business-hour cells framed.
- Daily room-hours chart: Σ room seconds per local day / 3600.
- Ranking: ≤ 12 rooms → single list; more → top 8 and bottom 8 by utilization.

### 7.3 Room-side audio quality

Uses the room lane's sent audio stream per record (classified / poor per CQD, call-ingest §7.6). Room poor rate =
poor / measured. Requires ≥ 3 measured streams (`MIN_QUALITY_SAMPLE`) before any rate is shown or a rule fires.

### 7.4 Device availability history

- Each snapshot stores state + `conn_changed_at`.
- Transition time = `conn_changed_at` when present (accurate), else snapshot time (`approx`).
- Between two snapshots with the same stamp, state is certainly unchanged.
- Stamp changed but state equal → at least one unseen down/up (`missed_flap`, counted as ≥ 1).
- After the last snapshot → unknown (hatched); days with no data → hatched, not "online".
- Business-hours availability = online seconds in business cells / known seconds in business cells; show one
  decimal and never round 99.x% up to 100%.

Patterns per device (thresholds as named constants):

| Pattern | Rule |
|---|---|
| Offline now | last interval offline and ongoing |
| Flapping | short outages (< 2 h) + missed flaps ≥ 3 per week |
| Switched off at night | ≥ 3 nights with an outage starting 16:00–03:00 and ending 05:00–11:00 local, lasting 4–18 h |
| Switched off at weekends | an outage covering Saturday noon and Sunday noon of the same weekend |
| Insufficient data | known time < 24 h |
| Stable | none of the above |

### 7.5 Attention rules (one evaluation per room; severity Critical > Warning > Cost > Info)

| Rule key | Severity | Condition |
|---|---|---|
| `acc_off` | Critical | room account disabled |
| `no_lic` | Critical | signed in on a room device, no Teams Rooms license |
| `offline` | Critical | device `healthStatus = offline` or connection `disconnected` |
| `critical` | Critical | device `healthStatus = critical` |
| `login` | Warning | Teams or Exchange login `disconnected` (Exchange down → no calendar, no one-touch join) |
| `periph` | Warning | non-optional peripheral `disconnected` (not `unknown`) |
| `updates` | Warning | any software component `updateAvailable` |
| `nonurgent` | Warning | `nonUrgent` with no other explaining rule |
| `flappy` | Warning | pattern Flapping in the last 30 days |
| `basic_multi` | Warning | Basic tier and > 1 room device on the account |
| `idle` | Warning | no connected record for `room_idle_days` (14) days; devices all enrolled less than 14 days ago → `new` (Info) instead |
| `poor` | Warning | room poor rate ≥ 10% over ≥ 3 measured streams in 30 days |
| `no_device` | Cost | device API working and no device signed in with a licensed account |
| `extra_lic` | Cost | room account holds licenses other than room tiers (person licenses); show monthly amount if prices are set |
| `night` / `weekend` | Info | patterns (intentional shutdown is fine; maintenance/updates may be delayed) |
| `wifi` | Info | room audio streams only over Wi-Fi |
| `basic_limit` | Critical (fleet) | assigned Basic licenses > 25 |

Each rule produces `{severity, rule, params}` (codes + parameters, no text); the UI renders title, detail and the
recommended action from label functions. Device-dependent rules (`offline`, `critical`, `login`, `periph`,
`updates`, `nonurgent`, `flappy`, `night`, `weekend`, `no_device`, `basic_multi`) are **suppressed** when the
device API is not `ok` or the device data is older than 24 h (`stale`); the UI shows them as "device data
unavailable" instead of silently passing.

### 7.6 Licensing view

Room license pool (assigned / purchased per tier), Basic usage vs 25, person licenses on room accounts.
Room accounts are not people: the licensing module marks them "meeting room" (never idle, excluded from waste);
the calls person table labels them "room".

### 7.7 Graceful degradation matrix

| Device API state | Inventory/health/peripherals/software | Availability history | Attention rules | Usage/quality/licensing |
|---|---|---|---|---|
| ok | live | grows | all | full |
| limited / unavailable | last known values with "as of <date>" badge; hidden after 30 days | frozen; "no new data since <date>" | device rules suppressed, one fleet notice | full |
| forbidden (permission) | permission card | frozen | suppressed | full |
| never available | section replaced by an info card explaining MC1183294 and the Pro Management portal | hidden | suppressed | full |

Signals that remain without the device API: rooms (directory), tiers (licenses), usage/utilization and room-side
quality (call records), room platform from call lanes (`roomSystem`, `surfaceHub`), idle detection, extra licenses.

## 8 UI

Tab "Rooms and devices" (`#rooms`), lazy-loaded. Designed for hundreds of rooms (reference tested with 300 rooms
/ 11,660 meetings synthetic: build 2.5 s, ≈ 7.6 KB per room when embedded). The rebuild serves the room list
through a paged API instead of embedding all rows.

1. Window chips 7 / 30 / 90 days: change usage/quality numbers only.
2. Fleet summary KPI strip: rooms, devices with problems (offline + critical), updates pending, business-hours
   utilization, meetings + calls (room-hours), poor audio rate, room licenses (assigned/purchased, Basic limit).
   Health distribution bar below (label + icon, never colour alone).
3. Attention: **one line per rule** ("Device offline · 20 rooms") with up to 24 room chips; clicking a chip filters
   the room list and opens that room. Single-room rules show the detail inline. Max 8 lines expanded.
4. Room list: search (name, account, location, model, serial; diacritic-folded), filters (Attention, Offline,
   Idle, Updates, Cost, Location), sortable columns (name, location, health, utilization, meetings, poor rate —
   window-dependent), first 50 + "Load more". Row expands to: room alerts, usage tiles, room-side quality, last 8
   meetings (person-level), device (serial, MAC, sign-in states, 30-day snapshot counts, last 7 days availability
   strip), peripherals, software (installed → available), remote operations history, account and licenses.
5. Usage: class mix, content share / camera rates, occupancy heatmap with business-hour frame, daily room-hours,
   rooms by utilization.
6. Device availability: KPIs (business-hours availability, outages, offline now, total downtime, flapping, night
   shutdown), device timeline (worst 40 devices, night 22–07 shaded, no night band at 90 days), day × hour offline
   rate (no-data hatched), device table.
7. Inventory: model distribution; software versions per model (firmware never mixed across vendors; outdated
   count and share); other Teams devices (phones, displays, SIP).
8. Licenses: room license pool, person licenses on room accounts.
9. Permission/info cards for device API, presence, catalogue + "next step" card.

Admin → Room locations: table of rooms with effective location and its source, inline edit, select rooms → bulk
assign / clear, search; every change audited before/after. Role editor shows an info box when rooms without
location exist ("not visible under location scope; fix in Admin → Room locations or in Entra").

States: no rooms → one line "No Teams Rooms accounts found (licensed or signed in on a room device)."; device API
unavailable → see §7.7; empty usage → "No meetings joined by rooms in this range."

### Permission transparency (shared component, `../../platform/permissions.md`)

- **"Which permissions does this need?"** popover in the module header (reference: `../user-lookup/DESIGN.md`):
  lists this module's permissions least-privileged first with alternatives as "(alternative: X or Y)", ✓ per held
  option from the token `roles` claim, ✗ if none held, required/optional, what it is used for, how to grant
  (Azure Portal → App registrations → the app → API permissions → Add → Grant admin consent), and "not needed"
  permissions (module off or not used by this module).
  Rows: `User.Read.All` (alternative: `Directory.Read.All`) and `LicenseAssignment.Read.All` (alternative:
  `Organization.Read.All` or `Directory.Read.All`) — required, room accounts/tiers; `CallRecords.Read.All` — required,
  usage/quality; `TeamworkDevice.Read.All` (alternative: `TeamworkDevice.ReadWrite.All`, flagged as broader) — required
  for device health, shown with its live status (ok / limited: beta API retiring, MC1183294); `Presence.Read.All` —
  optional; `Place.Read.All` — optional.
- **No live Graph calls at view time** in this module, so no request-trace popover. Instead each data section has
  a **"What fed this?"** popover: ingest step, Graph endpoint, permission, and last successful run time
  (inventory/licenses → directory step, `/users` + `/subscribedSkus`; usage/utilization/quality → step `calls`;
  device health/availability/inventory → step `rooms`, `beta /teamwork/devices` + parts, `TeamworkDevice.Read.All`,
  with the device API state; presence → `getPresencesByUserId`, `Presence.Read.All`; catalogue → `/places/microsoft.graph.room`,
  `Place.Read.All`). Presence and device probes run at **ingest** time, not view time; the popover therefore shows
  the last run's outcome per call type (status OK/403/404/error, request count, total ms) recorded in the run's
  `errors`/`meta`, not a live trace.

## 9 Access control

- Module `rooms`. Scope types: all rooms / selected locations (exact match on effective location — never use
  `lower()`; locale-specific case folding broke matching in the reference).
- Detail levels: full / masked / aggregate apply to **people** shown in room detail (organizers, participants of
  recent meetings). Room accounts are not people and remain visible (known decision: room account e-mail
  addresses remain visible in aggregate mode).
- Admin room locations requires module `admin`; POST through the platform admin action wrapper (CSRF + audit).
- Audit actions: `view_tab` (rooms, scope), `room_location` (before/after, room, source), `forbidden`.
- Drill-down to call search (`room=<key>`) only when the viewer also has module `calls` with non-aggregate view.

## 10 Logging events

| Event | Level | Context |
|---|---|---|
| `rooms.places` | INFO | rooms, pages, duration_ms |
| `rooms.places_failed` | WARNING | error |
| `rooms.devices_list` | INFO | devices, pages, duration_ms |
| `rooms.devices_detail` | INFO | devices, workers, partial, duration_ms |
| `rooms.device_partial` | WARNING | device_id, errors (part names) |
| `rooms.devices_unavailable` | WARNING | status, error (truncated) |
| `rooms.devices_forbidden` | ERROR | error |
| `rooms.presence` | INFO | accounts, presences, duration_ms |
| `rooms.presence_failed` | WARNING | error |
| `rooms.done` | INFO | rooms, devices, health distribution, errors |
| `rooms.failed` | ERROR | exc |
| `admin.room_location` | INFO | room, op (set/clear/bulk), count |
| `report.tab_build` | INFO | module rooms, rooms, duration_ms |

## 11 i18n notes

- Store raw Graph enums (health status, connection, freshness, presence availability, device type, operation
  type); translate in label functions: health (Healthy, Attention, Critical, Offline, Unknown), connection
  (Connected, No connection, No information), freshness (Up to date, Update available, Unknown), presence
  (In a meeting, In a call, Presenting, Busy, Do not disturb, Available, Away, Offline, Unknown), device types
  (Teams Rooms on Windows / on Android, Surface Hub, Touch console, Teams panel, IP phone, Teams display, Phone,
  SIP device, Analog (SIP)), operations (Restart, Diagnostics, Software update, Configuration, Management agent
  configuration), tiers (Pro, Premium, Standard, Basic, Shared).
- Peripheral and software component names are translatable labels keyed by Graph field names.
- Attention rule texts: title / detail / action templates with named parameters (device name, counts, dates).
- Business hours shown in locale time format; location values are data (not translated).

## 12 Testing & acceptance criteria

- [ ] Synthetic fleet (test DB only): 300 rooms, mixed tiers, 11k meetings; tab ≤ 3 s; room list API page ≤ 300 ms.
- [ ] 61 devices × 30 days hourly snapshots with stable / night / weekend / flapping / offline-now profiles:
      each pattern detected, no false positives on stable.
- [ ] `conn_changed_at` honoured: outage start equals the stamp, not the poll time.
- [ ] Device list returns 404 / 410 / 501 → device section degraded, all other sections intact, setting updated,
      WARNING logged; 403 → permission card, ERROR logged.
- [ ] One device part fails → other parts stored, previous value kept for the failed part.
- [ ] Basic: 26 assigned → fleet Critical; Basic account with 2 devices → Warning.
- [ ] Peripheral `unknown` never raises `periph`.
- [ ] Room poor rule needs ≥ 3 measured streams.
- [ ] Location scope: user scoped to one location sees only those rooms; diacritic location names match exactly.
- [ ] Admin bulk location change audited with before/after.
- [ ] Masked viewer: no person names in recent meetings; aggregate viewer: no recent meetings.
- [ ] Screenshots light/dark/390 px; attention chips keyboard accessible.

- [ ] "Which permissions does this need?" popover lists exactly: the 6 permissions above with their alternatives; ✓/✗ follow the token roles snapshot
      (test with a snapshot holding none, the least-privileged option, and only an alternative); grant steps shown.
- [ ] Every data section's "What fed this?" popover shows ingest step, endpoint, permission and the last
      successful `ingest_runs` time; device section shows the last device-API state (ok / limited / unavailable / forbidden) and per-endpoint status counts of the last rooms run.

## 13 Implementation steps for an agent

1. Ensure directory sync provides room-relevant columns (account_enabled, sku_ids, officeLocation, city).
2. Migration: tables and functions in §5, tier function, settings defaults, location override table, RLS/grants.
3. Pure derive helpers for device columns, peripherals, software, place rows.
4. Graph wrappers (beta device list + 4 parts, presence batches, places) with the error mapping in §4.
5. Permission features `rooms`, `room_devices` (with live probe), `room_presence` (optional), `room_places`
   (optional); package `rooms`.
6. Ingest step `rooms` (§6) with logs.
7. SQL: `v_rooms`, `v_room_calls`, `room_usage`, `room_hourly`, `room_business_hours`, `v_device_events`,
   `device_intervals`, `device_hourly`.
8. Rule engine (codes + params) and pattern detector (constants), degradation states (§7.7).
9. Tab builder + paged room list API + room detail API; renderer with label functions.
10. Admin room locations page + audit; role-editor info box.
11. Tests §12; screenshots; leak scan.

## 14 Open questions / future

- Post-retirement data source for device health: Teams Rooms Pro Management has no public Graph replacement
  announced; candidate partial sources to evaluate: Intune managed devices for Windows-based rooms
  (`DeviceManagementManagedDevices.Read.All`), vendor clouds. Decide only after MC1183294 completes.
- Booking vs actual use (ghost meetings): room mailbox calendars via `Calendars.Read` limited by an Exchange
  application access policy to room mailboxes only.
- Capacity vs attendees ("large room, small meetings") once `Place.Read.All` is granted and live-tested.
- Remote actions: out of scope while read-only; would need `TeamworkDevice.ReadWrite.All`, confirmation flows and
  audit — and the API is being retired.
- Snapshot cadence: more frequent device polls improve flap counts (4 requests per device per poll).
