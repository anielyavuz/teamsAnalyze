# Call Ingest — Design

Module key: `call_ingest` (backbone, no UI tab of its own) · Feature key: `calls` · Package: `calls`
Status of reference implementation: proven in production on a small tenant and load-tested with a fake Graph
(10,000+ synthetic records). Derive logic version in the reference: 9.

---

## 1 Purpose

Pull every Microsoft Teams call record (1:1 calls, meetings, group calls) from Microsoft Graph, store the raw
Graph responses unchanged as the single source of truth, and compute a small, versioned set of derived columns
and rows that every call/meeting module reads. The derive layer is pure (no I/O) so that any change in business
logic can be applied to the whole history by re-deriving from stored raw JSON, without calling Graph again —
including records older than Graph's 30-day retention.

Everything downstream (call analytics, meeting analytics, call search, rooms, about) depends on the contract in
this document. Downstream modules never call Graph for call data and never parse raw JSON themselves except
through the derived tables defined here.

## 2 Scope / Out of scope

In scope:
- Listing call records in a time window, fetching detail (participants) and sessions/segments per record.
- Parallel fetch workers with a shared process-wide throttle, shared 429 brake, token refresh, network retry.
- Idempotency: skip records whose Graph `version` and our `derive_version` are unchanged.
- Raw JSONB storage of three payloads per record; derived columns; participants; identities; identity-name history;
  per-stream media rows (audio/video/share quality facts).
- The derive layer contract: inputs, outputs, codes, parameters, UTC, diagnostics, versioning, reprocess.
- Run bookkeeping (`ingest_runs`) and the ingest log events.

Out of scope (owned elsewhere):
- Scheduling, CLI entry points, locking against concurrent runs → `../../platform/ingest-scheduler.md`.
- The generic Graph HTTP client (token cache, throttle, retry primitives) → `../../platform/graph-client.md`.
  This document specifies the call-record-specific usage and the limits it must respect.
- Service health incidents → `../service-health/DESIGN.md`.
- Directory users / license SKUs (room accounts, department/manager scope) → directory sync in the core platform
  (`../../platform/ingest-scheduler.md`, `../licensing/DESIGN.md`).
- `callEvents` (missed vs declined vs caller hung up) — not used; see §14.
- PSTN / Direct Routing call logs (`getPstnCalls`, `getDirectRoutingCalls`) — not used; see §14.
- Change notifications (push subscriptions) — future; see §14.
- Any message, transcript or recording content. This module reads metadata only.

## 3 Dependencies

- `../../platform/architecture.md` — layering rule: graph client → ingest → db → derive (pure) → read models.
- `../../platform/graph-client.md` — `get`, `get_all`, shared throttle, token provider, `GraphHTTPError`
  (status + error code), network retry, request statistics counters.
- `../../platform/database.md` — migrations (append-only), owner vs web connection, autocommit rule, RLS
  conventions, `settings` key/value table, `app_tz()` SQL function.
- `../../platform/logging.md` — structured log API, event-key conventions, timed blocks, run correlation (`run_id`).
- `../../platform/permissions.md` — feature map entry `calls`, token-role check, "skip when missing".
- `../../platform/scope-masking.md` — RLS policies on person-bearing tables created here.
- `../../platform/ingest-scheduler.md` — step runner, `--scheduled`, `--reprocess`, run windows.
- `../../platform/testing.md` — fake Graph layer, test DB cloning, load test harness.
- `../../platform/i18n.md` — derived data is language-neutral; display text comes from label functions.
- Consumers: `../call-analytics/DESIGN.md`, `../meeting-analytics/DESIGN.md`, `../call-search/DESIGN.md`,
  `../rooms-devices/DESIGN.md`, `../about-transparency/DESIGN.md`, `../service-health/DESIGN.md` (incident overlap).

## 4 Microsoft Graph endpoints & permissions

| Method | Endpoint | Version | Permission (least privileged first) | Alternatives | Why |
|---|---|---|---|---|---|
| GET | `/communications/callRecords?$filter=startDateTime ge {since} and startDateTime lt {until}` | v1.0 | `CallRecords.Read.All` (Application) | none | Lightweight list of records in the window; includes `id` and `version` |
| GET | `/communications/callRecords/{id}?$expand=participants_v2` | v1.0 | `CallRecords.Read.All` | none | Record detail + all participants (paged) |
| GET | `/communications/callRecords/{id}?$expand=sessions($expand=segments)` | v1.0 | `CallRecords.Read.All` | none | Sessions, segments, media streams (quality, device, network) |
| GET | nextLink URLs returned by the above | v1.0 | same | — | Pagination (see quirks) |

Only Application permission is supported for call records; delegated permission is not accepted by Graph.
Admin consent is required. No other permission is needed by this module.

### Limits (documented)

