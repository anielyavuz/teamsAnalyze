# Cisco device logs (credential-based local collection)

Module key: `cisco_logs` (viewing) · credential/rotation screens: `device_credentials` (admin) ·
Package: `rooms` (optional add-on) · Kind: `rooms` · Status: designed (Nov 2026), technical note — not built.

> Why this module exists: Microsoft Graph gives Teams Rooms **health and inventory** but **no raw device
> logs** (only `runDiagnostics`, which ships logs to Microsoft/TAC, not to us; and the device API is being
> retired, see `../rooms-devices/DESIGN.md`). Cisco RoomOS devices expose their full logs — RoomOS system logs
> **and the Microsoft Teams Android app logs (logcat)** — on the device's own HTTPS interface. This module
> reads them **incrementally every 30 seconds** with a per-device username/password, stores them, and shows
> each device's latest state.

## 1 Purpose

- Near-real-time view (≤ 1 min) of what every Cisco video device is doing and what went wrong: Teams sign-in
  lost, meeting join failed, camera/peripheral missing, network drop, reboot, software update.
- Keep a searchable, bounded history of device log lines and derived events in our database (devices keep only
  a few days and rotate quickly).
- Manage the device credentials this requires **safely**: encrypted at rest, write-only, audited, and
  rotatable one by one or in bulk (audit/compliance driven password changes).
- Never harm the device: no account lockouts, no session exhaustion, no heavy log bundles on a schedule.

## 2 Scope / Out of scope

In scope:
- Cisco RoomOS collaboration devices (Desk, Board, Room series) on the customer LAN, in Microsoft Teams Rooms
  (MTR on Android) or Webex mode. Verified on RoomOS 26.0.1 (Nov 2026) on a Desk Pro in MTR mode.
- Incremental reading of a configurable set of log files (default list in §6.4).
- Device latest-state view and derived events.
- Per-device credential entry, verify, replace, disable; single and bulk admin password rotation.
- Guardrails: stop on authentication failure, session hygiene, circuit breaker, firmware-change detection.

Out of scope (now):
- Other vendors (Logitech, Yealink, Poly, Neat…): same module shape, different adapters — see §14.
- Scheduled full log bundles (`/web/logs/bundle`): **on demand only**, never on a timer (CPU + ~15 MB each).
- Changing device configuration other than user/passphrase management.
- Webex Control Hub cloud APIs (alternative for cloud-registered devices, §14).
- Reading meeting content. Logs may contain names/meeting titles: see §7.4 privacy.

## 3 Dependencies

- `../../platform/secrets.md` — **mandatory** for credential storage (encryption, key handling, write-only).
- `../../platform/database.md`, `../../platform/logging.md`, `../../platform/auth-rbac.md`,
  `../../platform/scope-masking.md` (location scope for rooms), `../../platform/ui-shell.md`,
  `../../platform/ingest-scheduler.md` (collector loop differs: own 30 s cadence, §6.1), `../../platform/testing.md`.
- `../rooms-devices/DESIGN.md` — room/device inventory; devices are matched by serial number / MAC so log
  state appears next to Graph health in the room view. This module can also run without Graph (manual
  device list).
- `../../platform/permissions.md` transparency component — adapted for non-Graph sources (§8.5).

## 4 Device interfaces & permissions (instead of Microsoft Graph)

No Graph permission is involved. The "permission" is a **device account** with a role. All calls are HTTPS to
the device's IP/hostname (TLS; device certificates are often self-signed → certificate pinning per device, §6.3).

### 4.1 Endpoints (verified on RoomOS ce26.0.1.2, 2026-11-08)

