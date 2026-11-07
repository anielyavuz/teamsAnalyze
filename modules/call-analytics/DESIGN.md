# Call Analytics (1:1 calls) — Design

RBAC module key: `call_analytics` (tab) · also feeds the KPI strip and person table of the `calls` tab ·
Package: `calls` · Features: `calls` (required), `health` (optional section)

---

## 1 Purpose

Answer "how well do we pick up the phone?" for Teams 1:1 calls: volume, answered / missed / interrupted,
per-person answer rate, unique missed contacts, when calls are missed (day × hour), who we talk to outside the
organisation, and whether audio quality problems come from our network/devices or from a Microsoft service
incident. All numbers come from the call-ingest derived tables; this module makes no Graph requests.

## 2 Scope / Out of scope

In scope:
- 1:1 call KPIs (records with class `p2p`, see `../call-ingest/DESIGN.md` §7.1).
- Per-person statistics (incoming, answered, missed, answer rate, outgoing, meetings count, talk time).
- Unique missed contacts (deduplicated repeated attempts; reached / called back).
- Day × hour heatmap, calls by hour (stacked), missed rate by hour.
- External communication for 1:1 calls.
- Audio quality (CQD) KPIs and breakdowns for 1:1 calls.
- Microsoft service-incident overlay (during vs outside incidents).
- Drill-down links into call search with matching filters.

Out of scope:
- Meetings and group calls → `../meeting-analytics/DESIGN.md` (also hosts the combined view).
- Record list, search, CSV, detail drawer → `../call-search/DESIGN.md`.
- Incident ingest and relevance classification → `../service-health/DESIGN.md`.
- Callback analysis beyond the "reached within 60 min" rule, queues, PSTN, department breakdowns → §14.

## 3 Dependencies

- `../call-ingest/DESIGN.md` — tables `call_records`, `call_participants`, `identities`, `call_media_streams`;
  classification, outcome, `is_external`, CQD rules.
- `../service-health/DESIGN.md` — `service_issues` with relevance category; this module only consumes
  incidents of category `call` and their start/end.
- `../call-search/DESIGN.md` — drill-down target (`/api/calls` filter parameters).
- `../../platform/database.md` (SQL functions, `app_tz()`), `../../platform/auth-rbac.md`,
  `../../platform/scope-masking.md`, `../../platform/ui-shell.md` (lazy tab, KPI strip, heatmap, DataTable,
  chart conventions), `../../platform/i18n.md`, `../../platform/logging.md`, `../../platform/permissions.md`,
  `../../platform/testing.md`.

## 4 Microsoft Graph endpoints & permissions

None directly. Indirect requirements:

| Data | Endpoint (owner) | Version | Permission | Notes |
|---|---|---|---|---|
| Call records | `/communications/callRecords` (call-ingest) | v1.0 | `CallRecords.Read.All` | required; tab locked without it |
| Service incidents | `/admin/serviceAnnouncement/issues` (service-health) | v1.0 | `ServiceHealth.Read.All` | optional section; permission card inside the section when missing |

Data quirks relevant here (dated in call-ingest §4): missed = inferred (no flag); Graph cannot distinguish
missed / declined / caller-cancelled, so the label is "Not connected (missed or declined)"; records appear
15–30 min after the call; Graph keeps 30 days but the DB keeps everything, so ranges up to 90 days work once
history has accumulated.

## 5 Data model

No new raw tables. SQL read models (functions/views, `security_invoker = true`, all computed in SQL — the UI
never re-computes metrics):

