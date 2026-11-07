# Meeting Analytics (+ combined view) — Design

RBAC module keys: `meeting_analytics` (tab "Meeting analytics"), `combined_analytics` (tab "Call + meeting
analytics") · Package: `calls` · Features: `calls` (required), `health` (optional section)

---

## 1 Purpose

Describe how the organisation meets on Teams: how many meetings and group calls, when they happen, how long they
last, how many people attend, how often camera and screen sharing are used, how much of it involves external
parties, how much person-time is spent, and how good the audio is for the people in them. A second tab, the
combined view, puts 1:1 calls and meetings on the same axes for an overall load picture. No Graph requests;
everything is read from the call-ingest tables.

## 2 Scope / Out of scope

In scope:
- Meeting habits KPIs and distributions (duration, participant count).
- Meetings by local day × hour and by hour; linked meeting vs unlinked group call split.
- Camera and screen-share adoption (record-level presence).
- External participation in meetings.
- Person-hours (attendee seconds) as a load metric.
- Audio quality of user-side streams in meetings, incident overlay.
- Combined view with calls + meetings (§7.5, §8).
- Drill-down links to call search.

Out of scope:
- Planned vs actual (calendar), no-shows, late join/early leave per scheduled meeting (needs calendar /
  attendance reports) → §14.
- Meeting cost in money (licensing module multiplies person-hours by an hourly rate) → `../licensing/DESIGN.md`.
- Rooms joining meetings → `../rooms-devices/DESIGN.md`.
- Transcript/recording presence per meeting → `../transcripts/DESIGN.md`.
- 1:1 specifics (answer rate, missed contacts) → `../call-analytics/DESIGN.md`.

## 3 Dependencies

- `../call-ingest/DESIGN.md` — classification (§7.1), outcome, bots excluded from people, `derived.timeline`
  (`attendee_seconds`), `call_media_streams`, `is_external`.
- `../call-analytics/DESIGN.md` — shared SQL read models (`v_call_facts`, `call_hourly`, quality functions,
  `external_parties`, incident overlay), shared chart components and the drill-down helper.
- `../service-health/DESIGN.md` — incidents of category `call`.
- `../call-search/DESIGN.md` — drill-down target.
- Platform: `../../platform/ui-shell.md`, `../../platform/auth-rbac.md`, `../../platform/scope-masking.md`,
  `../../platform/i18n.md`, `../../platform/logging.md`, `../../platform/database.md`,
  `../../platform/permissions.md`, `../../platform/testing.md`.

## 4 Microsoft Graph endpoints & permissions

None directly.

| Data | Endpoint (owner) | Version | Permission | Notes |
|---|---|---|---|---|
| Meeting / group call records | `/communications/callRecords` (call-ingest) | v1.0 | `CallRecords.Read.All` | required |
| Incidents | `/admin/serviceAnnouncement/issues` (service-health) | v1.0 | `ServiceHealth.Read.All` | optional section |

Relevant quirks (details and dates in call-ingest §4):
- In meetings every session's far side is Microsoft's media server (`serviceEndpoint`, always "wired"); exclude it.
- Recording/transcription bots look like users in `participants_v2`; they are excluded from participant counts.
- Record `modalities` say a modality existed at some point; Graph does not give when or how long camera or screen
  share was on. Adoption metrics are therefore "meetings where X was used", never minutes.
- Live event / town hall attendees are not in call records.
- `failureInfo` can be present on perfectly good meetings (`stage=midcall`); meetings with it are
  `interrupted`, not failed.

## 5 Data model

No new tables. Reads `v_call_facts` (one row per record):

| Column | Use here |
|---|---|
| class (`p2p` / `meeting` / `group`) | meetings = `meeting` + `group` |
| start_at, hour_local, dow_local | time placement (local via `app_tz()`) |
| duration_s, connected, outcome_state | duration KPIs; only connected meetings in averages |
| has_join_url | linked vs group call |
| has_video | record modalities contain `video` |
| has_screenshare | record modalities contain `screenSharing` or `videoBasedScreenSharing` |
| people | distinct non-bot identities in `call_participants` |
| external_people | of those, `is_external` |

Additional read model added by this module:

| Object | Grain | Columns |
|---|---|---|
| `meeting_load(from, to)` | local day | meetings, attendee_seconds (sum of `derived.timeline.attendee_seconds`), distinct organizers |
| `meeting_organizers(from, to, limit)` | organizer | organizer_key, name, meetings, attendee_seconds, avg_people (person-level; aggregate mode removes) |

Constants (code, documented): duration buckets `< 15 min`, `15–30 min`, `30–60 min`, `1–1.5 h`, `> 1.5 h`
(half-open, upper bound exclusive); participant buckets `1`, `2`, `3–5`, `6–10`, `11+`.

## 6 Ingest / request flow

No ingest. `GET /api/tab/meeting_analytics` and `GET /api/tab/combined_analytics` follow the lazy-tab flow of
`../../platform/ui-shell.md`: auth → module/package check → scope (RLS) → build windows 7/30/90 days
(default 30; facts fetched once for 90 days and filtered for 30/7) → render fragment → masking → audit
`view_tab` → log `report.tab_build`.

## 7 Derived logic & metric definitions

Range: records with `start_at ∈ [from, to)`. "Meetings" = class `meeting` + `group` (call_type ≠ peerToPeer,
`unknown` included).

### 7.1 Meeting habits

| Metric | Definition |
|---|---|
| Meetings | count of meetings; note "{a} linked · {b} group calls" (linked = has join URL) |
| Average duration | mean `duration_s` over **connected** meetings |
| Average participants | mean `people` over all meetings (unique people, bots excluded) |
| Camera used | share of meetings with `has_video` |
| Screen shared | share of meetings with `has_screenshare` |
| With external participants | share of meetings with `external_people ≥ 1` |
| Person-hours | Σ `attendee_seconds` / 3600 (union of each person's device intervals; late join / early leave / device switch handled; two devices at once counted once) |
| Average person-hours per meeting | person-hours / connected meetings |

Distributions: duration buckets (counts + share), participant buckets (counts + share), single series `--s3`,
peak bar labelled, table equivalent.

### 7.2 When meetings happen

- Heatmap 7 × 24: cell = meetings starting in local day/hour; same scale rules as call analytics.
- Meetings by hour: single series columns; tooltip and table split linked vs group call.
- Optional daily line: meetings per local day and person-hours per day (`meeting_load`), two separate charts
  on a shared x-axis (never dual y-axes).

### 7.3 Audio quality (meetings)

Same definitions as `../call-analytics/DESIGN.md` §7.6 with the class filter `meeting + group`: user-side
(`NOT is_service`) classified audio streams; KPIs, 4 breakdowns (connection, platform, device, network),
incident during/outside comparison. Section note: "User audio streams in meetings (meeting server side
excluded)."

### 7.4 External participation (meetings)

KPI: "Meetings with external participants" = meetings with ≥ 1 external person / meetings. Distinct external
people. Top 10 external identities (by distinct meetings, then connected seconds) — person-level block.

### 7.5 Combined view (`combined_analytics`)

Everything over all classes:

| Element | Definition |
|---|---|
| Total card | 1:1 calls + meetings, stacked bar answered 1:1 (`--s1`) / missed 1:1 (`--s2`) / meetings (`--s3`), legend with shares |
| Missed rate | p2p missed / p2p calls (never includes meetings); tone bad ≥ 20% |
| Busiest hour | max(p2p calls + meetings) by local hour |
| Most missed hour | as in call analytics |
| Heatmap | cell = p2p calls + meetings; tooltip calls / missed / meetings |
| By-hour stacked | answered / missed / meeting |
| Missed rate by hour | p2p only, ≥ 3 calls per hour rule |
| Audio quality | all classes, user side only |
| Meetings section | meeting habits (7.1) + external communication over all records, with extra KPI "External 1:1 calls" (count of p2p with external participant / of N 1:1 calls) |

### 7.6 Minimum-sample rules

- Percent KPIs with denominator 0 render "-" (not 0%).
- Averages require ≥ 1 connected meeting; else "-".
- Breakdown poor rate: always show the stream count next to the rate.
- Aggregate detail level: module hidden when the viewer's scope has fewer than `min_group_size` (5) people.

### 7.7 Drill-down mapping

| Source | Filters |
|---|---|
| Meetings KPI | `type=meeting,group` |
| Linked / group call | `type=meeting` / `type=group` |
| Heatmap cell | `type=meeting,group&dow=D&hour=H` |
| Duration bucket | `type=meeting,group&min_dur=lo&max_dur=hi−1` |
| Participant bucket | `type=meeting,group&min_people=lo&max_people=hi` |
| Camera / screen share | (future filter `modality=video|share`; not in the reference API — add to call search) |
| With external | `type=meeting,group&ext=1` |
| Quality breakdown row | `type=meeting,group&qdim=…&qval=…[&quality=poor]` |
| Incident row | `incident=<id>` |

Clicked number must equal the search total. Organizer rows link only for full-detail people.

## 8 UI

Tab "Meeting analytics" (`#meeting_analytics`):
1. Filter row: 7 / 30 / 90 day chips (group-scoped), source note.
2. "1 · Meeting habits": KPI strip (meetings with linked/group note, average duration, average participants,
   camera used, screen shared, with external, person-hours), duration distribution + participant distribution
   side by side, heatmap, meetings by hour.
3. "2 · Audio quality": note, KPIs, 4 breakdown tables, incident card.
4. "External communication" (meetings).
5. Optional "Top organizers" table (person-level).

Tab "Call + meeting analytics" (`#combined_analytics`): "1 · Call and meeting volume" (total card, missed rate,
busiest hour, most missed, heatmap, stacked by hour + missed rate), "2 · Audio quality", "3 · Meetings"
(habits + external over all records).

States: empty → one grey line ("No meetings in this range." / "No calls or meetings in this range.");
locked → permission card for `calls`; health permission missing → card only in the incident block;
aggregate → person-level blocks removed. Colour rules and table equivalents as in call analytics.

### Permission transparency (shared component, `../../platform/permissions.md`)

- **"Which permissions does this need?"** popover in the module header (reference: `../user-lookup/DESIGN.md`):
  lists this module's permissions least-privileged first with alternatives as "(alternative: X or Y)", ✓ per held
  option from the token `roles` claim, ✗ if none held, required/optional, what it is used for, how to grant
  (Azure Portal → App registrations → the app → API permissions → Add → Grant admin consent), and "not needed"
  permissions (module off or not used by this module).
  Rows: `CallRecords.Read.All` (required), `ServiceHealth.Read.All` (optional — incident overlay). Same popover on
  both tabs (meeting and combined).
- **No live Graph calls at view time** in this module, so no request-trace popover. Instead each data section has
  a **"What fed this?"** popover: ingest step, Graph endpoint, permission, and last successful run time
  (habits/quality/external → step `calls`; incident card → step `health`).

## 9 Access control

- Module keys `meeting_analytics` and `combined_analytics` (separately grantable; when migrating roles that had
  the old single analytics module, grant both with the same scope/detail).
- Scope: people modules; RLS shows a meeting if ≥ 1 participant is in scope; out-of-scope colleagues are counted
  in `people` only if visible — note: under restricted scope, participant counts reflect in-scope + external
  participants only (document this in the UI footnote).
- Detail: full / masked / aggregate as platform. Person-level blocks: external table, top organizers.
- Audit: `view_tab`, `forbidden`.

## 10 Logging events

| Event | Level | Context |
|---|---|---|
| `report.tab_build` | INFO | module, windows, duration_ms, meetings, masked |
| `analytics.query_slow` | WARNING | query, duration_ms |

## 11 i18n notes

- "Meeting" (entity) vs "Meetings" (section); "group call" always lower-case in running text.
- Bucket labels are translatable strings (`< 15 min`, `1–1.5 h`); numeric bucket bounds are code.
- Person-hours formatted with locale decimal separator; durations via the shared duration label (`{h} h {m} min`).
- Footnote about restricted-scope participant counts must be translatable.

## 12 Testing & acceptance criteria

- [ ] Fixture with 20 meetings (12 linked, 8 group), known durations/people/modalities: every KPI exact.
- [ ] Bot-joined meeting: bot not counted in `people`; person-hours unchanged by the bot.
- [ ] Device switch (iOS → Mac within 15 s) counts the person once in person-hours.
- [ ] Unconnected group record excluded from average duration; still counted in "Meetings".
- [ ] `unknown` call_type with join URL classified as linked meeting.
- [ ] Combined missed rate equals call-analytics missed rate for the same range.
- [ ] Each drill-down link total equals the clicked number.
- [ ] Masked / aggregate leak scan passes; small scope hides the modules.
- [ ] Screenshots light/dark/390 px; palette validated.

- [ ] "Which permissions does this need?" popover lists exactly: `CallRecords.Read.All` (required), `ServiceHealth.Read.All` (optional); ✓/✗ follow the token roles snapshot
      (test with a snapshot holding none, the least-privileged option, and only an alternative); grant steps shown.
- [ ] Every data section's "What fed this?" popover shows ingest step, endpoint, permission and the last
      successful `ingest_runs` time on both tabs.

## 13 Implementation steps for an agent

1. Ensure call-analytics shared read models and drill-down helper exist (build them first if not).
2. Migration: `meeting_load`, `meeting_organizers` (security_invoker).
3. Builders for `meeting_analytics` and `combined_analytics` sharing section renderers with call analytics
   (parameterised by class filter).
4. Register both modules in RBAC and package `calls`; lazy-tab keys.
5. Add `modality` filter to call search if camera/share drill-down is wanted.
6. Tests §12, screenshots, leak scan.

## 14 Open questions / future

- Calendar data (`Calendars.ReadBasic.All` or room/user mailbox access policy): planned vs actual duration,
  no-shows, meeting load per person per week.
- Attendance reports (late join / early leave vs scheduled time): requires online-meeting artifact access and an
  application access policy.
- Show participant counts under restricted scope as "≥ n" or hide them?
- Meeting size vs room capacity (needs `../rooms-devices/DESIGN.md` Place catalogue).
