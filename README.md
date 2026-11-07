# teamsAnalyze — design documentation

teamsAnalyze is a self-hosted **Microsoft Teams analytics service**. It reads tenant data from
**Microsoft Graph** with an app-only identity (client credentials), stores the raw responses in
**PostgreSQL** (JSONB as the source of truth, plus derived tables), and serves a **server-rendered web
application** with Microsoft Entra ID sign-in, role-based access per module, row-level scope, masking,
an append-only access audit, a structured application log, and an English/Turkish (extensible) UI.

This repository contains **design documents only**. There is no code here. The documents are written so
that a coding agent can implement the whole system, step by step, without access to any earlier code.

The product is built **incrementally**: the platform first (with zero analytics modules), then one module
at a time, only when it is needed. The first module is *User lookup* (one person's licenses and effective
Teams policies).

## What the system never does

- It never reads **content**: no chat or channel message bodies, no meeting transcript text, no recording
  audio/video, no Copilot prompts/responses. Only counts and "exists / does not exist" metadata.
- It never stores Microsoft tokens or passwords of signed-in users.
- It never sends data anywhere except to the operator's own database (no telemetry).
- It never requests Graph permissions for modules that are not enabled.

## How the documents are organised

| Path | What it is |
|---|---|
| `README.md` | This file: overview, read order, module catalog |
| `ROADMAP.md` | Build order as small shippable steps, with prerequisites and definition of done |
| `CONVENTIONS.md` | Rules every change and every module must follow + the **module doc template** |
| `platform/architecture.md` | Components, data flow, process model, repository layout, module manifest |
| `platform/graph-client.md` | Token handling, throttling, retries, paging, `$expand` quirks, beta policy |
| `platform/database.md` | PostgreSQL schema rules, raw vs derived data, reprocess, migrations, roles, RLS |
| `platform/logging.md` | `app_log` design, event naming, levels, never-log list, scrubbing, log CLI |
| `platform/permissions.md` | Feature → Graph permission map, packages, token evaluation, probes, locked UI |
| `platform/auth-rbac.md` | Entra sign-in, sessions, CSRF, roles × modules, admin module, preview, audit |
| `platform/scope-masking.md` | **Security-critical**: scope types, detail levels, RLS + masking pipeline, leak tests |
| `platform/i18n.md` | English source strings, gettext/Babel catalogs, formatting, language-independent data, timezone |
| `platform/ui-shell.md` | Design tokens, layout, components, themes, accessibility, data-viz colour rules |
| `platform/ingest-scheduler.md` | Ingest runner, run ids, step isolation, cadence, freshness line |
| `platform/deployment.md` | Generic Linux/systemd + reverse proxy, environment variables, backups, deploy |
| `platform/testing.md` | Test database, fake Graph layer, leak tests, screenshots, acceptance gates |
| `modules/<folder>/DESIGN.md` | One design per module, all following the template in `CONVENTIONS.md` |

## Read order for an implementing agent

1. `README.md` (this file) and `CONVENTIONS.md` — fully. The conventions are binding.
2. `ROADMAP.md` — find the step you are implementing and its definition of done.
3. `platform/architecture.md` — the shape of the system and the module manifest contract.
4. The platform documents your step touches. For **Step 0** that is all of `platform/*.md`, in this
   order: database → logging → graph-client → permissions → auth-rbac → scope-masking → i18n → ui-shell →
   ingest-scheduler → testing → deployment.
5. The module's `DESIGN.md`, then its dependencies listed in its section 3.
6. Implement following the module's section 13 checklist. Verify against section 12 and `platform/testing.md`.
7. Update the documents when reality differs (see "Documentation rule" in `CONVENTIONS.md`).

When two documents disagree, the more specific one wins (module doc over platform doc), **except** for
security and privacy rules in `CONVENTIONS.md`, `platform/scope-masking.md` and `platform/auth-rbac.md`,
which no module may relax. Raise the conflict in the module doc's "Open questions" section.

## Module catalog

RBAC module keys are the folder name with `-` replaced by `_` (for example `user-lookup` → `user_lookup`)
unless the module document states otherwise.

| Module | Folder / design | Summary | Status |
|---|---|---|---|
| User lookup | [modules/user-lookup](modules/user-lookup/DESIGN.md) | One person's licenses, service plans and effective Teams policies (live, read-only) | designed |
| Licensing | [modules/licensing](modules/licensing/DESIGN.md) | Subscriptions, seats, assigned vs used licenses, idle-license candidates, cost | designed |
| Copilot | [modules/copilot](modules/copilot/DESIGN.md) | Microsoft 365 Copilot license usage and adoption (no prompts/content) | designed |
| Messaging | [modules/messaging](modules/messaging/DESIGN.md) | Daily message/activity counts per person and team from usage reports (no content) | designed |
| Transcripts | [modules/transcripts](modules/transcripts/DESIGN.md) | Whether meetings were transcribed/recorded — metadata only, never text or media | designed |
| Service health | [modules/service-health](modules/service-health/DESIGN.md) | Microsoft service incidents and their relevance to calls/meetings | designed |
| Call ingest | [modules/call-ingest](modules/call-ingest/DESIGN.md) | Call records (1:1 calls, meetings) ingest, identities, derive layer, media streams | designed |
| Call analytics | [modules/call-analytics](modules/call-analytics/DESIGN.md) | 1:1 call volume, answer rate, missed calls, audio quality, external communication | designed |
| Meeting analytics | [modules/meeting-analytics](modules/meeting-analytics/DESIGN.md) | Meeting habits, hourly volume, quality, external participation, person-hours | designed |
| Call search | [modules/call-search](modules/call-search/DESIGN.md) | Server-side paged search/filter/export of call and meeting records, record detail | designed |
| Rooms and devices | [modules/rooms-devices](modules/rooms-devices/DESIGN.md) | Teams Rooms fleet, device health, availability, utilisation, room quality | designed |
| About / transparency | [modules/about-transparency](modules/about-transparency/DESIGN.md) | What the system collects, which permissions it uses and why, traffic estimate | designed |

Platform screens that are **not** modules in this catalog but exist from Step 0: sign-in, Admin (users,
roles, settings, access audit, preview-as-user), Logs and permissions. See `platform/auth-rbac.md`,
`platform/logging.md` and `platform/permissions.md`.

## Glossary

| Term | Meaning |
|---|---|
| Module | A unit of RBAC: one rail entry / screen group with its own key (`user_lookup`, `rooms`, …) |
| Package | A unit of enablement: a set of modules + Graph features that an operator switches on or off |
| Feature | A Graph capability with its required permissions (`calls` → `CallRecords.Read.All`) |
| Scope | *Whose* data a viewer may see in a module (whole org, department, own team, self, locations) |
| Detail level | *How much* of each in-scope person is shown: `full`, `masked`, `aggregate` |
| Raw | Graph response stored verbatim (JSONB). Source of truth |
| Derived | Columns/tables computed from raw by pure functions; always recomputable (`reprocess`) |
| Run | One execution of one ingest step, a row in `ingest_runs` |
| Invocation | One OS process (CLI or web worker) — all its log lines share an invocation id |

## Licence and contributions

The licence of the implementation is decided by the repository owner (the reference implementation used
AGPL-3.0-or-later with an SPDX header in every source file). Documents in this repository must never
contain real tenant names, domains, GUIDs, people, e-mail addresses, IP addresses, host names or secrets;
use placeholders such as `contoso.com` and `<tenant-id>`.