| Limit | Value |
|---|---|
| callRecords requests, per app × tenant | 1,500 requests / 20 s (≈ 75 req/s) |
| callRecords requests, per tenant (all apps) | 10,000 requests / 20 s |
| Requests per single record | 40 / 20 s |
| List calls | 40 / 20 s |
| PSTN call log API, per app × tenant | 200 / 60 s (not used) |
| Retention | 30 days |

Cost per record: 2 requests + extra pages (participants page = 130 items, sessions page = 60 items).
List page = 60 records.

### Real-API findings (contradict or extend the docs)

| Date found | Finding | Consequence |
|---|---|---|
| 2026-09-25 | A window of exactly 720 hours returns `400 "The date provided in the filter query must be within the last 30 days and not in the future."` | Maximum window = **719 hours**. Clamp silently with a warning. |
| 2026-09-25 | Only **one** `$expand` per request: `$expand=participants_v2,sessions($expand=segments)` → `400 ... at least 2 items, but the maximum allowed is 1.` | Two GETs per record; cannot be optimised. |
| 2026-09-25 | Expanded relation paging: the first continuation link is under **`participants_v2@odata.nextLink`** / **`sessions@odata.nextLink`**, not `@odata.nextLink`. Subsequent pages use plain `@odata.nextLink`. | A standard paging loop silently loses participants in meetings with 130+ joins. Implement a dedicated "follow expanded" helper. |
| 2026-09-25 | The list response does **not** include `participants_v2` or `sessions`, and `type` is often `"unknown"` in the list. It **does** include `version`. | Classification must use the detail response. Use list `version` for idempotency. |
| 2026-09-25 | `/callRecords/{id}/participants_v2` as a sub-route is not documented. | Do not use it; use `$expand`. |
| 2026-09-25 | Fractional seconds have variable length: 7 digits, 5 digits, or none. | Use a tolerant ISO parser that normalises to microseconds; never a strict 3/6-digit parser. |
| 2026-09-25 | Records appear ~15–30 minutes after the call ends. | "No data" right after a call is normal; scheduler cadence of 15 min is enough. |
| 2026-09-26 | `organizer` / `participants` (deprecated, documented to stop returning data 2026-06-30) are **still populated**. | Use `organizer_v2` / `participants_v2` anyway: `organizer_v2` carries `userPrincipalName`; the old fields will go silent without error. |
| 2026-09-26 | A missed 1:1 call: `type=peerToPeer`, `startDateTime == endDateTime` (to the tick), `joinWebUrl=null`, `main-audio` media present but with no metrics (0 packets), **`failureInfo=null`**, `organizer_v2` = caller. | "Missed" is inferred (see §7). Graph cannot distinguish missed / declined / caller-cancelled. |
| 2026-09-26 | Caller/callee come from `sessions[0].caller/callee.associatedIdentity`, which is a **flat identity**, not an identitySet; it carries no kind. Its id may not match the `participants_v2` identity. | Match by id, then by display name, else build an identity (see §7.4). |
| 2026-09-26 | In group calls/meetings every session's other side is Microsoft's media server (`@odata.type` ends with `serviceEndpoint`); its network always reports `wired`. | Exclude service endpoints from all user-side statistics. |
| 2026-09-26 | The web client reports `connectionType=unknown` and no `subnet`; only a local IP. | "Unknown" connection rows are mostly web users; network is approximated by IPv4 /24. |
| 2026-09-26 | Recording/transcription bots appear in `participants_v2` as ordinary `user` identities: `displayName` = an application GUID, `userPrincipalName = ""`, `tenantId` = our tenant. The only discriminator is in sessions: `userAgent.role = customBot` (documented service role) and/or `userAgent.headerValue` containing `GraphCommunicationsClient-<agent>/<version>` and `appid:<guid>`. Graph does not distinguish recording from transcription. | Bots are not people (see §7.5). |
| 2026-10-03 | `failureInfo` is populated on **connected** meetings too (87 min, 1.5 M packets, `stage=midcall`, `reason=''`). Docs: "why the call or a part of it failed". | `failed` only when not connected; connected + failureInfo = `interrupted` (derive v9). |
| 2026-09-26 | Live event (town hall) attendees are not recorded in call records. | Document as a known gap. |

## 5 Data model

All tables live in the product schema (see `../../platform/database.md`). Timestamps are `timestamptz` in UTC.
No local time and no natural-language text is ever stored in derived data.

### 5.1 `ingest_runs` (shared with other sources; defined by platform, used here)

| Column | Meaning |
|---|---|
| id | run id; correlates all log lines of the run |
| source | `graph.callRecords` for this module |
| started_at, finished_at | wall-clock |
| window_start, window_end | requested Graph window |
| status | `running` · `ok` · `partial` (some records had fetch errors) · `failed` (aborted; rows written so far stay) |
| records_seen / inserted / updated / unchanged | counters |
| errors | JSON list of per-record errors `{id, errors[]}` and a final `{fatal}` if aborted |
| meta | `{hours, derive_version, workers, max_rps}` |

