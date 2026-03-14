# Notifications

## Channel 1: Triage Alerts + Workload Summary

```yaml
Webhook URL: https://default454af8f364734bce8aa5f9403e1d12.71.environment.api.powerplatform.com:443/powerautomate/automations/direct/workflows/2f988e1527cc416284dc27d3df468b58/triggers/manual/paths/invoke?api-version=1&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=LIjoyCCexv6xBCuFMnWU6mT73Sd2bkvOSMBHN85pf1Q

Content: workload summary + stale/SLA-breached tickets + manual triage alerts + errors
Send: Always (every triage run)
```

## Channel 2: Daily Sync (All Team Members)

```yaml
Webhook URL: https://default454af8f364734bce8aa5f9403e1d12.71.environment.api.powerplatform.com:443/powerautomate/automations/direct/workflows/7a239907b2cd4736982336aa81a7bbeb/triggers/manual/paths/invoke?api-version=1&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=w_BZ2oEcAxy07O45y-Is9afERVxBjDIbhBcelb91Oxg

Content: assignments made this run + stale/SLA-breached tickets
Send: Only if assignments were made OR stale tickets exist (and firstRunOfDay=true)
```

## Notification Recipients

```yaml
Tag in alerts:
  - name: Nipun Sahni
    email: nipun.sahni@eptura.com
  - name: Gaurav Kumar
    email: gaurav.kumar@eptura.com
```

## Stale / SLA-Breached Ticket Detection

Only check **active** tickets — status must be "New Issue" or "In Progress":
```
JQL: project = CM AND assignee = "<accountId>" AND status IN ("New Issue", "In Progress")
Fields: summary, priority, updated, labels, comment
```

For each ticket, compute hours since `updated` and flag as stale if:
- S1 or S2 (priority Highest/High): not updated in > 4 hours → SLA: 4h
- S3 (priority Medium): not updated in > 24 hours → SLA: 24h
- S4 (priority Low/Lowest): not updated in > 24 hours → SLA: 24h

### Under-Observation Exclusion

Before adding a ticket to the stale list, check if it should be excluded.
A ticket is **excluded** from stale reporting if BOTH conditions are true:

**Condition A — Ticket is under observation:**
Any of:
- Has a label matching (case-insensitive): `underobservation`, `under-observation`, `under_observation`, `monitoring`, `observation`
- Last comment from the assignee/team member contains any of: `under observation`, `monitoring`, `watching`, `observing`, `keeping an eye`, `will observe`, `under watch`

**Condition B — No pending update request:**
The last comment on the ticket (from anyone, including CS/customer) does NOT contain any of:
`please update`, `any update`, `update?`, `following up`, `looking for update`, `any progress`, `status update`, `can you update`, `please respond`, `waiting for update`, `need an update`, `please provide`

If BOTH A and B are true → **exclude** from stale list (ticket is legitimately quiet).
If A is true but B is false (someone asked for an update) → **include** (team needs to respond).
If A is false → **include** normally (not under observation).

To check comments, fetch the last 3 comments: `GET /rest/api/3/issue/{key}/comment?maxResults=3&orderBy=-created`

Format each stale ticket as: `[S{n}] {key} - {summary} | Last update: {X}h ago (SLA: {Y}h)`
Only include team members who have at least one stale ticket after exclusions.
Only run stale detection when `firstRunOfDay=true` (default).

## Channel 1 Payload Format

POST `{"text": "..."}` using **Python requests** (NOT curl) so newlines encode correctly.
Do NOT use HTML tags. The flow renders `text` as plain text.
Use actual newline characters (`\n`) in the Python string for line breaks.

IMPORTANT: Teams only renders paragraph breaks (blank lines between items).
Do NOT rely on single `\n` for line breaks — use `\n\n` between every distinct line.
Build the text as a list of sections, then join with `\n\n`:

