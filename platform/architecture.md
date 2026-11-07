# Architecture

## 1. Decisions at a glance

| Decision | Choice | Why |
|---|---|---|
| Language/runtime | Python ≥ 3.11 | Mature Graph/HTTP/PostgreSQL libraries; reference built in Python |
| Web framework | FastAPI (ASGI) behind a reverse proxy | Simple routing, dependency injection for auth/scope, good test client |
| Rendering | Server-side templates (Jinja2, autoescape on) + small vanilla JS | No SPA build chain; server decides what is sent (access control) |
| Database | PostgreSQL ≥ 15, one schema `teams` | JSONB for raw data, RLS, `security_invoker` views |
| DB driver | psycopg 3, dict rows, autocommit | Server-side binding; autocommit rule (`database.md`) |
| Sign-in | Entra ID OIDC, authorization code + PKCE via MSAL | Standard, no passwords stored |
| Graph identity | App-only client credentials (certificate preferred, secret allowed) | Call records/reports require application permissions |
| i18n | gettext catalogs managed with Babel | Standard tooling, plural/context support |
| External assets | None at runtime (fonts embedded, no CDN) | Works offline/air-gapped; no third-party requests from viewers |
| Process model | Web service (long-running) + collector (oneshot, timer-driven) | Isolation: a slow ingest never blocks the UI and vice versa |

## 2. Components

```
                 ┌──────────────── Microsoft cloud ────────────────┐
                 │  login.microsoftonline.com      graph.microsoft.com
                 └───────▲───────────────────────────▲─────────────┘
          OIDC (openid profile)│                      │ app-only token, REST
                         │                            │
 Browser ──HTTPS──▶ Reverse proxy ──▶ Web service ────┤ (live modules, probes)
                                    │  auth · rbac · scope · masking · UI shell
                                    │  module routes / tab builders
                                    │  restricted DB role (RLS applies)
                                    ▼
                              PostgreSQL  schema "teams"
                                    ▲
                                    │ owner/collector role (no RLS)
 systemd timer ──▶ Collector (CLI "ingest --scheduled")
                     permission check → enabled+permitted steps → raw JSONB → derive → derived tables
                     all steps write app_log + ingest_runs
```

| Component | Responsibility | Talks to |
|---|---|---|
| Config | Load and validate environment variables | — |
| App log | Structured logging to `app_log`, scrubbing, retention | DB (own connection) |
| DB layer | Connections (owner / web), migrations, transaction helper | PostgreSQL |
| Graph client | Token, throttle, retries, paging, errors | Graph, token endpoint |
| Permission registry | Features, packages, token evaluation, probes, `permission_checks` | Graph client, DB |
| Module registry | Loads module manifests; exposes modules, features, steps, routes | all platform parts |
| Auth | OIDC sign-in, sessions, CSRF | Entra, DB |
| RBAC | Users, roles, role-module grants, audit | DB |
| Scope & masking | Resolve scope, set RLS context, mask/aggregate rendering | DB (owner for resolution, web for data) |
| UI shell | Layout, tokens, components, i18n injection | templates |
| Collector | Ingest runner, step isolation, cadence decisions | Graph client, DB |
| Tools | Log reader, i18n tool, render check, leak scan, bootstrap admin | DB |

## 3. Data flow

**Ingest (collector):** timer → `ingest --scheduled` → `cli.start` → permission check (token roles +
probes) → plan (which steps run now and why, `schedule.plan`) → each step: `start_run` → list from Graph →
decide what to fetch (one SQL) → fetch details in a worker pool → write raw + derived on one thread, one
transaction per record → `finish_run` (ok/partial/failed + counters) → `cli.end` with exit code.

**Page request (web):** request → session lookup → user's module grants (from DB on every request) →
module manifest gate (package enabled, feature permitted, viewer has grant) → resolve view (scope + detail)
with the owner connection → open web-role transaction, set scope context → module builds a view model from
SQL (RLS-filtered) → masking/aggregation applied to the view model → template render → audit `view_tab` →
response. The shell (`GET /`) renders the rail and the first permitted module; other modules load lazily
via `GET /api/tab/{module}`.

**Live request (e.g. user lookup):** same gate → Graph client call(s) from the web process (shared throttle,
logged) → view model → render. Results are not persisted unless the module doc says so.

## 4. Module manifest (the plug-in contract)

The platform has no compile-time knowledge of modules. Each module registers one manifest object:

| Field | Meaning |
|---|---|
| `key` | RBAC module key, English snake_case, stable forever |
| `package` | Package key it belongs to |
| `label`, `description`, `icon` | Marked for translation; icon from the shell icon set |
| `order` | Position in the rail |
| `features` | Feature rows (permissions) this module adds (`permissions.md`) |
| `kind` | `people`, `rooms`, or `unscoped` — decides available scope types (`scope-masking.md`) |
| `ingest_steps` | Zero or more steps: key, function, cadence/min age, depends-on features (`ingest-scheduler.md`) |
| `tab_builder` | Function (web conn, view, request) → view model; template name |
| `routes` | Extra API routes (each declared with its required module grant and audit action) |
| `audit_actions` | Action names it writes, with their detail schema |
| `derive_version` | Current derive version (if it stores derived data) |
| `reprocess` | Function re-deriving all its rows from raw (if applicable) |
| `migrations_prefix` | Its tables' name prefix (documentation aid) |

The platform runs with an **empty** registry: the rail then contains only Admin, Logs and permissions
(platform screens), and the collector has no steps.

## 5. Proposed repository layout (implementation repo)

```
app/
  config.py              environment loading and validation
  applog.py              structured logger + writer thread + scrubber
  db.py                  connections, transaction helper, migration runner
  graph/                 client.py (token, throttle, request, paging), errors.py
  i18n/                  gettext loader, locale helpers, label helpers
  security/              markup.py (safe JSON embedding), masking.py, csrf.py
  platform/
    registry.py          module manifests
    permissions.py       features, packages, evaluation, probes
    auth.py              OIDC + sessions
    rbac.py              users, roles, grants, audit
    scope.py             view resolution, RLS context
    ingest.py            runner, plan, step isolation
    web.py               app factory, middleware, shell, admin, logs screens
  modules/
    <key>/manifest.py, graph_calls.py, derive.py, ingest.py, queries.py, views.py,
          templates/, static/, tests/
  templates/             shell, components
  static/                css (generated from theme tokens), js, fonts
migrations/NNN_<area>_<description>.sql     one global sequence
locales/<lang>/LC_MESSAGES/messages.po
tools/                   logs CLI, i18n tool, render_check, leak_scan, bootstrap_admin
tests/                   platform tests, fixtures, fake Graph
```

Rules: modules import platform code, never the other way round; modules do not import each other's
internals — a dependency on another module is through its SQL objects (views/functions) documented in that
module's §5. Graph access only through `app/graph`; SQL only in `db.py`, platform data-access functions and
each module's `queries.py`.

## 6. Settings

Runtime-editable settings live in the `settings` key/value table (see `database.md`); deployment
settings live in environment variables (`deployment.md`). A setting has one home, never both, except the
timezone (environment value wins, DB value is the fallback; see `i18n.md`).