A run stuck in `running` (process killed) is harmless; the scheduler marks runs older than a timeout as
`failed` (open item in the reference, required here).

### 5.2 `call_records` — one row per Graph callRecord

| Column | Kind | Source / rule |
|---|---|---|
| id | uuid PK | Graph id |
| version | bigint | Graph `version`; increases when Graph enriches the record |
| call_type | text | detail `type`: `peerToPeer` · `groupCall` · `unknown` |
| start_at, end_at | timestamptz | parsed `startDateTime` / `endDateTime` |
| duration_s | int | `end_at − start_at` in whole seconds; null if either missing |
| last_modified_at | timestamptz | Graph `lastModifiedDateTime` |
| join_url | text | Graph `joinWebUrl`; **empty string normalised to null** |
| organizer_key | text → identities.key | from `organizer_v2.identity` |
| **raw** | jsonb | detail response of `?$expand=participants_v2` (first page as returned) |
| **participants_raw** | jsonb | all participant pages concatenated |
| **sessions_raw** | jsonb | all sessions with segments and media, all pages concatenated |
| outcome_state | text | `connected` · `short` · `interrupted` · `unanswered` · `failed` (§7.2) |
| connected | bool | §7.2 |
| media_packets | bigint | sum of `packetUtilization` over all streams |
| quality | jsonb | record-level quality summary (§7.6) |
| derived | jsonb | flat view used by list rows and the detail drawer (§7.7) |
| derived_version | int | `DERIVE_VERSION` that produced the derived columns |
| fetch_errors | jsonb list | non-empty → record is re-fetched next run |
| first_seen_run, last_seen_run | bigint → ingest_runs | `last_seen_run` kept on reprocess (null input never overwrites) |
| first_seen_at, updated_at | timestamptz | bookkeeping |

Indexes: `start_at DESC`, `call_type`, `outcome_state`, GIN on `raw` (path ops), partial indexes for
"has poor audio" and "has recording bot" (used by call search), `organizer_key`.

### 5.3 `identities` — every identity seen in calls

| Column | Meaning |
|---|---|
| key | PK, `<kind>:<external id or name>` e.g. `user:<guid>`, `phone:+15550100`, `guest:<display name>`, `unknown:<guid>`, `endpoint:<name>` |
| kind | `user`, `guest`, `phone`, `acsUser`, `spoolUser`, `encrypted`, `onPremises`, `applicationInstance`, `application`, `device`, `unknown` |
| external_id, display_name, upn, tenant_id | latest non-null wins (null never overwrites) |
| attrs | jsonb, reserved for directory enrichment |
| first_seen_at, last_seen_at | by **call time**, not ingest time (LEAST / GREATEST on upsert) |

`user:<oid>` keys must equal directory-user keys (lower-case object id) so that call data joins directory
data (department, manager, room accounts).

### 5.4 `identity_names` — every (key, display_name, upn) ever observed

Used by masking (`../../platform/scope-masking.md`) so that renamed people are still masked in old records.
RLS enabled, no policy, `REVOKE ALL` from the web role (owner connection only).

### 5.5 `call_participants` — call × identity × role

| Column | Meaning |
|---|---|
| call_id, identity_key, role | PK; role ∈ `caller`, `callee`, `participant` |
| joins | number of times the identity joined (deduplicated rejoins) |
| platform | caller/callee platform for 1:1 calls (`windows`, `macOS`, `iOS`, `android`, `web`, …) |

Double-count rule: in a `peerToPeer` record the same two people appear both as caller/callee and in
`participants_v2`. Write **only** `caller` + `callee` rows for them; never `participant`. In group calls there
is no caller/callee; everyone is `participant`. Bots are never written here.

Rows for a call are deleted and rewritten on every save (participant lists can change between versions).

### 5.6 `call_media_streams` — one row per media stream (derived, rebuildable)

| Column | Meaning |
|---|---|
| call_id, start_at, call_type | denormalised from the call (start_at for range filters) |
| session_no | 1-based session ordinal |
| label | Graph media label (`main-audio`, `main-video`, `applicationsharing-video`, …) |
| direction | `callerToCallee` / `calleeToCaller` |
| side | `caller` / `callee` — the **sending** side |
| is_service | sending endpoint is a Microsoft service endpoint |
| platform, product | sender `userAgent.platform` / `productFamily` |
| person_id, person | sender `associatedIdentity` id / display name |
| connection_type | sender network `connectionType`, `unknown` → null |
| network | sender `subnet`, else IPv4 with last octet replaced by 0 (/24); null otherwise |
| public_ip | sender `reflexiveIPAddress` (sensitive; never shown to non-admins) |
| wifi_signal | sender network Wi-Fi signal strength |
| capture_device, render_device | sender device names; empty → null |
| packets | `packetUtilization` |
| jitter_ms, loss, rtt_ms | `averageJitter` (ISO duration → ms), `averagePacketLossRate` (fraction 0–1), `averageRoundTripTime` (→ ms) |
| classified | audio label AND packets > 500 |
| poor | classified AND (jitter > 30 ms OR loss > 0.10 OR rtt > 500 ms) |

