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

## ⛔ CRITICAL — stdout Safety (Read Before Writing Any Code)

The Oz agent runtime parses **every byte on stdout** as a JSON object. Any plain string on stdout — including progress messages, debug prints, f-strings, or raw API responses — crashes the runtime with:

```
json: cannot unmarshal string into Go value of type map[string]interface {}
```

**These rules are absolute and apply to the entire script, from line 1:**

1. **Every `print(...)` call must have `file=sys.stderr`** unless it is part of the Step 6 final markdown summary.
2. **The Step 6 markdown tables are the ONLY content permitted on stdout.** Nothing else — ever.
3. **Never print file contents to stdout.** When reading `team-config.md` or `notifications.md`, do NOT echo the file contents. Read silently into a variable.
4. **Never print API responses, JSON payloads, or dicts to stdout.** All `json.dumps(...)` output goes to stderr only.
5. **Never print f-strings containing `{` or `}` characters to stdout** (e.g. accountIds, URLs, payloads).
6. **Never use bare `print(variable)` at the top level of the script** — wrap every diagnostic in `file=sys.stderr`.

```python
import sys

# ✅ CORRECT — all diagnostics to stderr
print("Reading team-config...", file=sys.stderr)
print(f"Found {len(issues)} tickets", file=sys.stderr)
print(f"Assigning {key} to {name}", file=sys.stderr)

# ✅ CORRECT — only Step 6 summary tables to stdout
print("## Triage Complete")
print("| Ticket | Summary | Assigned To |")

# ❌ WRONG — crashes runtime
print("Reading team-config...")
print(f"Response: {response.json()}")
print(json.dumps(payload))
```

**If in doubt, add `file=sys.stderr`.** The Step 6 markdown tables are the only exception.

## Python Environment

**Required packages:** `requests` only. No other third-party packages are needed or permitted.

Install once if not present:
```bash
apt-get install -y python3-requests 2>/dev/null || pip install --break-system-packages requests
```

**IST time — use stdlib only, never `pytz`:**
```python
from datetime import datetime, timezone, timedelta

utc_now = datetime.now(timezone.utc)
ist_now = utc_now + timedelta(hours=5, minutes=30)  # NOT timezone.utc + timedelta(...)
ist_hour = ist_now.hour  # integer 0-23, used for shift-aware routing
today_ist = ist_now.strftime("%Y-%m-%d")  # use this for notification date headers
```

**WRONG — do not do this:**
```python
# WRONG: timezone.utc + timedelta(...) raises TypeError
datetime.now(timezone.utc + timedelta(hours=5, minutes=30))
```

**Jira comment body — ADF extraction:**
Jira's `GET /rest/api/3/issue/{key}/comment` returns each comment's `body` field as an **Atlassian Document Format (ADF) dict**, not a plain string. Calling `.lower()` directly on it raises `AttributeError: 'dict' object has no attribute 'lower'`. Always extract text from ADF before string operations:

```python
def extract_comment_text(body):
    """Extract plain text from Jira ADF comment body (or return '' if not a string)."""
    if isinstance(body, str):
        return body
    if not isinstance(body, dict):
        return ""
    # ADF structure: body.content[].content[].text
    texts = []
    for block in body.get("content", []):
        for inline in block.get("content", []):
            if inline.get("type") == "text":
                texts.append(inline.get("text", ""))
    return " ".join(texts)

# Usage — always extract before any string check:
raw_body = comment.get("body", "")
comment_text = extract_comment_text(raw_body).lower()
if "under observation" in comment_text:
    ...
```

Apply `extract_comment_text()` to every comment body access throughout the script — under-observation detection, pending update request detection, next-step inference, and blocked signal detection.

