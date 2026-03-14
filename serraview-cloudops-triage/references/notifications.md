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
  - name: Hritik Chaudhary
    email: hritik.chaudhary@eptura.com
  - name: Shilpa Goyal
    email: shilpa.goyal@eptura.com
```

## Stale / SLA-Breached Ticket Detection

Only check **active** tickets — status must be "New Issue" or "In Progress":
```
JQL: project = CM AND assignee = "<accountId>" AND status IN ("New Issue", "In Progress")
Fields: summary, priority, updated
```
For each ticket, compute hours since `updated` and flag as stale if:
- S1 or S2 (priority Highest/High): not updated in > 4 hours → SLA: 4h
- S3 (priority Medium): not updated in > 24 hours → SLA: 24h
- S4 (priority Low/Lowest): not updated in > 24 hours → SLA: 24h

Format each stale ticket as: `[S{n}] {key} - {summary} | Last update: {X}h ago (SLA: {Y}h)`
Only include team members who have at least one stale ticket.
Only run stale detection when `firstRunOfDay=true` (default).

## Channel 1 Payload Format

POST `{"text": "..."}` using **Python requests** (NOT curl) so newlines encode correctly.
Do NOT use HTML tags. The flow renders `text` as plain text.
Use actual newline characters (`\n`) in the Python string for line breaks.

```python
import requests, json

lines = []
lines.append("📊 Serraview Workload Summary - {YYYY-MM-DD}")
lines.append("")
# Triage result
lines.append("No new tickets in filter 55922.")  # or list each assignment
lines.append("")
# Stale section (firstRunOfDay=true only)
lines.append("🕐 STALE / SLA-BREACHED TICKETS")
lines.append("")
lines.append("Person Name:")
lines.append("• [S{n}] CM-XXXXX - Summary | Last update: {X}h ago (SLA: {Y}h)")
lines.append("• [S{n}] CM-XXXXX - Summary | Last update: {X}d ago (SLA: {Y}h)")
lines.append("")
lines.append("Next Person:")
lines.append("• ...")
lines.append("")
# Manual triage (if any)
lines.append("⚠️ MANUAL TRIAGE REQUIRED")
lines.append("• CM-XXXXX — Summary (reason)")
lines.append("")
# Errors (if any)
lines.append("❌ ERRORS")
lines.append("• CM-XXXXX: error message")
lines.append("")
# Workload
lines.append("⚠️ OVER CAPACITY")
lines.append("")
lines.append("• Person: {current}/{max} ⚠️  - over by {N} ticket(s)")
lines.append("")
lines.append("✅ ON TRACK")
lines.append("")
lines.append("• Person: {current}/{max} ({%})")
lines.append("• Person: {current}/{max} ({%})")
lines.append("")
lines.append("@Hritik Chaudhary @Shilpa Goyal")

text = "\n".join(lines)
requests.post(webhook_url, json={"text": text})
```

**Channel 1 Rules:**
- Always send; always include workload (OVER CAPACITY + ON TRACK)
- Always end with `@Hritik Chaudhary @Shilpa Goyal`
- Stale section: only when `firstRunOfDay=true` and stale tickets found
- Manual triage + errors: omit sections if empty
- Triage result: "No new tickets" if nothing in filter; otherwise list each assignment

## Channel 2 Payload Format

POST `{"text": "..."}` using **Python requests** (NOT curl).
Do NOT use HTML tags. Use actual newline characters (`\n`) in the Python string.

```python
lines = []
lines.append("📋 Serraview CloudOps Daily Sync - {YYYY-MM-DD}")
lines.append("")
# Assignments (if any)
lines.append("✅ New Assignments")
lines.append("")
lines.append("• CM-XXXXX → Assignee — Summary (reason)")
lines.append("• CM-XXXXX → Assignee — Summary (reason)")
lines.append("")
# Stale tickets (firstRunOfDay=true only)
lines.append("🕐 STALE / SLA-BREACHED TICKETS")
lines.append("")
lines.append("Person Name:")
lines.append("• [S{n}] CM-XXXXX - Summary | Last update: {X}h ago (SLA: {Y}h)")
lines.append("")
lines.append("Next Person:")
lines.append("• ...")

text = "\n".join(lines)
requests.post(webhook_url, json={"text": text})
```

**Channel 2 Rules:**
- Send ONLY if at least one section has content
- If no assignments AND no stale tickets → Do NOT send
- Stale tickets: same SLA-based detection as Channel 1 (active tickets only)
- Stale tickets included only when `firstRunOfDay=true`