Stream attribution rule: a stream is attributed to its **sending** side (its network, device, platform). The
measurement is taken at the receiver but reflects the sender's uplink.

Reference implementation filled this table with a database trigger calling an SQL unpacking function. For the
rebuild, the derive layer must produce these rows (pure function) and the save step writes them in the same
transaction, so CQD thresholds exist in **one place only** (the reference had them duplicated in Python and SQL —
a known maintenance hazard). RLS: a stream row is visible iff its call is visible.

### 5.7 Size guidance

Raw meeting JSON ≈ 3 KB + ~14 KB per participant session (measured on a small sample). Stored size ≈ 2× raw
(JSONB + derived + indexes). List response ≈ 1.6 KB per record. A single huge meeting may be several MB.

## 6 Ingest / request flow

Entry: the scheduler calls `ingest_calls(hours)` (default 24 for manual runs; 6 for scheduled runs every
15 minutes; a periodic wide sweep of 48 h once a day; a one-off backfill of 719 h on first install).

1. Permission gate: feature `calls` must be granted and the `calls` package enabled; otherwise log
   `ingest.skipped` / `ingest.module_off` and return without any request.
2. Clamp hours to 719. Compute `since = now − hours`, `until = now` (UTC, second precision, `Z` suffix).
3. `start_run(source=graph.callRecords, window, meta)` → `run_id`; log `ingest.start`.
4. List: GET the list with the `$filter` on `startDateTime` (both bounds), follow `@odata.nextLink` until done.
   Log `graph.list` (pages, records, duration).
5. Plan (one SQL query, no per-record queries): `ids_needing_fetch(listed)` returns ids where **any** of:
   - the record is not in the DB;
   - listed `version` > stored `version`;
   - stored `fetch_errors` is non-empty (retry);
   - stored `derived_version` < current `DERIVE_VERSION`.
   All other listed ids are "unchanged": one bulk update sets `last_seen_run = run_id`. Log `ingest.plan`.
6. Fetch in parallel: a pool of `GRAPH_WORKERS` (default 8) threads; at most `workers × 4` tasks queued at once
   (bounded memory). Each task (network only, no DB):
   - GET detail with `$expand=participants_v2`; follow `participants_v2@odata.nextLink` then `@odata.nextLink`.
     On failure record the error and fall back to the list item as `raw` with empty participants.
   - GET with `$expand=sessions($expand=segments)`; follow `sessions@odata.nextLink` then `@odata.nextLink`.
     On failure record the error and use empty sessions.
   - If more than one page was followed, log `graph.expand_pages` (INFO).
7. Write on a single thread as tasks complete: `save_record(raw, participants, sessions, errors, run_id)` in
   **one transaction**: derive → upsert identities → upsert identity names → upsert call row (insert vs update is
   reported by the database) → delete + insert participants → delete + insert media streams. A half-written call
   is impossible. Log derive diagnostics (§7.8). Count `inserted` / `updated`.
8. Progress: every 100 records log `ingest.progress` (done, total, rate, errors).
9. Finish: status `partial` if any record had fetch errors, else `ok`; log `ingest.done` with counters,
   records/s and Graph request statistics delta (requests, throttled, reauth). On any exception: log
   `ingest.failed` with traceback, finish run as `failed`, re-raise.

Shared throttle and resilience (implemented in the graph client, required behaviour):
- Global rate limiter: at most `GRAPH_MAX_RPS` (default 50, ceiling 75) request starts per second across all
  threads. Keep it low if other apps share the tenant quota.
- On 429/503/504: read `Retry-After` (seconds; fallback 2^attempt) and pause **all** threads until then
  (`graph.throttled`). Up to 5 attempts per request, then `graph.retries_exhausted`.
- Token: client-credentials token cached; refreshed `min(300 s, 10% of lifetime)` before expiry under a lock;
  on a 401, force-refresh once and retry (`graph.reauth`); a second 401 is a permanent error.
- Network errors (connection reset, timeout): 3 attempts with 1 s, 2 s backoff (`graph.network_retry` /
  `graph.network_error`).

Measured capacity (fake Graph, 200 ms latency): sequential 2.4 rec/s → 8 workers 17.6 rec/s; with a 50 rps cap
≈ 25 rec/s ≈ 90,000 records/hour. 5% random 429s: 400 records, 49 throttles, 0 loss. 15-second token lifetime:
400/400 records, 0 errors. A 30-day backfill of 100,000 records: ~14 min listing (list limit 40/20 s,
60 per page) + ~70 min details.

