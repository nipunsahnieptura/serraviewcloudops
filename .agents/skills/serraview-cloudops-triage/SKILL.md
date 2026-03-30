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
MCP Server: https://mcp.atlassian.com/v1/sse  (Atlassian MCP — preferred when available)
```

Read `.agents/skills/serraview-cloudops-triage/references/team-config.md` for team roster, routing rules, workload balancing, and ticket analysis signals.
Read `.agents/skills/serraview-cloudops-triage/references/notifications.md` for Teams webhook URL, recipients, and payload format.

## MCP vs REST API Strategy

**Always prefer the Atlassian MCP server** when it is connected to the agent runtime. Fall back to direct Jira REST API calls (curl / Python `requests`) only when MCP is unavailable or a specific call fails.

### Detecting MCP availability

At the start of every run, probe MCP availability with a lightweight call:

```python
MCP_AVAILABLE = False

def probe_mcp():
    """Return True if the Atlassian MCP server responds successfully."""
    global MCP_AVAILABLE
    try:
        # Use the MCP getAccessibleResources or equivalent lightweight tool.
        # In an agentic context the agent runtime exposes MCP tools directly;
        # call Atlassian:getAccessibleAtlassianResources and check for a valid cloudId.
        result = call_mcp_tool("Atlassian:getAccessibleAtlassianResources")
        if result and result.get("cloudId"):
            MCP_AVAILABLE = True
            print(f"MCP available — cloudId: {result['cloudId']}", file=sys.stderr)
            return True
    except Exception as e:
        print(f"MCP probe failed: {e} — falling back to REST API", file=sys.stderr)
    MCP_AVAILABLE = False
    return False

probe_mcp()
```

If the agent runtime natively exposes Atlassian MCP tools (e.g. `Atlassian:searchJiraIssuesUsingJql`, `Atlassian:getJiraIssue`, `Atlassian:editJiraIssue`, `Atlassian:transitionJiraIssue`), treat MCP as available without an explicit probe — just call them directly and catch exceptions.

### Known MCP Quirks (Eptura Atlassian tenant — confirmed in live execution)

These are not theoretical — they were observed during real triage runs and must be coded around:

| Quirk | Symptom | Fix |
|---|---|---|
| `editJiraIssue` assignee silent failure | Returns `"Issue updated successfully"` but assignee stays `"Unassigned"` on follow-up GET | **Always use REST** `PUT /rest/api/3/issue/{key}/assignee` for assignment. Never use MCP for this. |
| `transitionJiraIssue` requires assignee pre-set | Fails with `"Field Assignee is required"` even if ticket was just "assigned" via MCP | Pre-verify assignee via REST GET before calling any transition (MCP or REST) |
| `editJiraIssue` label update works | Labels (e.g. `ClopsManualTriage`) **do** persist correctly via MCP `update={"labels": [{"add": "..."}]}` | No workaround needed — MCP is safe for labels |
| `searchJiraIssuesUsingJql` returns `total: -1` | The `total` field is always `-1`; pagination uses `next_page_token` / `isLast` | Same pattern as REST fallback — never rely on `total`, use `isLast` |
| `searchJiraIssuesUsingJql` with `filter=55922` returns empty | MCP returned 0 issues for filter 55922 even when tickets exist in Jira — observed in live execution | Fall back to explicit `SERRAVIEW_SCOPE_JQL` hardcoded in Step 3. **Never widen to unscoped `project = CM` queries.** |

---

### MCP tool → REST API mapping

| Operation | MCP Tool (preferred) | REST API fallback |
|---|---|---|
| Search tickets by JQL | `Atlassian:searchJiraIssuesUsingJql` | `POST /rest/api/3/search/jql` |
| Fetch single issue | `Atlassian:getJiraIssue` | `GET /rest/api/3/issue/{key}` |
| **Assign issue** | ⚠️ **REST only** — see quirk note below | `PUT /rest/api/3/issue/{key}/assignee` |
| Transition issue | `Atlassian:transitionJiraIssue` | `POST /rest/api/3/issue/{key}/transitions` |
| Add label | `Atlassian:editJiraIssue` (labels field) | `PUT /rest/api/3/issue/{key}` |
| Add comment | `Atlassian:addCommentToJiraIssue` | `POST /rest/api/3/issue/{key}/comment` |
| Fetch filter JQL | `Atlassian:fetchAtlassian` (filter ARI) | `GET /rest/api/3/filter/55922` |
| Get transitions | `Atlassian:getTransitionsForJiraIssue` | `GET /rest/api/3/issue/{key}/transitions` |

> ⚠️ **Known MCP quirk — assignee does NOT persist via `editJiraIssue`:**
> Confirmed in live execution: `Atlassian:editJiraIssue` with `fields={"assignee": {"accountId": "..."}}` returns `"Issue updated successfully"` but a follow-up `getJiraIssue` call shows `"Unassigned"` — the assignment silently fails. As a direct consequence, `transitionJiraIssue` then fails with `"Field Assignee is required"`. **Always use the REST API for assignment** (`PUT /rest/api/3/issue/{key}/assignee`), even when MCP is otherwise available. Do not attempt to detect or work around this at runtime — just skip MCP for this operation entirely.
>
> Similarly, **`Atlassian:transitionJiraIssue` requires the assignee field to be already set**. Since assignment must go through REST, always complete the REST assign call and verify the assignee is non-empty before calling the transition (MCP or REST).

### Per-call fallback pattern

Wrap every MCP call so that a failure automatically retries via REST:

```python
def jira_search(jql, fields=None):
    """Search Jira issues — MCP first, REST fallback."""
    if MCP_AVAILABLE:
        try:
            result = call_mcp_tool("Atlassian:searchJiraIssuesUsingJql", jql=jql)
            # MCP returns issues directly; normalise to the same shape as REST
            return normalise_mcp_search(result)
        except Exception as e:
            print(f"MCP search failed ({e}), falling back to REST", file=sys.stderr)
    # REST fallback — always include fields: ["summary", "assignee"]
    return rest_search_jql(jql, fields=fields or ["summary", "assignee"])