```python
import requests

parts = []
parts.append("📊 Serraview Workload Summary - {YYYY-MM-DD}")

# Triage result
parts.append("No new tickets in filter 55922.")  # OR each assignment as its own part:
# parts.append("✅ CM-XXXXX → Assignee — Summary")

# Stale section (firstRunOfDay=true only, if any stale tickets)
parts.append("🕐 STALE / SLA-BREACHED TICKETS")
parts.append("Person Name:")         # person name on its own line
parts.append("• [S{n}] CM-XXXXX - Summary | Last update: {X}h ago (SLA: {Y}h)")
parts.append("• [S{n}] CM-XXXXX - Summary | Last update: {X}d ago (SLA: {Y}h)")
parts.append("Next Person:")
parts.append("• [S{n}] CM-XXXXX - Summary | Last update: {X}h ago (SLA: {Y}h)")
# ... repeat for each person

# Manual triage (omit section if empty)
parts.append("⚠️ MANUAL TRIAGE REQUIRED")
parts.append("• CM-XXXXX — Summary (reason)")

# Errors (omit section if empty)
parts.append("❌ ERRORS")
parts.append("• CM-XXXXX: error message")

# Workload (always include)
# Counts reflect active tickets only — under-observation tickets (with no pending requests) are excluded
# Three tiers based on load vs maxLoad:
#   OVER CAPACITY  → currentLoad > maxLoad
#   AT CAPACITY    → currentLoad == maxLoad
#   ON TRACK       → currentLoad < maxLoad
parts.append("⚠️ OVER CAPACITY")
parts.append("• Person: {current}/{max} ⚠️  - over by {N} ticket(s)")   # only when N > 0; append " (+{U} obs)" if U > 0
parts.append("🔴 AT CAPACITY")
parts.append("• Person: {current}/{max}")   # append " (+{U} obs)" if U > 0
parts.append("✅ ON TRACK")
parts.append("• Person: {current}/{max} ({%})")   # append " (+{U} obs)" if U > 0
# ...(one parts.append per person; omit a section header entirely if no one falls in that tier)

parts.append("@Nipun Sahni @Gaurav Kumar")

text = "\n\n".join(parts)   # double newline = paragraph break in Teams
requests.post(webhook_url, json={"text": text})
```

**Channel 1 Rules:**
- Always send; always include workload (OVER CAPACITY / AT CAPACITY / ON TRACK — omit empty tiers)
- Always end with `@Nipun Sahni @Gaurav Kumar`
- Stale section: only when `firstRunOfDay=true` and stale tickets found
- Manual triage + errors: omit sections if empty
- Triage result: "No new tickets" if nothing in filter; otherwise list each assignment

## Channel 2 Payload Format

POST `{"text": "..."}` using **Python requests** (NOT curl).
Do NOT use HTML tags. Use actual newline characters (`\n`) in the Python string.

```python
import requests

parts = []
parts.append("📋 Serraview CloudOps Daily Sync - {YYYY-MM-DD}")

# Assignments (if any)
parts.append("✅ New Assignments")
parts.append("• CM-XXXXX → Assignee — Summary (reason)")
parts.append("• CM-XXXXX → Assignee — Summary (reason)")
# ...(one parts.append per assignment)

# Stale tickets (firstRunOfDay=true only)
parts.append("🕐 STALE / SLA-BREACHED TICKETS")
parts.append("Person Name:")
parts.append("• [S{n}] CM-XXXXX - Summary | Last update: {X}h ago (SLA: {Y}h)")
parts.append("Next Person:")
parts.append("• [S{n}] CM-XXXXX - Summary | Last update: {X}h ago (SLA: {Y}h)")
# ...(one parts.append per person name, one per ticket)

text = "\n\n".join(parts)   # double newline = paragraph break in Teams
requests.post(webhook_url, json={"text": text})
```

**Channel 2 Rules:**
- Send ONLY if at least one section has content
- If no assignments AND no stale tickets → Do NOT send
- Stale tickets: same SLA-based detection as Channel 1 (active tickets only)
- Stale tickets included only when `firstRunOfDay=true`
