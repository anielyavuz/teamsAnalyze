# Module: Transcripts and recordings (metadata only)

> Module key: `transcripts` · Report tab: **Transcripts and recordings** · Data: scheduled per-meeting checks of whether a transcript / recording exists.
> **Content is never read.** Transcript text and recording media are not downloaded, stored or shown (owner decision 2026-10-17, after a short-lived content-backup phase was removed).

---

## 1 Purpose

For each finished internal Teams meeting: **was a transcript produced? was it recorded?** Then aggregate: share of meetings with transcription/recording, by day, week and organizer, and show where the app cannot check (missing tenant switch or access policy) and what to do about it.

## 2 Scope / Out of scope

**In scope:** per-meeting existence check for transcripts (required) and recordings (optional permission); exact rates over a clearly defined meeting set; access-problem reporting with remediation; chips on the call detail ("Transcript produced", "Recorded", "Bot joined, neither recording nor transcript").

**Out of scope:**

- Transcript content (`…/transcripts/{id}/content`, `…/metadataContent`) and recording content (`…/recordings/{id}/content`, `recordingContentUrl`) — never requested.
- Channel meetings and meetings organized outside the tenant (cannot be queried).
- Live/real-time transcription.

## 3 Dependencies

[architecture](../../platform/architecture.md) · [graph-client](../../platform/graph-client.md) (quiet 403/404 mode; secret query parameters not logged) · [database](../../platform/database.md) · [ingest-scheduler](../../platform/ingest-scheduler.md) · [permissions](../../platform/permissions.md) · [auth-rbac](../../platform/auth-rbac.md) · [scope-masking](../../platform/scope-masking.md) · [logging](../../platform/logging.md) · [i18n](../../platform/i18n.md) · [ui-shell](../../platform/ui-shell.md) · [testing](../../platform/testing.md) · [deployment](../../platform/deployment.md)

**Hard module dependency: [call-ingest](../call-ingest/DESIGN.md)** (call records, package `calls`). Candidate meetings come from call records: join URL, organizer, start/end, type. The scope rule reuses "call in scope". Without the calls module this module cannot run.

## 4 Microsoft Graph endpoints & permissions

| Method | Endpoint | Version | Permission | Alternatives | Why |
|---|---|---|---|---|---|
| GET | `/users/{organizerId}/onlineMeetings?$filter=JoinWebUrl eq '{joinUrl}'` | v1.0 | `OnlineMeetings.Read.All` | `OnlineMeetings.ReadWrite.All` (not recommended) | Resolve the online meeting behind a call record's join link |
| GET | `/users/{organizerId}/onlineMeetings/{meetingId}/transcripts` (paged) | v1.0 | `OnlineMeetingTranscript.Read.All` | — | Transcript metadata (id, created, ended) — existence only |
| GET | `/users/{organizerId}/onlineMeetings/{meetingId}/recordings` (paged) | v1.0 | `OnlineMeetingRecording.Read.All` (**optional**) | — | Recording metadata — existence only |

**Tenant prerequisites (besides permissions):**

1. **Tenant switch:** Teams admin center → Meetings → Meeting settings → Transcript API access → Microsoft Graph access = On (the reference documented the PowerShell equivalent on the meeting configuration; verify the current parameter name in Microsoft docs). Can take hours to apply. Without it: 403 with code `GraphAccessToTranscriptsDisabled` / "access to transcripts is disabled".
2. **Application access policy** for organizers (Teams PowerShell: create a policy with the app ID, grant it globally or per organizer). Takes up to 30 minutes. Without it: 403 mentioning "application access policy" — **per organizer**, so some organizers can work while others fail.

### Quirks and real-API findings

- **Why per-meeting and not `getAllTranscripts`:** the bulk tenant/user `getAllTranscripts` covers only **scheduled** meetings ("Meet now" missing: in a test a same-day Meet now was 0 in bulk, 1 per-meeting) and, **without an access policy, returns an empty list instead of an error** — indistinguishable from "no transcripts" (2026-10-16).
- **A join link is reused across sessions** (recurring meetings, chat "Meet now" used for days); the transcript/recording list returns items for **all** sessions. Assign each item to the call record whose start–end window (±2 min) contains the item's start time; otherwise every session would count all transcripts.
- 403/404 are **expected outcomes**: log at DEBUG via quiet mode, not as Graph errors.
- The join URL is a credential-like value: the request's query parameters must not be logged.
- Transcript existence ≠ recording: Graph call records show the same recorder bot for both (bot role `customBot`, agent name containing a recorder/communications client marker); only these endpoints separate them. Observed 2026-09-26: a bot joined a meeting that ended with **neither** transcript nor recording — so "bot joined" must never be counted as "recorded".
- Historical (content phase, removed): with speaker attribution disabled in the tenant, `Accept: text/vtt` returned 403 `SpeakerAttributionNotAllowed`; the plain-text transcript media type worked. Kept here only so nobody re-discovers it if content is ever reconsidered.
- 2026-10-16: in the reference tenant all recording lists returned 200 and empty.

## 5 Data model

**Per meeting session** (`transcript_meetings`, PK = call record ID, FK to call records):

| Column | Meaning |
|---|---|
| call_record_id | session (call record) |
| organizer_key | `user:<oid>` |
| meeting_id | Graph online-meeting ID (null if not found) |
| status | `ok` / `meeting_not_found` / `no_policy` / `disabled` / `forbidden` / `error` (language-neutral code) |
| error_code | Graph error code (no message text) |
| transcript_count, first_transcript_at | |
| recording_count | **null = not asked / no permission / failed**; 0 = asked, none |
| first_recording_at, recording_error | |
| attempts, checked_at, next_check_at (null = final), run_id | |

