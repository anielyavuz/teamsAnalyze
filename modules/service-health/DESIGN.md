# Module: Microsoft service health

> Module key: `health` (feature) — shipped inside the call-and-meeting analytics package by default; can be its own package. No dedicated tab: it powers sections in other tabs.

---

## 1 Purpose

Turn "audio quality dropped on Tuesday at 14:00" into an evidence-based sentence: *"Microsoft had an open Teams incident affecting calls at that time — it was not your network"*, or the opposite: *"Microsoft published no incident that could affect Teams calls in this period; the cause is most likely network, device or location."* Also explain late or missing usage-report numbers with Microsoft's reporting incidents.

## 2 Scope / Out of scope

**In scope:** ingest of Microsoft 365 service announcement issues (incidents and advisories) with history accumulation; relevance classification (`call`, `call_vdi`, `reports`, irrelevant); overlap with calls; quality during vs. outside incidents; incident banner on call detail; reporting-incident context for messaging/copilot/licensing.

**Out of scope:** Message Center change announcements (`/admin/serviceAnnouncement/messages`), service health overview (`/healthOverviews`) dashboards, alerting/notifications, root-cause analysis beyond time overlap.

## 3 Dependencies

[architecture](../../platform/architecture.md) · [graph-client](../../platform/graph-client.md) · [database](../../platform/database.md) · [ingest-scheduler](../../platform/ingest-scheduler.md) · [permissions](../../platform/permissions.md) · [auth-rbac](../../platform/auth-rbac.md) · [logging](../../platform/logging.md) · [i18n](../../platform/i18n.md) · [ui-shell](../../platform/ui-shell.md) · [testing](../../platform/testing.md)

Consumers: [call-analytics](../call-analytics/DESIGN.md) / [meeting-analytics](../meeting-analytics/DESIGN.md) and the call detail from [call-search](../call-search/DESIGN.md) (banner, quality-vs-incidents card), data from [call-ingest](../call-ingest/DESIGN.md) — main consumer; [messaging](../messaging/DESIGN.md) (reports incidents, last 45 days). Without the calls module the data can still be listed, but the main value (correlation) is absent.

## 4 Microsoft Graph endpoints & permissions

| Method | Endpoint | Version | Permission | Why |
|---|---|---|---|---|
| GET | `/admin/serviceAnnouncement/issues` (paged; optional `$filter` on `lastModifiedDateTime` for incremental runs) | v1.0 | `ServiceHealth.Read.All` | Incidents/advisories with service, classification, status, start/end, resolved flag, posts |

### Quirks and real-API findings

- **History depth ≈ 9–10 months** (first fetch 2026-09-26 returned 79 issues, oldest from December 2025). Accumulate in the DB; do not rely on Microsoft keeping older issues.
- **"Open" is not `isResolved = false`.** Microsoft sometimes sets an end time but leaves `isResolved=false` (observed: end in late August, still unresolved weeks later). Definition: **open = not resolved AND (no end time OR end time in the future).** A naive count gave 3 open issues; the correct count was 1.
- **Service name + time overlap alone is useless**: long-running unrelated advisories under "Microsoft 365 suite" (e.g. a community-admin auditing issue) matched every call; the word "chat" matched Copilot Chat issues. In 9 months the "Microsoft Teams" service had no call-quality incident at all.
- **`feature` / `featureGroup` are unreliable** for relevance (a speed-dial issue filed under Meetings, an emergency-calling issue under Calling). Classify on title + impact text instead.
- Issues are updated in place (status, end, new posts): upsert by ID, keep the latest `lastModifiedDateTime`.

## 5 Data model

**Issues** (`service_issues`, PK = issue ID such as `TM…` / `MO…`):

| Column | Meaning |
|---|---|
| service | e.g. "Microsoft Teams", "Microsoft 365 suite" |
| classification | `incident` / `advisory` |
| status | e.g. `serviceDegradation`, `serviceRestored`, `postIncidentReviewPublished` |
| origin | microsoft / thirdParty / customer |
| title, impact | `title`, `impactDescription` |
| feature_group, feature | stored but not used for relevance |
| start_at, end_at (null = open-ended), is_resolved, last_modified_at | |
| raw | full issue JSON including posts |
| first_seen_at, updated_at | |

**Settings:** `health_services` (default "Microsoft Teams,Microsoft 365 suite"), `incident_vdi` (off/on; default off).

**Classification function** (single source of truth, in SQL, immutable, with a dedicated test): returns `reports`, `call`, `call_vdi`, or null. **Classified view** adds category and `is_open`.