**No demo/simulation fallback — ever:** 
- If a required environment variable (`SV_JIRA_BASE_URL`, `SV_JIRA_EMAIL`, `SV_JIRA_API_TOKEN`) is missing → print a clear error to stderr and exit with code 1.
- If the triage script crashes or encounters a KeyError, AttributeError, or any unhandled exception → let the exception propagate (do NOT catch it with a broad `except Exception` that swallows it). The traceback is more useful than a silent demo.
- If any step fails → log the failure to stderr and continue with remaining steps where possible; never substitute fake/mock data for real Jira results.
- Never create a `demo_triage.py`, `demo_output.py`, or any simulation script. Real actions from real Jira API only.
- If the real triage produces no assignments and no stale tickets, that IS a valid result — report it honestly in Channel 1 ("No new tickets in filter 55922"). Do not invent example output to make the run look more productive.

**Script output — stdout safety rules:**
The agent runtime parses the script's stdout as a JSON stream. Certain output patterns crash the runtime with `json: cannot unmarshal string into Go value of type map[string]interface {}`. To prevent this:

> **See the `⛔ CRITICAL — stdout Safety` section at the top of this file for the full rule set.** The rules below are a summary; the top section takes precedence.

- **Redirect ALL script progress logging to stderr** — use `print("...", file=sys.stderr)` for every status message, debug output, and intermediate result. No exceptions.
- **Never echo file contents to stdout** — reading `team-config.md` or `notifications.md` must be silent; do NOT print the file content even partially.
- **Never print raw JSON payloads to stdout** — the Adaptive Card payload, Jira API responses, or any dict/list printed via `print(json.dumps(...))` will crash the runtime if it reaches stdout.
- **stdout is reserved for the Step 6 final triage summary only** — markdown tables and headers only. Everything else goes to stderr.
- **Avoid printing strings containing bare `{` or `}` to stdout** unless they are inside a markdown table cell.

```python
import sys

# All progress/debug output → stderr (safe, never parsed by runtime)
print("Fetching filter 55922...", file=sys.stderr)
print(f"Found {len(issues)} tickets", file=sys.stderr)

# Final summary → stdout (only markdown tables, no raw JSON)
print("## Triage Complete")
print("| Ticket | Summary | Assigned To |")
```

## REST API Patterns

**Always use `/rest/api/3/`** — do not fall back to `/rest/api/2/`. A 410 on the old `/rest/api/3/search` or `/rest/api/2/search` endpoint means those are deprecated — always use `POST /rest/api/3/search/jql` instead.

### Critical: `/rest/api/3/search/jql` behaves differently from the old `/search`

**Fields parameter** — the search endpoint behaviour on this tenant is specific:

- **Omitting `fields` entirely** returns issues with **only `id`** — no `key`, no `assignee`, no `summary`. This makes the result useless for workload counting or stale detection. Do NOT omit fields.
- **`"fields": ["summary", "assignee"]`** — confirmed working, returns `key`, `id`, `summary`, and `assignee.accountId`. Always use this as the minimum for all search calls.
- **Adding other fields** (e.g. `"status"`, `"priority"`, `"updated"`, `"labels"`, `"description"`) to the array causes **400 Bad Request** on this tenant.

**Rule: always use exactly `"fields": ["summary", "assignee"]` in the search payload.** Never omit it, never add other fields. For any additional field (status, priority, updated, labels, description), use a separate `GET /rest/api/3/issue/{key}?fields=...` call.

**Pagination** — the new `/search/jql` endpoint uses **cursor-based pagination**, NOT offset-based. There is no `startAt` parameter and no `total` field in the response:
- Response contains `issues[]`, `isLast` (boolean), and `nextPageToken` (string)
- To paginate: include `"nextPageToken": "<value from previous response>"` in the next request body
- Loop until `isLast == true`
- **Never check `data.get("total")` — it will always be 0 or absent. Check `len(data.get("issues", []))` instead.**

