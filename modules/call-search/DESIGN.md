# Call Search (records list, detail drawer, CSV) — Design

RBAC module key: `calls` (tab "Calls") · Package: `calls` · Feature: `calls`

---

## 1 Purpose

Let an authorised user find any call, meeting or group call in the full stored history, filter and sort it on
the server, page through results, export them as CSV, and open one record in a detail drawer with a person ×
device timeline and an event flow. It is also the landing point of every drill-down link from the analytics
modules: a clicked number must open a list whose total equals that number.

## 2 Scope / Out of scope

In scope:
- `GET /api/calls` (paged search), `GET /api/calls/{id}` (single record), `GET /api/calls.csv` (export).
- Filter grammar, validation, sort whitelist, limits, error contract.
- Name-search privacy (no side channel), masking of rows/CSV, audit of search/export/view.
- List row rendering and the detail drawer (summary chips, timeline, event flow, side panel, incident banner).
- URL state (hash) and drill-down chips.

Out of scope:
- KPI strip and person table of the Calls tab → `../call-analytics/DESIGN.md`.
- Metrics definitions → call/meeting analytics docs; this module only filters.
- Rate limiting of GET endpoints (platform; open item in the reference).

## 3 Dependencies

- `../call-ingest/DESIGN.md` — `call_records` (`derived`, `quality`, indexes), `call_participants`, `identities`,
  `call_media_streams`, classification, outcome codes, timeline/event codes.
- `../service-health/DESIGN.md` — incidents (category `call`) for the incident filter and the drawer banner.
- `../rooms-devices/DESIGN.md` — room keys for the `room` filter.
- `../call-analytics/DESIGN.md`, `../meeting-analytics/DESIGN.md` — drill-down sources.
- Platform: `../../platform/auth-rbac.md` (session, module check, audit), `../../platform/scope-masking.md`
  (RLS, masker levels, name variants, nonce markers, JSON-in-script helper), `../../platform/ui-shell.md`
  (topbar search/date segment/CSV placement, DetailPanel, DataTable), `../../platform/i18n.md`,
  `../../platform/logging.md`, `../../platform/database.md`, `../../platform/testing.md`.

## 4 Microsoft Graph endpoints & permissions

None. Reads only the database. Requires that feature `calls` (`CallRecords.Read.All`) has data; without the
permission the tab shows the permission card. Join URLs are stored and shown only in the detail side panel to
users with full detail (a join URL lets anyone join the meeting — never log it, never put it in CSV).

## 5 Data model

No new tables. Required indexes (call-ingest owns the migration; listed here because search depends on them):
`start_at DESC`, `id` (prefix range search), partial index on records with poor audio
(`quality.poor_audio_streams > 0`), partial index on records with a recording bot
(`derived.recording_or_transcript = true`), `call_participants(identity_key, call_id)`,
`call_media_streams(call_id)`.

Response shapes (illustrative):

```json
{"total": 1234, "page": 1, "size": 50, "has_more": true, "name_search": true,
 "rows": [{"id": "00000000-0000-0000-0000-000000000000", "start_ts": 1767225600, "html": "<li …>…</li>"}]}
```

```json
{"error": "bad_param", "param": "hour"}
```

Error codes: `unauthorized` (401), `forbidden` (403), `bad_param` (+`param`, 400), `not_found` (404), `server` (500).

## 6 Ingest / request flow

### 6.1 `GET /api/calls`

1. Start a masking context (fresh nonce for markers, `../../platform/scope-masking.md`).
2. Session → 401. Module `calls` not granted, package off, or the viewer's view of `calls` is aggregate /
   small-group → 403 (`forbidden` audited).
3. Validate filters (§7.1); any invalid → 400 `bad_param` with the parameter name.
4. Open the **web** connection (RLS); apply scope settings.
5. Resolve name search (§7.3) to identity keys.
6. `count(*)` with all filters, then the page: `ORDER BY <sort col> <dir> NULLS LAST, id <dir>` with
   `LIMIT size OFFSET (page−1)×size`. If `offset ≥ total` return empty rows.
7. Render each row (§8.2) server-side, translate, then mask with a per-row fast masker that only considers
   people whose names could appear in that row.
8. Audit `search_calls` (filters incl. q, total, rows, page, size, sort, order); log `report.calls_search`
   (filter keys and `q_len`, **not** q).

### 6.2 `GET /api/calls/{id}`

