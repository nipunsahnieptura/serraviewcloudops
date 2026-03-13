---
name: cloudops-serraview-triage
description: "Intelligent triage and assignment of Jira Change Management (CM) tickets for the CloudOps Serraview team. Automatically routes unassigned tickets based on domain, severity, team workload, and routing rules, then transitions them to Approved. Transitions already-assigned tickets to Approved without reassigning. Use when asked to triage Serraview CM tickets, run ticket triage, process filter 55922, assign or approve Change Management tickets for the Serraview team, or balance workload across Serraview engineers."
---
# Serraview CloudOps Ticket Triage
## Jira API Configuration
```yaml
Base URL: SV_JIRA_BASE_URL env var (https://eptura.atlassian.net)
Auth: Basic (email: SV_JIRA_EMAIL env var, token: SV_JIRA_API_TOKEN env var)
Transition ID for Approve: "31"
Filter ID: 55922
```
Read `references/team-config.md` for team roster, routing rules, workload balancing, and ticket analysis signals.
Read `references/notifications.md` for Teams webhook URL, recipients, and payload format.
## Workflow
### Step 1: Get Team Availability
Check the `onLeave` parameter. Remove those members from the available assignee pool for this run.
### Step 2: Query Current Workload
Query open tickets for all team members via Jira REST API:
```
project = CM AND assignee IN ("712020:e779f9ea-49c9-4573-8a3a-61a6264cd283", "639af1b47145571a7ea882d7", "6362e6fe59c794184bcc1a3e", "712020:b217ad2b-f35c-41e1-8ec5-73d5d58952d0", "712020:7f06dd3b-4d20-4e02-bb38-b3ff9ea66c64", "64238bb20152b5f4f9f2e7f9", "712020:4962c34a-ee1a-427c-aced-015675053cae", "62b8f3e8118b20bee2ba7228") AND status NOT IN (Done, Cancelled, "Under Observation")
```
Count active tickets per person. Flag anyone at or over their `maxLoad`.
### Step 3: Fetch Tickets from Filter 55922
Get all tickets from filter 55922 (Serraview_NewIssue_CM) via Jira REST API.
**Pre-check - Under Observation**: If a ticket's current status is "Under Observation", skip it entirely. Add to Skipped section with reason "Under Observation — excluded from triage". Do NOT assign, transition, or label.

**Bucket 1 - Already Assigned** (assignee IS NOT EMPTY):
- DO NOT reassign
- Call `transition_issue(issueKey, transitionId="31")` to move to Approved
- Output: Auto-Transitioned (Already Assigned)

**Bucket 2 & 3 - Unassigned** (assignee IS EMPTY):
For each ticket:
1. Fetch full description
2. Fetch last 3 comments (most recent first)
3. Check for blocked/Dev/QA signals (see `references/team-config.md` - Ticket Analysis)
4. Blocked or Dev/QA environment detected: apply `ClopsManualTriage` label, do NOT assign or transition
5. Routable: apply routing rules, assign, transition

On API timeout/error: log the error, skip that ticket, continue with remaining, add to Skipped (API Error).
### Step 4: Apply Routing Rules (priority order)
1. CLAUTO/automation-fix keywords in summary/description → Deevanshu Gakhar (primary), Shobhit Mishra (secondary if Deevanshu at maxLoad)
2. Severity S1 → Gaurav Kumar; S2 → Ankit Kumar Sinha (escalate to Gaurav if over maxLoad)
3. Database keywords → Yuan Yang (primary), Mridul Raina (secondary), Deevanshu Gakhar (tertiary)
4. Domain-based routing (see routing table in `references/team-config.md`)
5. Workload balancing
6. Skill/will matching
### Step 5: Execute Assignments
- **Blocked/Dev/QA**: Add label `ClopsManualTriage` via Jira API. DO NOT assign or transition.
- **Routable**:
  1. `transition_issue(issueKey, transitionId="31")`
  2. `update_issue(issueKey, assignee={email})` — use **email format** (not accountId); the Jira MCP tool resolves assignees by email
### Step 6: Output Summary
```
## Triage Complete

### Auto-Transitioned (Already Assigned)
| Ticket | Summary | Assignee |

### S2 Assigned (Senior Engineer)
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
Read `references/notifications.md` for webhook URL, recipients, and payload format.
- ALWAYS send Teams notification via webhook after every triage run
## Key Rules
- AUTO-ASSIGNS tickets — no confirmation required
- Filter 55922 targets "New Issue" status, non-S1 tickets
- Bucket 1: already assigned → transition only (no reassignment)
- Bucket 2: unassigned, routable → assign + transition
- Bucket 3: unassigned, blocked/Dev/QA → ClopsManualTriage label + Teams alert
- DB tasks restricted to Yuan Yang, Mridul Raina, and Deevanshu Gakhar only
- CLAUTO/automation tickets go to Deevanshu Gakhar (primary) or Shobhit Mishra (secondary)
- Respect incubation for Ankit Kumar Sinha (prioritize Critical/Exploratory over BAU)
- Tickets in "Under Observation" status are skipped entirely (not assigned, transitioned, or labelled) and excluded from workload counts
## Example Invocation
User: "Triage the unassigned tickets, Yuan Yang is on leave today"
1. Exclude Yuan Yang from available assignees
2. Query filter 55922 and current workloads
3. Transition already-assigned tickets (Bucket 1)
4. Analyze and route unassigned tickets (Buckets 2 & 3)
5. For DB tickets normally routed to Yuan Yang, escalate to Deevanshu Gakhar or flag for manual triage
6. Send Teams notification with triage summary
