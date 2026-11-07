# Module: Messaging (Teams usage reports)

> Module key: `messaging` · Report tab: **Messaging** · Data: scheduled ingest of Microsoft 365 Teams usage reports — **daily counts only, no message content.**

---

## 1 Purpose

Show Teams messaging volume and patterns per day, person and team: chat vs. channel messages, posts vs. replies, urgent messages, reactions and mentions per team, active people, weekday pattern, and the client platforms people use — without reading any message.

## 2 Scope / Out of scope

**In scope:** per-person daily activity, per-team daily activity, device/platform usage snapshot, KPIs and charts, people and team tables, context from Microsoft service-health "reports" incidents (when the service-health module is on).

**Out of scope (deliberate):**

- Message content and per-message APIs (`/chats/getAllMessages`, `/chats/{id}/messages`, channel messages). They are **protected APIs** (Microsoft request form + approval) and return content. Do not request `Chat.Read.All` / `ChannelMessage.Read.All` for this module.
- Hourly message distribution: the reports are daily only.
- Bot/app posts and e-mails sent to channels (not counted by the reports).
- Per-person monitoring metrics such as "out-of-hours messaging" (privacy; aggregate only if ever added).

## 3 Dependencies

[architecture](../../platform/architecture.md) · [graph-client](../../platform/graph-client.md) (302 → CSV download without Authorization; URL never logged) · [database](../../platform/database.md) · [ingest-scheduler](../../platform/ingest-scheduler.md) · [permissions](../../platform/permissions.md) · [auth-rbac](../../platform/auth-rbac.md) · [scope-masking](../../platform/scope-masking.md) · [logging](../../platform/logging.md) · [i18n](../../platform/i18n.md) · [ui-shell](../../platform/ui-shell.md) · [testing](../../platform/testing.md) · [deployment](../../platform/deployment.md)

Module dependencies: platform identities/directory (person key `user:<object id>`, names, scope); optional [service-health](../service-health/DESIGN.md) for report-delay incidents; optional calls module for a combined "messages + calls" daily view; licensing consumes last-message-day and Teams last-activity.

## 4 Microsoft Graph endpoints & permissions

| Method | Endpoint | Version | Permission | Why |
|---|---|---|---|---|
| GET | `/reports/getTeamsUserActivityUserDetail(date=YYYY-MM-DD)` | v1.0 | `Reports.Read.All` | Per person per day: chat, channel (post/reply), urgent messages, calls, meetings, audio/video/screen-share seconds, last activity, is licensed |
| GET | `/reports/getTeamsTeamActivityDetail(date=YYYY-MM-DD)` | v1.0 | `Reports.Read.All` | Per team per day: channel messages, posts, replies, reactions, mentions, urgent, active users/channels, guests, shared channels, external users |
| GET | `/reports/getTeamsDeviceUsageUserDetail(period='D30')` | v1.0 | `Reports.Read.All` | Snapshot: platforms each person used in last 30 days |

All three answer **302** with a `Location` to a short-lived download URL; the CSV is fetched from that URL **without an Authorization header**. The URL serves data without authentication for several minutes: **never log it**, never store it, never include it in error messages.

### Quirks and real-API findings

- **`date=` accepts at most 28 days back including today** — documentation says 30. `today-28` and older → 400 "Only support data for the past 28 days" (measured 2026-09-26). Fetch window: `today-27 … yesterday`.
- **24–48 h delay;** Microsoft revises the last ~3 days. A not-yet-processed day returns 0 rows and fills later.
- **New tenants return header-only CSVs for every report** for a while (even the Office 365 active-user report, which normally lists inactive licensed users too). Not a permission problem.
- **Only active people/teams appear** for a day; absence ≠ fetch failure. "Fetched but empty" vs "never fetched" is known only from the fetch log table.
- **CSV may start with a UTF-8 BOM.**
- **Header spelling, case and order are inconsistent** and differ from documentation (e.g. `Team type` vs `Team Type`, `Active users` vs `Active Users`, columns appended at the end, an undocumented `Is Deleted` in the team report, a double space in `Used  Chrome OS`). Normalize every header to snake_case and read by name, never by position. New `Used …` platform columns must be picked up automatically.
- **Concealed names are on by default** (Microsoft 365 admin center → Settings → Org settings → Services → Reports → "Display concealed user, group, and site names in all reports"). When on, UPN and user ID arrive as hashes and cannot be matched to directory or call data.
- Counts include only messages written in Teams clients.

## 5 Data model

| Table | Grain | Key columns |
|---|---|---|
| `usage_user_daily` | person × day | activity_date, user_key (`user:<oid>` or `hash:<value>`), upn, is_concealed, private_chat_msgs, team_chat_msgs, post_msgs, reply_msgs, urgent_msgs, calls, meetings_attended, meetings_organized, audio_s, video_s, screenshare_s, has_other_action, is_licensed, last_activity_date, report_refresh_date, raw |
| `usage_team_daily` | team × day | activity_date, team_key (team ID or `name:<team name>`), team_id, team_name, team_type, active_users, active_channels, guests, reactions, meetings_organized, post_msgs, reply_msgs, channel_msgs, urgent_msgs, mentions, active_shared_channels, active_external_users, last_activity_date, report_refresh_date, raw |
| `usage_device` | person × snapshot | snapshot_date, user_key, upn, is_concealed, platforms (text array), last_activity_date, period, raw |
| `usage_fetches` | report × day | report, activity_date, row_count (0 allowed), report_refresh_date, run_id — **the only source for "fetched, no activity" vs "never fetched"** |

