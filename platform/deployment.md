# Deployment and operations

Generic Linux deployment. No real hosts, addresses or tenant values appear here; use placeholders.

## 1. Topology

```
Internet / intranet ──HTTPS──▶ reverse proxy (TLS) ──HTTP 127.0.0.1:<WEB_PORT>──▶ teams-web.service
                                                                                   │
                                   teams-ingest.timer ─▶ teams-ingest.service ─────┤
                                                                                   ▼
                                                                    PostgreSQL (local socket)
```

- The web service listens on **loopback only**; only the reverse proxy is exposed. TLS terminates at the
  proxy; HSTS on. The proxy forwards `X-Forwarded-Proto/For`; the app trusts them only from
  `TRUSTED_PROXIES`.
- PostgreSQL listens on the local socket (or loopback). Data directory on a reliable filesystem (ext4/xfs;
  never on NTFS/FUSE mounts — permission bits and fsync semantics break PostgreSQL).
- Recommended layout: `/srv/teams-analyze/app` (code, read-only for the service user),
  `/srv/teams-analyze/venv`, `/srv/teams-analyze/env` (env file, mode 600), backups on another disk/host.
- One unprivileged OS user (e.g. `teamsanalyze`) runs both services.

## 2. Environment variables

| Variable | Required | Meaning |
|---|---|---|
| `GRAPH_TENANT_ID` | for collection | Entra tenant id (`<tenant-id>`) |
| `GRAPH_CLIENT_ID` | for collection | Collector app (client) id |
| `GRAPH_CLIENT_CERT_PATH` + `GRAPH_CLIENT_CERT_THUMBPRINT` | one of cert/secret | Certificate credential (preferred) |
| `GRAPH_CLIENT_SECRET` | one of cert/secret | Client secret credential |
| `AUTH_CLIENT_ID` | yes | Sign-in app (client) id |
| `AUTH_CLIENT_SECRET` or `AUTH_CLIENT_CERT_PATH` | yes | Sign-in app credential |
| `DATABASE_URL` | yes | Owner/collector connection string |
| `WEB_DATABASE_URL` | yes | Restricted `teams_web` connection string |
| `WEB_BASE_URL` | yes | Public base URL, e.g. `https://teams-analyze.contoso.com` (redirect URI = base + `/auth/callback`) |
| `WEB_HOST` / `WEB_PORT` | no | Bind address/port (default `127.0.0.1` / `8765`) |
| `TRUSTED_PROXIES` | no | Comma list of proxy addresses whose forwarded headers are trusted |
| `APP_TIMEZONE` | no | IANA zone for presentation and local bucketing (default `UTC`) |
| `APP_LANGUAGE` | no | Default UI language (`en`) |
| `SESSION_ABSOLUTE_HOURS` / `SESSION_IDLE_MINUTES` | no | Session limits (8 / 30) |
| `LOG_LEVEL` / `LOG_CONSOLE` | no | Log level (INFO) / mirror level to stderr (off) |
| `LOG_DEBUG_DAYS` / `LOG_KEEP_DAYS` | no | Log retention (7 / 90) |
| `GRAPH_WORKERS` / `GRAPH_MAX_RPS` | no | Detail worker pool (8) / request rate (50) |
| `APP_DEBUG_LOCALES` | no | Enable pseudo-locale `xx` (off) |

Environment variables already set in the process take precedence over the env file. Missing required
values → `config.missing` naming the variables, exit non-zero. The web service starts without collector
credentials (permissions screen says "collector not configured").

## 3. Entra app registrations (operator steps)

1. **Collector app** (`<collector-app>`): single tenant; credential (certificate preferred); API permissions →
   Microsoft Graph → Application permissions → only those listed on the permissions screen for enabled
   packages → Grant admin consent.
2. **Sign-in app** (`<signin-app>`): single tenant; platform Web; redirect URI
   `<WEB_BASE_URL>/auth/callback`; no API permissions beyond `openid`/`profile` (delegated, no admin consent
   needed); credential. Optionally "Assignment required" on its enterprise app.
3. Rotate credentials before expiry; the permissions screen shows credential expiry when known.

