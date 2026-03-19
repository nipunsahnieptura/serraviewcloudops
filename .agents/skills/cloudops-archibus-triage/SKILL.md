---
name: cloudops-archibus-triage
description: "Intelligent triage and assignment of Jira Change Management (CM) tickets for the CloudOps Archibus team. Automatically routes unassigned tickets based on domain, severity, team workload, and routing rules, then transitions them to Approved. Transitions already-assigned tickets to Approved without reassigning. Use when asked to triage Archibus CM tickets, run ticket triage, process filter 55937, assign or approve Change Management tickets for the Archibus team, or balance workload across Archibus engineers."
---
# Archibus CloudOps Ticket Triage
## Jira API Configuration
```yaml
Base URL: $SV_JIRA_BASE_URL (env var)
Auth: Basic base64(SV_JIRA_EMAIL:SV_JIRA_API_TOKEN)
Transition ID for Approve: "31"
Filter ID: 55937
```
Use Jira REST API directly via curl or Python requests — do NOT use MCP tools.
Read `.agents/skills/archibus-cloudops-triage/references/team-config.md` for team roster, routing rules, workload balancing, and ticket analysis signals.
Read `.agents/skills/archibus-cloudops-triage/references/notifications.md` for Teams webhook URL, recipients, and payload format.
## REST API Patterns
```bash
# Auth header
AUTH=$(echo -n "$SV_JIRA_EMAIL:$SV_JIRA_API_TOKEN" | base64)
HEADER="Authorization: Basic $AUTH"

# Get filter JQL
curl -s -H "$HEADER" "$SV_JIRA_BASE_URL/rest/api/3/filter/55937" | jq .jql

# Search tickets
curl -s -H "$HEADER" -H "Content-Type: application/json" \
  -X POST "$SV_JIRA_BASE_URL/rest/api/3/search/jql" \
  -d '{"jql":"filter=55937","fields":["summary","assignee","priority","description","comment","labels","customfield_15699","status"]}'

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

# Lookup account ID by email (run for any LOOKUP_REQUIRED entry in team-config.md)
curl -s -H "$HEADER" "$SV_JIRA_BASE_URL/rest/api/3/user/search?query=<email>&maxResults=1" | jq -r '.[0].accountId'
```
## Workflow
### Step 1: Get Team Availability
Check the `onLeave` parameter. Remove those members from the available assignee pool for this run.
Check `firstRunOfDay` parameter (default: **`false`**). Only if `true` will stale tickets be included in the Channel 2 notification. This prevents duplicate stale alerts when the skill runs multiple times per day.

**Account ID Resolution**: If any team member's `accountId` in `references/team-config.md` is `LOOKUP_REQUIRED`, resolve it using the lookup pattern above before proceeding with any JQL queries.
### Step 2: Query Current Workload
The Archibus category condition is: `"Category and Sub-category[Select List (cascading)]" = Archibus`
Verify by fetching filter 55937's JQL and extract the category clause if different.

**Query 1 — Active workload** (excludes done, cancelled, and under-observation tickets):
```
project = CM
AND assignee IN (<comma-separated accountIds from team-config.md>)
AND status NOT IN (Done, Cancelled, "Under Observation")
AND "Category and Sub-category[Select List (cascading)]" = Archibus
```

**Query 2 — Under Observation tickets** (check if any have pending requests):
```
project = CM
AND assignee IN (...same accountIds...)
AND status = "Under Observation"
AND "Category and Sub-category[Select List (cascading)]" = Archibus
```
For each "Under Observation" ticket, fetch the last 3 comments.
- If the last comment contains a pending update request (`please update`, `any update`, `update?`, `following up`, `looking for update`, `any progress`, `status update`, `can you update`, `please respond`, `waiting for update`, `need an update`, `please provide`) → **include in workload count** (requester is waiting on us)
- Otherwise → **exclude from workload count**, add to `obsCount` for the `(+X obs)` display suffix

Final workload count per person = Query 1 count + Under Observation tickets with pending requests.
Flag anyone at or over their `maxLoad`. Track `obsCount` per person separately.
### Step 3: Fetch Tickets from Filter 55937
Get all tickets from filter 55937 (Archibus_NewIssue_CM) via Jira REST API.
**Bucket 1 - Already Assigned** (assignee IS NOT EMPTY):
- DO NOT reassign
- POST to `/rest/api/3/issue/{issueKey}/transitions` with `{"transition":{"id":"31"}}` to move to Approved
- Output: Auto-Transitioned (Already Assigned)

**Bucket 2 & 3 - Unassigned** (assignee IS EMPTY):
For each ticket:
1. Fetch full description
2. Fetch last 3 comments (most recent first)
3. Check for blocked/Dev/QA/urgency signals (see `references/team-config.md` - Ticket Analysis)
4. Blocked or Dev/QA environment detected: apply `ClopsManualTriage` label, do NOT assign or transition
5. Routable: apply routing rules, assign, transition