```python
# Correct pagination pattern for /rest/api/3/search/jql
# ALWAYS include "fields": ["summary", "assignee"] — omitting fields returns only id with no key
def search_all_tickets(jql, headers, base_url, max_results_per_page=200):
    url = f"{base_url}/rest/api/3/search/jql"
    all_issues = []
    next_page_token = None

    while True:
        payload = {
            "jql": jql,
            "maxResults": max_results_per_page,
            "fields": ["summary", "assignee"]   # REQUIRED — never omit
        }
        if next_page_token:
            payload["nextPageToken"] = next_page_token

        response = requests.post(url, headers=headers, json=payload)
        response.raise_for_status()
        data = response.json()

        issues = data.get("issues", [])
        all_issues.extend(issues)

        if data.get("isLast", True) or not issues:
            break
        next_page_token = data.get("nextPageToken")

    return all_issues
```

```bash
# Auth header
AUTH=$(echo -n "$SV_JIRA_EMAIL:$SV_JIRA_API_TOKEN" | base64)
HEADER="Authorization: Basic $AUTH"

# Fetch filter 55922 metadata
# Correct URL: GET /rest/api/3/filter/{id}  — NOT /filter/{id}/jql
curl -s -H "$HEADER" "$SV_JIRA_BASE_URL/rest/api/3/filter/55922" | jq .jql

# Search tickets — always include fields: summary + assignee (omitting returns only id, no key)
curl -s -H "$HEADER" -H "Content-Type: application/json" \
  -X POST "$SV_JIRA_BASE_URL/rest/api/3/search/jql" \
  -d '{"jql":"filter=55922","maxResults":200,"fields":["summary","assignee"]}'

# Fetch full issue detail (safe — all fields work via this endpoint)
curl -s -H "$HEADER" \
  "$SV_JIRA_BASE_URL/rest/api/3/issue/CM-XXXXX?fields=summary,assignee,priority,labels,description,status,updated"

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

**Team maxLoad reference** — use these hardcoded values. `team-config.md` omits `maxLoad` for Michael Ola Soga Jr.; use 5 as his default. Never crash on a missing `maxLoad` key — always fall back to 5 for any member without one defined:

| Person | maxLoad |
|---|---|
| Gaurav Kumar | 10 |
| Deevanshu Gakhar | 5 |
| Yuan Yang | 5 |
| Ankit Kumar Sinha | 10 |
| Shobhit Mishra | 5 |
| Mashkoor Ahmad | 5 |
| Mridul Raina | 5 |
| Vikas Kumar | 5 |
| Michael Ola Soga Jr. | 5 |

In code: `max_load = config.get("maxLoad", 5)` — never `config["maxLoad"]`.

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
AND "Category and Sub-category[Select List (cascading)]" IN cascadeOption("Serraview")
```

**Always include `"fields": ["summary", "assignee"]` in the search payload** — omitting fields returns only `id` with no `key` or assignee data, making the result useless for workload counting. Use `issue["fields"]["assignee"]["accountId"]` to attribute each result to a team member. Paginate using `nextPageToken` / `isLast` (not `startAt`).

**Query 2 — Under Observation tickets** (decide inclusion individually):

```
project = CM
AND assignee IN (...same accountIds...)
AND status = "Under Observation"
AND "Category and Sub-category[Select List (cascading)]" IN cascadeOption("Serraview")
```

Same approach: include `"fields": ["summary", "assignee"]` in the search payload. For each returned issue key, fetch full detail via `GET /rest/api/3/issue/{key}/comment?maxResults=3&orderBy=-created` to check for pending update requests.
- If the last comment (from anyone) contains a pending update request (`please update`, `any update`, `update?`, `following up`, `looking for update`, `any progress`, `status update`, `can you update`, `please respond`, `waiting for update`, `need an update`, `please provide`) → **include in workload count** (requester is waiting on a response)
- Otherwise → **exclude from workload count**; add to that person's `obsCount` for the `(+X obs)` display suffix in notifications

Final workload count per person = Query 1 count + Under Observation tickets with pending requests.
Flag anyone at or over their `maxLoad`. Track `obsCount` (excluded under-observation tickets) separately per person.

> **Note**: The Jira "Under Observation" status is the primary signal here. The label-based under-observation keywords (`underobservation`, `under-observation`, `under_observation`, `monitoring`, `observation`) are used in Step 7a stale detection only — they are separate mechanisms.

