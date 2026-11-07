# Module: Copilot analytics

> Module key: `copilot` · Report tab: **Copilot analytics** · Data: scheduled ingest of Microsoft 365 Copilot usage reports (counts and dates only).
> **Interaction content is never read.**

---

## 1 Purpose

For organizations paying for Microsoft 365 Copilot licenses:

- Are Copilot licenses used? Who never used it, who stopped (**reclaim candidates**)?
- In which apps is Copilot adopted (Teams, Outlook, Word, Excel, PowerPoint, OneNote, Loop, Copilot Chat, Edge, Microsoft 365 app)?
- Which departments adopted it?
- **Teams opportunity:** people who attend many meetings but never use Copilot in Teams (training target; meeting recap requires transcription).
- Organization-wide daily trend of enabled/active users and prompts.

## 2 Scope / Out of scope

**In scope:** per-person Copilot usage report (last use per app, prompts in period, active days), org-wide daily trend, derived states, department aggregation, Teams opportunity (when the calls module is on), per-SKU usage feed to the licensing module.

**Out of scope:**

- Prompt/response content — the `interactionHistory` API (`/copilot/users/{id}/interactionHistory/getAllEnterpriseInteractions`) is deliberately not used and its permission is not requested.
- Unlicensed Copilot Chat users: not in Graph usage reports; would require Purview audit log search (future, §14).
- Copilot license assignment date / time-to-first-use (Graph does not provide a Copilot assignment date; the Microsoft 365 active-user report's assignment dates do not include Copilot).
- Prices and ROI.

## 3 Dependencies

[architecture](../../platform/architecture.md) · [graph-client](../../platform/graph-client.md) (report download: 200 or 302 → CSV) · [database](../../platform/database.md) · [ingest-scheduler](../../platform/ingest-scheduler.md) · [permissions](../../platform/permissions.md) · [auth-rbac](../../platform/auth-rbac.md) · [scope-masking](../../platform/scope-masking.md) · [logging](../../platform/logging.md) · [i18n](../../platform/i18n.md) · [ui-shell](../../platform/ui-shell.md) · [testing](../../platform/testing.md) · [deployment](../../platform/deployment.md)

Module dependencies: platform directory (department, scope, UPN match) — required; calls module (meetings attended in last 28 days) — optional, only for the Teams opportunity; licensing module consumes this module's "used in last N days" per UPN for the Copilot SKU row.

## 4 Microsoft Graph endpoints & permissions

| Method | Endpoint | Version | Permission | Alternatives | Why |
|---|---|---|---|---|---|
| GET | `/copilot/reports/getMicrosoft365CopilotUsageUserDetail(period='D28',version='v2')` | v1.0 | `Reports.Read.All` | — | Per licensed person: last activity per app, prompt counts, active days |
| GET | `/copilot/reports/getMicrosoft365CopilotUserCountTrend(period='D28',version='v2')` | v1.0 | `Reports.Read.All` | — | Org-wide per-day enabled/active users and prompts |

Response is CSV (either directly 200 or via 302 to a short-lived download URL — handle both; never send Authorization to the download URL and never log it).

### Quirks and real-API findings (observed 2026-09-28)

- **`version='v2'` must be passed explicitly.** Documentation says v2 is the default; in practice omitting `version` returns **v1** (no prompt-count or active-day columns) and `period='D30'` returned **0 rows**. With `version='v2'` the valid periods are `D7`, `D28`, `D90`, `D180` — use **D28**.
- **`$format=application/json` on v1.0 → 400** "JSON format is not supported". (Beta returns JSON but was empty in the reference tenant.) Use CSV.
- **Trend lags behind the person report:** a day with prompts in the person report showed 0 in the trend. Do not reconcile the two; label each with its own refresh date.
- **Only Copilot-licensed people appear.**
- No user ID column: match directory by **lower-cased UPN**; concealed-names setting breaks matching (same as messaging/licensing).
- Data lags 2–3 days; daily resolution.

## 5 Data model

**Per-person snapshot** (`copilot_user_activity`; PK snapshot date × UPN; same snapshot rewritten):

| Column | Meaning |
|---|---|
| snapshot_date | CSV `Report Refresh Date` |
| upn_lower, display_name | |
| last_activity | last activity in any Copilot app |
| teams_last, outlook_last, word_last, excel_last, powerpoint_last, onenote_last, loop_last, chat_last, chat_work_last, chat_web_last, edge_last, m365app_last | per-app last activity date |
| prompts_all, prompts_chat_work, prompts_chat_web | prompts submitted in the period |
| active_days, period_days | active usage days in the period |
| raw, run_id | CSV row as JSON (original headers) |

**Org trend** (`copilot_daily`; PK report date): refresh_date, any_enabled, any_active, prompts, `apps` (JSON map app → [enabled, active]), raw, run_id.

**Settings:** `copilot_idle_days` (default 30).

RLS: person rows visible only when the UPN matches an in-scope directory person; trend rows visible only to viewers with **organization-wide** scope (they are tenant-wide numbers that would otherwise leak outside a department scope).

## 6 Ingest / request flow

- Scheduled every **6 hours** (data is daily; 2 requests per run). Own ingest-run row, source `graph.copilot`.
- Gate: package `copilot` enabled and `Reports.Read.All` present; otherwise skip with `ingest.module_off` / `ingest.skipped`.
- Download both CSVs → normalize headers (BOM, case, spacing) → pure derivation → upsert person snapshot (delete+insert for the snapshot date) and trend rows (latest refresh wins per report date).
- Empty person report → warning `copilot.empty` (no Copilot licenses, or reports not produced yet); not a failure. A failure here never stops other ingest steps.

## 7 Derived logic & metric definitions

- **State per licensed person** (latest snapshot):
  - `never` — no last activity in any app.
  - `stale` — last activity older than `copilot_idle_days` (stopped using it).
  - `heavy` — active days in 28 ≥ **10**.
  - `active` — otherwise.
- **Reclaim candidates** = `never` + `stale`. The KPI note lists only non-zero parts ("1 person never used it"). No "N days unused since assignment" (assignment date unknown).
- **Used %** = (`heavy` + `active`) ÷ licensed people.
- **Per-app adoption** = people with that app's last activity within 28 days ÷ licensed people.
- **Teams opportunity** = attended ≥ **3** meetings in the last 28 days (calls module) **and** no Copilot-in-Teams use within 28 days. Hidden if the calls module is off.
- **Average active days** over licensed people.
- **Department table:** licensed, used, used %, prompts; groups below the platform's small-group threshold are suppressed.
- **Trend:** if every day is 0, show an explanation instead of an empty chart.

## 8 UI

Tab **Copilot analytics**: status line (snapshot date, last run) · KPI strip (licensed, using %, reclaim candidates with name tooltip, prompts in 28 days, average active days) · per-app adoption bars · department table (person-level marker) · daily trend chart + table equivalent (org scope only) · people table (person-level marker; state pill, per-app last use, prompts, active days, Teams opportunity chip) · Teams opportunity card.

Name tooltips on KPI cards must embed names only inside person-level markers so the aggregate view strips them and the masked view masks them.

States: permission missing → locked tab card (`Reports.Read.All`); module off → tab absent; empty report → note "No Copilot-licensed people in the report (or Microsoft has not produced it yet)"; trend all zero → explanatory note; concealed names → setup card.

## 9 Access control

- RBAC module `copilot`; new module enabled by default only for the system admin role (sensitive per-person data); other roles granted in Admin.
- Scope/masking: person rows RLS by directory UPN; trend only at org scope; names masked per platform; small departments suppressed.
- Audit: tab view, CSV export.

## 10 Logging events

`copilot.fetch` (INFO timed; rows per report, period, version), `copilot.empty` (WARNING), `copilot.no_permission` (ERROR), `copilot.failed` (ERROR exc), `copilot.saved` (INFO; persons, days, unmatched UPN count), `ingest.skipped` / `ingest.module_off`. Never log UPN lists, the download URL, or tokens.

## 11 i18n notes

App names are brand names (not translated) except generic ones ("Copilot Chat (work)", "Microsoft 365 app"). State labels and the KPI note are translatable with plurals. Dates via platform formatters.

## 12 Testing & acceptance criteria

- Fake v2 CSV with BOM and all states → correct state counts; v1-shaped CSV (missing prompt columns) is detected and logged as a warning, not silently zero.
- Request URL always contains `version='v2'` and `period='D28'`.
- 302 and 200 responses both parsed; download URL absent from logs.
- Department-scoped viewer: person rows filtered, trend hidden.
- Teams opportunity hidden when calls module off; correct when on (≥3 meetings, no Teams Copilot).
- Screenshots light/dark/390 px; locked and empty states.

## 13 Implementation steps for an agent

1. Migration: two tables, setting, RLS (person by directory UPN; trend org-scope only), role grant to system admin.
2. Register feature `copilot` (Reports.Read.All) and package `copilot`.
3. Graph report download supporting the `/copilot/reports` root.
4. Pure CSV → row derivation (per-app columns from a single app list constant).
5. Scheduler step (6 h), gates, logging.
6. Report builder: states, reclaim, adoption, departments, trend, opportunity.
7. Tab UI with person-level markers; i18n; tests; screenshots.
8. Expose "Copilot used in last N days per UPN" for the licensing module.

## 14 Open questions / future

- Unlicensed Copilot Chat usage via Purview audit (`CopilotInteraction` records; needs a different permission and has privacy implications) — phase 2, not planned.
- Longer history: the person report is a 28-day window; trends beyond that come only from accumulated snapshots.
- Whether to show prompts per person at all (some organizations consider it monitoring); possibly an admin setting to aggregate only.
