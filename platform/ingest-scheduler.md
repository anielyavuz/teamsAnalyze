# Ingest runner and scheduler

## 1. Modes

| Command | Behaviour |
|---|---|
| `ingest` | Manual: run all enabled+permitted steps now with their default windows |
| `ingest --only <step>[,<step>]` | Manual subset |
| `ingest --hours N` | Override the look-back window for windowed steps (bounded by each source's maximum) |
| `ingest --scheduled` | Timer mode: apply cadence rules (section 4), small windows |
| `reprocess [--module <key>]` | Re-derive from raw, no Graph (`database.md` §2) |

Manual runs and the timer use the same code path. No automation is installed until the operator enables
the timer (documented in `deployment.md`).

## 2. Invocation lifecycle

1. `cli.start`; load config; connect (owner role); apply pending migrations.
2. Prune logs; delete expired sessions.
3. Permission check (`permissions.md` §2). If the token cannot be obtained, log and exit 1 (no step can run).
4. Build the plan: for each registered step → `run` / `skip` with a reason code (`module_off`,
   `missing_permission`, `not_due`, `depends_failed`). Log one `schedule.plan` INFO with the whole plan.
5. Run steps sequentially in dependency order (e.g. `directory` before modules that use it).
6. Summary: if any step failed → `cli.steps_failed` (ERROR, `failed=[...]`), exit code 1; else 0.
   `cli.end` with `exit_code`. No report files are written in scheduled mode.

Concurrency guard: one collector at a time — a PostgreSQL advisory lock taken at start; a second
invocation logs `schedule.locked` (WARNING) and exits 0. Why: a long manual backfill and the timer must not
interleave writes or double the Graph load.

## 3. Steps and runs

A step is declared in a module manifest: `key`, `features` (required/optional), `depends_on` (step keys),
`cadence` (`every_run` or minimum age since last **successful** run, e.g. 6 h), `window` (default and
scheduled look-back), `run(ctx)`.

Each execution creates an `ingest_runs` row:

| Column | Meaning |
|---|---|
| `id` | run id, set into the logger so every log line carries `run_id` |
| `source` | step key, e.g. `graph.callRecords`, `graph.reports.messaging`, `graph.directory` |
| `started_at`, `finished_at` | |
| `window_start`, `window_end` | time window requested (windowed steps) |
| `status` | `running` / `ok` / `partial` / `failed` |
| `records_seen`, `records_inserted`, `records_updated`, `records_unchanged` | counters (meaning per step documented in the module doc) |
| `errors` | jsonb list of per-item errors (id, stage, status, code) — bounded (first 200) |
| `meta` | jsonb: derive version, request statistics, window reason, invocation id |

Status rules: `ok` all items fine; `partial` finished but some items failed (they will be retried by the
re-fetch rule); `failed` the step aborted (data written so far stays — one transaction per record);
`running` with no `finished_at` after the process ended = killed (harmless; next run continues).
A `no_permission` outcome is recorded as `failed` with `meta.reason = "no_permission"` but is **not**
counted as a failed step for the exit code (it is a configuration state, already visible in the UI).

## 4. Step isolation

- Each step runs inside its own try/except for `Exception` (not `BaseException`: Ctrl-C and system exit still
  stop the process). A failing step logs `<module>.failed` with traceback, closes its run as `failed`,
  logs `cli.step_failed`, and the **next step still runs**. Why: in the reference, one unreachable report
  host for a day stopped every later step and left gaps in unrelated data.
- Steps that depend on a failed step are skipped with `depends_failed`.
- Network outages: the Graph client retries briefly; a step keeps failing each cadence until the outage
  ends — the Logs screen groups the repeats.

## 5. Scheduled cadence (defaults; module docs may override with reasons)

| Kind of step | Cadence | Window |
|---|---|---|
| Directory | every run | full list |
| Event records (e.g. call records) | every run | last 6 h (records appear 15–30 min after the event and may be enriched later). Plus once a day a 48 h window to catch late version bumps outside the short window |
| Device/room state snapshots | every run | current state |
| Usage reports (daily CSVs) | min age 6 h | report-specific (e.g. last 4 days + never-fetched days) |
| Service health | min age 1 h | full list |

Timer: every 15 minutes (`OnCalendar=*:00/15`), first run 2 minutes after boot, `Persistent=true`, a
randomised delay of up to 30 s; oneshot service so a run in progress suppresses the next trigger; start
timeout 20 minutes. Units are specified in `deployment.md`.

Budget rule: each module doc states requests per scheduled run for its target scale; the sum must stay
well below Graph limits and finish within the 15-minute interval.

## 6. Freshness line

Every module screen shows in the top-bar status line: "Data as of <local time of last successful run of
the module's main step> · next update about every <cadence>". Rules:

- Source: latest `ingest_runs` with `status in (ok, partial)` for the module's main step (`partial` adds
  "some items pending").
- Warning colour if older than 2× cadence; danger colour + "collection failing since <time>" if the last
  N (≥ 3) runs failed.
- Live modules show "Live from Microsoft Graph · <time of this request>".
- In small-group aggregate views the freshness line is still shown but contains no counts.

## 7. Progress and observability

- `<module>.start` (window, plan counts), `<module>.plan` (to fetch / unchanged), `<module>.progress` every
  100 items or 30 s, `<module>.done` (counters, items/s, request stats, duration).
- `run.start` / `run.finish` from the runner itself.
- `logs run <id>` reconstructs a run's story.