| Object | Grain | Columns |
|---|---|---|
| `v_participation` | participant row × call | call_id, identity_key, role, joins, platform, start_at, duration_s, connected, outcome_state, call_type, display_name, upn, kind, tenant_id |
| `person_stats(from, to)` | identity | identity_key, name, upn, kind, incoming, incoming_answered, missed, answer_rate, outgoing, outgoing_answered, meetings, seconds, total |
| `person_stats_daily(from, to, tz)` | local day × identity | same counters per local day (for custom ranges) |
| `call_kpis(from, to)` | range | p2p_calls, answered, missed, interrupted, talk_seconds, incoming, incoming_answered |
| `call_kpis_daily(from, to, tz)` | local day | same per local day |
| `v_call_facts` | call | id, call_type, class, start_at, hour_local, dow_local (ISO 1 = Monday), duration_s, connected, outcome_state, has_join_url, has_video, has_screenshare, people, external_people |
| `call_hourly(from, to)` | local dow × hour | calls (p2p), answered (p2p connected), missed (p2p not connected), meetings (non-p2p) |
| `missed_contacts(from, to, gap_s, reach_s)` | contact episode | caller_key, callee_key, first_at, last_at, attempts, reached, reached_at, called_back, called_back_at |
| `quality_overview(from, to, class)` | range | streams, poor, unclassified, calls, avg jitter/loss/rtt |
| `quality_breakdowns(from, to, dims, limit, class)` | dim × value | key, streams, poor, calls, poor_calls, avg jitter/loss/rtt (single `GROUPING SETS` scan) |
| `external_parties(from, to, limit, class)` | external identity | key, name, kind, upn, calls, seconds, last_at |
| `quality_vs_incidents(from, to, class)` | during / outside | streams, poor, poor_rate, calls |
| `issues_in_window(from, to, category)` | incident | id, service, classification, status, title, impact, start_at, end_at, is_resolved, affected calls |

Settings used: `min_rate_sample` (3), `missed_gap_s` (600), `missed_reach_s` (3600), `answer_rate_green` (0.80),
`answer_rate_amber` (0.50), `poor_rate_red` (0.20), `poor_rate_amber` (0.05), `min_group_size` (platform, 5).

## 6 Ingest / request flow

No ingest. Request flow for the tab (lazy, see `../../platform/ui-shell.md`):

1. `GET /api/tab/call_analytics` → session check (401) → module + package check (403 + audit `forbidden`).
2. Resolve the viewer's scope and detail level; set RLS session settings (`../../platform/scope-masking.md`).
3. For each preset window (7, 30, 90 days; default 30) compute all sections. Facts are fetched once for the
   widest window and filtered in memory for narrower windows; everything else is one SQL call per window.
4. Render HTML fragment; apply masking/aggregate stripping; audit `view_tab`; log `report.tab_build`.

Calls tab KPI strip + person table use presets 1 day / 7 days / 1 month (default 1 month) and a custom range
(max 90 days back, earliest = first record in DB) served by the daily functions summed on the server (the
reference embedded per-day JSON and summed in the browser; the rebuild should call an API with from/to instead).

Performance target (10k calls, web role + RLS): tab build ≤ 2 s; quality breakdowns one scan per window.

## 7 Derived logic & metric definitions

All counts are over records whose `start_at ∈ [from, to)` (UTC). Local day/hour use `app_tz()`.

### 7.1 Volume and outcome (class `p2p` only)

| Metric | Definition |
|---|---|
| 1:1 calls | count of p2p records |
| Answered | p2p and `connected` (includes `short` and `interrupted`) |
| Missed | p2p and NOT `connected` (`unanswered` + `failed`) |
| Interrupted | p2p and `outcome_state = interrupted` (subset of answered; shown as a neutral note, not a failure) |
| Missed rate | missed / 1:1 calls; KPI tone "bad" when ≥ 20% |
| Average call | mean `duration_s` of answered p2p calls |
| Talk time | sum `duration_s` of connected records |
| Busiest hour | local hour with max p2p calls |
| Most missed hour | local hour with max missed (tie → fewer calls first); "-" if no missed calls |

Reference pitfall to avoid: a combined KPI counted `NOT connected` across **all** records (meetings included) as
"missed". Missed is defined on p2p only. (Unconnected group records are reported in meeting analytics.)

### 7.2 Per-person (`person_stats`)

From `v_participation` in range:
- incoming = rows with role `callee`; incoming_answered = callee rows of connected calls;
  missed = incoming − incoming_answered;
