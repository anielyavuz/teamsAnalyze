# Testing

Testing proves behaviour on the **real code paths** (real SQL, real HTTP client code, real templates) with
fake inputs. Every module doc §12 lists its fixtures and numbered acceptance criteria; this document
defines the shared method and tools.

## 1. Test layers

| Layer | What | Tooling |
|---|---|---|
| Pure unit | derive functions, parsers, label helpers, masking engine, permission evaluation | plain unit tests; no DB/network; fixed "now" |
| SQL | views/functions return the expected numbers for embedded patterns | test database copy |
| Integration | ingest step end-to-end through the Graph client with the fake transport | test DB + fake Graph |
| Web | routes, auth, RBAC, scope, masking, audit | ASGI test client against test DB, web role |
| Security | leak tests, RLS checks, CSRF, injection payloads | section 5 |
| UI | screenshots, DOM probes | headless browser |
| Load | throughput and request counts at target scale | fake Graph with latency |

## 2. Test database

- Never write fake data into a real database.
- Create a copy for a test session: `createdb -T <db> <db>_test` (template copy) or an empty DB +
  migrations. Point the test at it by overriding `DATABASE_URL` / `WEB_DATABASE_URL` in the test process
  environment (env vars take precedence over the env file). Drop it afterwards.
- Verify writes from a separate connection (autocommit pitfalls, `database.md` §3).
- Web tests use the real `teams_web` role so RLS is exercised; a fixture helper creates test users, roles
  and grants.

## 3. Fake Graph transport

Replace the Graph client's HTTP session factory with a fake that serves fixtures in **real response
shapes**:

- Collections paged (default 60 per page) with `@odata.nextLink`; `$expand` continuation under
  `<relation>@odata.nextLink` on the first page, `@odata.nextLink` after.
- 302 with `Location` to a fake download host, serving CSV with a UTF-8 BOM, headers in inconsistent
  case/order/spacing.
- Error injection by rule: 429 with `Retry-After` (random x %), 503/504, 401 once (expired token), 403 with
  realistic error bodies (`error.code`, `innerError.code`), 404, connection errors, timeouts.
- Token endpoint fake with configurable `expires_in` (e.g. 15 s to test refresh) and `roles` claim.
- Configurable latency per request (load tests) and a request log (method, path, params) to assert request
  counts.
- Fixture builders generate realistic objects with known embedded patterns (e.g. "missed calls peak at
  lunch", "one network has poor audio", "external participants in 20 % of meetings").

The fake asserts that no request to the download host carries an `Authorization` header.

## 4. What each kind of test must prove

- **Throttle/token:** with 5 % random 429 and a 15 s token, a 400-record ingest loses 0 records, no
  unbounded loop on permanent 401, `graph.throttled` and `graph.reauth` logged.
- **Idempotency:** running the same ingest twice changes nothing and fetches no unchanged details
  (assert request count).
- **Re-fetch rule:** version bump, previous error, derive version bump each cause exactly one re-fetch.
- **Reprocess:** after a derive version bump, all rows updated without any Graph request; verified from a
  separate session.
- **Step isolation:** a step raising an exception does not stop later steps; exit code 1;
  `cli.steps_failed` logged.
- **Permissions:** feature marked `missing` → step skipped (`ingest.skipped`), no 403 requests, locked card
  rendered; `limited` → yellow note; package disabled → "not required", no probe.
- **Logging:** expected events present for the scenario; leak query on `app_log` returns 0 after error
  scenarios that include report URLs and tokens in exception text.

## 5. Security tests (required before any people/rooms screen ships)

Run as part of CI and by the security reviewer:

1. **RLS:** as `teams_web` with no scope context every RLS table returns 0 rows; with a scope, row counts
   equal the expected set; all views have `security_invoker`; web role cannot DDL or delete audit rows.
2. **Leak matrix** (`scope-masking.md` §9): every module × {full, masked, aggregate, small group} × scope
   types × endpoints (shell, tab, API, export, record detail, preview). Scan responses for every name/UPN/oid
   of people not at full detail, case-insensitive, in all escaping variants.
3. **Aggregate:** person-level endpoints return 403; HTML has no person-level blocks.
4. **Search side channel:** masked/out-of-scope name returns a response identical to a random string.
5. **Injection:** display names containing `</script>`, `<!--`, `-->`, quotes, `&`, U+2028 — page structure
   unchanged (DOM probe counts sections/scripts), masking intact.
6. **Auth:** unauthenticated → 401/redirect; wrong tenant → denied; disabled user → session ends
   immediately; CSRF-less POST → 403; lockout scenarios refused; preview uses target's view for lazy tabs
   and APIs; audit rows written for each.
7. **Headers:** CSP, nosniff, no-store present on authenticated responses.

## 6. UI checks

- Headless browser screenshots for each touched screen: light, dark (`data-theme="dark"`), phone 390 px
  rendered **inside an iframe** (headless browsers enforce a minimum window width around 500 px), plus
  states: empty, locked, limited, aggregate, small group, error, popovers open.
- If shared CSS changed: every screen.
- Languages: `en`, `tr`, pseudo-locale `xx` (unmarked text visible).
- DOM probes: inject a small script, dump the DOM, assert behaviour (filters, sorting, drawer, lazy load)
  without a full browser test framework.
- Data-viz colours run through the contrast validator for light (`#FFFFFF`) and dark (`#11141B`) surfaces;
  record PASS in the module doc.
- Render-check tool: render a screen with a fixed "now" and fixed data, diff against a stored reference to
  prove refactors do not change output.

## 7. Load tests

Fake Graph with ~200 ms latency per request. Measure records/s, requests/s, concurrency, 429/401 counts,
rows written, memory; compare to the module's budget (doc §6). Web: build time per tab at 10,000 records for
full, masked and scoped viewers; target < 2 s per tab and < 1 s for the shell.

## 8. Acceptance gate (per step)

A step/module is accepted when: its doc §12 criteria pass; sections 4–6 relevant items pass; leak matrix
extended and green; screenshots reviewed; load result recorded; independent security review recorded for
security-relevant changes.