def jira_get_issue(key, fields=None):
    """Fetch a single issue — MCP first, REST fallback."""
    if MCP_AVAILABLE:
        try:
            result = call_mcp_tool("Atlassian:getJiraIssue", issueIdOrKey=key)
            return normalise_mcp_issue(result)
        except Exception as e:
            print(f"MCP getIssue({key}) failed ({e}), falling back to REST", file=sys.stderr)
    field_str = ",".join(fields or ["summary","assignee","priority","labels","description","status","updated"])
    return rest_get_issue(key, fields=field_str)


def jira_assign(key, account_id):
    """Assign an issue — always REST (MCP editJiraIssue silently fails for assignee)."""
    # NOTE: Do NOT attempt MCP for assignment. Atlassian:editJiraIssue accepts the call
    # but the assignee field does not persist on this tenant. REST is the only reliable path.
    rest_assign(key, account_id)
    # Verify the assignment actually took (guard against silent failure)
    verify = rest_get_issue(key, fields="assignee")
    if verify.get("fields", {}).get("assignee") is None:
        raise RuntimeError(f"Assignment of {key} to {account_id} did not persist after REST call")


def jira_transition(key, transition_id="31"):
    """Transition an issue — MCP first, REST fallback.
    
    IMPORTANT: Atlassian:transitionJiraIssue will fail with 'Field Assignee is required'
    if the issue is unassigned. Always call jira_assign() and verify before calling this.
    """
    # Pre-check: confirm assignee is set before attempting transition
    issue = rest_get_issue(key, fields="assignee")
    if issue.get("fields", {}).get("assignee") is None:
        raise RuntimeError(f"Cannot transition {key}: assignee is not set. Call jira_assign() first.")
    if MCP_AVAILABLE:
        try:
            call_mcp_tool("Atlassian:transitionJiraIssue",
                          issueIdOrKey=key,
                          transitionId=transition_id)
            return
        except Exception as e:
            print(f"MCP transition({key}) failed ({e}), falling back to REST", file=sys.stderr)
    rest_transition(key, transition_id)


def jira_add_label(key, label):
    """Add a label — MCP first, REST fallback."""
    if MCP_AVAILABLE:
        try:
            call_mcp_tool("Atlassian:editJiraIssue",
                          issueIdOrKey=key,
                          update={"labels": [{"add": label}]})
            return
        except Exception as e:
            print(f"MCP addLabel({key}, {label}) failed ({e}), falling back to REST", file=sys.stderr)
    rest_add_label(key, label)
```

### MCP response normalisation

MCP responses may differ in shape from raw REST JSON. Normalise before passing to shared logic:

```python
def normalise_mcp_search(mcp_result):
    """Normalise MCP search result to match REST /search/jql shape."""
    # MCP may return a list directly or wrap in {"issues": [...]}
    if isinstance(mcp_result, list):
        return mcp_result
    return mcp_result.get("issues", [])