- **answer rate** = incoming_answered / incoming, only p2p (callee rows exist only in p2p); null when incoming = 0;
- outgoing = role `caller`; outgoing_answered = caller rows of connected calls;
- meetings = role `participant` (meetings + group calls; does not affect the rate);
- seconds = sum duration of connected calls in any role; total = incoming + outgoing + meetings.
- Order: total desc, seconds desc, name. Table limited to the top 50 rows; KPIs always from the full range.
- Room accounts are labelled "room" (they are not people for staffing purposes).
- Answer-rate bar tone: ≥ 80% green, ≥ 50% amber, below red.
- Minimum sample: show the rate for any incoming ≥ 1, but always show `answered/incoming` next to it; rankings
  such as "lowest answer rate" include only people with incoming ≥ 5 (new rule, not in the reference).

### 7.3 Unique missed contacts (approved design pending owner sign-off; reference had a prototype)

Purpose: three rapid redials should not look like three missed calls.
- Episode: consecutive missed p2p calls with the same ordered pair (caller → callee) where each attempt starts
  within `missed_gap_s` (600 s) of the previous attempt in the episode.
- `reached`: a connected p2p call between the same two people, either direction, starting within
  `missed_reach_s` (3600 s) after the episode's last attempt. `reached_at` = its start.
- `called_back`: a p2p call (connected or not) from the original callee to the original caller within
  `missed_reach_s`. `called_back_at` = its start.
- KPIs: unique missed contacts, % reached within 60 min, % called back, median time to callback.
- Per-person column "unique missed" next to "missed". The answer rate itself stays call-based (unchanged).
- Parameters are settings; changing them needs no reprocess (pure SQL over derived tables).

### 7.4 Day × hour and by-hour charts

- Heatmap 7 × 24 (ISO weekday rows, Monday first unless locale first weekday differs; hours 00–23 local):
  cell = p2p calls starting in that local day/hour; tooltip: calls, answered, missed. Row totals at right.
  Colour: single-hue sequential scale, 5 buckets + zero, bucket edges = ceil(vmax × i / 5); dark theme inverted
  so "few" stays close to the background.
- Calls by hour: stacked columns answered (`--s1`) + missed (`--s2`), every 3rd hour labelled.
- Missed rate by hour: only hours with ≥ `min_rate_sample` (3) p2p calls are drawn; others show
  "small sample (<3 calls)" or "no calls" in the tooltip; y-axis fixed 0–100%. If no hour qualifies, show the
  explanatory empty state instead of an empty chart.

### 7.5 External communication (p2p)

- External = `is_external(kind, tenant_id)` (call-ingest §7.9). Kinds shown as: another organisation (`user`
  from another tenant), guest, phone, unidentified (`unknown`), ACS (`acsUser`, `spoolUser`).
- KPIs: share of 1:1 calls with ≥ 1 external participant; distinct external people.
- Table: top 10 external identities by distinct calls, then talk seconds: name, kind (+ UPN if different),
  records, connected duration, last contact (local). Person-level → aggregate mode removes it.

### 7.6 Audio quality (p2p)

Stream universe: `call_media_streams` rows with `classified AND NOT is_service` in p2p records in range.
- KPIs: measured streams (in N calls), poor stream rate (tone: ≥ 20% bad, < 5% ok), average jitter (threshold
  30 ms), average loss (threshold 10%), average RTT (threshold 500 ms), unclassified (audio streams ≤ 500 packets).
- Breakdowns (4 tables): connection type, platform, microphone/device (capture device), network (subnet or /24).
  Columns: value ("unknown" for null), streams, poor rate bar (≥ 20% red, ≥ 5% amber, else green, number always
  shown), avg jitter, avg loss %, avg RTT. Top 12 per dimension ordered by streams desc.
- Footnote: streams attributed to the sending side; meeting server excluded; web client reports no connection
  type, which explains most "unknown" rows.
- Empty: "No measurable audio streams in this range." plus, when unclassified > 0, "N audio streams but none passed
  the CQD threshold (500+ packets) — short or unconnected calls."
- Note: tables count **streams**, drill-down totals count **calls**; label both clearly.

### 7.7 Service-incident overlay

- Incident ↔ call overlap: incident of category `call` with `incident.start ≤ coalesce(call.end, call.start)`
  AND `coalesce(incident.end, now) ≥ call.start`.
- "Open" incident (from service-health): not resolved AND (no end OR end in the future). Microsoft sometimes sets
  an end time but leaves `isResolved = false`.
