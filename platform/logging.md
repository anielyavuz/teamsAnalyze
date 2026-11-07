# Logging

All diagnosis starts from the application log. Every feature writes to it; a feature without log events
is not finished (`CONVENTIONS.md` §1).

## 1. Table `app_log`

| Column | Type | Meaning |
|---|---|---|
| `id` | bigserial PK | order of insertion |
| `ts` | timestamptz | event time (clock time at emit, not insert) |
| `level` | text | `DEBUG` / `INFO` / `WARNING` / `ERROR` / `CRITICAL` (checked) |
| `component` | text | `graph`, `db`, `ingest`, `auth`, `admin`, `web`, `cli`, `permissions`, `schedule`, or a module key |
| `event` | text | machine key `<component>.<what>` |
| `message` | text | one English line for humans |
| `context` | jsonb | structured fields (ids, counts, status, parameters); default `{}` |
| `duration_ms` | numeric | for timed events |
| `error` | jsonb | `{type, message, traceback}` for exceptions |
| `invocation` | uuid | one per OS process (CLI run or web worker) |
| `run_id` | bigint | `ingest_runs.id` when inside a step (no FK: logs outlive runs) |
| `request_id` | text | per web request (random), also returned as `X-Request-Id` |
| `user_id` | bigint | signed-in app user for web events (no FK) |
| `command` | text | CLI command line (arguments scrubbed) or `web` |
| `host`, `pid`, `thread` | | where it happened |

Indexes: `ts desc`; `(level, ts desc)` partial for WARNING+; `(invocation, id)`; `run_id` partial;
`(event, ts desc)`; `request_id` partial; GIN on `context` (jsonb_path_ops).

Views: `v_invocations` (one row per invocation: start, last, seconds, command, lines, warnings, errors,
run ids, finished = has `cli.end`), `v_log_problems` (WARNING+ newest first).