Column meanings (Microsoft definitions): **chat** = messages in 1:1, group and meeting chats; **channel** = posts + replies in team channels; urgent = messages marked urgent.

**User key rule:** if the report's user ID is a GUID → `user:<lower-case GUID>`, concealed = UPN lacks `@`; otherwise `hash:<value>`, concealed = true. Concealed rows are kept but never joined to people.

Derived (SQL, not application code): daily totals view (chat/channel/urgent/total + active people), combined activity view (messages + calls per day, if calls module), per-person stats function for a range (chat, channel, urgent, total, active days, average per active day, last activity) with names from identities.

RLS: person-day and device rows by the viewer's scope (person key); team rows visible per platform rule (team-level, no person names; decide in platform scope doc whether department scope sees all teams).

## 6 Ingest / request flow

Every scheduled run (e.g. every 15 min is fine; most runs fetch only 4 days):

1. For each daily report (user, team): days = today-27 … yesterday. Fetch = days never fetched **+ the last 4 days always** + (if the report has **never** returned a row) all days that came back empty.
2. Per day: request → 302 → download (no Authorization) → normalize → in one transaction delete that day's rows and insert the new ones, write `usage_fetches` (even with 0 rows).
3. **Empty-day rule:** while a report has never produced data, empty days are re-asked every run; after the first non-empty response, previously empty days are fetched once more, then settle. Real empty days (weekends) after go-live are not re-asked forever.
4. Device report: one `period='D30'` request per run (or daily), stored as a snapshot.
5. A single day's error (e.g. 400) does not stop other days → run `partial`. 403 stops the whole messaging ingest (others unaffected) with `usage.no_permission`.
6. Own ingest-run row, source `graph.reports`: `records_seen` = days fetched, `records_inserted` = rows written, `records_unchanged` = days skipped.

Cost reference: first fill ~114 requests (28 days × 2 reports × request+download + device); steady state ~18.

## 7 Derived logic & metric definitions

- **Total messages** = chat + channel.
- **Active people (period)** = distinct person keys with ≥1 message in range.
- **Per person per day** = total ÷ (active people × days with data).
- **Busiest day** = max daily total.
- **Change vs previous period** shown only if the previous equal-length period is fully fetched.
- **Weekday pattern** = average total per weekday over the range.
- **Person × day heatmap** = top 15 people by total; 5 buckets + zero.
- Data freshness = latest `report_refresh_date`; last 3 days marked provisional.
- Rates on small samples are not shown (platform rule).

## 8 UI

Tab **Messaging**, top to bottom: range buttons (7 / 14 / 28 days, default 14; changes everything) · KPI strip (total, chat, channel, active people, per person per day, busiest day) · daily stacked columns chat + channel (weekends shaded) with table equivalent · weekday columns (peak labelled) · person × day heatmap (single-hue scale, reversed in dark theme) · people table (chat, channel with post/reply, urgent, total, share bar, active days, daily average, last activity) · teams table (channel messages, posts, replies, reactions, mentions, active users) · "reports" service incidents from the last 45 days as context (if service-health on).

People without a display name show the UPN local part with the full UPN below. Concealed people show a "concealed" badge and the hash.

States: permission missing → locked tab card (`Reports.Read.All`); never produced data → note "Microsoft has not produced Teams usage reports for this tenant yet"; concealed names detected → red setup card with the admin-center path; days pending → provisional marker.

## 9 Access control

- RBAC module `messaging`.
- RLS on person-day and device rows by scope; aggregate KPIs computed over visible rows only.
- Masking: person names in tables, heatmap and tooltips go through the platform masking pipeline (person-level markers); aggregate view shows counts only; small groups suppressed.
- Audit: tab view, CSV export.

## 10 Logging events

`usage.start` (INFO; days), `usage.day` (DEBUG; report, date, rows), `usage.day_error` (WARNING; report, date, status), `usage.no_permission` (ERROR), `usage.done` (INFO timed; days fetched, rows, unchanged), `usage.concealed` (WARNING; report, concealed rows), `report.download` (DEBUG; report, rows, bytes, ms — **no URL**), `report.download_failed` (ERROR; status — **no URL**), `ingest.skipped`. The graph client must scrub `Location` headers and download URLs from every log path; add a leak query/test.

## 11 i18n notes

"Chat" / "Channel" / "Urgent" terminology consistent everywhere and translated; weekday names and dates via platform formatters; plural forms for people/days.

## 12 Testing & acceptance criteria

- Fake report layer: 302 + CSV with BOM, inconsistent headers, double-space header, extra columns; 400 for `today-28`; header-only responses during "new tenant" phase then data → empty-day rule refetches once.
- Download request carries no Authorization header; the URL never appears in logs.
- Concealed rows get `hash:` keys and the setup card appears.
- KPIs equal SQL totals over the full range (not over truncated tables).
- Scope: department viewer sees only in-scope people.
- Screenshots: light/dark/390 px, 7/14/28 days.

## 13 Implementation steps for an agent

1. Migration: four tables + views/function + RLS.
2. Feature `usage` (Reports.Read.All), package `messaging`.
3. Graph client report download (shared with licensing/copilot) with URL scrubbing.
4. Pure header normalization and row derivation (user, team, device); user key rule.
5. Ingest with fetch-window, last-4-days and empty-day rules; fetch log; logging.
6. SQL aggregates; tab UI; masking markers; i18n.
7. Tests (§12), screenshots, module notes with quirks.

## 14 Open questions / future

- Longer history only accumulates in the database (Graph gives 28 days): consider a retention setting.
- Team-level scope rules for department-scoped viewers.
- Optional per-team drill-down (still counts only).