Malformed uuid → 400. Out of scope and non-existent return the **same** 404 body. Response `{row}` (row HTML
incl. drawer content). Audit `view_call`.

### 6.3 `GET /api/calls.csv`

Same filters and sort, max 10,000 rows, UTF-8 with BOM, comma separator, header in the viewer's language,
names masked exactly like the page, formula-injection escaping (prefix `'` for cells starting with `= + - @`
or tab/CR), filename `calls-YYYYMMDD-HHMM.csv` (local time). Columns: start (local), type, outcome, duration
(s), participants (count), participant names (masked, `; `-joined), external (yes/no), poor audio (yes/no),
recording bot (yes/no), record id. No join URL. Audit `export_calls` (filters, rows); log `report.calls_export`.

Performance (10k synthetic calls, web role + RLS): count + page 1–150 ms; partial indexes took the poor-audio
filter from 531 ms to 7 ms; end-to-end 50 rows 150–280 ms (full detail), 310–440 ms (masked);
row render ≈ 3 ms/row.

## 7 Derived logic & metric definitions

### 7.1 Filter grammar (query parameters; all optional)

| Param | Type / values | Meaning |
|---|---|---|
| `q` | text ≤ 100 chars, no NUL | name/UPN/key (incl. phone) of a **full-detail** person, or record id prefix (4–32 hex, dashes optional) |
| `from`, `to` | unix seconds, 0…4102444800 | `from` inclusive, `to` exclusive; default `to = now+1 s`, `from = to − 30 days`; `from > to` → 400 |
| `type` | csv of `p2p`, `meeting`, `group` | class (call-ingest §7.1) |
| `outcome` | csv of `ok` (connected+short), `interrupted`, `missed` (unanswered), `failed` | outcome codes |
| `person` | identity key (validated pattern) | record has this participant |
| `dir` | `in` / `out` | with `person`: role callee / caller |
| `room` | identity key | record has this room account as participant |
| `ext` | `1` | ≥ 1 external participant (`is_external`) |
| `quality` | `poor` | record has ≥ 1 poor user-side stream (see `qdim` interaction) |
| `rec` | `1` | recording/transcription bot present |
| `incident` | `1` or incident id | overlaps any `call` incident / that incident |
| `min_dur`, `max_dur` | seconds 0…10,000,000 | duration bounds inclusive |
| `dow` | 1–7 (ISO, 1 = Monday, local `APP_TIMEZONE`) | start weekday |
| `hour` | 0–23 local | start hour |
| `min_people`, `max_people` | 0…100,000 | participant row count bounds (same counting as analytics `people`) |
| `qdim` | `all`, `connection`, `platform`, `device`, `network` | record has a classified user-side audio stream (in the dimension value) |
| `qval` | text ≤ 200; only with `qdim ≠ all`; `''` = unknown/null | dimension value |
| `sort` | `start` (default), `duration`, `people` | whitelist |
| `order` | `desc` (default), `asc` | |
| `page` | 1…100,000 (default 1) | |
| `size` | default 50, max 200 | |

Combination rules:
- `quality=poor` with `qdim` → the **stream** in that dimension must be poor (same rule as analytics tables);
  without `qdim` → record-level poor flag.
- Incident overlap: `incident.start ≤ coalesce(call.end, call.start)` AND `coalesce(incident.end, now) ≥
  call.start`; resolve the incident set once per query (materialised CTE) for the requested range.
- `dow`/`hour` use the same local-time expression as `v_call_facts` (verified 0 difference vs `call_hourly`
  over 40 cells × 2 classes in the reference).
- Hash-only UI shortcuts: "All" / "Talked" (`outcome=ok,interrupted`) / "Missed" (`outcome=missed,failed`).

### 7.2 Sort

Columns: start → `start_at`; duration → `duration_s`; people → `derived.participant_count`. Always add `id` as
a tiebreaker in the same direction for stable paging.

### 7.3 Name search privacy (side-channel rule)

`q` and `person` may only match people the viewer sees with **full** detail in this module. Matching:
case-insensitive substring on display name, UPN and key, plus a diacritic-folded comparison (Turkish dotless i
etc. folded the same way in SQL and in the client). Masked or out-of-scope people never enter the candidate set,
so a search for their name returns exactly what a non-existent name returns. If the viewer has no full-detail
person (`name_search=false`), `q` is interpreted only as a record-id prefix; with `self_view` the viewer can
still search their own name. `%` and `_` are escaped (not wildcards). The person picker lists only full-detail
people (otherwise hidden).

