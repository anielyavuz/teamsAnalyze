# Conventions

Binding rules for every platform component and every module. A change that violates one of these is not
done, even if it works. Each rule states the decision and why.

## 1. Logging is mandatory

- Every feature, module, CLI entry point and external call writes to the structured application log
  (`app_log`, see `platform/logging.md`). **A feature without logging is not finished.**
- Debugging starts from the log, never from "run this and paste the output". Why: the operator's server is
  not the developer's machine; the log is the only shared view.
- Event keys are `<component>.<what_happened>` — lower case ASCII, English, stable (they are queried, the
  message is not). Messages are one English line for humans. Everything you might query later goes into the
  structured context, not into the message.
- Start and end of every job at INFO with counters and `duration_ms`; every external request at least
  DEBUG; self-healing problems WARNING; failures ERROR with traceback; uncaught exceptions CRITICAL
  (automatic).
- The pure derive layer never logs; it returns a `diagnostics` list which the caller logs.
- After adding events: run the feature, confirm the events are written, list them in the module doc §10.

## 2. Permissions map per Graph feature

Every Graph capability is declared as a **feature** with its required application permissions
(least-privileged first; `A|B` = either is enough), its module/package, an "optional" flag and a one-line
"why" (see `platform/permissions.md`). Required for every new endpoint:

- [ ] A feature row exists; the feature belongs to exactly one package.
- [ ] Ingest steps and live requests are gated on the feature being enabled **and** permitted; a missing
      permission skips the step (`ingest.skipped`, WARNING) instead of producing 403 errors.
- [ ] If a role is not sufficient on its own (licence-dependent, tenant switch, beta API availability),
      a **live probe** is defined; expected refusals are requested in quiet mode (no ERROR log).
- [ ] The UI shows the locked card for the feature/section, tested by marking it `missing` in a test DB.
- [ ] **Permission transparency:** every module screen that calls Graph (live or via ingest) exposes the
      "Which permissions does this need?" popover (A); every **live** screen also exposes the per-section
      "What did this use?" popover (B) generated from the request trace — never hard-coded text. Spec:
      `platform/permissions.md` §8 and `platform/ui-shell.md` §6; reference implementation:
      [modules/user-lookup/DESIGN.md](modules/user-lookup/DESIGN.md).

## 3. Least privilege

- Request only permissions of **enabled** packages. A fresh install enables only the core package.
- Prefer the narrowest permission (`User.Read.All` over `Directory.Read.All`). List broader alternatives
  only as accepted fallbacks.
- No delegated Graph permissions for data collection; the sign-in app requests only `openid profile`.
- Never use protected/content APIs (chat messages, channel messages, transcript content, recording
  content, Copilot interaction history). This is a product promise, not an optimisation.
- The web process connects to PostgreSQL with a restricted role (no DDL, no superuser, subject to RLS).

## 4. Database

- **Raw is the source of truth.** Store Graph responses verbatim in JSONB. Everything derived must be
  recomputable from raw without calling Graph (`reprocess`).
- **Migrations are append-only.** New numbered file; never edit an applied migration; a fix is a new
  migration. Each migration runs in its own transaction.
- **Autocommit connections** for the collector and CLI. Explicit transactions only where atomicity is
  needed (`with transaction():`). Why: with autocommit off, the first SELECT opens an implicit transaction
  and later "transactions" silently become savepoints that are never committed.
- **Verify writes from a separate session** (another process or `psql`), never from the writing connection.
- **Calculations live in SQL** (views/functions) so the UI, exports and any BI tool get one definition.
  Python does not re-implement a metric. If a threshold must exist in both derive code and SQL, the
  module doc names both places and they change together.
- No per-row queries in loops; batch reads (`ids_needing_fetch`) and batch writes.

## 5. Pure derive layer

- Derive functions take raw JSON and return derived structures. **No network, no database, no logging,
  no clock reads** (pass "now" in). The same functions run during ingest and during reprocess.
- Derived data contains **no natural-language text and no local times**: only codes, parameters and UTC.
  Text and local time are produced at render time by label helpers (see `platform/i18n.md`).
- Changing derive logic means: bump the module's `DERIVE_VERSION`, add a row to the version history in the
  module doc, run `reprocess` for that module.

## 6. No content, privacy by default (KVKK / GDPR)

- Never read, store, log or display message text, transcript text, recording media, Copilot prompts.
- Person-level monitoring metrics (presence, after-hours activity, per-person performance) default to
  **aggregate** detail. Person-level detail is a conscious role grant.
- `aggregate` views enforce a minimum group size (k-anonymity, default 5).
- Self-view: a person may see their own data in modules they are allowed to open (operator can disable).
- Who looked at whose data is audited (append-only).
- Retention: each module states how long its person-level data is kept; operators can shorten it.
- The operator (customer) is the data controller. Documents and UI must not claim legal compliance; they
  describe what is processed and why.

## 7. Internationalisation

- Source language is **English**. Every user-visible string is marked for translation; no bare UI strings.
- Never build sentences from fragments; use named placeholders and plural forms.
- Numbers, percentages, dates and times are formatted only through the locale helpers.
- Stored data is language-independent (codes, not labels; UTC, not local time). Audit entries store
  structured data, not rendered text.