### Step 3: Fetch Tickets from Filter 55922

Get all tickets from filter 55922 via `POST /rest/api/3/search/jql`.

**Use the two-call pattern** — only `["summary", "assignee"]` are safe in the search `fields` array; all other fields cause 400 errors. Get keys and assignee from search, then fetch per-issue detail separately:

```python
# Step 1: Get all ticket keys and assignee from filter
# Always include fields: ["summary", "assignee"] — omitting returns only id with no key
search_payload = {"jql": "filter=55922", "maxResults": 200, "fields": ["summary", "assignee"]}
# Paginate using nextPageToken / isLast
# Each issue object will have: issue["key"], issue["fields"]["summary"], issue["fields"]["assignee"]

# Step 2: For each key, fetch full details via GET
# GET /rest/api/3/issue/{key}?fields=summary,assignee,priority,labels,description,status,updated
```

**Bucket 1 — Already Assigned** (assignee IS NOT EMPTY in the fetched issue detail):
- DO NOT reassign
- POST to `/rest/api/3/issue/{issueKey}/transitions` with `{"transition":{"id":"31"}}` to move to Approved
- Output: Auto-Transitioned (Already Assigned)

> Filter 55922 targets "New Issue" status, non-S1 tickets. S1 tickets are handled by a separate event-based trigger and will not appear here.

**Bucket 2 & 3 — Unassigned** (assignee IS EMPTY or null):
For each ticket:
1. Full description and last 3 comments already available from the GET issue call above — used for blocked signal detection; Step 7a fetches a separate 5-comment set for stale inference
2. Check for blocked/Dev/QA signals (see `references/team-config.md` — Ticket Analysis)
3. Blocked or Dev/QA environment detected → apply `ClopsManualTriage` label; do NOT assign or transition
4. Routable → apply routing rules (Step 4), assign, then transition (Step 5)

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
- Query with `maxResults=200 ORDER BY updated ASC`. Paginate using `nextPageToken` / `isLast` (NOT `startAt` / `total`).
- If any comment fetch fails → include the ticket in the stale list (fail-safe; never silently drop).

**`updated` field guard — REQUIRED:**
The `updated` field from `GET /rest/api/3/issue/{key}` may return an empty string or be absent for some tickets. Never crash on this. Apply this guard:

```python
raw_updated = issue_data.get("fields", {}).get("updated", "")
if not raw_updated:
    # Fail-safe: treat as maximally stale (epoch = always over SLA)
    updated_utc = datetime.fromtimestamp(0, tz=timezone.utc)
else:
    updated_utc = datetime.fromisoformat(raw_updated.replace('Z', '+00:00'))
```

Tickets with a missing or empty `updated` field must be **included** in the stale list (not skipped, not logged as "Unknown"). Use the ticket's actual key from the search results.

For each team member, query their **active** tickets:

```
JQL: project = CM AND assignee = "<accountId>" AND status IN ("New Issue", "In Progress") AND "Category and Sub-category[Select List (cascading)]" IN cascadeOption("Serraview") ORDER BY updated ASC
```

**Always include `"fields": ["summary", "assignee"]` in the search payload** — omitting returns only `id` with no `key`. Paginate using `nextPageToken` / `isLast`. For each returned issue `key`, fetch full detail via `GET /rest/api/3/issue/{key}?fields=summary,priority,updated,labels,status` to get the fields needed for SLA calculation.

**Workload / stale reconciliation — REQUIRED:**
After stale detection completes, cross-check against the Step 2 workload counts. If a person has N stale tickets but their Step 2 workload count is 0 (or less than N), the Step 2 workload query failed silently. In that case, use `max(step2_count, stale_count)` as the displayed workload for that person, and flag the workload query failure in the Errors section of the Step 6 output and Channel 1 notification. Never display a workload of 0 for someone who has confirmed stale tickets.

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

