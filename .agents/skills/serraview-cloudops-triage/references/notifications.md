# Notifications

## Channel 1: Triage Alerts + Workload Summary

```yaml
Webhook URL: https://default454af8f364734bce8aa5f9403e1d12.71.environment.api.powerplatform.com:443/powerautomate/automations/direct/workflows/2f988e1527cc416284dc27d3df468b58/triggers/manual/paths/invoke?api-version=1&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=LIjoyCCexv6xBCuFMnWU6mT73Sd2bkvOSMBHN85pf1Q

Notify on:
  - Any tickets marked for manual triage (ClopsManualTriage label)
  - Any API errors or transition failures
  - Workload summary (always include after each triage run)

Do NOT notify on:
  - Successful assignments (even if over capacity)
```

## Channel 2: Daily Sync (All Team Members)

```yaml
Webhook URL: https://default454af8f364734bce8aa5f9403e1d12.71.environment.api.powerplatform.com:443/powerautomate/automations/direct/workflows/7a239907b2cd4736982336aa81a7bbeb/triggers/manual/paths/invoke?api-version=1&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=w_BZ2oEcAxy07O45y-Is9afERVxBjDIbhBcelb91Oxg

Notify on:
  - All assignments made in this triage run
  - Stale tickets (open tickets with no status change for > 3 days)

Do NOT notify on:
  - Manual triage alerts
  - API errors
  - Workload summary
```

## Notification Recipients

```yaml
Tag in alerts:
  - name: Hritik Chaudhary
    email: hritik.chaudhary@eptura.com
  - name: Shilpa Goyal
    email: shilpa.goyal@eptura.com
```

## Channel 1 Payload Format

Send a JSON body with a single `text` field containing the pre-formatted message as a plain text / markdown string.

```json
{"text": "<formatted message — see template below>"}
```

### Channel 1 Text Template

Build the `text` value using this structure (omit empty sections):

```
🔔 Serraview CloudOps Triage — <date>

⚠️ Manual Triage Required
• CM-XXXXX — <summary> (<reason>)
  Link: https://eptura.atlassian.net/browse/CM-XXXXX

❌ Errors
• CM-XXXXX: <error message>

📊 Workload Summary
• Yuan Yang: 4/5 (80%) ✅
• Ankit Kumar Sinha: 8/10 (80%) ✅
• Mashkoor Ahmad: 6/5 ⚠️ OVER CAPACITY
...(all team members)

@Hritik Chaudhary @Shilpa Goyal
```

- If no manual triage tickets: omit the ⚠️ section
- If no errors: omit the ❌ section
- Always include 📊 Workload Summary
- Always include the @mention line at the end

## Channel 1 Send Rules

- Always send (workload is always included)
- Manual triage and error sections are optional (omit if empty)

## Channel 2 Payload Format

Send a JSON body with a single `text` field containing the pre-formatted message.

```json
{"text": "<formatted message — see template below>"}
```

### Channel 2 Text Template

```
📋 Serraview CloudOps Daily Sync — <date>

✅ New Assignments
• CM-XXXXX → <Assignee> — <summary> (<reason>)
  Link: https://eptura.atlassian.net/browse/CM-XXXXX

🕐 Stale Tickets (>3 days no update)
• CM-XXXXX (<Assignee>, <N> days) — <summary>
  Link: https://eptura.atlassian.net/browse/CM-XXXXX
```

- If no assignments: omit ✅ section
- If no stale tickets (or not firstRunOfDay): omit 🕐 section

## Channel 2 Send Rules

- Send ONLY if assignments were made OR stale tickets exist (and firstRunOfDay=true)
- If both sections would be empty → Do NOT send
- `stale_tickets`: query `project = CM AND assignee IS NOT EMPTY AND status NOT IN (Done, Cancelled) AND updated <= -3d`
- **Stale tickets once per day only** — include only when `firstRunOfDay=true` (default). Skip on subsequent same-day runs.