Known window gap: the list filters by `startDateTime`. A record that started 3 h ago whose `version` increases
now is not seen by a 2 h window. Mitigation: scheduled window 6 h + daily 48 h sweep. Preferred improvement:
start the window at the last successful run's `window_end` minus a safety margin.

### Reprocess

`reprocess()` re-derives every stored record from `raw` / `participants_raw` / `sessions_raw` with the current
derive code, through the same `save_record` path (run id null → `last_seen_run` preserved), without any Graph
request. Also deletes orphan bot identities left by older derive versions (identity whose display name equals
its external id, empty UPN, referenced by no call, organizer or directory user). Logs `db.reprocess`
(records, duration, orphan_bot_identities). Must be runnable without producing any report file.

Critical database rule: the connection must be in autocommit mode so that each `save_record` transaction is a
real transaction. In the reference, a non-autocommit connection turned every transaction into a savepoint of
one implicit outer transaction: reprocess was never persisted and an interrupted ingest lost everything. Always
verify writes from a separate session.

## 7 Derived logic & metric definitions

The derive layer is a set of pure functions: input = (detail dict, participants list, sessions list,
fetch_errors list); output = `{columns, identities, participation, media_streams, diagnostics}`. No network,
no database, no logging, no clock, no locale, no environment variables or time zone. Output contains only codes,
numbers, ids, names as received, and UTC ISO timestamps or offsets in seconds.

`DERIVE_VERSION` is an integer constant in the derive module. Any change to derive output semantics increments
it and appends a row to the version history table in this document's companion changelog, then a reprocess is
run. History of the reference (keep numbering continuity or restart at 1 with this table as v0 lineage):

