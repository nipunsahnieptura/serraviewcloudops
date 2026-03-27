---
name: cloudops-serraview-triage
description: "Intelligent triage and assignment of Jira Change Management (CM) tickets for the CloudOps Serraview team. Automatically routes unassigned tickets based on domain, severity, team workload, and routing rules, then transitions them to Approved. Transitions already-assigned tickets to Approved without reassigning. Use when asked to triage Serraview CM tickets, run ticket triage, process filter 55922, assign or approve Change Management tickets for the Serraview team, or balance workload across Serraview engineers."
---
# Serraview CloudOps Ticket Triage

## Jira API Configuration

```yaml
Base URL: $SV_JIRA_BASE_URL (env var)
Auth: Basic base64(SV_JIRA_EMAIL:SV_JIRA_API_TOKEN)
Transition ID for Approve: "31"
Filter ID: 55922
```

Use Jira REST API directly via curl or Python requests — do NOT use MCP tools.
Read `.agents/skills/serraview-cloudops-triage/references/team-config.md` for team roster, routing rules, workload balancing, and ticket analysis signals.
Read `.agents/skills/serraview-cloudops-triage/references/notifications.md` for Teams webhook URL, recipients, and payload format.

## REST API Patterns

**Always use `/rest/api/3/`** — do not fall back to `/rest/api/2/`. A 410 response indicates a network/proxy error, not an API version mismatch. Retry with exponential backoff; do not switch API version.

```bash
# Auth header
AUTH=$(echo -n "$SV_JIRA_EMAIL:$SV_JIRA_API_TOKEN" | base64)
HEADER="Authorization: Basic $AUTH"

# Fetch filter 55922 metadata (to extract the Serraview category JQL condition)
# Correct URL: GET /rest/api/3/filter/{id}  — NOT /filter/{id}/jql
curl -s -H "$HEADER" -H "Content-Type: application/json" \
  "$SV_JIRA_BASE_URL/rest/api/3/filter/55922" | jq .jql

# Search tickets (always POST to /rest/api/3/search/jql — not GET /rest/api/3/search)
curl -s -H "$HEADER" -H "Content-Type: application/json" \
  -X POST "$SV_JIRA_BASE_URL/rest/api/3/search/jql" \
  -d '{"jql":"filter=55922","fields":["summary","assignee","priority","description","comment","labels","customfield_15699"],"maxResults":200,"startAt":0}'

# Transition issue
curl -s -H "$HEADER" -H "Content-Type: application/json" \
  -X POST "$SV_JIRA_BASE_URL/rest/api/3/issue/CM-XXXXX/transitions" \
  -d '{"transition":{"id":"31"}}'

# Assign issue
curl -s -H "$HEADER" -H "Content-Type: application/json" \
  -X PUT "$SV_JIRA_BASE_URL/rest/api/3/issue/CM-XXXXX/assignee" \
  -d '{"accountId":"ACCOUNT_ID"}'

# Add label
curl -s -H "$HEADER" -H "Content-Type: application/json" \
  -X PUT "$SV_JIRA_BASE_URL/rest/api/3/issue/CM-XXXXX" \
  -d '{"update":{"labels":[{"add":"ClopsManualTriage"}]}}'
```

## Workflow

### Step 1: Get Team Availability

Check the `onLeave` parameter. Remove those members from the available assignee pool for this run.

Check `firstRunOfDay` parameter (default: **`false`**). Only if `true` will stale ticket detection run and stale tickets be included in notifications. This prevents duplicate stale alerts when the skill runs multiple times per day. If `firstRunOfDay` is not passed, assume `false` — stale detection is skipped entirely.

Determine the **current IST time** at run start. This is required for shift-aware routing (Step 4):
- Vikas Kumar is available only 04:00–12:00 IST
- Michael Ola Soga Jr. handles Serraview S1/S2 only during 19:00–05:00 IST (overnight)
- Michael handles Boeing tickets at any time regardless of shift

### Step 2: Query Current Workload

First, fetch filter 55922 to confirm the Serraview category condition:

```bash
curl -s -H "$HEADER" "$SV_JIRA_BASE_URL/rest/api/3/filter/55922" | jq .jql
```

The Serraview category condition extracted from this filter is:

```
"Category and Sub-category[Select List (cascading)]" IN cascadeOption("Serraview")
```

Use this exact string in all workload queries below. If the filter JQL returns a different condition for Serraview (e.g. a `cf[XXXXX]` field ID variant), extract and substitute that exact string — do not guess or reformulate it.

**Query 1 — Active workload** (excludes Done, Cancelled, and Under Observation status):

