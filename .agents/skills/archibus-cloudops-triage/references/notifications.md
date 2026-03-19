# Notifications — Archibus CloudOps

## Channel 1: Triage Alerts + Workload Summary

```yaml
Webhook URL: https://default454af8f364734bce8aa5f9403e1d12.71.environment.api.powerplatform.com:443/powerautomate/automations/direct/workflows/b69181a7858f4aa5b6b3d8f218dab4aa/triggers/manual/paths/invoke?api-version=1&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=OnQTgTG83xYTGEKNEQkJDuPezqwtyAiKjzDdANNYxbg-

Content: workload summary + stale/SLA-breached tickets + manual triage alerts + errors
Send: Always (every triage run)
```

## Channel 2: Daily Sync (All Team Members)

```yaml
Webhook URL: PLACEHOLDER — add Archibus daily sync Teams webhook here

Content: assignments made this run + stale/SLA-breached tickets
Send: Only if assignments were made OR stale tickets exist (and firstRunOfDay=true)
```

## Notification Recipients

```yaml
Tag in alerts:
  - name: Nipun Sahni
    email: nipun.sahni@eptura.com
  - name: Maninder Singh
    email: maninder.singh@eptura.com
```

## Stale / SLA-Breached Ticket Detection

Only check **active** tickets — status must be "New Issue" or "In Progress" with Archibus category filter:
```
JQL: project = CM AND assignee = "<accountId>" AND status IN ("New Issue", "In Progress")
     AND "Category and Sub-category[Select List (cascading)]" = Archibus
Fields: summary, priority, updated, labels, comment, status
```

For each ticket, compute elapsed time since `updated` and flag as stale if:

**S1 / S2 (Highest/High priority) — calendar hours, 24/7:**
- Flag if not updated in > 4 calendar hours → SLA: 4h
- Weekends do NOT reduce the elapsed time (S1/S2 are critical, no weekend grace)

**S3 / S4 (Medium/Low priority) — business hours only (Mon–Fri):**
- Flag if not updated in > 24 business hours → SLA: 24h
- **Weekend adjustment**: subtract 24h for each Saturday and Sunday that falls within the elapsed period
- Calculation: `business_hours = calendar_hours - (weekend_days_in_range × 24)`
- A ticket updated on Friday will NOT be stale on Monday unless `business_hours > 24`
- Example: updated Friday 3 PM, checked Monday 9 AM = 42 calendar hours − 48h (Sat+Sun) = −6h → NOT stale
- Example: updated Thursday 9 AM, checked Monday 9 AM = 96 calendar hours − 48h = 48h > 24h → STALE

To count weekend days in range: count distinct calendar days (midnight-to-midnight) between `updated` and `now` whose weekday is Saturday (5) or Sunday (6).

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

Format each stale ticket as:
`[S{n}] {key} - {summary} | {current_status} | Last update: {X}h ago (SLA: {Y}h)`
`➡️ Next: {inferred_next_step}`

For S3/S4 show adjusted business hours in the "Last update" field (not raw calendar hours).
Include the ticket's current Jira status in the `{current_status}` field (e.g. `In Progress`, `New Issue`).
The `{inferred_next_step}` is a short, actionable phrase derived from the ticket description and last 5 comments (see SKILL.md Step 7a for inference rules).

Only include team members who have at least one stale ticket after exclusions.
Only run stale detection when `firstRunOfDay=true` (default).

## Channel 1 Payload Format

POST `{"text": "..."}` using **Python requests** (NOT curl) so newlines encode correctly.
Do NOT use HTML tags. The flow renders `text` as plain text.

IMPORTANT: Teams only renders paragraph breaks (blank lines between items).
Do NOT rely on single `\n` for line breaks — use `\n\n` between every distinct line.
Build the text as a list of sections, then join with `\n\n`:

```python
import requests

parts = []
parts.append("📊 Archibus Workload Summary - {YYYY-MM-DD}")

# Triage result
parts.append("No new tickets in filter 55937.")  # OR each assignment as its own part:
# parts.append("✅ CM-XXXXX → Assignee — Summary")

# Stale section (firstRunOfDay=true only, if any stale tickets)
parts.append("🕐 STALE / SLA-BREACHED TICKETS")
parts.append("Person Name:")         # person name on its own line
parts.append("• [S{n}] CM-XXXXX - Summary | {Status} | Last update: {X}h ago (SLA: {Y}h)")
parts.append("➡️ Next: {inferred_next_step}")
parts.append("Next Person:")
parts.append("• [S{n}] CM-XXXXX - Summary | {Status} | Last update: {X}h ago (SLA: {Y}h)")
parts.append("➡️ Next: {inferred_next_step}")
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

parts.append("@Nipun Sahni @Maninder Singh")

text = "\n\n".join(parts)   # double newline = paragraph break in Teams
requests.post(webhook_url, json={"text": text})
```

**Channel 1 Rules:**
- Always send; always include workload (OVER CAPACITY / AT CAPACITY / ON TRACK — omit empty tiers)
- Always end with `@Nipun Sahni @Maninder Singh`
- Stale section: only when `firstRunOfDay=true` and stale tickets found
- Manual triage + errors: omit sections if empty
- Triage result: "No new tickets" if nothing in filter; otherwise list each assignment

## Channel 2 Payload Format

POST `{"text": "..."}` using **Python requests** (NOT curl).
Do NOT use HTML tags. Use actual newline characters (`\n`) in the Python string.

```python
import requests

parts = []
parts.append("📋 Archibus CloudOps Daily Sync - {YYYY-MM-DD}")

# Assignments (if any)
parts.append("✅ New Assignments")
parts.append("• CM-XXXXX → Assignee — Summary (reason)")
# ...(one parts.append per assignment)

# Stale tickets (firstRunOfDay=true only)
parts.append("🕐 STALE / SLA-BREACHED TICKETS")
parts.append("Person Name:")
parts.append("• [S{n}] CM-XXXXX - Summary | {Status} | Last update: {X}h ago (SLA: {Y}h)")
parts.append("➡️ Next: {inferred_next_step}")
parts.append("Next Person:")
parts.append("• [S{n}] CM-XXXXX - Summary | {Status} | Last update: {X}h ago (SLA: {Y}h)")
parts.append("➡️ Next: {inferred_next_step}")
# ...(one parts.append per person name, then one pair of parts.append per ticket: ticket line + next step line)

text = "\n\n".join(parts)   # double newline = paragraph break in Teams
requests.post(webhook_url, json={"text": text})
```

**Channel 2 Rules:**
- Send ONLY if at least one section has content
- If no assignments AND no stale tickets → Do NOT send
- Stale tickets: same SLA-based detection as Channel 1 (active tickets only, weekend-adjusted for S3/S4)
- Stale tickets included **only when `firstRunOfDay=true`** (default is `false`) — this ensures stale alerts appear at most once per day even when the skill runs multiple times