**Overlap view:** call × issue where the issue (category `call`, or `call_vdi` when `incident_vdi=on`) overlaps the call interval (open issue extends to now). **Quality comparison function** (range): poor-stream rate during vs. outside overlapping incidents, over classified, non-service streams.

No person data; no RLS needed on issues (overlap views inherit call RLS via security-invoker).

## 6 Ingest / request flow

- In every scheduled run (cheap; usually 1–2 pages) or hourly. Own ingest-run row, source `graph.health`.
- Gate: `ServiceHealth.Read.All` present and package enabled; else skip with warning.
- Fetch all pages → upsert each issue (raw + columns). Failure does not stop other ingest steps.

## 7 Derived logic & metric definitions

**Relevance classification** (rules in order; text = title + impact, case-insensitive, whole-word):

1. **`reports`** — text mentions report/reports/reporting/usage data/analytics **and** teams/usage/active users/admin center(s). Checked first ("Teams usage report…" is not a call incident).
2. Service not in the call-relevant set → **null**. For "Microsoft 365 suite" the text must also mention "teams", else null.
3. **Excluded (null)** regardless: emergency calling (emergency/E911/911/112), contacts/speed dial/address book, transcripts/recordings/recap/registrations (content or form issues, not quality).
4. **`call_vdi`** — VDI/Citrix/virtual desktop/AVD/Horizon specific; counts as `call` only if `incident_vdi=on`.
5. **`call`** — strong terms (audio, video, voice, screen sharing, PSTN, latency, jitter, packet, echo, joining a meeting, calls dropping…), or weak terms (call, meeting, webinar, town hall, live event) **only when** no off-topic context (chat, channel, file, calendar, notes, invitation…) is present, or a Teams access/sign-in outage.
6. Otherwise null (irrelevant).

Reference first-fetch distribution (79 issues): ~20 reports, ~4 call, ~55 irrelevant (before the later stricter exclusions). After tightening, no real calls overlapped an incident — the expected, correct result for that period.

**Open** = not resolved and (end null or end > now).

**Quality during vs outside** = poor streams ÷ classified streams, split by whether the call overlapped a `call` incident; show also the number of affected calls per incident.

## 8 UI

No own tab. Sections:

- **Call detail:** amber incident banner when the call overlaps a `call` incident: issue ID, title, status, interval.
- **Call analytics → Audio quality:** "Microsoft service incidents" card: poor-audio rate during vs outside incidents + incident list with affected-call counts. Empty: "Microsoft published no incident that could affect Teams calls in this range; the cause is most likely network, device or location."
- **Messaging (and other report-based tabs):** list of `reports` incidents from the last 45 days as context for delayed/missing numbers.
- Amber (`--incident` token) is reserved for Microsoft incidents.

States: permission missing → section-level locked card with `ServiceHealth.Read.All`; permission removed later → keep showing stored issues with "from the last permitted fetch, may be outdated".

## 9 Access control

- Feature key `health`; no own RBAC module — visible inside the modules that show it (calls, messaging).
- Issues are tenant-wide Microsoft data (no person data); overlap results follow call scope.
- No audit beyond the host tab's view audit.

## 10 Logging events

`health.list` (INFO timed; issues, pages), `health.done` (INFO; total, open, open call-relevant), `health.no_permission` (ERROR), `health.failed` (ERROR exc), `ingest.skipped`.

## 11 i18n notes

Issue titles/impact text are Microsoft's English and are shown as-is (not translated); category labels, banner and card texts translated. Dates via platform formatters in app time zone.

## 12 Testing & acceptance criteria

- Classification test table: at least one fixture per rule (reports-first, suite-without-teams, emergency, speed dial, recordings, VDI with setting off/on, strong term, weak term with/without off-topic context, irrelevant service).
- Open definition: end set + unresolved + end in the past → not open.
- Overlap: open-ended issue overlaps calls up to now; VDI incidents excluded unless setting on.
- Locked section when permission missing; stale note when removed.
- Screenshots of banner and card (light/dark/390 px).

## 13 Implementation steps for an agent

1. Migration: issues table, settings, classification function, classified view, overlap view (security invoker), quality comparison function.
2. Feature `health` in the permission map; attach to the calls package (or own package).
3. Graph call (paged), upsert, scheduler step, logging.
4. Classification test suite.
5. UI sections in calls and messaging tabs; i18n; screenshots.

## 14 Open questions / future

- Keyword classification can misfire (e.g. a Teams messaging-link issue counted as `call`); consider a manual override per issue (admin marks relevant/irrelevant).
- Message Center announcements as change context.
- Feeding incidents into a future AI explanation layer.