```
project = CM
AND assignee IN ("712020:e779f9ea-49c9-4573-8a3a-61a6264cd283", "639af1b47145571a7ea882d7", "6362e6fe59c794184bcc1a3e", "712020:b217ad2b-f35c-41e1-8ec5-73d5d58952d0", "712020:7f06dd3b-4d20-4e02-bb38-b3ff9ea66c64", "64238bb20152b5f4f9f2e7f9", "712020:4962c34a-ee1a-427c-aced-015675053cae", "62b8f3e8118b20bee2ba7228", "712020:2f76ab05-db2b-4d65-b0d0-9568aff61366")
AND status NOT IN (Done, Cancelled, "Under Observation")
AND <serraview-category-condition-from-filter-55922>
```

**Query 2 — Under Observation tickets** (decide inclusion individually):

```
project = CM
AND assignee IN (...same accountIds...)
AND status = "Under Observation"
AND <serraview-category-condition>
```

For each "Under Observation" ticket returned, fetch the last 3 comments:
- If the last comment (from anyone) contains a pending update request (`please update`, `any update`, `update?`, `following up`, `looking for update`, `any progress`, `status update`, `can you update`, `please respond`, `waiting for update`, `need an update`, `please provide`) → **include in workload count** (requester is waiting on a response)
- Otherwise → **exclude from workload count**; add to that person's `obsCount` for the `(+X obs)` display suffix in notifications

Final workload count per person = Query 1 count + Under Observation tickets with pending requests.
Flag anyone at or over their `maxLoad`. Track `obsCount` (excluded under-observation tickets) separately per person.

> **Note**: The Jira "Under Observation" status is the primary signal here. The label-based under-observation keywords (`underobservation`, `under-observation`, `under_observation`, `monitoring`, `observation`) are used in Step 7a stale detection only — they are separate mechanisms.

### Step 3: Fetch Tickets from Filter 55922

Get all tickets from filter 55922 (Serraview_NewIssue_CM) via Jira REST API.

**Bucket 1 — Already Assigned** (assignee IS NOT EMPTY):
- DO NOT reassign
- POST to `/rest/api/3/issue/{issueKey}/transitions` with `{"transition":{"id":"31"}}` to move to Approved
- Output: Auto-Transitioned (Already Assigned)

> Filter 55922 targets "New Issue" status, non-S1 tickets. S1 tickets are handled by a separate event-based trigger and will not appear here.

**Bucket 2 & 3 — Unassigned** (assignee IS EMPTY):
For each ticket:
1. Fetch full description
2. Fetch last 3 comments (most recent first) — used for blocked signal detection in this step only; Step 7a fetches a separate 5-comment set for stale inference
3. Check for blocked/Dev/QA signals (see `references/team-config.md` — Ticket Analysis)
4. Blocked or Dev/QA environment detected → apply `ClopsManualTriage` label; do NOT assign or transition
5. Routable → apply routing rules (Step 4), assign, then transition (Step 5)

On API timeout/error: log the error, skip that ticket, continue with remaining, add to Skipped (API Error).

### Step 4: Apply Routing Rules (priority order, highest first)

Execute rules in this exact order — stop at the first match:

**Rule 0 — Boeing keyword override (any time, any engineer):**
IF summary OR description contains "Boeing" (case-insensitive) → assign to **Michael Ola Soga Jr.** regardless of all other rules. Do NOT assign Boeing tickets to anyone else.

**Rule 1 — CLAUTO / automation-fix keyword override:**
IF summary or description contains `clauto` or `automation-fix` (case-insensitive) → assign to **Deevanshu Gakhar** (Principal). Secondary: **Shobhit Mishra** (CLAUTO only) if Deevanshu is on leave or over maxLoad.

**Rule 2 — Severity S1:**
S1 tickets should not appear in filter 55922 (handled separately). If one does appear, assign to **Gaurav Kumar**. If Gaurav is on leave or over maxLoad and current IST time is 04:00–12:00 → try **Vikas Kumar**. If current IST time is 19:00–05:00 → try **Michael Ola Soga Jr.**

**Rule 3 — Severity S2:**
Assign to **Ankit Kumar Sinha** (Top Performer). If Ankit is over maxLoad → escalate to **Gaurav Kumar**. If IST time is 04:00–12:00 and both are unavailable → try **Vikas Kumar**. If IST time is 19:00–05:00 and others unavailable → try **Michael Ola Soga Jr.**

