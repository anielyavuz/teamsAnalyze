# Module: User lookup

> Status: **build step 1** (first production module).
> Module key: `user_lookup` · Location in the UI: **Admin → User lookup** · Data: **live, read-only Microsoft Graph; nothing stored except audit + logs.**

---

## 1 Purpose

Let an administrator answer, in one screen and without opening three Microsoft portals:

- "Which licenses does this person have, and which service plans inside each license are on or switched off?"
- "Is a license assigned **directly** or **inherited from a group** (and which group)? Is there an assignment error?"
- "Is the subscription behind the license active, or suspended / in warning? Is it a free or self-service SKU the tenant never purchased?"
- "Which Teams policies are effective for this person — a direct assignment, a group assignment, or the organization default (Global)?"
- "Is the person Enterprise Voice enabled, which Teams feature types do the licenses grant, which phone numbers are assigned?"

Every section must also be able to tell the admin **what the app actually did to get the data** (endpoint, permission, HTTP outcome, duration) and **which Graph permissions the app has or lacks**. The explanation is generated from the same trace the code produced, so what the screen says and what the code did can never drift apart.

This module is intentionally standalone: it ships with only the platform minimum (sign-in, RBAC admin module, logging, permission map, Graph client, audit). It has no database tables of its own and no ingest job.

## 2 Scope / Out of scope

**In scope**