- Card: poor-audio rate during incidents vs outside (streams, poor, calls), and the list of incidents
  overlapping the window with affected call counts. No incidents → "In this range Microsoft published no incident
  that could affect Teams calls; the cause of quality problems is most likely the network, device or location."
- Permission missing → permission card in the section only (the rest of the tab works).
- Relevance classification is keyword-based (owned by service-health); expect occasional false positives.

### 7.8 Drill-down mapping (to call search)

Every clickable number links to the call search with filters whose `total` equals the clicked number
(acceptance test). Mapping:

| Source | Filters |
|---|---|
| 1:1 calls KPI | `type=p2p` + range |
| Answered / Missed | `type=p2p&outcome=ok,interrupted` / `type=p2p&outcome=missed,failed` |
| Heatmap cell | `type=p2p&dow=D&hour=H` |
| Hour column (answered/missed segment) | `type=p2p&hour=H&outcome=…` |
| Quality breakdown row | `type=p2p&qdim=<dim>&qval=<value>` (+ `quality=poor` for the poor cell) |
| Incident row | `incident=<id>`; overall "during incidents" → `incident=1` |
| Person row | `person=<key>&dir=in` (only for people the viewer sees with **full** detail) |
| External KPI | `type=p2p&ext=1` |

External-party table rows never link (the key would leak phone numbers/ids). Links are produced server-side by
one helper that also checks: viewer has module `calls` and the view is not aggregate/small-group.

## 8 UI

Tab "Call analytics" (route `#call_analytics`), lazy-loaded:
1. Filter row: chips "Last 7 / 30 / 90 days" (group-scoped: switching here does not affect other tabs), source
   note "call records in the database · full accumulated history, not only Graph's 30 days".
2. Section "1 · Call volume": KPI strip — 1:1 call card with answered/missed stacked bar, missed rate, average
   call, busiest hour, most missed hour; unique missed contacts card (§7.3); heatmap; by-hour stacked columns +
   missed rate chart side by side.
3. Section "2 · Audio quality": note "Audio streams in 1:1 calls.", KPI strip, 4 breakdown tables, incident card.
4. Section "External communication".

Calls tab (owned by call-search for the list) hosts from this module: KPI strip (1:1 calls, answered, missed,
answer rate, talk time, meetings count), per-person table (grouped header "1:1 call": incoming / answered /
missed / answer rate / outgoing; separate "Meetings" column), and compact insight cards (heatmap, by-hour,
quality summary with jitter/loss/RTT trio + connection/platform/network rows).

Rules: colours are bound to entities (answered `--s1`, missed `--s2` red, meeting `--s3`), text never in series
colour, every chart has a "Show as table" equivalent, tooltips also on keyboard focus and set via textContent.
Empty: single grey line, no big empty boxes. Locked: permission card (feature `calls`) replaces the tab body.
Aggregate view: person table, external table and person drill links removed; module hidden when the scope has
fewer than `min_group_size` people.

### Permission transparency (shared component, `../../platform/permissions.md`)

- **"Which permissions does this need?"** popover in the module header (reference: `../user-lookup/DESIGN.md`):
  lists this module's permissions least-privileged first with alternatives as "(alternative: X or Y)", ✓ per held
  option from the token `roles` claim, ✗ if none held, required/optional, what it is used for, how to grant
  (Azure Portal → App registrations → the app → API permissions → Add → Grant admin consent), and "not needed"
  permissions (module off or not used by this module).
  Rows: `CallRecords.Read.All` (required — call records), `ServiceHealth.Read.All` (optional — incident overlay).
- **No live Graph calls at view time** in this module, so no request-trace popover. Instead each data section has
  a **"What fed this?"** popover: ingest step, Graph endpoint, permission, and last successful run time
  (volume/quality/external sections → step `calls`, `/communications/callRecords`, `CallRecords.Read.All`;
  incident card → step `health`, `/admin/serviceAnnouncement/issues`, `ServiceHealth.Read.All`).

## 9 Access control

- Module key `call_analytics` (tab). Calls-tab KPIs/person table require `calls`.
- Scope: people modules (all / department / team / self). RLS limits records to those with ≥ 1 in-scope
  participant; participant rows of out-of-scope colleagues are invisible (they cannot leak into person tables).