**Rule 4 — Database keywords:**
IF summary or description contains database keywords (`database`, `db`, `schema`, `migration`, `sql`, `dba`, `rds`, `postgres`, `db-change`) → assign to **Yuan Yang** (primary). If Yuan is on leave or over maxLoad → **Mridul Raina** (secondary). If both unavailable → **Deevanshu Gakhar** (tertiary). Do NOT assign DB tasks to general Serraview engineers.

**Rule 5 — Domain-based routing (see routing table in `references/team-config.md`):**
Apply workload balancing and skill/will matching. Never fail to assign — if all engineers in domain are over capacity, assign to the person with the least overage and add an overload note.

**Shift constraints for Vikas and Michael (non-Boeing):**
- Vikas Kumar: only assignable during 04:00–12:00 IST; do not assign outside his shift
- Michael Ola Soga Jr.: only assignable for non-Boeing Serraview work during 19:00–05:00 IST

### Step 5: Execute Assignments

**Blocked/Dev/QA tickets:**
PUT `/rest/api/3/issue/{issueKey}` with `{"update":{"labels":[{"add":"ClopsManualTriage"}]}}`.
Do NOT assign or transition.

**Routable tickets — always assign BEFORE transitioning:**
1. PUT `/rest/api/3/issue/{issueKey}/assignee` with `{"accountId":"ACCOUNT_ID"}` — use the accountId from `references/team-config.md`
2. POST `/rest/api/3/issue/{issueKey}/transitions` with `{"transition":{"id":"31"}}`

> Order matters: assign first, then transition. This ensures the ticket is never briefly in Approved status without an assignee.

If the assignment call fails: do not transition. Log the error, add to Skipped (API Error), continue.

### Step 6: Output Summary

```
## Triage Complete

### Auto-Transitioned (Already Assigned)
| Ticket | Summary | Assignee |

### Boeing Assigned
| Ticket | Summary | Assigned To | Reason |

### S2 Assigned (Senior Engineer)
| Ticket | Summary | Assigned To | Reason |

### Standard Assigned
| Ticket | Summary | Assigned To | Reason | Capacity Note |

### Manual Triage Required (ClopsManualTriage label applied)
| Ticket | Summary | Reason |

### Updated Workload
| Person | Before | After | Obs (+N) | Status |

### Skipped
| Ticket | Reason |

### Skipped (API Error)
| Ticket | Error |
```

Output notes:
- **Capacity Note** column: populate with "Assigned despite capacity — all assignees overloaded" when applicable; leave blank otherwise.
- **Obs (+N)** column: show the `obsCount` for each person (under-observation tickets excluded from load). Show `0` if none. Example: `+2 obs`.
- **Status** column — use **exactly** these three values, matching `references/notifications.md`:
  - `OVER CAPACITY` → currentLoad > maxLoad
  - `AT CAPACITY` → currentLoad == maxLoad
  - `ON TRACK` → currentLoad < maxLoad
  - Never use "UNDER", "OK", "HEALTHY", or any other label.
- Omit the Boeing Assigned section if no Boeing tickets were processed this run.

### Step 7: Send Notifications

Read `references/notifications.md` for webhook URLs, payload format, and stale/SLA detection rules.

---

**7a. Detect stale/SLA-breached tickets (ONLY when `firstRunOfDay=true`; skip entirely otherwise):**

**Determinism — do this first, before any query:**
- Compute `now_utc = datetime.now(timezone.utc)` ONCE. Use it for every elapsed time calculation in this step. Never call `datetime.now()` again during stale detection.
- Parse every Jira `updated` field as UTC: `datetime.fromisoformat(value.replace('Z', '+00:00'))`
- Query with `maxResults=200 ORDER BY updated ASC`. Paginate if `total > 200`.
- If any comment fetch fails → include the ticket in the stale list (fail-safe; never silently drop).

For each team member, query their **active** tickets (status IN ("New Issue", "In Progress") only).
Check `elapsed_hours = (now_utc - updated_utc).total_seconds() / 3600` against SLA thresholds using weekend-adjusted business hours for S3/S4 (see `references/notifications.md` for full weekend adjustment calculation and examples).

For each ticket that breaches its SLA threshold:
1. Fetch full description and last **5** comments (most recent first) — this is a separate fetch from the 3-comment fetch done in Step 3
2. Apply **under-observation exclusion** using both label keywords AND comment keywords (see `references/notifications.md` for full keyword lists and the two-condition logic)
3. For tickets that pass the exclusion check, **determine the next step** by analysing the description and comments:

**Next-step inference rules** (apply in order, use the first match):
- Last comment is from a customer/requester containing a question or update request → `"Respond to [first name only] — [3–5 word topic, no verbatim quote]"`
  Example: `"Respond to Jessiqa — awaiting status on broken reports"`
