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

For each team member, query their open tickets:
```
JQL: project = CM AND assignee = "<accountId>" AND status NOT IN (Done, Cancelled)
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

POST `{"text": "<html>"}` with `Content-Type: application/json`.
Use `<br>` for every line break. Use `<b>` for bold headers. Do NOT use `\n`.

```
<b>📊 Serraview Workload Summary - {YYYY-MM-DD}</b><br>
<br>
{IF no new tickets from filter 55922:}
No new tickets in filter 55922.<br>
{ELSE list each assignment/auto-transition on its own line:}
✅ CM-XXXXX → Assignee — Summary<br>
<br>
{IF stale tickets exist (firstRunOfDay=true only):}
<b>🕐 STALE / SLA-BREACHED TICKETS</b><br>
<br>
<b>Person Name:</b><br>
• [S{n}] CM-XXXXX - Summary | Last update: {X}h ago (SLA: {Y}h)<br>
• [S{n}] CM-XXXXX - Summary | Last update: {X}d ago (SLA: {Y}h)<br>
<br>
<b>Next Person:</b><br>
• ...<br>
<br>
{IF manual triage tickets exist:}
<b>⚠️ MANUAL TRIAGE REQUIRED</b><br>
• CM-XXXXX — Summary (reason)<br>
<br>
{IF API errors:}
<b>❌ ERRORS</b><br>
• CM-XXXXX: error message<br>
<br>
{IF anyone over capacity:}
<b>⚠️ OVER CAPACITY</b><br>
<br>
• Person: {current}/{max} ⚠️  - over by {N} ticket(s)<br>
<br>
<b>✅ ON TRACK</b><br>
<br>
• Person: {current}/{max} ({%})<br>
• Person: {current}/{max} ({%})<br>
...<br>
<br>
<at>Hritik Chaudhary</at> <at>Shilpa Goyal</at>
```

**Channel 1 Rules:**
- Always send; always include workload (OVER CAPACITY + ON TRACK)
- Always end with `<at>Hritik Chaudhary</at> <at>Shilpa Goyal</at>`
- Stale section: only when `firstRunOfDay=true` and stale tickets found
- Manual triage + errors: omit sections if empty
- Triage result: "No new tickets" if nothing in filter; otherwise list assignments

## Channel 2 Payload Format

POST `{"text": "<html>"}` with `Content-Type: application/json`.
Use `<br>` for every line break. Use `<b>` for bold headers.

```
<b>📋 Serraview CloudOps Daily Sync - {YYYY-MM-DD}</b><br>
<br>
{IF assignments were made:}
<b>✅ New Assignments</b><br>
<br>
• CM-XXXXX → Assignee — Summary (reason)<br>
• CM-XXXXX → Assignee — Summary (reason)<br>
<br>
{IF stale tickets and firstRunOfDay=true:}
<b>🕐 STALE / SLA-BREACHED TICKETS</b><br>
<br>
<b>Person Name:</b><br>
• [S{n}] CM-XXXXX - Summary | Last update: {X}h ago (SLA: {Y}h)<br>
<br>
<b>Next Person:</b><br>
• ...<br>
```

**Channel 2 Rules:**
- Send ONLY if at least one section has content
- If no assignments AND no stale tickets → Do NOT send
- Stale tickets: same SLA-based detection as Channel 1
- Stale tickets included only when `firstRunOfDay=true`