**Transcripts** and **recordings** (PK = Graph item ID): call_record_id, meeting_id, organizer_key, created_at, ended_at, raw (metadata as returned), first_seen_at, run_id. No content columns. Do not follow any content URL found in `raw`; consider stripping content URLs before storing (§14).

RLS on all three: visible if the call record is in the viewer's scope (same rule as calls).

## 6 Ingest / request flow

Runs after the calls ingest in each scheduled run.

1. **Candidates:** internal organizer, has join URL, `groupCall` (meeting), ended more than 5 minutes ago, started within the last **28 days**, and either never checked or `next_check_at` due. Max **400 per run** (≈2–3 requests each).
2. Per candidate (parallel workers, network only): find meeting by join URL → list transcripts → (if recordings feature on) list recordings. A recording failure never changes the transcript result and never erases earlier recording info (keep previous values).
3. Map exceptions to status: transcripts-disabled code/message → `disabled`; 403 with "application access policy" → `no_policy`; other 403 → `forbidden`; 404 → `meeting_not_found`; network/retry exhaustion → `error`.
4. Session matching (±2 min) before saving.
5. **Recheck schedule** (pure function): `ok` and `meeting_not_found` → end + 10 min, + 1 h, + 6 h, + 24 h, then final. Access problems → every 6 h (error: 1 h) until the meeting leaves the 28-day window — fixes itself once the policy is granted.
6. Any access problem marks the run `partial` (so "0 transcripts" is not mistaken for truth).

## 7 Derived logic & metric definitions

- **Meeting set (numerator and denominator identical):** internal-organizer, linked, finished meeting sessions in range.
- **Checked** = status `ok`. **Could not check** = access statuses + not found + error — shown separately, never as 0%.
- **Transcript rate** = sessions with ≥1 transcript ÷ checked; hidden when checked < **3**.
- **Recording rate** = sessions with recording_count ≥ 1 ÷ sessions with recording_count not null.
- **Together:** both / transcript only / recording only / neither; plus "bot joined but nothing produced" (from call-record bot detection).
- Daily, weekly and per-organizer breakdowns use the same definitions.

## 8 UI

Tab **Transcripts and recordings**: top counters (meetings, checked, could not check) · three cards: Transcript (rate + detail), Recording (rate + detail), Together (breakdown) · access problems card: one row per status with count and the remediation text (tenant switch steps; access-policy commands with placeholders `<app id>`; permission names) · two daily charts (transcript, recording) with table equivalents · weekly table · per-organizer table (person-level) · transcript list (person-level; organizer, meeting time, transcript duration — **no text, no read link**).

Call detail chips (only when this module is visible to the viewer and the view is not aggregate): "Transcript produced", "Recorded", "Bot joined, neither recording nor transcript"; otherwise the calls module's generic "Recording / transcript bot joined" chip.

States: permission missing → locked tab card listing `OnlineMeetings.Read.All`, `OnlineMeetingTranscript.Read.All` (+ optional recording permission) and the two tenant prerequisites; recordings permission missing → recording card shows "not available" note; nothing checked yet → note-line; calls module off → module cannot be enabled (settings explain the dependency).

## 9 Access control

- RBAC module `transcripts`; by default only the system admin role.
- RLS: call-in-scope rule on all three tables; organizer table and lists are person-level (masked / removed in aggregate view).
- Audit: tab view. There is **no** content view action (the reference's old "view transcript" audit label may remain only for historic rows).

## 10 Logging events

`transcript.check` (INFO timed; candidates, ok, per-status counts, requests), `transcript.meeting` (DEBUG; status, counts — no join URL), `transcript.disabled` / `transcript.no_policy` / `transcript.forbidden` / `transcript.error` (WARNING, **one per status per run**, not per meeting), `transcript.recordings_failed` (WARNING), `transcript.done` (INFO), `transcript.failed` (ERROR exc), `graph.expected` (DEBUG, from the graph client). Never log join URLs, meeting subjects, or any content URL.

## 11 i18n notes

Status codes stored language-neutral; status names and remediation texts translated at render. PowerShell cmdlet names, portal menu paths in English UI terms (Microsoft's own labels), permission names untranslated.

## 12 Testing & acceptance criteria

- Pure recheck-schedule function: each status and boundary (window exit → final).
- Session matching: one join URL, three sessions, two transcripts → each counted on the right session only.
- 403 disabled / no policy / other, 404, network error → correct status; no ERROR rows for expected 403/404.
- Recording 403 while transcripts OK → transcript result saved, recording_count stays null, previous recording info preserved.
- Rates hidden under 3 checked; "could not check" never folded into 0%.
- No request ever to a `/content` path (assert in fake Graph).
- Join URL absent from logs (leak test).
- Screenshots incl. access-problems card and locked tab.

## 13 Implementation steps for an agent

1. Verify [call-ingest](../call-ingest/DESIGN.md) exposes candidate query fields and the call-in-scope function.
2. Migration: three tables, RLS, role grant to system admin.
3. Features `transcripts` (required pair) and `recordings` (optional); package `transcripts` with dependency on `calls`.
4. Graph functions (find meeting with secret params, list transcripts, list recordings) + status mapping.
5. Pure functions: recheck schedule, session matching.
6. Scheduler step with candidate cap, parallel workers, single-threaded DB writes, logging.
7. Tab UI, call-detail chips, i18n, tests, screenshots.

## 14 Open questions / future

- Change-notification subscriptions (webhooks) for transcripts/recordings instead of polling.
- Show the organizer access-policy gap in the global permissions panel too.
- Strip `recordingContentUrl` / transcript content URLs from stored raw JSON (raw-as-truth rule vs. minimization).
- Content reading stays out of scope; any revival requires an explicit owner decision and a separate privacy design.
