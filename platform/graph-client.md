# Graph client

All Microsoft Graph traffic — collector and web — goes through one client module. No other code opens
HTTP connections to Graph or to the token endpoint. Why: throttling, token refresh, retry rules, logging
and scrubbing must be identical everywhere, and a single place makes fake-Graph testing possible.

## 1. Authentication (app-only)

- Flow: OAuth 2.0 client credentials against `https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token`
  with scope `https://graph.microsoft.com/.default`. Use MSAL's confidential client.
- Credential: **certificate** (recommended for production) or client secret. Configuration via environment
  (`deployment.md`). Missing configuration → `config.missing` (ERROR) listing the variable names, exit.
- Token failure → `graph.token_failed` (ERROR) with HTTP status and the AADSTS code only (never the body
  verbatim if it may echo credentials; scrubber still applies).
- Delegated permissions are not used for collection (call records support application permissions only).

### Token provider

| Rule | Value / behaviour |
|---|---|
| Cache | One token per process, shared by all threads, guarded by a lock (only one refresh at a time) |
| Proactive refresh | When `now > expires_at − margin`, `margin = min(300 s, 10 % of lifetime)` |
| Reactive refresh | On HTTP 401, force a **new** token once per request attempt chain, then retry. A forced refresh must bypass any library cache (MSAL would otherwise return the same cached token) |
| Permanent 401 | Second 401 after forced refresh → permanent error (no loop) |
| Logging | `graph.token` INFO on acquire/refresh with `ttl_s`, `forced`; `graph.reauth` WARNING on 401 |
| Claims | Expose decoded claims (`roles`, `appid`, `app_displayname`, `tid`) for the permission check — decoding only, no signature validation needed (token came straight from Entra over TLS) |

Why proactive + reactive: a long ingest outlives a ~60–90 min token; the reference lost every remaining
request to 401 before refresh existed.

## 2. Shared throttle

One process-wide throttle object used before **every** request (including probes):

- **Rate limit:** at most `GRAPH_MAX_RPS` request starts per second (default 50). Implemented as a
  "next allowed start time" advanced by `1/rps` under a lock; callers sleep outside the lock.
- **Global pause:** on 429, 503 or 504 the throttle is paused until `now + Retry-After` for **all** threads.
  Why: if only the throttled thread waits, the others keep hitting the limit and 429s multiply.
- `Retry-After` parsing: seconds (float). Missing/invalid → exponential `2^attempt` seconds.
- Documented limits that motivate the default (call records): 1,500 requests / 20 s per app per tenant
  (≈75/s), 10,000 / 20 s per tenant across apps, 40 / 20 s per record, 40 / 20 s for the list call; PSTN
  records 200 / 60 s. Other APIs have their own limits; module docs state theirs.
- Keep the default below the documented limit: other apps share tenant quotas.

## 3. Request function

Single internal `request(method, url, params, body, quiet=(), secret_params=False, raw=False)`:

| Situation | Behaviour |
|---|---|
| 200 (or 2xx with body) | Return parsed JSON |
| 302 with `Location` (report downloads) | Return the location to the caller-specific download helper; **never log it** |
| 401 | Force token refresh once, retry |
| 429 / 503 / 504 | Global pause (Retry-After), retry up to `MAX_RETRIES` (5) |
| Status in `quiet` | Log `graph.expected` at DEBUG with status + short body, raise `GraphHTTPError` |
| Other 4xx/5xx | Log `graph.error` at ERROR (status, short URL, params unless secret, body ≤ 400 chars), raise `GraphHTTPError` |
| Connection error / timeout | Up to 3 attempts with 1 s, 2 s waits: `graph.network_retry` WARNING, then `graph.network_error` ERROR and re-raise |
| Retries exhausted | `graph.retries_exhausted` ERROR, raise |

- Timeouts: connect+read 60 s for data, 30 s for probes and token.
- Every attempt logs `graph.request` at DEBUG: method, short URL (base stripped, `beta:` prefix for beta),
  params (or `<redacted>` when `secret_params`), status, attempt, `duration_ms`.
- `GraphHTTPError` carries `status`, `code` (`error.innerError.code` or `error.code`), `message`, and a
  truncated body. Callers branch on `status`/`code`, never on message text.
- Request statistics (requests, throttled, reauth) are counted per process and logged in each step's
  `*.done` event.

### Quiet (expected) errors

Some refusals are normal outcomes: a probe that confirms a licence is missing, a meeting organiser without
an access policy, a user without a mailbox. The caller passes the expected status codes in `quiet`; the
client logs at DEBUG and still raises so the caller can record the state. Why: the Logs screen must show
real problems, not hundreds of expected 403s.

### Secret parameters

Parameters that grant access by themselves (for example a meeting join URL used as a filter) are passed
with `secret_params=True`; they are logged as `<redacted>`.

## 4. Paging

- Standard collections: follow `@odata.nextLink` until absent; the next link already contains the query,
  so parameters are dropped after the first page. Record the page count for logging.
- **`$expand` paging quirk:** when a relation is expanded, the continuation of the *expanded* collection
  comes under `<relation>@odata.nextLink` (for example `participants_v2@odata.nextLink`) on the first
  response; subsequent pages use the normal `@odata.nextLink`. A generic loop misses it and silently loses
  data (observed with 130+ participants). Provide a helper `follow_expanded(first_items, next_link)`.
- Log multi-page expansions at INFO (`graph.expand_pages`, relation, id, pages, items): they indicate very
  large meetings that approach per-record limits.
- Observed page sizes (call records): list 60, participants 130, sessions 60. Do not rely on them.

## 5. `$expand` and query rules worth knowing

- Many endpoints accept **only one** `$expand` per request (call records: two expands → 400 "maximum allowed
  is 1"). Fetch separately; this cannot be optimised.
- List endpoints often omit relationships and some properties; fetch the item individually when needed.
- `$filter` support is endpoint-specific; module docs list the supported operators they rely on.
- `$search` on directory objects requires header `ConsistencyLevel: eventual` (and `$count=true` for some
  queries).
- Timestamps have a variable number of fractional digits (0–7). Parse with a tolerant ISO parser that
  normalises fractions; never rely on a strict parser.

## 6. Beta endpoints policy

- Use `v1.0` by default. A `beta` endpoint is allowed only when no v1.0 equivalent exists, and then:
  - the feature using it is marked **beta** in the permission registry and has a live probe;
  - the probe's failure sets the feature to `limited` (not `missing`) with a detail code, and the UI
    explains it without blaming the operator;
  - the module doc records the dependency, the date checked and any Microsoft retirement notice;
  - the module degrades gracefully (section hidden with a note), other sections unaffected.
- Never mix beta response fields into v1.0-derived logic without a fallback.

## 7. Downloads (reports)

Usage report endpoints answer 302 with a short-lived, **unauthenticated** download URL. Rules:

- Follow the redirect manually; send **no** `Authorization` header to the download host.
- Never log the URL, the `Location` header or exception text that contains it (the scrubber masks known
  report hosts and download paths as a second line of defence).
- Decode CSV as `utf-8-sig` (BOM). Normalise headers (case, spacing, order) before reading by name.

## 8. Concurrency

- Detail fetching uses a bounded thread pool (`GRAPH_WORKERS`, default 8) with a bounded queue (≤ 4× workers)
  so memory stays flat on large windows.
- Database writes stay on one thread (the coordinator), which consumes completed futures.
- The web process uses the same client; live modules must keep request fan-out small and bounded.

## 9. Testing hooks

The client uses one HTTP session object created by a factory; tests replace the factory with the fake
Graph transport (`testing.md`). The throttle and token provider must be resettable between tests.