> **Note on `notifications.md` stale query instructions:** `notifications.md` references `startAt=0` and checking `total > 200` for pagination — those instructions are for the old `/search` endpoint. For `POST /rest/api/3/search/jql`, use `nextPageToken` / `isLast` instead (see REST API Patterns above). The `updated` field guard above also supersedes the bare `fromisoformat` call in `notifications.md`.

Group remaining stale tickets by person. Include the inferred next step for each ticket. Each ticket key must be a clickable hyperlink using `RichTextBlock` / `linked_key()` as described in Step 7b — do not render the key as plain text.

---

**7b. Send Channel 1 (always send, every run):**

Use Python `requests` library (NOT curl) to POST to Channel 1 webhook.
Build the message as a Python list of strings joined with `\n\n` — no HTML tags.

**Jira ticket links — key text as clickable hyperlink:**

Teams Adaptive Card `TextBlock` does not support inline hyperlinks. To make `CM-XXXXX` itself a clickable link, the card body must use **`RichTextBlock`** with `TextRun` elements instead of a single `TextBlock`. Switch the card body to a list of `RichTextBlock` elements, one per logical paragraph:

```python
JIRA_BASE_URL = os.environ.get("SV_JIRA_BASE_URL", "https://eptura.atlassian.net")

def ticket_url(key):
    return f"{JIRA_BASE_URL}/browse/{key}"

def rich_text(text):
    """Plain text run — no link."""
    return {"type": "TextRun", "text": text}

def linked_key(key):
    """Ticket key as a clickable hyperlink TextRun."""
    return {"type": "TextRun", "text": key,
            "selectAction": {"type": "Action.OpenUrl", "url": ticket_url(key)}}

def para(*runs):
    """One RichTextBlock paragraph from a list of TextRun dicts."""
    return {"type": "RichTextBlock", "inlines": list(runs)}
```

Build each ticket line as a `RichTextBlock` where the key is a linked `TextRun` and surrounding text is plain `TextRun`:

```python
# Stale ticket — key is the link, rest is plain text
body_elements.append(para(
    rich_text("• [S" + str(sn) + "] "),
    linked_key(key),
    rich_text(f" - {summary} | {status} | Last update: {elapsed}h ago (SLA: {sla}h)")
))
body_elements.append(para(rich_text(f"  ➡️ Next: {next_step}")))

# Assignment line
body_elements.append(para(
    rich_text("✅ "),
    linked_key(key),
    rich_text(f" → {assignee_name} — {summary}")
))

# Manual triage line
body_elements.append(para(
    rich_text("• "),
    linked_key(key),
    rich_text(f" — {summary} ({reason})")
))

# For plain text sections (workload, headers, footer @mentions) use a plain TextBlock:
body_elements.append({"type": "TextBlock", "text": plain_text_section, "wrap": True})
```

The final card payload `body` is a **mixed list** of `RichTextBlock` (for ticket lines with links) and `TextBlock` (for all other sections). The `msteams.entities` array for @mentions attaches to the card level as before — it works alongside `RichTextBlock` elements without change:

```python
payload = {
    "type": "message",
    "attachments": [{
        "contentType": "application/vnd.microsoft.card.adaptive",
        "content": {
            "type": "AdaptiveCard",
            "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
            "version": "1.4",
            "body": body_elements,   # mixed TextBlock + RichTextBlock list
            "msteams": {"width": "Full", "entities": entities}
        }
    }]
}
```

**Build strategy:** Start `body_elements = []`. Append a `TextBlock` for the header and each non-ticket section (stale person header, workload, footer). Append `RichTextBlock` paragraphs for each ticket line. This replaces the previous `parts` list / `"\n\n".join(parts)` approach — do not join into a single string anymore.

**CRITICAL — Teams @mentions in Adaptive Cards:** Plain `@Name` text does NOT trigger real mentions in Microsoft Teams Adaptive Cards. You MUST use the `mention()` helper function from `references/notifications.md` for every team member reference. This produces `<at>Name</at>` tags in the text AND registers the corresponding entity in `msteams.entities`. Missing this causes the message to send but @mentions are silent — no notification reaches the person.
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