### 7.4 Row model (derived from `derived`)

- Title: p2p "Caller → Callee"; linked meeting "Meeting"; group "Group call"; p2p-converted group
  (timeline has `call_started` + `p2p_to_group`) "Caller → Callee · group call".
- Meta: local start, duration label, participant count, outcome label (call-ingest §11), badges: external,
  poor audio, recording/transcription, incident, interrupted (neutral badge "Talked · interrupted").
- Data attributes for client filtering in the offline (CLI) report only: id, class, state, duration, people,
  poor, rec, incident.

### 7.5 Detail drawer content

| Part | Content |
|---|---|
| Summary chips | class, duration, people, device count (if > people), coverage "Audio + Camera + Screen sharing" (only those used), audio quality (worst, poor x / y streams) |
| Incident banner | for each overlapping `call` incident: id, title, status, interval (amber `--incident`) |
| Timeline | one row per person × device lane; bar = only the intervals that device was connected (`segments`); under the label: coverage "Audio", "Camera ↑ sent / ↓ watched", "Screen sharing ↑ sent / ↓ watched / not measured"; lanes with no measurement greyed; right side: jitter · loss · RTT, connection type, microphone; tooltip with packet counts; bots as hatched lanes labelled by bot type; footnote explaining that Graph does not say when sharing happened |
| Event flow | time-ordered events from `timeline.events` rendered by label functions with local `HH:MM:SS` (`start + offset_s`) |
| Side panel | participants (masked), class, organizer, modalities used, outcome (with first failure stage/reason), record id, join URL (full detail only), "Copy link" (`#calls?id=<id>`) |

Coverage direction rule: for camera/share, "sent" if the lane's outgoing stream had packets, "watched" if
incoming had packets; if declared in modalities but no packets measured, show "not measured" — never claim
"sent" without data. Zero-duration records: no bars, events only.

## 8 UI

### 8.1 Calls tab list area

- Topbar on this tab: search box (debounce 300 ms), date segment (1 day / 7 days / 1 month / Custom; default
  1 month; remembered per browser), CSV button.
- Filters panel (collapsible) + active-filter chips (each removable); drill-down chips for `dow`, `hour`,
  `qdim/qval`, `min/max_people`, `incident=<id>`, time range `t0/t1` render as readable chips
  ("Tuesday 14:00–14:59", "Network: 192.0.2.0", "Incident TM000000").
- Sort selector (date / duration / participants), "Load more" (50 per page), total count line.
- URL state in the hash: `#calls?q=…&type=…&outcome=…&w=…&id=…&dow=…&hour=…&qd=…&qv=…&pmin=…&pmax=…&incident=…`.
  Opening `#calls?id=<id>` fetches `/api/calls/{id}` and opens the drawer.
- Drawer: right-side panel (`DetailPanel`), focus-trapped, Esc closes, row stays highlighted.
- Clipboard: `navigator.clipboard` may hang; fall back to `execCommand('copy')` after 1.5 s.
- CSS prefix for this module's classes: `cl-`, drawer `rb-`, timeline `tl-` (avoid collisions).

### 8.2 States

- Empty result: one grey line "No records match these filters." + "Clear filters" link.
- No data at all: call-ingest empty line.
- Error from API: inline error with "Retry".
- Locked (permission missing): permission card for `calls`.
- Aggregate view: list hidden; message "Your role shows aggregate figures only for this module."
- Offline CLI report variant (if kept): latest 200 records embedded, client-side filtering; person/direction,
  room, external, CSV and copy-link hidden.

### Permission transparency (shared component, `../../platform/permissions.md`)

- **"Which permissions does this need?"** popover in the module header (reference: `../user-lookup/DESIGN.md`):
  lists this module's permissions least-privileged first with alternatives as "(alternative: X or Y)", ✓ per held
  option from the token `roles` claim, ✗ if none held, required/optional, what it is used for, how to grant
  (Azure Portal → App registrations → the app → API permissions → Add → Grant admin consent), and "not needed"
  permissions (module off or not used by this module).
  Rows: `CallRecords.Read.All` (required), `ServiceHealth.Read.All` (optional — incident filter/banner).
- **No live Graph calls at view time** in this module, so no request-trace popover. Instead each data section has
  a **"What fed this?"** popover: ingest step, Graph endpoint, permission, and last successful run time
  (list and drawer → step `calls`; incident banner/filter → step `health`). Shown in the list header and the drawer footer.

## 9 Access control