def normalise_mcp_issue(mcp_result):
    """Normalise a single MCP issue to match REST GET /issue/{key} shape."""
    # MCP typically returns the issue dict directly — pass through if fields present
    if "fields" in mcp_result:
        return mcp_result
    # Some MCP tools flatten fields to top level — re-wrap for compatibility
    key = mcp_result.get("key") or mcp_result.get("id")
    return {"key": key, "id": mcp_result.get("id"), "fields": mcp_result}
```

> **Tenant quirks still apply when using REST fallback.** All constraints in the REST API Patterns section below (fields array restriction, cursor pagination, ADF extraction, etc.) apply to REST fallback calls only — MCP tools are not subject to those raw API constraints.

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
- If a required environment variable (`SV_JIRA_BASE_URL`, `SV_JIRA_EMAIL`, `SV_JIRA_API_TOKEN`) is missing AND MCP is also unavailable → print a clear error to stderr and exit with code 1. If MCP is available, the REST env vars are used only by fallback helpers; their absence is non-fatal unless MCP also fails.
- If the triage script crashes or encounters a KeyError, AttributeError, or any unhandled exception → let the exception propagate (do NOT catch it with a broad `except Exception` that swallows it). The traceback is more useful than a silent demo.
- If any step fails → log the failure to stderr and continue with remaining steps where possible; never substitute fake/mock data for real Jira results.
- Never create a `demo_triage.py`, `demo_output.py`, or any simulation script. Real actions from real Jira API only.
- If the real triage produces no assignments and no stale tickets, that IS a valid result — report it honestly in Channel 1 ("No new Serraview tickets in filter 55922"). Do not invent example output to make the run look more productive. **Do NOT widen the scope to pull non-Serraview CM tickets just to have something to show.**

**Script output — stdout safety rules:**
The agent runtime parses the script's stdout as a JSON stream. Certain output patterns crash the runtime with `json: cannot unmarshal string into Go value of type map[string]interface {}`. To prevent this:

- **Redirect all script progress logging to stderr**, not stdout: use `print("...", file=sys.stderr)` for all status messages, debug output, and intermediate results.
- **Never print raw JSON payloads to stdout** — the Adaptive Card payload, Jira API responses, or any dict/list printed via `print(json.dumps(...))` will crash the runtime if it reaches stdout.
- **stdout is reserved for the final triage summary only** — the Step 6 markdown table output. Everything else goes to stderr.
- **Avoid printing strings containing bare `{` or `}` to stdout** unless they are inside a markdown code block context.

```python
import sys

# All progress/debug output → stderr (safe, never parsed by runtime)
print("Fetching filter 55922...", file=sys.stderr)
print(f"Found {len(issues)} tickets", file=sys.stderr)

# Final summary → stdout (only markdown tables, no raw JSON)
print("## Triage Complete")
print("| Ticket | Summary | Assigned To |")
```

## REST API Patterns (Fallback — used when MCP is unavailable or a call fails)

**Always use `/rest/api/3/`** — do not fall back to `/rest/api/2/`. A 410 on the old `/rest/api/3/search` or `/rest/api/2/search` endpoint means those are deprecated — always use `POST /rest/api/3/search/jql` instead.

The REST helper functions used by the fallback wrappers in the MCP Strategy section:

```python
import requests, base64, os, sys

def _rest_headers():
    email = os.environ["SV_JIRA_EMAIL"]
    token = os.environ["SV_JIRA_API_TOKEN"]
    creds = base64.b64encode(f"{email}:{token}".encode()).decode()
    return {"Authorization": f"Basic {creds}", "Content-Type": "application/json"}

def rest_get_issue(key, fields="summary,assignee,priority,labels,description,status,updated"):
    base_url = os.environ["SV_JIRA_BASE_URL"]
    r = requests.get(f"{base_url}/rest/api/3/issue/{key}?fields={fields}", headers=_rest_headers())
    r.raise_for_status()
    return r.json()

def rest_assign(key, account_id):
    base_url = os.environ["SV_JIRA_BASE_URL"]
    r = requests.put(f"{base_url}/rest/api/3/issue/{key}/assignee",
                     headers=_rest_headers(), json={"accountId": account_id})
    r.raise_for_status()

