# Roadmap — build order

The system is built in small, shippable steps. Each step ends in a deployable state that is useful on its
own and leaves the system secure. **No step may weaken a rule of an earlier step.** A step is done only
when its definition of done (DoD) is met, including logging, i18n, tests and documentation.

Common DoD for every step (in addition to the step's own list):

- All conventions in `CONVENTIONS.md` hold; new log events are listed in the relevant doc.
- English source strings marked; Turkish catalog complete; pseudo-locale shows no unmarked text.
- Screens touched have light, dark and 390 px screenshots reviewed.
- Migrations apply cleanly on an empty database **and** on a copy of the previous step's database.
- Security-relevant changes passed the review gate (`CONVENTIONS.md` §8).
- Docs updated (module doc version history, platform docs if the platform changed).

---

## Step 0 — Platform minimum (zero analytics modules)

Goal: an operator can install the service, sign in with Microsoft, manage users and roles, see the Graph
permission status and the application log — with no analytics module present.

Build exactly this (each item references its spec):

1. **Configuration loader** — environment variables / env file, fail fast with a clear `config.missing`
   event (`platform/deployment.md`).
2. **Application log** — `app_log` table, writer with background batching, scrubbing, retention, log
   reader CLI (`platform/logging.md`).
3. **Database layer** — connection helpers (owner/collector, restricted web role), migration runner,
   `schema_migrations`, `settings`, `ingest_runs`, the restricted role and default privileges
   (`platform/database.md`).
4. **Graph client** — token provider, shared throttle, retries, paging helpers, quiet mode, structured
   errors; no feature-specific calls yet (`platform/graph-client.md`).
5. **Permission framework** — feature/package registry (empty of module features except core), token role
   evaluation, `permission_checks`, enabled packages setting (default: core only)
   (`platform/permissions.md`).
6. **Auth and RBAC** — Entra sign-in (auth code + PKCE), server sessions, CSRF, users/roles/role-modules,
   time-limited grants, lockout protection, append-only `access_audit`, bootstrap CLI to create the first
   administrator (`platform/auth-rbac.md`).
7. **Admin screens** — users, roles matrix, role editor (view + scope/detail columns present but only
   `all`/`full` selectable until Step 3), settings (enabled packages, timezone display), access audit,
   preview-as-user (`platform/auth-rbac.md`).
8. **Module registry** — manifest contract and loading (`platform/architecture.md`); the app runs with an
   empty registry.
9. **i18n** — gettext/Babel catalogs (en source, tr), locale helpers, language switch, pseudo-locale,
   extraction/check tool (`platform/i18n.md`).
10. **UI shell** — tokens, rail, top bar, cards, tables, drawer, empty/locked/error states, theme switch
    (`platform/ui-shell.md`).
11. **Logs and permissions screen** — permission table + admin-consent instructions, grouped log viewer
    (`platform/logging.md`, `platform/permissions.md`).
12. **Ingest runner** — CLI `ingest` and `ingest --scheduled` with zero registered steps, step isolation,
    run ids, exit codes; systemd timer unit documented (`platform/ingest-scheduler.md`).
13. **Deployment** — systemd units, reverse proxy, backups, deploy script contract
    (`platform/deployment.md`).
14. **Test harness** — test-DB helper, fake Graph transport, headless screenshot script, leak-scan helper
    (empty fixtures) (`platform/testing.md`).

Prerequisites: a Linux host with PostgreSQL ≥ 15, Python ≥ 3.11, an Entra app registration for sign-in
and one for data collection (may be created later; Step 0 works without Graph permissions).

Definition of done:

- Fresh install → migrations apply → bootstrap CLI grants the first admin → admin signs in.
- A user without roles signs in and sees an empty-state page, nothing else; `/admin` returns 403 and is
  audited as `forbidden`.
- Removing one's own admin role or disabling oneself is refused and audited (`change_denied`).
- Disabling a user ends their sessions immediately.
- Permission screen shows the collector app identity and "no features required" with only core enabled;
  with a test-only feature registered (needs `User.Read.All`) and a fake token lacking it, the feature
  shows as missing with the consent instructions card; with the package disabled it shows "not required".
- Without any collector credentials configured, the web app still starts; the permissions screen says the
  collector is not configured.
- `ingest --scheduled` runs with zero steps, writes `cli.start`/`schedule.plan`/`cli.end`, exits 0.
- Log reader shows the invocation; leak query on `app_log` returns 0.
- Restricted web role cannot run DDL, cannot delete from `access_audit`, reads 0 rows from any RLS table
  when no scope is set.

## Step 1 — User lookup (`modules/user-lookup`)

Goal: an authorised person searches a user and sees their licenses, service plans and effective Teams
policies, live from Graph.

Prerequisites: Step 0. Permissions `User.Read.All`, `TeamsUserConfiguration.Read.All` (+ optional group
read for group names) granted to the collector app; package enabling it switched on.

Notes: live requests from the web process use the platform Graph client (same throttle and logging). Until
Step 3 exists the module is **unscoped** (all-or-nothing per role, like Admin): grant it only to roles that
may see every user. Every lookup is audited (who looked up whom).

DoD: module doc §12 acceptance criteria pass; locked state shown when `TeamsUserConfiguration.Read.All` is
missing; audit row per lookup; no lookup result is persisted except the audit/log metadata described in
the module doc.

## Step 2 — About / transparency (`modules/about-transparency`)

Goal: every viewer can see what the system collects, which Graph permissions it uses and why, which
granted permissions are unused (least privilege), and what it never reads.

Prerequisites: Step 0 (permission registry), Step 1 (at least one feature to describe).
DoD: content is generated from the permission registry and module manifests (no hand-maintained list);
unused granted permissions are listed; module doc §12 passes.

## Step 3 — Platform: directory sync, scope and masking

Goal: the platform can restrict people data by scope and detail level. Required before any module that
shows people in lists or aggregates.

Build: directory sync step (core feature `directory`), `directory_users`, `identity_names`, scope
resolution, RLS helper functions and policies pattern, masking engine, aggregate/small-group handling,
self-view, role editor scope/detail controls, preview with scope, leak-test harness with fixtures
(`platform/scope-masking.md`).

DoD: all leak tests in `platform/testing.md` §5 pass on a synthetic directory with hard names; role editor
warns about people without department; independent security review recorded.

After Step 3, User lookup may optionally gain scoped access (see its doc §14).

## Step 4 onward — suggested order

Pick the next module by need; the order below respects dependencies and builds shared ingest early.
The owner's next interest is video conferencing / rooms, so track A is recommended first.

| Step | Module | Prerequisites | Why here |
|---|---|---|---|
| 4 | `licensing` | Step 3 | First stored people module; reuses directory data; low request volume |
| 5 | `call-ingest` | Step 3 | Shared foundation for calls, meetings, rooms, transcripts; no UI except freshness and logs |
| 6 | `rooms-devices` (track A) | 4 (room licences), 5 (room meetings and quality) | Video-conferencing/rooms need |
| 7 | `call-search` | 5 | Record list, detail drawer, export; establishes paged API + masking for rows |
| 8 | `call-analytics` | 5, 7 (drill-down links) | 1:1 call KPIs |
| 9 | `meeting-analytics` | 5, 7 | Meeting KPIs, person-hours |
| 10 | `service-health` | 5 (correlation), Step 0 | Explains quality drops; small volume |
| 11 | `messaging` | Step 3 | Usage report pipeline (302 + CSV) |
| 12 | `copilot` | 4, 11 (shares report pipeline) | Copilot adoption |
| 13 | `transcripts` | 5 (meetings), Step 3 | Metadata only; needs tenant-side policy setup |

Every module step DoD: the module doc's §12 acceptance criteria, plus:

- Feature rows and package declared; locked state screenshot with permission marked missing.
- Ingest step registered (if any), isolated, gated, logged with run id; freshness line visible.
- RLS policy on every person-bearing table; views are `security_invoker`; leak tests extended with the new
  screens at full/masked/aggregate and small group.
- Load test result for the module's stated scale (doc §6 budget) recorded in the module doc.

## Later platform work (not scheduled)

- Change notifications (webhooks) instead of polling where Graph supports them.
- Entra group → role mapping; scope by Entra group or administrative unit.
- Per-module retention jobs with anonymisation.
- Rate limiting on search/tab GET endpoints.
- Installer/packaging and pre-upgrade automatic backup.