- Last comment is from the assignee/team saying they are investigating or monitoring → `"Add progress update / move ticket forward"`
- Ticket has no comments at all → `"Initial assessment needed"`
- Description or comments mention a blocker, dependency, or "waiting for" → `"Resolve blocker: [3–5 word description, no verbatim quote]"`
- Description or comments mention a pending deployment or change window → `"Confirm deployment window / schedule change"`
- Status is "New Issue" with no team comment yet → `"Assign and begin triage"`
- Default (none of the above match) → `"Review and update ticket status"`

**IMPORTANT**: Never include verbatim comment text in the next step. Summarise in your own words in at most 5 words. Do not quote or paraphrase sentences from the ticket.

Group remaining stale tickets by person. Include the inferred next step for each ticket.
See `references/notifications.md` for exact format string and display rules.

---

**7b. Send Channel 1 (always send, every run):**

Use Python `requests` library (NOT curl) to POST to Channel 1 webhook.
Build the message as a Python list of strings joined with `\n\n` — no HTML tags.

**CRITICAL — Teams @mentions in Adaptive Cards:** Plain `@Name` text does NOT trigger real mentions in Microsoft Teams Adaptive Cards. You MUST use the `mention()` helper function from `references/notifications.md` for every team member reference. This produces `<at>Name</at>` tags in the text AND registers the corresponding entity in `msteams.entities`. Missing this causes the message to send but @mentions are silent — no notification reaches the person.

Include:
- Triage result (assignments made this run, or "No new tickets in filter 55922")
- Stale tickets section — **only when `firstRunOfDay=true` AND stale tickets were found** (omit section entirely otherwise)
- Manual triage alerts — omit section if empty
- Errors — omit section if empty
- Full workload summary (OVER CAPACITY / AT CAPACITY / ON TRACK — omit empty tiers; include `(+N obs)` suffix per person where applicable)

Channel 1 is always sent regardless of content. See `references/notifications.md` for exact Python Adaptive Card template including `mention()` usage and footer @mentions.

---

**7c. Send Channel 2 (conditional):**

Use Python `requests` library (NOT curl) to POST to Channel 2 webhook.

**CRITICAL — Teams @mentions:** Same rule as Channel 1 — always use `mention()` from `references/notifications.md`. Never use plain `@Name` text.

Send ONLY if at least one of the following is true:
- Assignments were made this run, OR
- Stale tickets exist AND `firstRunOfDay=true`

If both conditions are false — do NOT send Channel 2.

Include: assignments from this run + stale tickets (stale section only when `firstRunOfDay=true`).
See `references/notifications.md` for exact Python Adaptive Card template.

---

## Key Rules

- AUTO-ASSIGNS tickets — no confirmation required
- Filter 55922 targets "New Issue" status; S1 tickets are excluded (handled by separate trigger)
- Bucket 1: already assigned → transition only (no reassignment)
- Bucket 2: unassigned, routable → **assign first, then transition**
- Bucket 3: unassigned, blocked/Dev/QA → ClopsManualTriage label + Teams alert via Channel 1
- Boeing tickets → Michael Ola Soga Jr. at any time, regardless of all other rules
- DB tasks restricted to Yuan Yang, Mridul Raina, and Deevanshu Gakhar only
- CLAUTO/automation tickets → Deevanshu Gakhar; Shobhit Mishra as secondary
- Vikas Kumar: AU-hours resource, only assignable 04:00–12:00 IST
- Michael Ola Soga Jr.: non-Boeing Serraview only during 19:00–05:00 IST shift
- Stale detection runs only when `firstRunOfDay=true` (default is `false`)
- Respect incubation for Ankit Kumar Sinha (prioritise Critical/Exploratory over BAU)
- Never fail to assign — if all domain engineers are over capacity, assign with overload note

## Example Invocation

```
User: "Triage the unassigned tickets, Yuan Yang is on leave today"
```

1. Exclude Yuan Yang from available assignees
2. Note current IST time for shift-aware routing
3. Query filter 55922 and current workloads (Queries 1 + 2)
4. Transition already-assigned tickets (Bucket 1)
5. Analyse and route unassigned tickets (Buckets 2 & 3)
   - Boeing keyword in any ticket → Michael Soga regardless of shift
   - DB tickets normally routed to Yuan Yang → escalate to Mridul Raina, then Deevanshu Gakhar
6. Assign first, then transition for each routable ticket
7. Send Channel 1 (always); skip stale detection since `firstRunOfDay` not passed (defaults to `false`)
8. Send Channel 2 only if assignments were made

> To include stale detection and daily sync alerts, pass `firstRunOfDay=true` on the first run of each day.