`app_log` is readable by the web role only through the Logs module/screen (role-gated); it contains no
person data beyond ids and is not RLS-scoped. **Therefore person names, UPNs and e-mail addresses must not
be written to the log** — use keys/ids. (Exception: auth events may contain the signing-in user's own UPN.)

## 2. Writer

- Logger API per component: `debug/info/warning/error/critical(event, message, exc=False, **ctx)` and a
  `timed(event, message, level="INFO", **ctx)` context manager that records `duration_ms`, lets the body add
  context, and logs at ERROR with traceback if the body raises (then re-raises).
- Emit is non-blocking: records go into an in-memory queue; a **background thread with its own DB
  connection** inserts in batches (≤ 200 rows or every 0.5 s). Why: logging must not slow parallel ingest
  (measured equal throughput with and without INFO logging) and must survive a failing main transaction.
- On process exit the queue is flushed (bounded wait, e.g. 5 s). If the table does not exist yet (fresh
  DB before migrations), the writer retries briefly, then falls back to stderr.
- `LOG_LEVEL` (default INFO) filters what is written; `LOG_CONSOLE=<level>` mirrors that level and above to
  stderr (systemd journal).
- Uncaught exceptions in main and worker threads → `cli.crash` / `cli.thread_crash` (CRITICAL, traceback).
- Process start: `cli.start` (command, version, Python); end: `cli.end` (duration, exit code, crashed).
  Tools that only **read** the log do not emit start/end (they must not pollute it).
- Retention at collector start: delete DEBUG older than `LOG_DEBUG_DAYS` (7) and others older than
  `LOG_KEEP_DAYS` (90) via a SQL function; logged as `db.log_pruned` with the count.

## 3. Levels

| Level | When |
|---|---|
| DEBUG | each external request (short URL, status, duration, attempt); per-record detail; data oddities. Off by default |
| INFO | start/end of a job with counters and duration; plan decisions; progress every N records; token acquire; migration applied |
| WARNING | self-recovering problems: 429/401 retry, one record/day failed, unexpectedly empty data, step skipped for missing permission |
| ERROR | a part of the job failed: permanent request error, missing permission at request time, step aborted (with traceback) |
| CRITICAL | uncaught exception (automatic) |

## 4. Event naming

- `<component>.<what_happened>`, lower case ASCII snake case after the dot: `graph.throttled`,
  `ingest.record_error`, `auth.login_ok`, `rooms.devices_unavailable`.
- Keys are stable API: renaming one is a breaking change noted in the doc.
- Standard suffixes: `.start`, `.plan`, `.done` (INFO, counters), `.failed` (ERROR, traceback),
  `.skipped` (WARNING), `.no_permission` (ERROR), `.empty` (WARNING), `.progress` (INFO).
- Every module doc §10 lists its events; the platform events are listed in section 7.

## 5. Never log

- Access tokens, refresh/id tokens, client secrets, certificate material, `Authorization` headers,
  session tokens, CSRF tokens, OIDC `code`/`state`/`nonce` values.
- Report download URLs / `Location` headers (they grant unauthenticated data access for minutes).
- Parameters marked secret (e.g. meeting join URLs).
- Any content (message text, transcript text) — it is never fetched anyway.
- Person names/UPNs/e-mails (use keys), except the signing-in user's UPN in `auth.*` events.
- Search query text typed by users: log only its length (`q_len`) and which filters were used.

## 6. Scrubbing (defence in depth)

Applied to message, context (recursively) and error text **before** truncation (so truncation cannot split
a token and defeat the pattern):

- Context keys (case-insensitive) `authorization`, `token`, `access_token`, `refresh_token`, `id_token`,
  `client_secret`, `secret`, `password`, `location`, `cookie`, `set-cookie`, `code`, `state` → `***`.
- Text patterns: `Bearer <token>`; JWT-like `eyJ…` strings; query parameters `token=`, `access_token=`,
  `sig=`, `signature=`, `sas=`, `code=`; any `reports*.office.com` host with or without scheme (regional
  hosts included); report download paths `/data/v<N>/download…` (they also appear host-less in HTTP library
  error messages).
- When adding a new sensitive field anywhere, extend the scrubber and the leak query in the same change.

Leak query (must return 0; run in tests and after deploys):
count rows where the concatenation of `context`, `message` and `error` matches (case-insensitive) any of
`reports[a-z0-9-]*\.office\.com`, `/data/v[0-9.]+/download`, `eyJ[A-Za-z0-9_-]{5,}`,
`Bearer [A-Za-z0-9]`, `(token|sig|access_token|signature|code)=[^*]`.

## 7. Platform events

| Event | Level | Meaning |
|---|---|---|
| `cli.start` / `cli.end` | INFO | process start/end (`exit_code`, `duration_ms`, `crashed`) |
| `cli.crash` / `cli.thread_crash` | CRITICAL | uncaught exception |
| `cli.step_failed` / `cli.steps_failed` | ERROR | one ingest step failed / summary of failed steps |
| `config.missing` | ERROR | required environment variable missing |
| `db.migrate` / `db.migrate_failed` / `db.migration_modified` / `db.schema_behind` | INFO/ERROR | migrations |
| `db.connect_failed` | ERROR | cannot connect |
| `db.reprocess` | INFO | reprocess counts |
| `db.log_pruned` | INFO | retention |
| `run.start` / `run.finish` | INFO/WARNING/ERROR | ingest run lifecycle (status decides level) |
| `schedule.plan` | INFO | which steps run and why others are skipped |
| `ingest.skipped` / `ingest.module_off` | WARNING/INFO | permission missing / package disabled |
| `graph.token` / `graph.token_failed` / `graph.reauth` | INFO/ERROR/WARNING | token lifecycle |
| `graph.request` | DEBUG | each request |
| `graph.throttled` | WARNING | 429/503/504 global pause |
| `graph.expected` | DEBUG | quiet error |
| `graph.error` / `graph.retries_exhausted` | ERROR | permanent failure |
| `graph.network_retry` / `graph.network_error` | WARNING/ERROR | connection errors |
| `graph.expand_pages` | INFO | multi-page expanded relation |
| `permissions.checked` / `permissions.missing` / `permissions.limited` | INFO/WARNING/INFO | permission check |
| `auth.*` | INFO/WARNING | login ok/failed/denied, logout, forbidden, csrf_failed, change_denied (mirrors audit) |
| `admin.*` | INFO | admin changes (mirrors audit, without names) |
| `web.request_failed` | ERROR | unhandled exception in a request (request id, route) |
| `web.tab_build` | INFO | module tab built (module, duration, rows, mode) |

## 8. Log reader CLI

`logs` tool (reads only; does not emit start/end):

| Command | Shows |
|---|---|
| `logs` | summary: last invocations (duration, warnings, errors, runs) + last problems |
| `logs last` | all lines of the latest invocation |
| `logs problems [N]` | last N WARNING+ |
| `logs run <id>` | lines of an ingest run |
| `logs inv <uuid-prefix>` | lines of an invocation |
| `logs req <request-id>` | lines of a web request |
| `logs event <prefix>` | events whose key starts with prefix |
| `logs grep <text>` | text in message/context/error |

Options: `--level <LEVEL>`, `--limit N` (default 200), `--detail` (full context + traceback). Colours only
on a TTY.

## 9. Logs screen (web)

Part of the platform "Logs and permissions" screen (role-gated). Server returns grouped events: identical
(level, component, event, normalised message with numbers/uuids replaced) are grouped with occurrence list
`[id, ts, invocation/run]`; context/traceback included only for samples (all if ≤ 6, else first, last and
newest 3). Left: invocations list (collapses ≥ 3 consecutive identical outcomes); right: timeline. Filters:
text, level (all / warning+ / error), time (24 h / 3 d / 30 d), component chips. Text is set via
`textContent`; JSON embedded via the safe helper. DEBUG is not shown (use the CLI).