On API timeout/error: log the error, skip that ticket, continue with remaining, add to Skipped (API Error).
### Step 4: Apply Routing Rules (priority order)
1. **Apollo/GraphQL keyword override**: summary/description contains `apollo`, `graphql`, `onsite api` → Alex Plotkin regardless of other rules
2. **S1 (Critical)**: Santosh Prasad (primary), Dany Silva (secondary). Notify Maninder Singh.
3. **S2 (High)**: Route to senior engineers (Santosh, Dany, Alex) based on domain
4. **Urgency signals**: `URGENT`, `ASAP`, `customer escalated`, `blocking`, `down`, `outage`, `503` → Santosh or Dany
5. **Domain-based routing** (see routing table in `references/team-config.md`)
6. **Workload balancing**
7. **US timezone preference**: if ticket requires US-hours coordination and was created outside IST business hours → prefer Alex Plotkin or Dany Silva
### Step 5: Execute Assignments
- **Blocked/Dev/QA**: PUT `/rest/api/3/issue/{issueKey}` with `{"update":{"labels":[{"add":"ClopsManualTriage"}]}}`. DO NOT assign or transition.
- **Routable**:
  1. POST `/rest/api/3/issue/{issueKey}/transitions` with `{"transition":{"id":"31"}}`
  2. PUT `/rest/api/3/issue/{issueKey}/assignee` with `{"accountId":"ACCOUNT_ID"}` — use the **accountId** from `references/team-config.md`
### Step 6: Output Summary
```
## Triage Complete

### Auto-Transitioned (Already Assigned)
| Ticket | Summary | Assignee |

### S1/S2 Assigned (Senior Engineer)
| Ticket | Summary | Assigned To | Reason |

### Standard Assigned
| Ticket | Summary | Assigned To | Reason |

### Manual Triage Required (ClopsManualTriage label applied)
| Ticket | Summary | Reason |

### Updated Workload
| Person | Before | After | Status |

### Skipped
| Ticket | Reason |

### Skipped (API Error)
| Ticket | Error |
```
### Step 7: Send Notifications
Read `.agents/skills/archibus-cloudops-triage/references/notifications.md` for webhook URLs, payload format, and stale/SLA detection rules.

**7a. Detect stale/SLA-breached tickets** (only when `firstRunOfDay=true`):
For each team member, query their **active** tickets (status IN ("New Issue", "In Progress") with Archibus category filter).
Check elapsed time since `updated` field against SLA thresholds using weekend-adjusted hours for S3/S4 (see `references/notifications.md`).

For each ticket that breaches its SLA threshold:
1. Fetch full description and last 5 comments (most recent first)
2. Apply **under-observation exclusion** (see `references/notifications.md` for keywords)
3. For tickets that pass the exclusion check, **determine the next step** by analyzing the description and comments:

**Next-step inference rules** (apply in order, use the first match):
- Last comment is from a customer/requester containing a question or update request → `"Respond to [first name only] — [3–5 word topic, no verbatim quote]"`
  Example: `"Respond to Luke — awaiting WAR deployment status"`
- Last comment is from the assignee/team saying they are investigating or monitoring → `"Add progress update / move ticket forward"`
- Ticket has no comments at all → `"Initial assessment needed"`
- Description or comments mention a blocker, dependency, or "waiting for" → `"Resolve blocker: [3–5 word description, no verbatim quote]"`
- Description or comments mention a pending deployment or change window → `"Confirm deployment window / schedule change"`
- Status is "New Issue" with no team comment yet → `"Assign and begin triage"`
- Default (none of the above match) → `"Review and update ticket status"`

**IMPORTANT**: Never include verbatim comment text in the next step. Summarise in your own words using at most 5 words. Do not quote or paraphrase sentences from the ticket.

Group remaining stale tickets by person. Include the inferred next step for each ticket.
See `references/notifications.md` for exact format.

**7b. Send Channel 1** (always send):
Use Python `requests` library (NOT curl) to POST to Channel 1 webhook.
Build the message as a Python list of strings joined with `\n` — no HTML tags.
Include: triage result, stale tickets (if firstRunOfDay), manual triage alerts (if any), errors (if any), full workload summary.
See `references/notifications.md` for exact Python template.

**7c. Send Channel 2** (conditional):
Use Python `requests` library (NOT curl) to POST to Channel 2 webhook.
Send ONLY if assignments were made OR stale tickets exist (and firstRunOfDay=true).
Include: assignments from this run + stale tickets.
See `references/notifications.md` for exact Python template.
If both lists are empty — do NOT send.
## Key Rules
- AUTO-ASSIGNS tickets — no confirmation required
- Filter 55937 targets "New Issue" status, Archibus category tickets
- Bucket 1: already assigned → transition only (no reassignment)
- Bucket 2: unassigned, routable → assign + transition
- Bucket 3: unassigned, blocked/Dev/QA → ClopsManualTriage label + Teams alert
- Apollo/GraphQL keyword tickets always go to Alex Plotkin
- S1 critical tickets always go to Santosh Prasad (Dany Silva backup)
- Stage/Staging and Prod/Production are VALID environments — do NOT flag these
## Example Invocation
User: "Triage the Archibus tickets, Santosh is on leave today"
1. Exclude Santosh from available assignees
2. Resolve any LOOKUP_REQUIRED account IDs
3. Query filter 55937 and current workloads
4. Transition already-assigned tickets (Bucket 1)
5. Analyze and route unassigned tickets (Buckets 2 & 3)
6. For PROD deployment tickets normally routed to Santosh, escalate to Dany Silva
7. Send Teams notifications