def rest_transition(key, transition_id="31"):
    base_url = os.environ["SV_JIRA_BASE_URL"]
    r = requests.post(f"{base_url}/rest/api/3/issue/{key}/transitions",
                      headers=_rest_headers(), json={"transition": {"id": transition_id}})
    r.raise_for_status()

def rest_add_label(key, label):
    base_url = os.environ["SV_JIRA_BASE_URL"]
    r = requests.put(f"{base_url}/rest/api/3/issue/{key}",
                     headers=_rest_headers(), json={"update": {"labels": [{"add": label}]}})
    r.raise_for_status()
```

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

First, fetch filter 55922 to confirm the Serraview category condition.

Use `jira_search` (MCP-first wrapper) or, if you need the raw filter JQL, call:
- **MCP:** `Atlassian:fetchAtlassian` with the filter ARI (`ari:cloud:jira::<cloudId>:filter/55922`)
- **REST fallback:** `GET /rest/api/3/filter/55922` → extract `.jql`

```bash
# REST fallback only
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

**Use `jira_search(jql)` for both queries** — it tries MCP first and falls back to REST automatically. When using the REST fallback, always include `"fields": ["summary", "assignee"]` in the search payload; omitting returns only `id` with no `key` or assignee data. Use `issue["fields"]["assignee"]["accountId"]` to attribute each result to a team member. Paginate using `nextPageToken` / `isLast` (not `startAt`).

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

Get all tickets from filter 55922. **Every ticket processed by this skill MUST belong to the Serraview category. This is a hard scope boundary — never process tickets outside it.**

#### Mandatory Serraview scope JQL (hardcoded fallback)

```
SV_SCOPE_JQL = (
    'project = CM'
    ' AND status = "New Issue"'
    ' AND "Category and Sub-category[Select List (cascading)]" IN cascadeOption("Serraview")'
    ' AND priority != Highest'   # S1 excluded — handled by separate trigger
    ' ORDER BY created ASC'
)
```

This JQL is the guaranteed Serraview-scoped query. Use it whenever filter 55922 is unavailable or returns zero results unexpectedly.

#### Fetch strategy — filter first, explicit JQL fallback

```python
SERRAVIEW_SCOPE_JQL = (
    'project = CM AND status = "New Issue"'
    ' AND "Category and Sub-category[Select List (cascading)]" IN cascadeOption("Serraview")'
    ' AND priority != Highest ORDER BY created ASC'
)

def fetch_filter_tickets():
    """
    Primary: filter=55922 via MCP or REST.
    Fallback: explicit Serraview-scoped JQL if filter returns 0 results or errors.
    NEVER fall back to an unscoped CM query.
    """
    issues = []
    try:
        issues = jira_search("filter=55922")
        print(f"Filter 55922 returned {len(issues)} tickets", file=sys.stderr)
    except Exception as e:
        print(f"Filter 55922 failed ({e}), switching to explicit Serraview JQL", file=sys.stderr)

    if not issues:
        # Filter returned empty or failed — use the explicit scope JQL
        # This is safe: the JQL is hardcoded to Serraview category only
        print("Using explicit Serraview-scoped JQL fallback", file=sys.stderr)
        issues = jira_search(SERRAVIEW_SCOPE_JQL)
        print(f"Explicit JQL returned {len(issues)} tickets", file=sys.stderr)

    # ── POST-FETCH SCOPE GUARD ─────────────────────────────────────────────
    # Even if filter 55922 returned results, verify each ticket belongs to Serraview.
    # This guards against filter misconfiguration or MCP returning stale/wrong scope.
    safe_issues = []
    rejected = []
    for issue in issues:
        key = issue.get("key", "UNKNOWN")
        detail = jira_get_issue(key, fields="summary,assignee,priority,labels,description,status,updated,customfield_10188")
        category_field = detail.get("fields", {}).get("customfield_10188")  # Category cascade field
        category_str = str(category_field).lower() if category_field else ""
        if "serraview" not in category_str:
            # Double-check via summary/description keyword as a secondary signal
            summary = detail.get("fields", {}).get("summary", "").lower()
            desc = str(detail.get("fields", {}).get("description", "") or "").lower()
            sv_keywords = ["serraview", "svlive", "sv-", "serra"]
            if not any(kw in summary or kw in desc for kw in sv_keywords):
                print(f"SCOPE REJECT: {key} does not belong to Serraview — skipping", file=sys.stderr)
                rejected.append(key)
                continue
        safe_issues.append((key, detail))

    if rejected:
        print(f"Scope guard rejected {len(rejected)} non-Serraview tickets: {rejected}", file=sys.stderr)

    return safe_issues
```