- Detail levels: full (names), masked ("Employee #n", stable per request), aggregate (marked person-level blocks
  removed). Every person-level block is wrapped in the platform's person-level markers.
- Audit: `view_tab` (module, scope summary), `forbidden`. Drill-down clicks audit in call search.

## 10 Logging events

| Event | Level | Context |
|---|---|---|
| `report.tab_build` | INFO | module, windows, duration_ms, rows (person, external), masked |
| `analytics.small_sample` | DEBUG | metric, hour, n |
| `analytics.query_slow` | WARNING | query, duration_ms (> 2 s) |
| `report.permission` | INFO | feature, status (when rendering a locked section) |

## 11 i18n notes

- Context-disambiguated strings: "Answered" (series) vs "call state|Answered"; "Missed" vs "missed rate".
- Pluralised: "{n} day(s)", "{n} audio stream(s) … none passed the CQD threshold".
- Percentages via locale formatter (e.g. `50%` en vs `%50` tr); thresholds rendered with the same formatter.
- Weekday names and first weekday from the locale; hours as `HH:00` in 24 h or locale 12 h format.
- External kind labels translated; ACS kept as-is.

## 12 Testing & acceptance criteria

Synthetic dataset (in a cloned test DB, never in production) with embedded patterns, as in the reference:
missed rate high at lunch and after 17:00 (e.g. 36% / 50% vs 15% otherwise); poor-stream rate wired < Wi-Fi <
mobile (≈ 0% / 61% / 81%); one problematic network (≈ 91% poor); external participants.

- [ ] SQL finds every embedded pattern within ±1 percentage point.
- [ ] Missed KPI counts only p2p NOT connected; a connected meeting with failureInfo never counts as missed.
- [ ] `call_hourly` totals over 7 × 24 equal `call_kpis` for the same range.
- [ ] Missed rate chart omits hours with < 3 calls; with no qualifying hour, the explanatory empty state renders.
- [ ] Unique missed contacts: 3 attempts within 10 min = 1 contact; attempt 11 min after = new contact;
      connected call either direction within 60 min → reached.
- [ ] Every drill-down link's search `total` equals the clicked number (all heatmap cells, both classes).
- [ ] Masked viewer: no real name/UPN in the fragment (case-insensitive scan, all name variants).
- [ ] Aggregate viewer: no person table, no external table, no person links; small scope → module hidden.
- [ ] Missing `ServiceHealth.Read.All`: only the incident card shows a permission card.
- [ ] Light/dark/390 px screenshots; palette validated by the dataviz validator in both themes.

- [ ] "Which permissions does this need?" popover lists exactly: `CallRecords.Read.All` (required), `ServiceHealth.Read.All` (optional); ✓/✗ follow the token roles snapshot
      (test with a snapshot holding none, the least-privileged option, and only an alternative); grant steps shown.
- [ ] Every data section's "What fed this?" popover shows ingest step, endpoint, permission and the last
      successful `ingest_runs` time (incident card uses the health run).

## 13 Implementation steps for an agent

1. Confirm call-ingest tables and `call_class`, `is_external` exist.
2. Migration: `v_participation`, `person_stats`, `person_stats_daily`, `call_kpis(_daily)`, `v_call_facts`,
   `call_hourly`, `missed_contacts`, quality functions (single GROUPING SETS scan), `external_parties`,
   incident overlap view (if not provided by service-health) — all `security_invoker`.
3. Settings defaults (§5).
4. Tab builder (data) and renderer (fragment) separated; renderer has no SQL.
5. Drill-down link helper shared with meeting analytics and rooms.
6. Calls-tab KPI strip + person table + custom-range API (`GET /api/calls/kpis?from&to`).
7. Register module `call_analytics` in RBAC and in package `calls`.
8. Tests from §12, screenshots, leak scan per user × module.

## 14 Open questions / future

- Owner approval of "unique missed contacts" parameters (10 min / 60 min) and whether to show it on the Calls tab.
- Weekly per-person trend; queue / call-centre metrics if PSTN or call queues appear.
- Department dimension once directory department is populated (already available for scope).
- `callEvents` to split "not connected" into missed / declined / cancelled.
- Pre-aggregated hourly table if RLS cost dominates at 100k+ calls.