| Purpose | Method + path | Auth | Official? | Cost on device |
|---|---|---|---|---|
| Web session sign-in | `POST /web/signin/open` (form fields `username`, `password`) → session cookie | form | web UI internal | creates a session |
| List log files | `GET /web/logs/files/current` (and `/historical`) → JSON `[{name, size, date}]` | session cookie | web UI internal | tiny |
| Read one log file | `GET /web/logs/file/current/<path>` — supports **HTTP Range** (`206 Partial Content`) | session cookie | web UI internal | file read only (1.27 MB in 0.14 s) |
| Full log bundle (on demand) | `GET /web/logs/bundle?type=anonymous` · `?type=full` (incl. call history) · no type = without call history | session cookie | web UI internal | compresses everything (~15 MB, ~4 s) |
| Status / config read | `GET /getxml?location=/Status/...` | HTTP Basic | **xAPI (official)** | tiny |
| Commands | `POST /putxml` with `<Command>…</Command>` | HTTP Basic | **xAPI (official)** | tiny |
| List / end sessions | `xCommand Security Session List` / `Terminate SessionId: <id>` | HTTP Basic | xAPI | tiny |
| Users / passwords | `xCommand UserManagement User Add|Modify|Delete|List|Get|Unblock`, `User Passphrase Set|Change` | HTTP Basic, Admin | xAPI | tiny |
| Live events (optional) | WebSocket `wss://<device>/ws` (JSON-RPC xAPI, feedback subscriptions) | Basic/session | xAPI | push, no polling |
| Push logs (alternative) | `xConfiguration Logging External Mode/Server/Protocol: SyslogTLS` | — | xConfiguration | device pushes |

### 4.2 Real-device findings (dated, keep updated)

| Date | Finding | Consequence |
|---|---|---|
| 2026-11-08 | `/web/*` endpoints **reject HTTP Basic** (302 to `/web/signin`); only the form sign-in works | Collector keeps one cookie session per device |
| 2026-11-08 | `Security Session` limits: `MaxSessionsPerUser = 20`, `MaxTotalSessions = 30`; `InactivityTimeout = 0` (sessions **never** expire) | Signing in every poll would exhaust sessions in minutes and block real admins. **Reuse one session**, terminate it on shutdown |
| 2026-11-08 | `/web/signout` returns 404 | End sessions with `xCommand Security Session Terminate` |
| 2026-11-08 | HTTP Basic xAPI calls (`/getxml`, `/putxml`) do **not** appear in `Session List` | Use Basic xAPI for status/commands; web session only for log files |
| 2026-11-08 | `MaxFailedLogins` is 0 on the test device, but it is configurable (`FailedLoginsLockoutTime` 60 s) | Customers may have lockout on → **one failed attempt and stop** (§6.2) |
| 2026-11-08 | The device writes every web command into its own log (`User <u>/web/vega about to execute command … from <ip>`) | Expected noise; filter our own collector lines in derivation |
| 2026-11-08 | A session with `Origin internal/xows`, user `__extension_xapi_sundroid__` exists permanently (the Teams Android extension) | Never terminate sessions we did not create |
| 2026-11-08 | Teams Android logs are under `extensions/msteams/logcat*.log`; rotation siblings `.first`, `.previous`, `.truncated` | Rotation handling (§6.4) |
| 2026-11-08 | Roles available to `User Add`: Admin, Audit, Integrator, RoomControl, User | — |
| 2026-11-08 | A suffix Range larger than the file (`bytes=-N`, N > size) returns **416** (not the whole file as RFC 7233 allows) | Always request explicit ranges computed from the listed size |
| 2026-11-08 | **External syslog tested** (`Logging External Mode On, Protocol Syslog`, UDP to a test receiver): works immediately, RFC 5424 over UDP with journald structured data, ~300–400 messages / 2 min (~1 KB/s) from RoomOS processes only (`main`, `metaproxy`, `gui`, `wpa_supplicant`, `kernel`, `systemd`, `vega`, `nginx`, `weston`, `video`). `xCommand Logging AddEvent Text: … Type: Info|Warning|Error` markers arrive. **No Teams Android (logcat) lines at all** | Syslog does **not** replace file collection for Teams logs (see §14) |
| 2026-11-08 | ⚠️ **Side effect:** enabling external syslog restarts the device's `syslog-ng`; at that moment the Teams Android logcat files (`extensions/msteams/logcat*.log`) **stopped being written** and did **not** resume after the setting was reverted (observed ≥ 10 min; RoomOS logs kept flowing; Teams app signed in and running) | **Never toggle `Logging External` on a production device as part of this module.** If a customer enables syslog, schedule it in a maintenance window and verify Teams log files resume (device restart may be required) |
| 2026-11-08 | **Tested with temporary users:** `Audit` and `Integrator` can sign in to the web UI but get **403** on `/web/logs/files`, `/web/logs/file` (Range) and `/web/logs/bundle`; both **can** read xAPI status (`/getxml` 200) | **Reading logs requires the Admin role.** See §6.5 recommendation |