> **Critical:** If `fetch_filter_tickets()` returns zero tickets after the scope guard, that is a valid result — report "No new Serraview tickets in filter 55922" in Channel 1. Do NOT widen the scope to include non-Serraview tickets.

**Two-call pattern (REST fallback within `jira_search`)** — only `["summary", "assignee"]` are safe in the search `fields` array on this tenant; all other fields cause 400 errors. Get keys from search, then fetch per-issue detail separately via `jira_get_issue()`.

**Bucket 1 — Already Assigned** (assignee IS NOT EMPTY in the fetched issue detail):
- DO NOT reassign
- Call `jira_transition(issueKey, "31")` to move to Approved (MCP first, REST fallback)
- Output: Auto-Transitioned (Already Assigned)

> Filter 55922 targets "New Issue" status, non-S1 Serraview tickets. S1 tickets are handled by a separate event-based trigger and will not appear here. If a non-Serraview ticket somehow passes the scope guard, skip it and log to Skipped with reason "Out of scope — not Serraview".

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
Call `jira_add_label(issueKey, "ClopsManualTriage")` (MCP first, REST fallback).
Do NOT assign or transition.

**Routable tickets — always assign BEFORE transitioning:**
1. Call `jira_assign(issueKey, account_id)` — always uses `PUT /rest/api/3/issue/{key}/assignee` (REST only; MCP silently fails for assignee on this tenant)
2. `jira_assign()` internally verifies the assignee persisted via a follow-up REST GET; if not, it raises `RuntimeError` — catch this, log to Skipped (API Error), and skip to the next ticket without transitioning
3. Call `jira_transition(issueKey, "31")` — attempts MCP first (`Atlassian:transitionJiraIssue`), falls back to REST. The wrapper pre-checks that assignee is set; it will refuse to transition and raise if the field is empty (guards against the MCP assign silent-failure scenario)

> Order matters: assign → verify → transition. **Never call the transition before confirming the assignee field is non-empty.** `Atlassian:transitionJiraIssue` returns `"Field Assignee is required"` if assignee is unset, which is misleading — the real fix is always to repair the assignment first.

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

### Skipped (Out of Scope)
| Ticket | Summary | Reason |

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
JQL: project = CM AND assignee = "<accountId>" AND status IN ("New Issue", "In Progress") ORDER BY updated ASC
```

**Use `jira_search(jql)` (MCP-first wrapper)** for each per-person stale query. When using the REST fallback, always include `"fields": ["summary", "assignee"]` in the search payload — omitting returns only `id` with no `key`. Paginate using `nextPageToken` / `isLast`. For each returned issue `key`, call `jira_get_issue(key, fields="summary,priority,updated,labels,status")` (MCP first, REST fallback) to get the fields needed for SLA calculation.

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

> **Note:** Teams webhook notifications always use direct HTTP POST via `requests` — MCP has no Teams webhook tool. The MCP strategy applies only to Jira operations above.

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

> **Note:** Same as Channel 1 — Teams webhook calls always use direct HTTP POST via `requests`, not MCP.

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
- **Serraview scope is mandatory** — only process tickets where Category = Serraview. If filter 55922 is unavailable, fall back to the hardcoded `SERRAVIEW_SCOPE_JQL` in Step 3. Never fall back to unscoped `project = CM` queries.
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
2. Probe MCP availability (`Atlassian:getAccessibleAtlassianResources`) — set `MCP_AVAILABLE`
3. Note current IST time for shift-aware routing
4. Query filter 55922 and current workloads via `jira_search()` (MCP first, REST fallback)
5. Transition already-assigned tickets (Bucket 1) via `jira_transition()` wrapper
6. Analyse and route unassigned tickets (Buckets 2 & 3)
   - Boeing keyword in any ticket → Michael Soga regardless of shift
   - DB tickets normally routed to Yuan Yang → escalate to Mridul Raina, then Deevanshu Gakhar
7. Assign via `jira_assign()`, then transition via `jira_transition()` for each routable ticket
8. Send Channel 1 (always via `requests` POST — not MCP); skip stale detection since `firstRunOfDay` not passed (defaults to `false`)
9. Send Channel 2 only if assignments were made

> To include stale detection and daily sync alerts, pass `firstRunOfDay=true` on the first run of each day.