- Internal keys (module, package, role, setting, event, scope type) are English identifiers and never
  translated. See `platform/i18n.md`.

## 8. Security review gate

Any change touching **authentication, sessions, CSRF, RBAC, scope/RLS, masking, the restricted DB role,
embedding of data into HTML/JS, exports, or audit** must pass an independent review (a second agent or
person who did not write it) **before** it is deployed. The reviewer runs the leak tests in
`platform/testing.md` and records the outcome in the change description. Hiding something with CSS/JS is
never access control: the server must not compute or send what the viewer may not see.

## 9. Scale rules

Target: a tenant with thousands of meetings per 30 minutes and tens of thousands of users.

- Ask for every new loop/request: for 10,000 records, how many HTTP requests, how many SQL queries, how
  many MB? Write the answer in the module doc.
- Graph detail requests run in a bounded worker pool; DB writes happen on one thread.
- All Graph traffic goes through the shared throttle; a 429 pauses **all** workers.
- Pages show bounded lists; KPIs are computed in SQL over the full range (a list limit must never change a
  KPI).
- Large lists are served by paged APIs, not embedded in the page.
- No per-user caching of rendered HTML (scope and detail differ per viewer).

## 10. Testing method

- Never write fake data into a real database. Copy it (`createdb -T <db> <db>_test`), point the test run
  at the copy, drop it afterwards.
- Graph is faked at the HTTP layer with realistic shapes (pages of 60, `@odata.nextLink`, `$expand`
  next-link keys, 302 + CSV, 429 + Retry-After, 401, 403), so the real code path runs.
- Embed known patterns in fake data and assert that SQL finds exactly them.
- UI: headless browser screenshots in light, dark and phone width (390 px, inside an iframe), for every
  screen touched; data-viz colours checked by the validator, never by eye.
- Leak tests for every people-showing screen × every detail level. Details: `platform/testing.md`.

## 11. Code hygiene (for the implementation repository)

- One place per concern: all Graph access in the Graph client, all SQL in the data-access layer of the
  platform or the module, all labels in label helpers, all colours in the theme.
- Secrets only in environment variables / an env file with mode 600, never in code or logs.
- No real tenant data, names, domains, IPs or hostnames in code, tests, fixtures or docs; use
  `contoso.com`, `<tenant-id>`, `user1@contoso.com`.
- Every new source file carries the project licence SPDX header.

## 12. Documentation rule

Docs are part of the change. After each feature or fix update the module doc (data model, flow, UI,
events, version history) and, if the platform changed, the platform doc. Any **observed Graph behaviour
that contradicts Microsoft documentation** is written into the module doc §4 "Known quirks" **with the
date observed**. Open decisions go to §14, not into code comments.

---

## MODULE DOC TEMPLATE

Every module lives in `modules/<folder>/DESIGN.md` and uses **exactly** these headings, in this order.
Write "None" rather than deleting a heading. No SQL DDL or source code; tables of columns, prose and short
illustrative JSON shapes are fine.

```
# <Module name>  (module key: `<key>`, package: `<package>`)

## 1. Purpose
What question it answers, for whom (roles), and why it exists. 3–8 lines.

## 2. Scope / Out of scope
Bullet lists. Out of scope must include any content API deliberately not used.

## 3. Dependencies
Platform docs it relies on (by file) and other modules (by key) that must exist first, with the reason.

## 4. Microsoft Graph endpoints & permissions
Table: | Method | Endpoint | v1.0/beta | Permission (least-privileged first; alternatives) | Why |
Then: feature rows to add to the permission map (key, needs, optional, why), live probes if any,
and "Known quirks" — each with the date it was observed and how the code handles it.

## 5. Data model
Tables and columns in prose/tables (name, type family, meaning, source: raw/derived/ingest).
Raw JSONB columns, derived columns, indexes needed, RLS policy for each person-bearing table,
retention, DERIVE_VERSION history table. No SQL.

## 6. Ingest / request flow
Step-by-step (list → decide what to fetch → fetch → save). Cadence when scheduled, window sizes,
idempotency, what happens on 403/429/empty. For live (non-stored) modules: the request flow per screen.
Request/record budget for 10,000 records.

## 7. Derived logic & metric definitions
Every metric: exact definition, numerator/denominator, filters, minimum sample rules, where it is
computed (derive function or SQL object). Edge cases.

## 8. UI
Screens and components (using platform/ui-shell.md components), sort/filter/export, empty state,
error state, locked (missing permission) state, aggregate/small-group state.

## 9. Access control
RBAC module key, whether it is a people module and/or room module, supported scope types, behaviour at
full / masked / aggregate, self-view, which elements are person-level blocks, audit actions written
(name + structured detail).

## 10. Logging events
Table: | Event | Level | When | Context keys |. Plus module-specific never-log items.

## 11. i18n notes
Domain terms that need context (pgettext), label helpers to add, formatting specifics.

## 12. Testing & acceptance criteria
Fake-Graph fixtures needed, embedded patterns and expected SQL results, leak tests, screenshots,
performance target. Numbered acceptance criteria, each verifiable.

## 13. Implementation steps for an agent
Ordered checklist (migrations → Graph client calls → feature rows → derive → ingest → SQL →
UI → access control → logging → i18n → tests → docs).

## 14. Open questions / future
Undecided items with options and a recommendation.
```