## 5 Data model (described; no DDL)

All tables in schema `teams`, RLS per `scope-masking.md` (location scope via the device's room). Secret
columns only in `device_credentials` (owner/collector role only).

**`cisco_devices`** — one row per device: `id`, `display_name`, `host` (IP or FQDN), `port` (443),
`serial`, `mac`, `product`, `software_version`, `room_key` (link to rooms module, nullable), `location`,
`collector_site` (which on-prem collector reaches it), `tls_fingerprint` (pinned), `enabled`, `added_by`,
`added_at`, `updated_at`.

**`device_credentials`** — one active row per device and purpose (`collector`; optionally `admin` for
rotation): `device_id`, `purpose`, `username`, `secret_ciphertext`, `secret_nonce`, `secret_key_version`,
`secret_set_at`, `secret_set_by`, **`status`** (`unverified` / `ok` / `auth_failed` / `locked_out` /
`disabled`), `status_since`, `last_verified_at`, `failure_code`, `failure_detail` (no secret), `pending_*`
columns used only during rotation (§6.6). Exposed to the web role only through view `v_device_credential_status`
(no secret columns).

**`credential_audit`** — append-only: `at`, `actor` (user id or `collector`), `device_id`, `purpose`, `action`
(`create`, `replace`, `test`, `test_failed`, `auth_failed_stop`, `disable`, `enable`, `delete`, `rotate_*`,
`secret_reveal`), `detail` (structured, never the secret), `ip`.

**`device_sessions`** — our own sessions: `device_id`, `remote_session_id` (from `Session List` when known),
`opened_at`, `last_used_at`, `closed_at`, `close_reason`. Cookie value itself is kept **in collector memory
only**, never stored.

**`device_log_cursors`** — per device × file: `file`, `offset`, `size_seen`, `head_hash` (hash of first 256 bytes
to detect rotation), `last_read_at`, `last_bytes`, `errors`.

**`device_log_lines`** — partitioned by day: `device_id`, `file`, `ts` (device timestamp, UTC), `level`
(V/D/I/W/E/F or n/a), `process`, `pid`, `tag`, `message` (masked, §7.4), `line_hash` (dedup key with device+file),
`ingested_at`. Default retention **14 days** (setting `device_log_retention_days`).

**`device_events`** — derived, small: `device_id`, `ts`, `code` (language-independent, e.g.
`teams_signed_out`, `meeting_join_failed`, `camera_lost`, `peripheral_disconnected`, `network_down`,
`reboot`, `software_update`, `collector_auth_failed`), `severity`, `params` (jsonb), `source_line_hash`,
`rule_version`. Retention **180 days**.

**`device_state`** — one row per device, the "latest state" the UI shows: `reachable`, `last_contact_at`,
`last_ok_poll_at`, `teams_signed_in`, `in_call`, `peripherals` (jsonb), `software_version`,
`open_issue_codes` (array), `collector_status` (`ok` / `paused_auth` / `offline` / `unsupported_firmware` /
`disabled`), `updated_at`.

**`rotation_jobs`** / **`rotation_items`** — §6.6.

Sizes: a busy device writes roughly 1–3 MB/day across the default file set; with masking and dedup, ~10–30k
lines/day. 500 devices × 14 days ≈ 100–200 M lines → partitioning by day and dropping whole partitions is
required, not row deletes.

## 6 Collection flow

### 6.1 Placement and cadence

- Devices are on the customer LAN; a cloud server cannot reach them. Run a **collector** process on-prem
  (same network/VLAN as the devices) that writes to the central database (or posts to the central API). One
  collector per site; `cisco_devices.collector_site` assigns devices.
- Own loop, not the 15-minute Graph scheduler: every device is polled every **30 s** with **jitter**
  (spread evenly over the 30 s window, ± 3 s) so 500 devices ≈ 17 devices/s, never a burst.
- Worker pool sized for the site (e.g. 16 workers); per-device timeouts 10 s connect / 20 s read; at most
  **one in-flight request per device**.

### 6.2 Guardrails (owner requirement: "on an error, never try again by itself")

1. **Authentication failure = stop that device.** Any of: sign-in not accepted, HTTP 401/403 on xAPI,
   redirect to `/web/signin` immediately after a fresh sign-in → set credential `status = auth_failed`,
   `collector_status = paused_auth`, write `credential_audit(auth_failed_stop)`, log `cisco.auth_failed`
   (ERROR, once). **No retry, no backoff retry, no "try again next cycle".** The device is skipped (DEBUG
   `cisco.auth_blocked`) until an admin **saves a new password** or presses **"Test once"** (exactly one
   attempt, audited). Never try alternative/default passwords. Never fall back to another account.
2. **Lockout awareness.** If the device reports a locked/blocked account (`locked_out`), behave as (1) and show
   the device's `FailedLoginsLockoutTime`. `xCommand UserManagement User Unblock` is an explicit admin action,
   never automatic.
3. **Circuit breaker.** If ≥ 3 devices **or** ≥ 5 % of a site's devices hit `auth_failed` within 10 minutes
   (typical after a wrong bulk rotation or a central policy change), pause **all new sign-ins** for that site,
   raise CRITICAL `cisco.circuit_open`, and require an admin to close it. Already-signed-in sessions keep
   reading.
4. **Session hygiene.** Exactly **one** web session per device per collector. Reuse the cookie for every
   poll. Re-sign-in only when the session is really gone (302 to sign-in) — and that re-sign-in counts as an
   authentication attempt under rule 1. Before the first sign-in after a collector start, call
   `Security Session List` (Basic xAPI) and terminate **only** sessions whose `UserName` is the collector
   account **and** `RemoteAddress` is this collector (leftovers of a crash). Never touch other sessions
   (admins, `internal/xows`). On shutdown terminate our session. Alarm if our account ever holds > 2 sessions.
5. **Network errors are not auth errors.** Timeout / refused / TLS reset → mark `offline`, exponential backoff
   30 s → 1 → 2 → 5 → 10 min (cap), credentials untouched.
6. **Firmware drift.** Unexpected response shape (HTML instead of JSON, 404 on a known path, missing fields) →
   `collector_status = unsupported_firmware` for that device, stop polling it, log WARNING
   `cisco.unsupported_firmware` with the software version, alert once. These `/web/*` paths are not an official
   API and may change on RoomOS upgrades.
7. **Bounded reads.** Per poll at most 512 KB per file and 2 MB per device; if a file grew more, read the newest
   part and record a `gap` event (so a log storm cannot overload device, network or DB).
8. **TLS pinning.** First successful contact stores the device certificate fingerprint (admin confirms in UI);
   a changed fingerprint stops the device (`tls_changed`) until confirmed — credentials are never sent to an
   unverified endpoint.
9. **Kill switch.** Setting `cisco_logs_enabled = off` stops all collectors' polling within one cycle.

### 6.3 One poll (steady state)

1. `GET /web/logs/files/current` with the session cookie → list (1 request).
2. For each watched file (§6.4): compare `size` with the cursor `offset`.
   - `size > offset` → `GET /web/logs/file/current/<file>` with `Range: bytes=<offset>-` (cap per §6.2.7).
   - `size < offset` or `head_hash` changed → **rotated**: read the remainder of the rotated sibling
     (`<file>.previous`) from the old offset, then the new file from 0.
   - unchanged → nothing.
3. Split complete lines (keep a partial last line in memory for the next poll), parse (§7.1), mask (§7.4),
   dedup by `line_hash`, insert in one batch, advance cursors **in the same transaction**.
4. Update `device_state` and derived events (§7.2).
Typical steady state: 1–3 requests and a few KB per device per poll.

Status (Teams sign-in, call state, peripherals) is cheaper and more reliable via Basic xAPI
`/getxml?location=/Status/...` every 60 s, or via the WebSocket feedback stream (§14) — use logs for *why*,
status for *what now*.

### 6.4 Default watched files

| File | Why |
|---|---|
| `extensions/msteams/logcat-ipphone.log` | The Teams app itself (`com.microsoft.skype.teams.ipphone`): sign-in, call/meeting state, app errors |
| `extensions/msteams/logcat-warning.log` | Warnings and errors from all Android components — best signal/noise |
| `extensions/msteams/logcat-admin-agent.log` | Teams admin agent: management, updates, configuration from Teams admin center |
| `eventlog/main.log` | RoomOS main process: peripherals, USB, audio routing |
| `eventlog/application.log` | RoomOS application events |
| Optional | `logcat.log` (everything, noisy), `logcat-camera-hal.log`, `logcat-codec.log`, `eventlog/all.log`, `wpa_supplicant.log` (Wi-Fi), `dhclient.log` |

Configurable per site; the list endpoint shows what exists on each firmware.

### 6.5 Credential lifecycle

- **Add**: admin enters host, username, password (write-only form) → encrypted per `secrets.md` → status
  `unverified` → admin presses **Test once** (sign-in + `Session List` + terminate the test session) → `ok` or
  `auth_failed`. Polling starts only from `ok`.
- **Recommended account (tested 2026-11-08):** log files are only readable with the **Admin** role, so the
  collector cannot run with a low-privilege role. Recommendation:
  1. Create a **dedicated collector user with the Admin role** on every device (`xCommand UserManagement User
     Add Username: <collector> Role: Admin Passphrase: <unique> PassphraseChangeRequired: False`) — **never
     reuse the device's main `admin` account**. Benefits: its password can be rotated/disabled without touching
     the people's admin account; the device's own logs show every collector action under a distinct name;
     the collector's sessions are easy to identify and clean up (§6.2.4).
  2. Unique password per device (generated, ≥ 24 chars), stored per `platform/secrets.md`; rotate on a schedule
     (e.g. every 90 days) and immediately after any staff change with access.
  3. Optional split for least privilege: an **Audit** account for status reads over Basic xAPI (`/getxml`,
     verified to work) and the Admin collector account only for log-file reads. Doubles the credentials to
     manage; worth it only if the customer requires it.
  4. Because the collector account is Admin, treat the collector host as a privileged system (hardened,
     patched, network access to device VLAN only, KEK kept off the database server).
  5. Device-pushed syslog is **not** an alternative for Teams logs (tested, §4.2/§14): it carries RoomOS logs
     only.
  The people's `admin` credential is stored only if the operator wants the system to rotate that account too
  (§6.6).
- **Replace**: new value → `unverified` → Test once → `ok`. Replacing clears `auth_failed`.
- **Disable / delete**: polling stops; delete overwrites the ciphertext row (crypto-shred) and keeps the audit.
- **Bulk import**: CSV of host/username/password uploaded over HTTPS, encrypted immediately, file discarded
  (never stored), then batched "Test once" with the circuit breaker active.

### 6.6 Password rotation (single and bulk)

Commands (Admin role, Basic xAPI):
- Set another user's password: `xCommand UserManagement User Passphrase Set Username: <u> NewPassphrase: <new>
  [YourPassphrase: <admin current>] [SecurityAction: InvalidateAllUserSessions|None]`
- Change own password: `xCommand UserManagement User Passphrase Change OldPassphrase: <old> NewPassphrase: <new>`

Per device (one `rotation_item`):
1. Generate a strong unique password (≥ 24 chars, device policy permitting) — or take the admin-supplied one —
   and store it **encrypted as pending** (`pending_ciphertext…`, key version) **before** touching the device.
2. Pause polling for the device; terminate our web session.
3. Execute `Passphrase Set` (or `Change`) using the current credential. `SecurityAction` default `None`
   (do not kick other admins) — operator can choose `InvalidateAllUserSessions` for compliance rotations.
4. Verify: one sign-in with the new password, then terminate that session.
5. Success → promote pending to active, `rotated_at`, audit `rotate_ok`, resume polling.
   Command failed → keep old as active, discard pending, audit `rotate_failed`, resume with old.
   **Command succeeded but verification failed → `rotate_unknown`: stop the device, keep BOTH old and pending
   encrypted, CRITICAL alert, manual resolution.** Never loop.

Bulk job (`rotation_jobs`: scope = list/site/all, requested_by, optional approved_by, mode generated/same,
status):
- **Dry run** first (reachability + current credential status; no changes).
- **Canary**: the first 1–3 devices alone; continue only if all `rotate_ok`.
- Then batches (e.g. 10 at a time), **halt automatically** on the first `rotate_unknown` or when failures
  exceed 2 % — remaining items stay `not_started`.
- Optional four-eyes: a second admin approves the job before execution (setting).
- Optional maintenance window; skip devices currently `in_call` and retry them at the end of the job (they are
  not authentication failures).
- Per-device unique passwords recommended; "same password for all" allowed but warned.
- Report: per device result, never the passwords. Export of new passwords only via break-glass (`secrets.md` §7).

## 7 Derived logic

### 7.1 Parsing

- Android logcat lines (as written by RoomOS): `<ISO-8601 ts+offset> <process>[<pid>]: <L> <Tag>: <message>`
  where `L` ∈ V/D/I/W/E/F. Example: `… zygote64[1682]: I CallStateBroadcastReceiver: onReceive: …APP_INCALL_STATE`.
- RoomOS event logs: `<ISO ts> <process>[<pid>]: <message>` (no level → `n/a`).
- Timestamps converted to UTC; lines without a parseable timestamp inherit the previous line's ts (stack traces).
- Multi-line messages are joined while the line does not start with a timestamp.

### 7.2 Event rules

Versioned rule table (`rule_version`): pattern (file + tag + regex) → `code`, `severity`, extracted params.
Start set: Teams signed out / sign-in failed, meeting join failed, call ended unexpectedly, camera/microphone not
found, peripheral (touch panel) disconnected, network link down/up, DHCP failure, Wi-Fi roam failure, reboot
detected (log restarts / uptime reset), software update started/finished, admin-agent configuration applied.
Rules are reviewed against real logs before enabling; unknown error bursts (≥ N `E` lines/min from one tag)
produce a generic `error_burst` event. Our own collector access lines are excluded.

### 7.3 Latest state

`device_state` = last successful poll + status xAPI + latest events per code (open until a matching "resolved"
event, e.g. `camera_lost` → `camera_found`). `open_issue_codes` drives the fleet list badges.

### 7.4 Privacy

Teams logs can contain display names, e-mail addresses, meeting titles and phone numbers. Default: mask e-mail
addresses, phone numbers and quoted display names before storage (`may_contain_names` classification, see
`scope-masking.md`), keep raw only if the operator enables `device_logs_store_unmasked` (audited). Log viewers
are scoped by room location. Full bundles are downloaded as `type=anonymous` unless an Admin explicitly asks for
`full` (audited, reason required).

## 8 UI

1. **Fleet list** (`cisco_logs`): device, room, status chip (`ok` / `offline` / `password rejected` /
   `unsupported firmware` / `disabled`), Teams signed in, in call, open issues, last contact ("12 s ago"),
   last error event. Filters by site/location/status. Auto-refresh every 30 s.
2. **Device detail**: state header; event timeline (24 h / 7 d); log viewer (file, level, tag, text search,
   time range; newest first; live "follow" toggle polling our DB, not the device); cursor health per file
   (last read, bytes/min, gaps).
3. **Credentials** (`device_credentials`, Admin only): table of devices × purposes with status, set-on/by,
   last verified; actions Add / Replace / Test once / Disable / Delete; bulk import. **Password rejected**
   rows are red with the reason and the only actions are *Replace* or *Test once*.
4. **Rotation**: wizard — select devices → mode (generated unique / same) → SecurityAction → dry run → canary →
   run; live progress; final report; approval step when four-eyes is on.
5. **Circuit breaker banner** on all screens of the site while open, with "Close (resume sign-ins)".
6. **Transparency popover** ("What does this use?", platform component): instead of Graph permissions it lists
   the device endpoints used (§4.1), the account and role in use per device (username only), poll cadence,
   last poll duration/bytes, and the guardrail state. Same rendering conventions as `permissions.md` §8.
Empty states: no devices → "Add a device"; no credentials → link to Credentials; collector not seen for > 2
minutes → banner "Collector <site> offline since …".

## 9 Access control

| Action | Who |
|---|---|
| View fleet, device state, events | `cisco_logs` module, scoped by room location |
| View log lines | `cisco_logs` + detail level `full` (masked/aggregate viewers see events only) |
| Download full bundle on demand | Admin; `type=full` additionally requires a reason |
| Add/replace/test/disable/delete credentials | Admin (`device_credentials`) |
| Rotation (single/bulk) | Admin; bulk optionally needs a second approver |
| Close circuit breaker, Unblock account | Admin |
| Break-glass reveal | Admin + `secret_reveal_enabled` (default off) |

Audit actions (`access_audit` + `credential_audit`): `device_add`, `device_update`, `credential_create`,
`credential_replace`, `credential_test`, `credential_disable`, `credential_delete`, `rotation_create`,
`rotation_approve`, `rotation_run`, `rotation_item_result`, `circuit_close`, `bundle_download`,
`log_view` (device + time range), `secret_reveal`.

## 10 Logging events

`cisco.poll.done` (DEBUG per poll; INFO summary per minute per collector: devices, requests, bytes, lines,
errors), `cisco.auth_failed` (ERROR, once per stop), `cisco.auth_blocked` (DEBUG), `cisco.offline` /
`cisco.online` (WARNING/INFO on transitions only), `cisco.session.opened` / `.terminated` / `.leftover_cleaned`,
`cisco.session.excess` (WARNING), `cisco.rotated` (log file rotation), `cisco.gap`, `cisco.unsupported_firmware`,
`cisco.tls_changed`, `cisco.circuit_open` (CRITICAL) / `.circuit_closed`, `cisco.rotation.*`
(`started`, `canary_ok`, `halted`, `item_ok`, `item_failed`, `item_unknown`, `done`), `secrets.*`.
Never log: passwords, pending passwords, cookies, `Authorization`, request bodies of sign-in or passphrase
commands, unmasked log lines.

## 11 i18n notes

Event codes and statuses are language-independent; labels in the catalog. Device log text is shown as-is
(not translated). Rotation report and status chips are translated.

## 12 Testing & acceptance criteria

Fake device server (test double) reproducing §4.1: form sign-in with cookie, 302 on missing session, file list,
Range reads with 206, rotation (`.previous`), session list/terminate, passphrase commands, configurable
`MaxFailedLogins` lockout, slow/timeout responses, changed response shapes.

Must pass:
- **Lockout safety**: with a wrong password, the collector makes **exactly one** sign-in attempt per device and
  then none for ≥ 1 hour of simulated polling; credential `auth_failed`; one ERROR log; UI shows "password
  rejected". With lockout configured at 3 failures, the account is never locked.
- **Session safety**: 1,000 polls → the collector account never holds more than 1 session; after collector
  crash + restart only our leftovers are cleaned; `internal/xows` and other users' sessions untouched.
- Circuit breaker opens at the threshold and blocks new sign-ins site-wide; closing requires Admin.
- Network outage → backoff, no auth attempts counted, credential stays `ok`, resumes automatically.
- Rotation: canary halt; `rotate_unknown` stops device and keeps both secrets; no password ever appears in logs,
  audit, HTML or API responses (grep test with known test passwords).
- Secrets: ciphertext copied to another device row fails (AAD); collector refuses to start without KEK;
  web role cannot select secret columns.
- Incremental read: no duplicate or lost lines across 100 rotations; partial last line handled.
- Load: 500 simulated devices at 30 s → steady ≤ 3 requests/device/poll, CPU and DB write rate recorded.
- Real-device smoke test (lab device only): N polls, `Session List` shows one collector session; wrong
  password once → one failed login in the device's `auth.log`, no lockout.

**Lab result (2026-11-08, Desk Pro, RoomOS 26.0.1, MTR mode)** — prototype of §6.2/§6.3 against the real device:

| Check | Result |
|---|---|
| 6 polls × 30 s, files `logcat-ipphone.log` + `logcat-warning.log`, one sign-in reused | ✅ |
| Collector sessions on the device during the run | **max 1** (baseline: only `internal/xows`) |
| Requests per poll | 1–3 (list + Range reads) |
| Bytes per poll / duration | 7–33 KB / 0.35–0.7 s |
| New lines per poll (device idle, Teams app running) | 0–150 |
| Wrong password: sign-in attempts | **exactly 1**, then 5 polls skipped (`auth_failed`) |
| Device `auth.log` | one `Web login failed: <user> from <collector ip>` (+ the matching PAM line); no lockout |
| After shutdown (`Session Terminate` of own sessions) | sessions back to baseline |


## 13 Implementation steps for an agent

1. Implement `platform/secrets.md` (scheme, KEK loading, AAD, startup KAT, scrubber entries, leak test).
2. Migrations: tables of §5 (partitioned `device_log_lines`), view `v_device_credential_status`, RLS/REVOKE.
3. Device adapter (`cisco_roomos`): sign-in, session reuse, list/read with Range, Basic xAPI status/commands,
   session list/terminate, typed errors (`AuthError`, `NetworkError`, `UnsupportedFirmware`, `TlsChanged`).
4. Guardrail state machine (§6.2) with unit tests first (lockout and session tests of §12).
5. Collector loop: jittered scheduler, worker pool, cursors, rotation handling, bounded reads, batch insert.
6. Parser + masking + dedup; rule engine with versioned rules; `device_state` updater.
7. Admin screens: devices, credentials (write-only forms, Test once), circuit breaker.
8. Fleet list + device detail + log viewer; transparency popover.
9. Rotation: single, then bulk with dry run/canary/halt/approval; report.
10. Retention jobs: drop `device_log_lines` partitions older than the setting; events retention.
11. Docs: update this file's §4.2 with every new firmware finding; About page data-source table.

## 14 Open questions / future

- ~~Minimum role for `/web/logs`~~ — **answered 2026-11-08: Admin only** (`Audit`, `Integrator` → 403). `User`
  and `RoomControl` not tested (lower than Audit/Integrator for this purpose). Re-test on new RoomOS versions.
- **WebSocket event stream** (`wss://<device>/ws`, xAPI feedback): not yet tested; would replace status polling
  and give instant events.
- ~~Syslog as the credential-free mode~~ — **tested 2026-11-08: syslog carries RoomOS logs only, no Teams
  Android logs**, and enabling it disrupted local Teams log writing (§4.2). Conclusion: file collection (this
  design) stays the only way to get Teams Android logs. Syslog may still be used *in addition* for RoomOS logs
  by customers who already run a SIEM — configured by the customer, never toggled by this module. Open: does a
  device restart restore the Teams logcat files after a syslog change? (lab re-test).
- Webex Control Hub API for cloud-registered devices (logs/diagnostics in the cloud).
- Other vendors via the same adapter interface (Logitech Sync, Yealink, Poly Lens, Neat Pulse) — each has its
  own management API and credential model.
- Default retention values (14 d lines / 180 d events) to be confirmed by the owner.