- Search people by display name, UPN or e-mail (prefix search via Graph `$search`).
- Open one person: profile header, licenses with service plans, assignment source and state, subscription status, Teams user configuration (effective policy assignments, feature types, Enterprise Voice, phone numbers, account type).
- Resolve group display names for group-inherited licenses/policies (optional permission; falls back to the group ID).
- Per-section "What did this use?" info box built from the per-request trace.
- "Which permissions does this need?" panel with ✓/✗ derived from the app token's roles.
- Audit entry for every search and every opened person; structured log events without personal query text.
- Friendly product/service-plan names (built-in map, optionally extended from Microsoft's published CSV).

**Out of scope (deliberately)**

- **Any write action**: assigning/removing licenses, assigning policies, enabling Enterprise Voice, assigning numbers. (So `TeamsPolicyUserAssign.ReadWrite.All` and any `*.ReadWrite.*` permission are **not** requested.)
- **Policy settings** (what a given meeting/messaging policy actually allows). Microsoft Graph does not expose Teams policy settings; only Teams PowerShell does (`Get-CsTeamsMeetingPolicy`, etc.). See §14.
- Usage/activity of the person (belongs to the licensing / messaging / copilot modules).
- Bulk export, scheduled sync, caching of lookup results, storing Graph responses.
- Non-admin access, scope (department/manager) filtering, masking — this screen bypasses scope by design and is therefore admin-only (§9).
- Sign-in logs, MFA/authentication methods, group membership lists, mailbox data.

## 3 Dependencies

| Platform doc | What this module needs from it |
|---|---|
| [architecture](../../platform/architecture.md) | Module registration (feature key, package, admin sub-tab), layering rules (Graph access only via the Graph client) |
| [graph-client](../../platform/graph-client.md) | Shared token cache, throttle, retry on 429/503/504 with `Retry-After`, one forced re-auth on 401, **"quiet" mode** where 400/403/404 are returned as typed errors (status, error code, message) and logged at DEBUG as expected outcomes, not as errors; support for custom request headers (`ConsistencyLevel`); a way to read the app token's `roles` claim without a network call |
| [permissions](../../platform/permissions.md) | Feature → permission map with `A|B` alternatives (least privileged first), optional flag, package membership, "granted / missing / not needed" evaluation from token roles |
| [auth-rbac](../../platform/auth-rbac.md) | Entra SSO session, the **admin module** check, CSRF for any POST (this module uses GET only), `audit()` helper writing to the append-only access audit table |
| [logging](../../platform/logging.md) | Structured event logger (`component.event` keys, context as JSON, `timed` helper, secret scrubbing) |
| [i18n](../../platform/i18n.md) | Translatable UI strings, plural forms, number/date formatting helpers |
| [ui-shell](../../platform/ui-shell.md) | Admin page layout, tokens (colors, spacing), card, pill, table, `<details>` disclosure, note-line empty state, light/dark themes, mobile layout |
| [scope-masking](../../platform/scope-masking.md) | Only to document that this module is **outside** the scope/masking pipeline (admin only) |
| [database](../../platform/database.md) | Settings table (package enable flags); optional SKU catalog table (§5) |
| [testing](../../platform/testing.md) | Fake Graph layer, screenshot procedure (light, dark, 390 px), test database procedure |
| [deployment](../../platform/deployment.md) | App registration permissions list, admin consent procedure |
| [ingest-scheduler](../../platform/ingest-scheduler.md) | Not used (no ingest). Only relevant if the optional SKU catalog refresh becomes a scheduled job |

## 4 Microsoft Graph endpoints & permissions

All requests use the app's own identity (client credentials, application permissions). All are `GET`.

### 4.1 Endpoint table

| Step key | Method | Endpoint | Version | Permission (least privileged first) | Alternatives | Why |
|---|---|---|---|---|---|---|
| `search` | GET | `/users?$search=…&$select=…&$orderby=displayName&$top=15&$count=true` + header `ConsistencyLevel: eventual` | v1.0 | `User.Read.All` | `Directory.Read.All` | Find people by name, UPN or e-mail |
| `user` | GET | `/users/{id}?$select=id,displayName,userPrincipalName,mail,jobTitle,department,officeLocation,usageLocation,accountEnabled,userType,createdDateTime,assignedLicenses,licenseAssignmentStates` | v1.0 | `User.Read.All` | `Directory.Read.All` | Profile, assigned licenses, direct vs group source, assignment errors |
| `license_details` | GET | `/users/{id}/licenseDetails` | v1.0 | `User.Read.All` | `Directory.Read.All` | Service plans inside each license and their provisioning status |
| `skus` | GET | `/subscribedSkus?$select=skuId,skuPartNumber,capabilityStatus` | v1.0 | `LicenseAssignment.Read.All` | `Organization.Read.All`, `Directory.Read.All` | Product part number of license IDs, subscription status (Enabled / Warning / Suspended / LockedOut) |
| `teams_config` | GET | `/admin/teams/userConfigurations/{id}` | v1.0 | `TeamsUserConfiguration.Read.All` | — | Effective Teams policy assignments, feature types, Enterprise Voice, phone numbers, account type |
| `group` | GET | `/groups/{id}?$select=displayName` | v1.0 | `GroupMember.Read.All` | `Group.Read.All`, `Directory.Read.All` | Display name of the group a license/policy is inherited from (**optional**) |

### 4.2 Required vs optional permissions (shown in the "Which permissions" panel)

| Requirement | Required? | Used for |
|---|---|---|
| `User.Read.All` \| `Directory.Read.All` | required | Search, profile, assigned licenses, service plans |
| `TeamsUserConfiguration.Read.All` | required | Teams policies, feature types, phone numbers, Enterprise Voice |
| `LicenseAssignment.Read.All` \| `Organization.Read.All` \| `Directory.Read.All` | optional | Product name of a license the person's license list does not name (e.g. a suspended subscription), subscription status |
| `GroupMember.Read.All` \| `Group.Read.All` \| `Directory.Read.All` | optional | Group name when a license or policy comes from a group |

**Explicitly not needed:** `TeamsPolicyUserAssign.ReadWrite.All` (that one *assigns* policies; this module only reads). The panel says so, because admins searching for "Teams policy permission" usually find that one first.

In the permission map (platform `permissions`), register two features:

- `user_lookup` — needs `User.Read.All|Directory.Read.All` **and** `TeamsUserConfiguration.Read.All`; package: see §9.
- `user_lookup_groups` — needs `GroupMember.Read.All|Group.Read.All|Directory.Read.All`; `optional: true`.
- The `skus` step is optional inside `user_lookup` (the screen degrades gracefully); if the licensing module is later enabled, its own feature already requires one of those permissions.

### 4.3 Response shapes (illustrative, trimmed)

`GET /users/{id}` (licensing fields):

```json
{
  "id": "00000000-0000-0000-0000-000000000001",
  "displayName": "Pat Example",
  "userPrincipalName": "pat@contoso.com",
  "usageLocation": "US",
  "accountEnabled": true,
  "userType": "Member",
  "assignedLicenses": [{ "skuId": "<sku-guid>", "disabledPlans": [] }],
  "licenseAssignmentStates": [
    { "skuId": "<sku-guid>", "assignedByGroup": null, "state": "Active", "error": "None",
      "disabledPlans": [], "lastUpdatedDateTime": "2026-10-01T08:00:00Z" },
    { "skuId": "<sku-guid>", "assignedByGroup": "<group-guid>", "state": "Active", "error": "None",
      "disabledPlans": ["<plan-guid>"] }
  ]
}
```

`GET /admin/teams/userConfigurations/{id}`:

```json
{
  "id": "00000000-0000-0000-0000-000000000001",
  "userPrincipalName": "pat@contoso.com",
  "accountType": "user",
  "isEnterpriseVoiceEnabled": false,
  "featureTypes": ["Teams", "AudioConferencing", "PhoneSystem"],
  "telephoneNumbers": [{ "telephoneNumber": "+15555550100", "assignmentCategory": "primary" }],
  "effectivePolicyAssignments": [
    { "policyType": "TeamsMeetingPolicy",
      "policyAssignment": { "displayName": "Contoso Restricted Meetings", "assignmentType": "group",
                            "policyId": "<policy-id>", "groupId": "<group-guid>" } }
  ]
}
```

### 4.4 Quirks and real-API findings

Record these in the module's notes; several contradict or go beyond the documentation.

1. **`$search` requires `ConsistencyLevel: eventual`** as a request header **and** `$count=true`; without them Graph returns 400. `$orderby=displayName` works only in the same advanced-query mode.
2. **`$search` is word-prefix tokenized.** The query matches the *start of tokens* in the property, not substrings. Example: searching `Sales` finds "Sales Desk" and "Pat Salesman", but **not** a display name written as a single concatenated word such as "ContosoSales". The documentation suggests tokens also split on casing changes; in the reference implementation (November 2026) a camel-case concatenation did **not** match on its second part. The UI hint must say "start of a name or address" and give an example that works.
3. **Search syntax**: each clause is `"property:term"`, combined with `OR`: `"displayName:{q}" OR "userPrincipalName:{q}" OR "mail:{q}"`. Double quotes and backslashes in user input must be removed before building the clause (they break or escape the expression). Input length capped at 100 characters, minimum 2.
4. **`effectivePolicyAssignments` lists only direct and group assignments.** A policy type that is absent means the person gets the **organization default (Global)** for that type. The UI must therefore render a fixed list of main policy types and fill missing ones as "Global / organization default" — otherwise the admin reads "no meeting policy".
5. **Policy *settings* are not in Graph.** Only the assignment (name, ID, source) is returned. Settings require Teams PowerShell.
6. **`/admin/teams/userConfigurations/{id}` returns 404** when the person has no Teams license or Teams has not provisioned them yet (new users can take a while). This is a normal state, not an error.
7. **Suspended subscriptions hide service plans.** When a subscription's `capabilityStatus` is `Suspended` (or `LockedOut`), `licenseDetails` no longer lists that SKU's service plans although `assignedLicenses` still contains it. Name the SKU from `subscribedSkus` and explain the missing plans.
8. **Free / self-service SKUs are absent from `subscribedSkus`** (examples: `TEAMS_FREE`, `TEAMS_EXPLORATORY`, `FLOW_FREE`, `POWER_BI_STANDARD`). They do appear in `assignedLicenses` and usually in `licenseDetails` (which carries `skuPartNumber`). A built-in map of their well-known Microsoft SKU IDs gives the part number when `licenseDetails` does not. Label them "not a purchased subscription (free / self-service)".
9. **`licenseAssignmentStates` may contain a SKU that is no longer in `assignedLicenses`** (removal or group processing in progress). Show it with a note, not as an active license.
10. **The same SKU can have several assignment states** (e.g. direct *and* via one or more groups). Show one source line per state.
11. **A newly granted permission is not visible until the token renews** (up to about an hour) or the service restarts; the ✓/✗ panel reads the cached token.
12. **Without `usageLocation`, licenses cannot be assigned**; flag it in the header.

## 5 Data model

**No module tables.** Lookups are live: nothing from Graph is written to the database and nothing is cached across requests (the same person opened twice makes the requests twice). This avoids a second copy of directory data with its own retention and RLS obligations.

What *is* persisted, via platform facilities:

| Where | What | Notes |
|---|---|---|
| Access audit (platform `auth-rbac`) | action `user_search` with result count; action `user_lookup` with target UPN (or object ID if unresolved) and the per-step outcome map (`{step: status}`) | Append-only. Query text is **not** stored |
| App log (platform `logging`) | events in §10 | No query text, no names, no UPNs |
| Settings (platform `database`) | package enabled flag | |

### 5.1 In-memory structures (per request)

**Trace item** — one per Graph call:

| Field | Meaning |
|---|---|
| step | step key from §4.1 (`search`, `user`, `license_details`, `skus`, `teams_config`, `group`) |
| status | `ok` / `denied` (403) / `not_found` (404) / `error` (anything else, after retries) |
| http | HTTP status code (200 on success) |
| ms | wall-clock duration in milliseconds, including retries |
| count | number of results when meaningful (search hits, plan list, SKU list), else empty |

**Step catalog** (static): step key → method, endpoint template (with `{id}` and `…`, never real values), permission requirement string `A|B|C`, translatable purpose sentence. The info box and the permissions panel are both rendered from this catalog, so text and behavior stay in sync.

**License row** (derived, one per SKU appearing in any of `assignedLicenses`, `licenseAssignmentStates`, `licenseDetails`):

| Field | Source |
|---|---|
| sku_id | any of the three lists |
| part_number | `licenseDetails` > `subscribedSkus` > built-in free-SKU ID map |
| friendly_name | friendly-name map (§5.2) by part number, else empty |
| sources | list of `{group_id or null, state, error, last_updated, disabled_plan_count}` from `licenseAssignmentStates` |
| in_assigned | SKU is in `assignedLicenses` |
| in_details | SKU is in `licenseDetails` |
| subscription_status | `capabilityStatus` from `subscribedSkus`; *absent* = not a purchased subscription; *unknown* = `skus` step failed |
| plans | `servicePlans` from `licenseDetails` (`servicePlanName`, `provisioningStatus`, `appliesTo`), Teams-related first |

**Policy row** (derived, fixed main types first, then any extra types Graph returned, sorted):

| Field | Meaning |
|---|---|
| type | Graph `policyType` (e.g. `TeamsMeetingPolicy`) |
| name | `displayName` → else `policyId` → else "?"; "Global" for missing types |
| source | `direct` / `group` / `global` (missing type) |
| group_id | when source is group |

Main policy types (always rendered, translated label in parentheses): `TeamsMeetingPolicy` (Meetings), `TeamsMessagingPolicy` (Messaging), `TeamsCallingPolicy` (Calling), `TeamsAppSetupPolicy` (App setup), `TeamsAppPermissionPolicy` (App permission), `TeamsChannelsPolicy` (Teams and channels), `TeamsEventsPolicy` (Events: webinars, town halls), `TeamsMeetingBroadcastPolicy` (Live events), `TeamsUpdateManagementPolicy` (Update management), `TeamsFeedbackPolicy` (Feedback), `TeamsVoiceApplicationsPolicy` (Voice applications), `TeamsCallParkPolicy` (Call park), `CallingLineIdentity` (Caller ID), `OnlineVoiceRoutingPolicy` (Voice routing), `TenantDialPlan` (Dial plan), `TeamsEmergencyCallingPolicy` (Emergency calling).

### 5.2 Friendly names for SKUs and service plans

- **Built-in map (ships with the module):** part number → product name for the common Teams-relevant SKUs (Teams Essentials, Teams Exploratory, Teams free, Teams Rooms Basic/Pro, Teams Premium, Teams Phone Standard, Audio Conferencing, Calling Plans, Phone Resource Account, Microsoft 365 Copilot, Business Basic/Standard/Premium, Office 365 E1/E3/E5, Microsoft 365 E3/E5/F3, Office 365 F3, Power Automate Free, Fabric Free). Product names are brand names and are **not translated**.
- **Teams service plan map:** plan name → readable label for Teams-related plans (`TEAMS1`, `TEAMS_FREE`, `MCOIMP`, `MCOSTANDARD`, `MCOEV`, `MCOMEETADV`, `MCOPSTN1`, `MCOPSTN2`, `MCOPSTNC`, `MCO_VIRTUAL_APPT`, `TEAMSPRO_*`, `MCO_TEAMS_IW`, `TEAMS_ADVCOMMS`, `Teams_Room_*`, `MESH_IMMERSIVE_FOR_TEAMS`). A plan counts as Teams-related if it is in this map, or its upper-cased name starts with `TEAMS` or `MCO`, or contains `TEAMS`.
- **Free SKU ID map:** well-known Microsoft SKU IDs → part number for SKUs that never appear in `subscribedSkus`: `16ddbbfc-09ea-4de2-b1d7-312db6112d70` = `TEAMS_FREE`, `710779e8-3d4a-4c88-adb9-386c958d1fdf` = `TEAMS_EXPLORATORY`, `a403ebcc-fae0-4ca2-8c8c-7a907fd6c235` = `POWER_BI_STANDARD`, `f30db892-07e9-47e9-837c-80727f46fd3d` = `FLOW_FREE`.
- **Optional extended catalog:** Microsoft publishes "Product names and service plan identifiers for licensing" (a documentation page on Microsoft Learn with a downloadable CSV; columns: Product_Display_Name, String_Id, GUID, Service_Plan_Name, Service_Plan_Id, Service_Plans_Included_Friendly_Names). If the owner wants complete names, add an admin-triggered import (or a build-time bundled copy) into a small catalog table: `sku_id`, `string_id` (part number), `product_display_name`, `service_plan_id`, `service_plan_name`, `service_plan_friendly_name`, `source_file_date`, `imported_at`. Lookup order: catalog → built-in map → technical name. The runtime must **never** fetch the CSV from the internet on a page view. Unknown names always fall back to the technical name; never block rendering.

## 6 Ingest / request flow

No ingest. Two request flows, both `GET` on the admin page (`tab=lookup`).

### 6.1 Search (`q` present, no `id`)

1. Session + admin module check (platform). Feature `user_lookup` must be enabled (package on); otherwise render the "module off" note and stop.
2. Normalize `q`: trim, cut to 100 characters; if shorter than 2 characters, show only the form.
3. Read granted roles from the cached app token (no network call). Failure → empty set + `lookup.roles_unknown` warning; ✓/✗ shows all as missing.
4. One Graph call `search` with `$top=15` through the trace. Result fields: `id, displayName, userPrincipalName, jobTitle, department, accountEnabled, userType`.
5. Audit `user_search` (results count; no query text).
6. Render results list (each row links to `?tab=lookup&id=<object id>&q=<q>` so "Back to results" works).

### 6.2 Open a person (`id` present)

1. Same gate as 6.1. Validate `id` as a GUID (36 chars, hex and dashes); otherwise ignore it (treat as absent).
2. Step `user` (sequential, because every later step needs a confirmed object ID). If it fails → render the failure state (§8) with the info box of the `user` step, audit, stop.
3. In parallel (max 3 workers): `license_details`, `skus`, `teams_config`. Each goes through the trace independently; one failure never stops the others.
4. Build the part-number map: free-SKU ID map, then `subscribedSkus`, then `licenseDetails` (later overrides earlier). Build the subscription-status map from `subscribedSkus` (or "unknown" if that step failed).
5. Collect group IDs from `licenseAssignmentStates[].assignedByGroup` and `effectivePolicyAssignments[].policyAssignment.groupId`. Resolve at most **20** groups sequentially via `group`. On the **first 403**, stop resolving (the permission is missing; further calls would also fail) and show IDs.
6. Derive license rows and policy rows (§7).
7. Audit `user_lookup` with target UPN (fallback: object ID) and `{step: status}` map.
8. Render header, licenses section, Teams policies section, each with its own info box.

Request budget per opened person: 4 calls + up to 20 group calls (typical: 4–6). Expected latency: dominated by the slowest of the three parallel calls.

## 7 Derived logic & metric definitions

### 7.1 License row status text (first matching rule wins for the extra note)

| Condition | Display |
|---|---|
| subscription status *absent* (SKU not in `subscribedSkus`, step succeeded) | grey note "not a purchased subscription (free / self-service)" |
| subscription status not `Enabled` and not unknown (`Warning`, `Suspended`, `LockedOut`) | red pill "subscription {status}" + note "The tenant's subscription is not active, so this license gives no services; Microsoft no longer lists its service plans." |
| SKU present only in `licenseAssignmentStates`, not in `assignedLicenses` | grey note "Listed in the assignment history but not currently assigned (being removed or processed)." |

### 7.2 Assignment source line (one per assignment state)

- Pill with `state` (`Active`, `ActiveWithSolution`, `Disabled`, `Error`…); pill is "bad" when state is not `Active` **or** `error` is set and not `None`.
- "direct" when `assignedByGroup` is null, else "via group {name or ID}".
- Assignment error text (e.g. `CountViolation`, `MutuallyExclusiveViolation`, `DependencyViolation`, `ProhibitedInUsageLocationViolation`, `UniquenessViolation`) shown in red when not `None`.
- "{n} plan(s) switched off" when `disabledPlans` is non-empty.
- No state at all → "source unknown".

### 7.3 Service plans

- Status class from `provisioningStatus`: `Success` → ok (green); `Disabled` → off (grey); `PendingActivation`, `PendingInput`, `PendingProvisioning` and anything else → warn (amber).
- Teams-related plans shown in a visible table; all other plans collapsed under "{n} other service plan(s)".
- Sorting: Teams-related first, then by plan name.

### 7.4 Teams policies

- Policy source: `assignmentType` lower-cased (`direct` / `group`); missing type → `global`.
- Summary line: "All policy types use the organization default (Global): no direct or group assignment." or "{n} policy type(s) have a direct or group assignment; the rest use the organization default (Global)."
- Feature types rendered as chips; Enterprise Voice "on/off"; phone numbers comma-separated, "—" if none; account type as returned.

### 7.5 Header flags

- `accountEnabled = false` → red pill "account disabled".
- `userType = Guest` → pill "guest".
- `usageLocation` empty → red pill "no usage location", tooltip "Licenses cannot be assigned without a usage location".
- Created date formatted with the platform date formatter (UTC → configured app time zone).

### 7.6 Info box ("What did this use?")

For a list of step keys, one entry per step:

- Line 1: `METHOD endpoint-template` in monospace.
- Line 2: translated purpose sentence.
- Line 3: "Permission:" cell rendered with the **permission cell rule** in §8.2.1 (same rule as the permissions panel).
- Line 4 (only if the step ran in this request): outcome text (`OK` / `permission missing (403)` / `not found (404)` / `error`), duration "N ms", "×k" when the step ran k times (groups), and "· N results" when a count exists and the step succeeded. If any run of the step failed, show the first failure instead of the last success.

Sections and their steps: Results → `search`; Licenses → `user`, `license_details`, `skus`, `group`; Teams policies → `teams_config`, `group`; person not readable → `user`.

## 8 UI

### 8.1 Screen layout (Admin → User lookup sub-tab)

1. **Top row**: search form (label "Name, UPN or e-mail", input `type=search`, `minlength=2`, `required`, `autocomplete=off`, placeholder like "e.g. Pat or pat@contoso.com", button "Search") and, right-aligned, the **"Which permissions does this need?"** disclosure.
2. One-line description: "Shows the person's licenses (with service plans, direct or group assignment) and effective Teams policies, live from Microsoft 365."
3. **Results** section (after a search): heading + info box; list of clickable rows: display name (bold) · UPN · muted "job title · department" plus "disabled" / "guest" markers. Three columns on desktop, stacked on mobile.
4. **Person view** (after opening): "← Back to results" link when `q` is present; header card (name, UPN, job/department/office, usage location, created, flags); **Licenses (n)** section with one card per license (title = friendly name or part number or SKU ID; subtitle = part number or SKU ID in monospace; right side = source lines and notes; Teams plan table; collapsed other plans); **Teams policies** section with a key-value card (feature chips, Enterprise Voice, phone numbers, account type), the summary note-line, and the policy table (Policy type + technical type in small monospace · Effective policy · Source). Global rows are dimmed.

### 8.2 Permissions transparency UI — MUST be reproduced exactly

The owner requires this UI to be reproduced **exactly** in the new environment. Everything in §8.2 is mandatory, not a suggestion.

#### 8.2.0 Popover mechanics (shared by A and B)

- A button-like summary chip with a round "i" info icon + label; clicking opens a popover. Implemented with native `<details>`/`<summary>` — **no JavaScript required**; keyboard accessible by default.
- The popover **overlays** the content (absolutely positioned, right-aligned to its button, width ≈ **720 px** max, card background, border, **card/flyout shadow**). It does not push the page down.
- On phone width (≤ 700 px) the popover becomes **full-width inside 16 px gutters** (left/right 16 px), still overlaying.
- Colors, radius and shadow come from platform tokens only.

#### 8.2.1 Permission cell rendering rule (used in A's table and in every B entry)

1. The **least-privileged option is always written first**.
2. Alternatives follow in **muted, smaller text** as "(alternative: X or Y)".
3. **Every option the app actually holds gets a green ✓ next to it.**
4. The row/cell gets a **leading ✓** (green) if **any** option is held, otherwise a **leading red ✗**.
5. If the first (recommended) option is **not** held but an alternative is, the first option is shown **greyed** (the leading ✓ is still green because an alternative is held).
6. "Held" is read from the current access token's `roles` claim (no extra request).

#### 8.2.2 A — "Which permissions does this need?" (top right of the page)

Placed at the top right, next to the search form. Popover contents, **in this order**:

1. One-line statement: "Every lookup is a live, read-only Microsoft Graph request made with the app's own identity (client credentials). Nothing is written to the database or changed in Microsoft 365."
2. Table with columns **Application permission | required/optional | Used for**, one row per requirement, permission cell per §8.2.1:

| Application permission | | Used for |
|---|---|---|
| `User.Read.All` (alternative: `Directory.Read.All`) | required | Search people, profile, assigned licenses and service plans |
| `TeamsUserConfiguration.Read.All` | required | Teams policy assignments, Teams features, phone numbers |
| `LicenseAssignment.Read.All` (alternative: `Organization.Read.All` or `Directory.Read.All`) | optional | Product name of a license that the person's license list does not name |
| `GroupMember.Read.All` (alternative: `Group.Read.All` or `Directory.Read.All`) | optional | Group name when a license or policy comes from a group |

3. **How to grant:** Azure Portal → Microsoft Entra ID → App registrations → this app → API permissions → Add a permission → Microsoft Graph → Application permissions → select the permission → Grant admin consent.
4. **Legend:** "✓ = this app already has it (read from the current access token's roles claim, no extra request), ✗ = missing. A new permission appears after the token renews (up to about an hour) or after the service restarts."
5. **Not needed:** "`TeamsPolicyUserAssign.ReadWrite.All` (that one assigns policies — a write permission; this screen only reads). Policy settings themselves are not in Microsoft Graph; they are only available in Teams PowerShell (`Get-CsTeamsMeetingPolicy` …)."
6. **Who can use it:** "Only people with the Admin module. The lookup bypasses the report's department/manager scope, so it is not offered as a report tab. Every search and lookup is written to the access log."

The table rows are generated from the step catalog / permission map, not typed separately, so they cannot drift from what the code requests.

#### 8.2.3 B — "What did this use?" (per section)

- Same popover style, placed at the right of the section heading of **Results**, **Licenses** and **Teams policies** (and on the "person could not be read" message, for the `user` step).
- Built from the **real per-request trace of this page load** (§5.1, §7.6). For each step:
  - `METHOD endpoint` in monospace (endpoint template; never real IDs or query text),
  - one-line purpose,
  - "Permission:" cell per §8.2.1,
  - the **actual outcome**: `OK` / `permission missing (403)` / `not found (404)` / `error`, duration in **ms**, **×N** when the step ran N times (group lookups), and the **result count** when there is one.
- A step that is in the section's list but did not run in this request shows endpoint, purpose and permission only (no outcome line).
- **Never hard-code** outcomes or descriptions: what is described must equal what was executed.

### 8.3 Components

- Info box: native `<details>` with a summary chip ("i" badge + label). The body is an absolutely positioned flyout (max 720 px, right-aligned) on desktop and a full-width block (16 px gutters) on mobile. Keyboard accessible by default; no JavaScript required.
- Pills (ok / bad / neutral), status chips for plan provisioning (ok / off / warn), feature chips — all from platform tokens; no hard-coded colors.
- Use the platform prefix convention for class names (e.g. `ul-*`) to avoid collisions.

### 8.4 States

| State | Trigger | Rendering |
|---|---|---|
| Module off | feature/package disabled | single note-line: "User lookup is turned off (Admin → Settings)." No form |
| Idle | no `q`, no `id` | form + description only |
| Query too short | `q` < 2 chars | form only (browser validation also prevents submit) |
| No results | search ok, 0 hits | note-line "No one matched "{q}"." plus hint "Search matches the start of a name or address." |
| Search 403 | `search` denied | red message "Search failed: permission missing (403)" + info box showing ✗ `User.Read.All` |
| Search other error | 400/5xx after retries | red message "Search failed: error" + info box |
| Person 404 | `user` not found | red message "The person could not be read: not found (404)" + `user` info box |
| Person 403 | `user` denied | same pattern with "permission missing (403)" |
| No licenses | user ok, all three license lists empty | note-line "No licenses are assigned to this person." |
| licenseDetails 403 | `license_details` denied | red message "Licenses could not be read: permission missing." |
| subscribedSkus failed | `skus` denied/error | licenses still render; part numbers come from licenseDetails/free map; subscription notes omitted (status "unknown"); info box shows ✗ |
| Suspended / Warning / LockedOut subscription | `capabilityStatus` ≠ Enabled | red pill + explanation; plan table empty is expected |
| Free / self-service SKU | not in subscribedSkus | grey note, still listed with plans if licenseDetails has them |
| Assignment error | `error` ≠ None or state ≠ Active | red pill + error code |
| Being removed | only in assignment states | grey note |
| No usage location | empty `usageLocation` | red header pill with tooltip |
| Disabled account / guest | flags | header pills |
| Teams not provisioned | `teams_config` 404 | red message "Microsoft Teams has no configuration for this person (no Teams license, or not yet provisioned)." |
| Teams config 403 | `teams_config` denied | red message "Teams policies could not be read: the app needs TeamsUserConfiguration.Read.All." |
| Teams config other error | 5xx etc. | "Teams policies could not be read (error)." |
| All Global | no assignments | summary note says all types use Global; table still shows all main types dimmed |
| Group names missing | `group` 403 | group IDs shown in monospace; info box shows ✗ for the group permission and the outcome of the first (failed) call |
| Token roles unknown | token read failed | ✗ for all permissions; lookups still attempted |

All error messages are one line; no large empty boxes (platform rule).

## 9 Access control

- **RBAC:** reachable only by users with the platform **admin module**. It is a sub-tab of the Admin area, not a report tab, and is not grantable to report roles. Reason: the lookup reads the whole directory live and therefore **bypasses scope (RLS) and masking**; there is no safe way to scope a live directory search.
- **Feature/package key:** `user_lookup` (+ optional `user_lookup_groups`). In step 1 it ships as its own package (`user_lookup`) so it can be enabled without any other module. When the licensing module ships later, the owner may move both features into the licensing package (open question §14). If the package is off: no form, no Graph calls, permission shown as "not needed".
- **Scope / masking:** not applied (admin only). The page must **not** reuse report rendering helpers that assume masking, and must escape every Graph-sourced string (names, UPNs, group names, policy names, phone numbers) with HTML escaping.
- **Input validation:** `id` must match a GUID pattern; `q` capped at 100 chars; quotes/backslashes stripped before building `$search`.
- **Audit actions** (platform audit helper, module = admin):

| Action | Context fields | Never |
|---|---|---|
| `user_search` | `results` (count), client IP per platform convention | the query text |
| `user_lookup` | `target` (UPN of the opened person; object ID if unreadable), `steps` (`{step: status}`), IP | response contents |

- Translatable audit labels: "Searched users", "Looked up a user's licenses/policies".
- GET only; no state change, so no CSRF token required. Rate limiting of admin GETs is a platform concern (open question).

## 10 Logging events

Component: `lookup`. All keys lower-case ASCII.

| Event key | Level | When | Context |
|---|---|---|---|
| `lookup.search` | INFO (timed) | each search | `results`, `qlen` (length only), `status`, `duration_ms` |
| `lookup.user` | INFO (timed) | each opened person | `found` (bool), `licenses` (count), `policies_assigned` (count of non-Global), `steps` (`{step: status}`), `requests` (count), `target_id` (object ID only), `duration_ms` |
| `lookup.step_failed` | WARNING | a step ended in `error` (not 403/404) | `step`, `status`, `code` (Graph error code) |
| `lookup.roles_unknown` | WARNING | token roles could not be read | exception |
| `graph.expected` | DEBUG | quiet 400/403/404 (from graph client) | method, path template, status, code |

**Never log:** the search text, display names, UPNs, e-mail addresses, phone numbers, group names, policy names, tokens. Expected 403/404 must not create ERROR rows (the graph client's quiet mode). Add a leak test that runs searches with a recognizable marker string and asserts the marker does not appear anywhere in the log table.

## 11 i18n notes

- All UI strings translatable (EN source, TR translation), including step purposes, status texts, policy type labels, notes and the permissions panel.
- Plural forms for "{n} result(s)", "License(s) ({n})", "{n} plan(s) switched off", "{n} other service plan(s)", "{n} policy type(s) have…".
- **Not translated:** Microsoft product names, service plan names, policy type technical names, Graph enum values shown as data (`Active`, `Suspended`, `Success`, `CountViolation`), permission names, endpoint paths.
- Numbers (ms, counts) and dates via platform formatters.
- The search placeholder example must use a contoso-style address and a name with only ASCII, and the "no results" hint must explain prefix matching in both languages.

## 12 Testing & acceptance criteria

**Unit (pure functions, no network):**

- License derivation: direct + group on same SKU; SKU only in assignment states; suspended SKU (in assigned, missing in details, status Suspended); free SKU (not in subscribedSkus, part number from free map); `skus` failed (status unknown, no subscription notes); disabled plans count; error code rendering.
- Policy derivation: empty assignments → 16 Global rows; one direct + one group; an extra policy type outside the main list is appended sorted; missing displayName falls back to policyId.
- Teams-plan classification rule.
- Search clause builder strips `"` and `\`, caps length.
- Info box: picks first failure over last success; "×k"; count only on success.

**Integration (fake Graph layer from platform testing):**

- Search sends `ConsistencyLevel: eventual` and `$count=true`; 400 without them is simulated to prove the header is present.
- 403 on `teams_config` does not prevent licenses from rendering; 404 renders "not provisioned".
- Group resolution stops after the first 403 (exactly one group call made with 25 group IDs present); never more than 20 calls.
- 429 with `Retry-After` on `license_details` is retried and the trace duration includes the wait.
- Invalid `id` (non-GUID) makes zero Graph calls.
- Package off → zero Graph calls.

**Security / privacy:**

- Non-admin user → 403/redirect per platform; no Graph call made.
- Audit row written for every search (no query text column populated) and every lookup (target present).
- Log leak test (§10).
- HTML-escaping test with a display name containing `<script>` and quotes.

**Permissions transparency acceptance (MUST, §8.2):**

- Panel A opens without JavaScript, overlays content (does not shift layout), is right-aligned ≈720 px on desktop and full-width within 16 px gutters at 390 px; its six parts appear in the specified order with the four rows exactly as listed.
- Permission cell rule, tested with token-role fixtures: (a) none held → leading red ✗, first option normal, alternatives muted; (b) only `Directory.Read.All` held → leading green ✓, first option greyed, green ✓ next to `Directory.Read.All`; (c) least-privileged held → leading ✓ and ✓ next to it; (d) several held → ✓ next to each.
- Panel B on Results, Licenses and Teams policies lists exactly the steps executed in that request with their real outcome, ms, ×N and count: with a fake Graph returning 403 for groups after 1 call and 404 for Teams config, the boxes show "permission missing (403)" for `group` (×1) and "not found (404)" for `teams_config`; changing the fake changes the boxes with no code change.
- No outcome text or permission state is hard-coded (code review check: all rows come from the step catalog + trace + token roles).

**UI acceptance:** screenshots in light, dark and 390 px width for: idle, results, no results, full person (direct + group license, suspended, free SKU, assignment error, no usage location), Teams not provisioned, Teams 403, group 403, permissions panel open, info box open (desktop flyout and mobile block). No horizontal page scroll at 390 px.

**Done when:** an admin can search by name prefix, UPN or e-mail, open a person, and every state in §8.4 renders as specified; every Graph call is visible in an info box with the correct ✓/✗; logs and audit match §9–§10.

## 13 Implementation steps for an agent

1. Confirm platform minimum exists: SSO, admin module check, audit helper, logger, permission map, Graph client with quiet mode + custom headers + token roles reader. If quiet mode or token-roles reading is missing, add them in the Graph client (not in this module).
2. Register features `user_lookup` and `user_lookup_groups` in the permission map (needs, optional flag, purpose) and the package `user_lookup`.
3. Add Graph client functions: `search_users(text, top)`, `get_user_licensing(id)`, `get_license_details(id)`, `list_subscribed_skus_min()` (select `skuId,skuPartNumber,capabilityStatus`), `get_teams_user_config(id)`, `get_group_name(id)` — all via the quiet request path.
4. Implement the step catalog and the trace (step, status, http, ms, count).
5. Implement pure derivation functions: part-number map, subscription map, license rows, policy rows, Teams-plan classifier. Add the built-in name maps (§5.2).
6. Implement the flows `search(q)` and `lookup(id)` (§6), with the 3-way parallel step and the capped/short-circuit group resolution.
7. Add logging events (§10) and verify they appear; run the leak test.
8. Add the admin sub-tab route (GET), input validation, package gate, audit calls.
9. **MUST — permissions transparency UI (§8.2):** implement the shared popover (native `<details>`, overlay, ≈720 px right-aligned, card shadow, full-width with 16 px gutters on phone), the permission cell rendering rule (§8.2.1), panel A with its six parts in order, and panel B on Results / Licenses / Teams policies driven only by the request trace. Use the platform's shared permission-panel component if [permissions](../../platform/permissions.md) provides it; otherwise build it there (not inside this module) so other modules can reuse it.
10. Render: form, results, person header, licenses, policies, all states in §8.4. Use platform tokens and component classes only.
11. Add all strings to i18n (EN + TR), plurals, check no untranslated literals.
12. Write unit + fake-Graph integration tests (§12).
13. Take screenshots (light/dark/390 px) of every state; fix layout issues.
14. Update the deployment doc's permission list (two required, two optional) and the module notes with the quirks in §4.4.
15. Security review of the route (admin gate, escaping, audit) before going live.

## 14 Open questions / future

- **Reusable permission transparency pattern:** the "Which permissions does this need?" panel, the permission cell rendering rule and the trace-driven "What did this use?" box (§8.2) should be reusable by every module that calls Graph. [platform/permissions.md](../../platform/permissions.md) defines the shared component (inputs: requirement list from the permission map, token roles, optional trace); this module is its first consumer.

- **Package placement:** keep `user_lookup` as its own package, or fold it into the licensing package once that module ships (the reference had it inside "License analytics")?
- **Policy settings via a PowerShell sidecar (optional):** a separate, isolated worker running Teams PowerShell with certificate-based app auth could return settings for the effective policy (`Get-CsTeamsMeetingPolicy -Identity <name>` etc.). Would need: a separate service boundary, read-only Teams admin role for the app, cached per policy (not per user), clear "from PowerShell" provenance in the info box, and its own permission/role checks in the panel. Not planned until requested.
- **Write actions** (assign policy, assign license, fix usage location): out of scope. If ever added they need a different permission set, a confirmation flow, CSRF, and an audit with before/after — design separately.
- **Extended name catalog:** bundle Microsoft's CSV at build time vs. admin upload; how often to refresh.
- **Cross-links:** from licensing/copilot person rows to this screen for admins only (must not leak the link to non-admins).
- **Rate limiting** of admin GET endpoints (platform-wide).
- **Search on more properties** (`mailNickname`, `employeeId`, phone) — check `$search` support per property before adding.
- **Show license assignment dates** per SKU (not available on these endpoints; the Microsoft 365 active-user report has assignment dates per service — would couple this module to the licensing ingest).