- Module `calls`; aggregate/small views → 403 for all three endpoints.
- Scope via RLS on the web connection (records with ≥ 1 in-scope participant; out-of-scope colleagues' rows
  invisible). Masking levels for names in rows, drawer and CSV; out-of-scope colleague → "Out of scope",
  external → "External participant" for masked viewers per platform policy.
- All JSON embedded in HTML via the platform JSON-in-script helper; markers carry per-request nonces; unmatched
  marker → fail closed.
- Person keys (oids) in attributes/selectors only for full-detail people.
- Audit actions: `search_calls`, `export_calls`, `view_call`, `forbidden`. Audit detail is structured
  (filters dict), never free text.

## 10 Logging events

| Event | Level | Context |
|---|---|---|
| `report.calls_search` | INFO | filter_keys, q_len, total, rows, page, size, sort, order, duration_ms |
| `report.calls_export` | INFO | filter_keys, q_len, rows, duration_ms |
| `report.call_view` | INFO | found (bool), duration_ms (no id of out-of-scope records) |
| `auth.search_calls` / `auth.export_calls` / `auth.view_call` | INFO | module, ip, q_len, filter_keys |
| `api.bad_param` | INFO | param |
| `api.error` | ERROR | endpoint, exc |

Never log `q`, `person`, names or join URLs.

## 11 i18n notes

- Row and drawer text produced by label functions from codes; event times formatted with locale time incl.
  seconds; CSV headers in the viewer's language, values (dates) in ISO for machine-friendliness or locale
  (decide once; reference used the viewer's locale for headers only).
- Search folding is language-independent (same fold table in SQL and JS).
- Masking labels ("Employee #n", "Out of scope", "External participant") must be translatable (they were
  hard-coded Turkish in the reference — known defect).

## 12 Testing & acceptance criteria

- [ ] Every filter: valid value works, invalid value → 400 with the right `param`.
- [ ] `total` is over all pages; paging is stable under equal sort keys.
- [ ] Drill-down: for every heatmap cell and every analytics number, search total = clicked number.
- [ ] Name search: searching a masked or out-of-scope person's exact name returns the same response as a
      random string; `name_search=false` viewers can search by id prefix only.
- [ ] Injection: a participant whose display name contains marker-like text, `</script>`, quotes or U+2028
      cannot break masking (12 payloads × full/masked/scoped/aggregate).
- [ ] CSV: BOM present, ≤ 10,000 rows, formula cells escaped, masked names, no join URL.
- [ ] `/api/calls/{id}` returns identical 404 for out-of-scope and missing ids.
- [ ] Audit rows written for search/export/view with structured filters.
- [ ] 10k calls: page request ≤ 300 ms full, ≤ 500 ms masked.
- [ ] Drawer: p2p-converted-to-group shows `call_started`, `p2p_to_group`, `device_switched`; missed call shows
      ringing + not connected and no bars.

- [ ] "Which permissions does this need?" popover lists exactly: `CallRecords.Read.All` (required), `ServiceHealth.Read.All` (optional); ✓/✗ follow the token roles snapshot
      (test with a snapshot holding none, the least-privileged option, and only an alternative); grant steps shown.
- [ ] Every data section's "What fed this?" popover shows ingest step, endpoint, permission and the last
      successful `ingest_runs` time (list header and drawer).

## 13 Implementation steps for an agent

1. Confirm call-ingest indexes; add search-specific partial indexes if missing.
2. Filter parser (pure, whitelist, typed errors) + unit tests.
3. SQL builder (parameterised; column names only from whitelists) + `search`, `by_id`, `export`.
4. Name-resolution step honouring masker level; person picker filter.
5. Row renderer + drawer renderer from `derived` (no SQL in renderers).
6. Endpoints with auth, module check, masking context, audit, logs.
7. Client: hash state, chips, debounce, load more, drawer, CSV link building from current filters, clipboard fallback.
8. Drill-down chip rendering for analytics parameters.
9. Tests §12 incl. leak and injection suites.

## 14 Open questions / future

- Rate limiting per user for search/export (each request writes audit + log rows).
- `modality` filter (video / share) for meeting-analytics drill-down.
- Keyset pagination for very deep pages (OFFSET cost) if needed beyond 10k results.
- Custom-range KPI API to replace embedded per-day JSON (≈ 800 KB at 10k calls in the reference).
- Preview-as-user: list currently loads with the admin's own session in the reference — must use the previewed
  user's view.
