# Database

## 1. Engine and layout

- PostgreSQL ≥ 15 (needs `security_invoker` views). One database, one application schema **`teams`**.
  Other data sources may later live in their own schemas in the same database without collisions.
- Database created with UTF-8 encoding. Recommended `LC_CTYPE = C.UTF-8` so SQL `lower()`/`upper()`
  behave like Python for ASCII; collation may be the operator's locale for sorting. **Never compare scope
  values (departments, locations) through `lower()`**: Turkish dotted/dotless I differs between Python and
  PostgreSQL ("İstanbul" stopped matching in the reference). Scope values match exactly.
- All timestamps are `timestamptz` and stored in UTC. Local time appears only in presentation (see
  `i18n.md`) or in SQL views that explicitly call the timezone function below.

## 2. Raw and derived data

| Layer | Rule |
|---|---|
| Raw | Graph responses stored verbatim in JSONB columns (`raw`, `<relation>_raw`). Multi-page relations are stored merged into one array. Never edited after write except by a newer fetch of the same object |
| Extracted columns | Frequently filtered scalars (ids, timestamps, type codes) copied from raw into typed columns for indexing |
| Derived | Computed by the module's pure derive functions: typed columns + one `derived` JSONB for presentation-ready structures. Codes and UTC only |
| Derived version | Every row with derived data has `derived_version int`. The module's `DERIVE_VERSION` constant is the current value |
| SQL layer | Metrics are SQL views/functions over extracted+derived columns. Python never re-computes a metric |

Indexes: B-tree on time and filter columns; `GIN (raw jsonb_path_ops)` only where ad-hoc containment
queries on raw are expected (document why in the module doc). Partial indexes for selective UI filters.

### Re-fetch rule (per record)

A stored record's detail is fetched again when **any** of: not in DB; list `version`/`lastModified` newer
than stored; previous fetch recorded errors (`fetch_errors` not empty); stored `derived_version` lower than
code. Determined for the whole page/window in **one** query, and unchanged ids are "touched" (last seen run)
in one batch update. Modules without versions document their own rule.

### Reprocess