| Version | Change |
|---|---|
| 1 | initial |
| 2 | quality summary excludes the meeting server side; caller/callee matched to participants by id then name; unmatched tenant-less identity = `unknown` (external) |
| 3 | timeline (person × device lanes, events, per-lane audio); diagnostics list |
| 4 | per-lane media coverage: sent/received packets for audio, camera, screen share |
| 5 | person seconds / attendee seconds (union of a person's device intervals) |
| 6 | bots are not people; `bots[]`, `recording_or_transcript`; lanes/events mark bots |
| 7 | room/device platform codes (`roomSystem`, `surfaceHub`, `ipPhone`, `holoLens`) |
| 8 | language-neutral derived data: no labels, no local time; events = `{offset_s, kind, code, p}` |
| 9 | connected + failureInfo = `interrupted`; `failed` only when not connected |

### 7.1 Record classification (used by every module, must be identical everywhere)

| Class | Code | Rule |
|---|---|---|
| 1:1 call | `p2p` | `call_type = peerToPeer` |
| Meeting (linked) | `meeting` | `call_type ≠ peerToPeer` (including `unknown`) AND `join_url` not null |
| Group call (unlinked) | `group` | `call_type ≠ peerToPeer` AND `join_url` null |

"Meetings" in analytics = `meeting` + `group`. A 1:1 call that Teams converts into a group call (e.g. when a
participant transfers the call to another device) is a `groupCall` record whose first session is a
user-to-user leg; the list row shows "A → B · group call" (timeline event `p2p_to_group`). Expose the class as a
generated column or SQL function `call_class(call_type, join_url)`; never re-implement the rule in UI code.
(Reference inconsistency to avoid: one daily view counted only `groupCall` as meetings and treated an empty
`join_url` differently from null.)

### 7.2 Outcome

Inputs: `seconds` = duration, `packets` = sum of `packetUtilization` over all streams,
`failures` = non-empty `failureInfo` (`stage` or `reason` present) on any session or segment.

- `connected` = packets > 0 OR seconds > 0
- state:
  - not connected and failures → `failed`
  - not connected → `unanswered` (missed or declined; Graph cannot tell)
  - connected and failures → `interrupted` (call held, with an interruption)
  - connected and seconds < 5 → `short`
  - otherwise → `connected`
- `outcome.failures` keeps `[{stage, reason}]` (stage ∈ `callSetup`, `midcall`, …). The same failure can be
  present on both session and segment — deduplicate before counting interruptions.

Derived counting rules (all modules): **missed = NOT connected**, **answered = connected**.
`interrupted` and `short` are answered.

### 7.3 Participants and joins

For each `participants_v2` item, build an identity record from the identitySet: check known kinds in order
(`user, guest, phone, acsUser, spoolUser, encrypted, onPremises, applicationInstance, application, device`,
then any other non-`@` key); the first with a display name, UPN or id wins. Key = `<kind>:<id or name>`.
Same key repeated → one person with `joins = count`. `participant_count` = number of distinct people (bots
excluded). `join_count` = raw participant entries.

### 7.4 Caller / callee (1:1 only)

From `sessions[0].caller` and `.callee` (`associatedIdentity`, flat object):
1. id matches a participant's external id → use that participant's identity;
2. else display name matches a participant's display name → use it (diagnostic `endpoint_matched_by_name`);
3. else build: kind `user` if id and tenantId present, else `unknown` (counted as external); key
   `<kind>:<id>` or `endpoint:<name>` (diagnostic `endpoint_unmatched`).
Platform = endpoint `userAgent.platform`. Joins = participant joins or 1.

### 7.5 Bots

A session endpoint is a bot if not a service endpoint and (`userAgent.role = customBot` OR header value matches
`GraphCommunicationsClient-<agent>`). Bot type by agent (case-insensitive): `callrecorderbot` → `recorder`,
`rtaf` → `media`, other → `bot`. Fallback without session data: a participant whose display name equals its own
id and whose UPN is empty → `unnamed`. Bots are excluded from people, participant count, person seconds,
external counts, `call_participants` and `identities`. `derived.bots[] = {app_id, type, agent, joins}`;
`recording_or_transcript = any bot of type recorder`. Third-party compliance-recording bots are also `customBot`
and are treated the same way.

### 7.6 Record quality summary (`quality`)

Over all audio streams (label contains `audio`) whose **sending side is not a service endpoint**:
- `unclassified_audio_streams`: packets ≤ 500 (e.g. a muted participant's outgoing stream). Excluded from the
  denominator.
- `audio_streams`: classified streams; `poor_audio_streams`: classified streams exceeding any CQD threshold
  (jitter > 30 ms, packet loss > 0.10, RTT > 500 ms).
- `worst`: max jitter/loss/rtt over classified streams.
- `connection_types`: distinct non-unknown connection types of non-service sides.

Decision for the rebuild (resolves an open item of the reference): the reference counted service-side streams in
the record-level poor badge while analytics counted user-side only, so a meeting could be "poor" in the list but
not in analytics. Here both use user-side streams only, computed from the same stream rows (§5.6).

CQD thresholds (one constant set, one place):

| Metric | Field | Poor if |
|---|---|---|
| Jitter | `averageJitter` (ISO 8601 duration, e.g. `PT0.004S` = 4 ms) | > 30 ms |
| Packet loss | `averagePacketLossRate` (fraction) | > 0.10 |
| Round trip | `averageRoundTripTime` (ISO duration) | > 500 ms |
| Precondition | `packetUtilization` | > 500, else unclassified |

### 7.7 Timeline (`derived.timeline`)

- Lanes: one per (person id or name, platform) across sessions, service endpoints excluded; bots get platform
  `bot` and person null. Lane intervals = session start/end; intervals of the same lane merge when the gap ≤ 5 s
  (`EDGE_S`). Lane fields: `person`, `person_id`, `platform` (raw code), `offset_s`, `dur_s`,
  `segments[{offset_s, dur_s}]`, `audio` (the lane's **sent** main audio stream with most packets:
  packets, jitter_ms, rtt_ms, loss, classified, poor, connection, device), `media{audio|video|share:
  {sent, recv, declared}}`, `bot`.
- Media coverage: `declared` from session modalities (present at some point); `sent`/`recv` from stream
  packets by direction. Labels: `applicationsharing*` or `screen` → share, other `video` → camera, `audio` → audio.
  Graph does not say **when** or **how long** screen sharing or camera was on — UI must not draw it inside bars.
- Events `{offset_s, kind, code, p}` (sorted by time then kind order ring/start < missed < join/bot < switch <
  info < leave < end):
  - first user↔user leg starting within 5 s of call start: duration < 1 s → `call_ringing` + `not_connected`;
    else `call_started` (`p`: caller, caller_platform, callee, callee_platform) and, if the record is not p2p,
    `p2p_to_group` at the leg end. Otherwise `meeting_started` (join URL) or `group_call_started`.
  - `device_switched` (`p`: person, from, to): same person, one lane ends and another starts within ±15 s
    (`SWITCH_GAP_S`), not at call end.
  - `joined` / `left` (`p`: person, platform) when more than 5 s from the call edges and not part of a switch.
  - `bot_joined` / `bot_left` (`p`: bot type).
  - `ended` (`p`: duration_s) when duration ≥ 1 s.
- `person_seconds{name: s}`: per person, union of all their lane segments (two devices at once count once);
  `attendee_seconds` = sum. Bots excluded. Used for meeting cost / person-hours.
- Zero-duration records: no lanes drawn, events only.

`derived` also carries: id, version, type, start, end (UTC ISO), duration_seconds, modalities, join_url,
organizer `{name, upn, kind}`, participants `[{name, upn, kind, joins}]`, participant_count, bots,
recording_or_transcript, join_count, quality, outcome, sides `{caller, callee}` (`{name, upn, platform}`),
timeline, errors.

### 7.8 Diagnostics

The derive layer returns `diagnostics: [{code, ...params}]`; it never logs. The save step logs each as
`derive.<code>`: `endpoint_matched_by_name`, `endpoint_unmatched`, `bot_detected` (bot_type, agent, app_id)
at INFO; `no_sessions`, `no_participants`, `connected_without_measured_audio` at DEBUG.

### 7.9 External identity

`is_external(kind, tenant_id)`: `guest`, `phone`, `acsUser`, `spoolUser`, `unknown` → always external;
`user` → external iff tenant_id is not null and ≠ home tenant (settings `home_tenant`, synced from configuration
on every migrate). Known unverified edge: how B2B guest accounts (`#EXT#`) appear in call records.

## 8 UI

No tab. This module surfaces only through:
- Run status in the shell freshness line ("last call ingest N min ago · status") — `../../platform/ui-shell.md`.
- Permission card for feature `calls` on every call/meeting tab when the permission is missing.
- Logs tab (platform) shows `ingest.*` / `graph.*` events.
- Empty state on consumer tabs when no run has ever completed: one grey line "No call data yet. The first
  import runs after the permission is granted; records appear 15–30 minutes after a call ends."

### Permission transparency (shared component, `../../platform/permissions.md`)

- **"Which permissions does this need?"** popover in the module header (reference: `../user-lookup/DESIGN.md`):
  lists this module's permissions least-privileged first with alternatives as "(alternative: X or Y)", ✓ per held
  option from the token `roles` claim, ✗ if none held, required/optional, what it is used for, how to grant
  (Azure Portal → App registrations → the app → API permissions → Add → Grant admin consent), and "not needed"
  permissions (module off or not used by this module).
  Rows: `CallRecords.Read.All` (Application, required; no alternative; delegated not supported).
- **No live Graph calls at view time** in this module, so no request-trace popover. Instead each data section has
  a **"What fed this?"** popover: ingest step, Graph endpoint, permission, and last successful run time
  (shown on every consuming call/meeting tab; this module provides the provenance record
  `{step: calls, endpoint: /communications/callRecords, permission: CallRecords.Read.All, last_ok_at}`).

## 9 Access control

- No RBAC module key of its own; consumers gate on `calls`, `call_analytics`, `meeting_analytics`,
  `combined_analytics`, `rooms`.
- RLS (see `../../platform/scope-masking.md`): `call_records` visible iff at least one participant identity is
  in the viewer's scope (`call_in_scope`); `call_participants` rows visible only for in-scope identities plus
  external identities; `call_media_streams` visible iff the call is visible; `identities` filtered by scope;
  `identity_names` owner-only. All views `security_invoker = true`.
- Audit: none (system process). Reprocess via CLI writes `cli_*` audit entries per platform rules.

## 10 Logging events

| Event | Level | Context |
|---|---|---|
| `ingest.start` | INFO | hours, since, until, workers, max_rps |
| `graph.list` | INFO | since, until, records, pages, duration_ms |
| `ingest.plan` | INFO | seen, todo, unchanged |
| `graph.request` | DEBUG | url (short), status, attempt, duration_ms |
| `graph.expand_pages` | INFO | relation, id, pages, items |
| `graph.throttled` / `graph.reauth` / `graph.network_retry` | WARNING | status, retry_after, attempt |
| `graph.error` / `graph.retries_exhausted` / `graph.network_error` | ERROR | status, body (≤400 chars), url |
| `ingest.record` | DEBUG | id, result, type, participants, sessions, fetch_ms, duration_ms |
| `ingest.record_error` | WARNING | id, errors |
| `ingest.progress` | INFO | done, total, rate, errors |
| `derive.<code>` | INFO/DEBUG | id + diagnostic params |
| `ingest.done` | INFO | counts, seconds, records_per_s, graph stats delta |
| `ingest.failed` | ERROR | exc, counts |
| `ingest.skipped` / `ingest.module_off` | WARNING / INFO | feature, needed |
| `db.reprocess` | INFO | derive_version, records, orphan_bot_identities, duration_ms |
| `run.start` / `run.finish` | INFO/WARNING/ERROR | platform run bookkeeping |

Never log tokens, Authorization headers, full join URLs with tokens, or participant lists at INFO.

## 11 i18n notes

- Derived data is language-neutral: outcome codes, failure stage codes, bot types, platform codes, event codes
  with parameters, UTC timestamps, second offsets. Display text is produced only by label functions in the
  presentation layer (see `../../platform/i18n.md`).
- Label catalogue owned by this module (English source strings): outcomes (`Call connected`, `Very short call`,
  `Call held, with an interruption`, `Not connected (missed or declined)`, `Failed`), failure stages
  (`during setup`, `mid-call`, `unknown stage`), bot types (`Recording / transcription bot`,
  `Microsoft media bot`, `App / bot`, `Unnamed app / bot`), platforms (`roomSystem` → `Room system`,
  `ipPhone` → `IP phone`, others proper nouns), event sentences with named parameters.
- Local time conversion (`APP_TIMEZONE`) happens only when rendering.
- Log messages are English; event keys ASCII.

## 12 Testing & acceptance criteria

Fake Graph layer (`../../platform/testing.md`) must reproduce: 60-item list pages with `@odata.nextLink`;
participants paged at 130 with the first continuation under `participants_v2@odata.nextLink`; sessions paged at
60 under `sessions@odata.nextLink`; 400 on two `$expand` items; 400 on a 720 h window; 429 with `Retry-After`;
503; 401 once (expired token); connection reset; variable fractional-second digits.

Acceptance:
- [ ] A meeting with 300 participant joins and 150 sessions is stored with all 300 / 150 items.
- [ ] Re-running the same window performs exactly 1 list request series and 0 detail requests (all unchanged);
      a version bump of one record triggers exactly that record's 2+ requests.
- [ ] Incrementing `DERIVE_VERSION` causes all records in the window to be re-fetched by ingest, and
      `reprocess` re-derives all records with 0 Graph requests; verified from a separate DB session.
- [ ] 5% random 429: no record lost; all threads paused (no request started during a pause).
- [ ] Token lifetime 15 s during a 400-record run: 0 errors, ≥1 `graph.reauth` or refresh.
- [ ] Missed call fixture (equal start/end, zero packets, null failureInfo) → `unanswered`, connected false.
- [ ] Connected meeting with `failureInfo{stage: midcall, reason: ''}` → `interrupted`, connected true.
- [ ] 1:1 call → exactly 2 participant rows (caller, callee); no `participant` row.
- [ ] Caller id not in participants but same display name → no duplicate person.
- [ ] Recording bot (user identity, GUID name, empty UPN, `customBot` in session) → not a person;
      `recording_or_transcript = true`; event `bot_joined`.
- [ ] Group call: service endpoint streams excluded from `quality` and flagged `is_service` in stream rows.
- [ ] Stream rows produced by derive equal (both directions, set difference 0) the reference SQL unpacking on
      the same fixture set.
- [ ] Derive functions are deterministic: same input → byte-identical output; no import of network/db/log modules.
- [ ] Kill the process mid-run: rows written so far persist; run stays `running` until the timeout marks it failed.
- [ ] Load test: ≥ 17 records/s with 8 workers and 200 ms fake latency; DB write thread never the bottleneck.

- [ ] "Which permissions does this need?" popover lists exactly: `CallRecords.Read.All` (required); ✓/✗ follow the token roles snapshot
      (test with a snapshot holding none, the least-privileged option, and only an alternative); grant steps shown.
- [ ] Every data section's "What fed this?" popover shows ingest step, endpoint, permission and the last
      successful `ingest_runs` time; with no successful run it says "never".

## 13 Implementation steps for an agent

1. Read the platform docs listed in §3; confirm the graph client provides shared throttle, token refresh,
   429 brake and network retry (implement there if missing, not here).
2. Migration: `call_records`, `identities`, `identity_names`, `call_participants`, `call_media_streams`,
   indexes, `is_external`, `call_class`, ISO-duration helper, `home_tenant` setting sync; RLS policies and
   web-role grants per `../../platform/scope-masking.md`.
3. Derive module: tolerant ISO parser, ISO duration → ms, identity parsing, outcome, caller/callee matching,
   bots, quality summary, stream rows, timeline, person seconds, diagnostics; `DERIVE_VERSION = 1` (or continue
   at 10) with the history table.
4. Unit tests for every derive rule in §7 with JSON fixtures shaped like real Graph responses (no real ids).
5. Graph wrappers: `list_records(since, until)`, `fetch_detail(id)`, `fetch_sessions(id)`, `follow_expanded`.
6. DB layer: `ids_needing_fetch` (single query), `touch_many`, `save_record` (single transaction),
   `reprocess`, run bookkeeping.
7. Ingest orchestration with bounded parallel queue and single writer; counters; logs from §10.
8. Register feature `calls` (needs `CallRecords.Read.All`) in the permission map and package `calls`.
9. Scheduler wiring: scheduled 6 h window every 15 min, daily 48 h sweep, `--reprocess`, first-install 719 h.
10. Fake-Graph integration tests and the load test from §12; verify every write from a separate session.
11. Update the about module data-source table (endpoint, permission, version, freshness, retention).

## 14 Open questions / future

- Change notifications (`/communications/callRecords` subscription) for near-real-time; requires a public HTTPS
  endpoint, validation handshake and periodic renewal; keep the periodic pull as a safety net.
- `callEvents` to distinguish missed / declined / caller-cancelled (permission not used today).
- PSTN and Direct Routing logs for organisations with phone numbers (separate 200/60 s limit).
- Window start from last successful `window_end` instead of a fixed lookback.
- Verify B2B guest (`#EXT#`) tenant attribution in real data.
- Pre-aggregated daily/hourly tables if RLS cost (~9 µs per call row) becomes the bottleneck at 100k+ calls.
- Interruption count metric: requires deduplicating session/segment failures first.