## 4. systemd units (specification)

**teams-web.service**: `Type=simple`; `User`/`Group` service user; `WorkingDirectory` app dir;
`EnvironmentFile` env file; `ExecStart` = ASGI server bound to `WEB_HOST:WEB_PORT` with the app factory;
`Restart=on-failure`; hardening `NoNewPrivileges=true`, `PrivateTmp=true`, `ProtectSystem=strict`,
`ProtectHome=true`, `ReadWritePaths` limited to what is needed (nothing in the app dir);
`After/Requires` PostgreSQL; `RequiresMountsFor` the data mount when on a separate disk.

**teams-ingest.service**: `Type=oneshot`; same user, env and hardening; `ExecStart` = `ingest --scheduled`;
`TimeoutStartSec=20min`; `Nice=10`, `IOSchedulingClass=best-effort`, `IOSchedulingPriority=7`;
`After=network-online.target` + PostgreSQL.

**teams-ingest.timer**: `OnBootSec=2min`, `OnCalendar=*:00/15`, `Persistent=true`, `AccuracySec=30s`,
`RandomizedDelaySec=30s`, `WantedBy=timers.target`.

Status: `systemctl status`, `journalctl -u teams-ingest` for process-level output; application detail is
in `app_log` (`logs` CLI).

## 5. Reverse proxy (requirements, any proxy)

TLS 1.2+, HTTP→HTTPS redirect, HSTS, request body limit (e.g. 1 MB), gzip/brotli for HTML/JSON, timeouts
≥ 60 s for tab builds, `X-Forwarded-*` headers, no caching of authenticated responses, access log without
query strings (search text) if the proxy supports it.

## 6. Database setup

1. Create roles `teams_owner` (LOGIN) and `teams_web` (LOGIN, NOBYPASSRLS) — or let the first migration
   create `teams_web` when run as a role with CREATEROLE.
2. Create the database owned by `teams_owner`, UTF-8, `LC_CTYPE=C.UTF-8`; enable data checksums at cluster
   init where possible.
3. Authentication: local peer mapping or SCRAM passwords; never `trust`.
4. Run migrations (`migrate` CLI). Grant TEMP on the database to `teams_web`.
5. Bootstrap the first admin: `admin grant-admin <upn>`.

## 7. Deploy procedure (upgrade)

The repository provides a deploy script; contract:

1. Show what will change (dry-run diff of files).
2. **Backup first** (section 8) — abort if the backup fails.
3. Sync code (excluding env files, local data, reports, screenshots); never edit files on the server by hand
   (the next sync overwrites them).
4. Install dependencies if the lock file changed.
5. Compile translation catalogs.
6. Run migrations as the owner (stop the timer during migrations: `systemctl stop teams-ingest.timer`).
7. Restart the web service; start the timer.
8. Health check: `GET /healthz` (process up, DB reachable, schema version current; no auth, no data) returns
   200 within 30 s; else roll back code and report.
9. Log `deploy.done` / `deploy.failed` (via the CLI) with version.

Never run `reprocess` and a manual large backfill concurrently with the timer (the advisory lock prevents
overlap, but stop the timer for long manual jobs anyway).

## 8. Backups and restore

- Daily logical dump (`pg_dump -Fc`) of the database to another disk or host, retention e.g. 14 daily + 8
  weekly; dumps encrypted at rest when off-host (they contain personal data).
- Before every upgrade: an extra dump.
- Restore test quarterly: restore into a scratch database, run the verification queries in
  `database.md` §8, count rows of key tables.
- The env file and certificates are backed up separately and securely (not with the dumps).
- What a backup does **not** cover: Graph data older than Graph's retention that was never collected.

## 9. Operations quick reference

| Task | Command |
|---|---|
| What happened? | `logs`, `logs problems`, `logs last --detail` |
| Run collection now | `ingest` (or `ingest --only <step>`) |
| Re-derive after a derive change | `reprocess --module <key>` |
| Check permissions | admin screen button or next `ingest` |
| Emergency admin | `admin grant-admin <upn>` |
| Disable a user | `admin disable <upn>` (ends sessions) |
| Health | `GET /healthz` |