`reprocess [--module <key>]` re-derives every row from raw **without calling Graph** (including rows older
than Graph's retention). It runs each module's `reprocess` function in batches (e.g. 500 rows), one
transaction per batch, logs `db.reprocess` with counts and duration. It never writes report files.
Procedure when derive logic changes: bump `DERIVE_VERSION`, add a row to the module doc's version history,
deploy, run reprocess, verify from a separate session.

## 3. Connections

| Connection | Role | Used by | RLS |
|---|---|---|---|
| Owner | schema owner (`teams_owner`, not a superuser in production) | migrations, collector, CLI, web internal work (scope resolution, masking name index) | bypasses (owner) |
| Web | `teams_web` restricted role | every web request that reads module data or writes users/sessions/audit/settings | enforced |
| Log writer | owner role, separate connection | app log writer thread | n/a |

- **Autocommit on** for all connections. Use explicit `transaction()` blocks for atomic multi-statement
  writes (one record = one transaction: record row + child rows + identities). With autocommit off, the
  first SELECT opens an implicit transaction and later blocks become savepoints that are never committed —
  the reference lost every reprocess this way until it was found by checking from another session.
- Web module reads run inside an explicit **read-only transaction** per module build so the scope context
  is transaction-local (`scope-masking.md` §4).
- Connection pooling is allowed for the web role. On return to the pool a connection is reset
  (`RESET ALL`, discard temp objects). Scope context must never survive a request.
- Every connection sets `app.tz` (session setting) from `APP_TIMEZONE`.
- Owner connection inside the web process is used only by platform internals, never handed to module
  tab builders.

## 4. Migrations

- Files `migrations/NNN_<area>_<description>.sql`, one global three/four-digit sequence. `<area>` is
  `platform` or a module key. Example: `014_platform_scope_rls.sql`, `031_rooms_devices.sql`.
- Applied list in `teams.schema_migrations(version, filename, applied_at, checksum)`.
- Runner applies pending files in order, **each in its own transaction**; a failure rolls back that file
  only and stops. Logged as `db.migrate` (INFO) per file and `db.migrate_failed` (ERROR).
- **Append-only**: never edit an applied file. The runner stores a checksum and refuses to start if an
  applied file changed (`db.migration_modified`, ERROR). A fix is a new migration.
- Migrations run from the deploy procedure and at collector start; the web process does not run them
  (it checks the latest version and refuses to start if the DB is behind: `db.schema_behind`).
- Each migration that creates a table holding person data also, in the same file: enables RLS, creates the
  web policy (or none for owner-only tables), and sets grants (section 6).
- Each migration that creates a view creates it `WITH (security_invoker = true)`.
- Migrations must be idempotent where cheap (`IF NOT EXISTS` for roles/extensions) but never rely on it.

## 5. Platform tables (created in Step 0 / Step 3)

| Table | Purpose | Spec |
|---|---|---|
| `schema_migrations` | applied migrations | this doc |
| `settings` | key/value runtime settings (`key text PK`, `value text`, `updated_at`, `updated_by`) | below |
| `ingest_runs` | one row per ingest step execution | `ingest-scheduler.md` |
| `app_log` (+ views) | structured application log | `logging.md` |
| `permission_checks` | per-feature permission status | `permissions.md` |
| `app_users`, `roles`, `role_modules`, `user_roles`, `app_sessions`, `access_audit` | auth/RBAC/audit | `auth-rbac.md` |
| `directory_users`, `identity_names` | directory sync, name index for masking (Step 3) | `scope-masking.md` |

Platform settings keys (all English, values as text): `enabled_packages` (comma list; **missing = core
only**), `min_group_size` (default 5), `self_view` (`on`/`off`, default `on`), `timezone` (fallback IANA
name), `default_language`, `app_name`/`app_id`/`tenant_id`/`app_roles` (written by the permission check),
`home_tenant` (tenant id used to classify external identities). Settings changes are audited
(`settings_update` with before/after).

Helper SQL functions (platform): `app_tz()` → `app.tz` session setting, else `settings.timezone`, else
`UTC`; `iso_ms(text)` ISO-8601 duration → milliseconds; `arr(jsonb)` → the value if it is an array else
`[]` (guards `jsonb_array_elements` against scalars).

## 6. Roles and grants

| Role | Attributes | Grants |
|---|---|---|
| `teams_owner` | LOGIN, owns schema and objects, not superuser | everything in `teams` |
| `teams_web` | LOGIN, NOSUPERUSER, NOCREATEDB, NOCREATEROLE, **NOBYPASSRLS** | USAGE on schema; SELECT on tables by default; EXECUTE on functions; INSERT/UPDATE/DELETE only on `app_users`, `roles`, `role_modules`, `user_roles`, `app_sessions`; INSERT on `access_audit`; INSERT/UPDATE on `settings`; USAGE on sequences; TEMP on the database (scope key table) |

- `ALTER DEFAULT PRIVILEGES FOR ROLE teams_owner IN SCHEMA teams GRANT SELECT ON TABLES TO teams_web` and
  EXECUTE on functions, so new module tables are readable — **therefore every person-bearing table must
  have RLS enabled in the migration that creates it**, otherwise it is readable unfiltered.
- Owner-only tables (secrets of identity like historical names, raw log context) : enable RLS, create
  **no** policy, and `REVOKE ALL ... FROM teams_web`. Why both: RLS-without-policy denies rows; the revoke
  also denies existence-revealing errors and functions.
- `access_audit` is append-only: rules/triggers turn UPDATE and DELETE into no-ops/errors for every role
  except an explicit maintenance role used for retention (none by default).
- Authentication to PostgreSQL: local socket with peer mapping, or password over TLS; never trust auth
  in production.

## 7. Row-level security rules

- Policies are written **for `teams_web` only** (`TO teams_web`). The owner is not affected (no `FORCE`).
- Policies call platform helpers: `in_scope(key)`, `scope_all()`, and module helpers such as
  `call_in_scope(call_id)`. Helpers that must read RLS-protected tables to decide (avoiding recursive
  policy evaluation) are `SECURITY DEFINER` with a fixed `search_path = teams, pg_temp` and are as small
  as possible.
- **Missing scope context means deny** (zero rows), never "everything".
- Views over RLS tables must be `security_invoker = true`; otherwise they run with the owner's rights and
  bypass RLS. A CI check lists all views in `teams` and fails if any lacks it.
- Functions returning person data (`SECURITY INVOKER` by default) inherit RLS from the tables they read.
  A `SECURITY DEFINER` function must never return person rows.
- Organisation-wide aggregates that cannot be filtered per person (e.g. tenant daily trend reports) get a
  policy visible **only when `scope_all()` is true**.

## 8. Verification queries (run after each migration step)

- Every table in `teams` holding a person key column has `relrowsecurity = true`.
- Every view in `teams` has `security_invoker=true` in `reloptions`.
- As `teams_web` with no scope context: `SELECT count(*)` on each RLS table returns 0.
- As `teams_web`: DDL fails; `DELETE FROM access_audit` affects 0 rows.
