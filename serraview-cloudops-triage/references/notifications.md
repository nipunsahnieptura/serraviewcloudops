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

```json
{
  "type": "triage_alert",
  "notify": [
    {"name": "Hritik Chaudhary", "email": "hritik.chaudhary@eptura.com"},
    {"name": "Shilpa Goyal", "email": "shilpa.goyal@eptura.com"}
  ],
  "manual_triage": [
    {
      "key": "CM-XXXXX",
      "url": "https://eptura.atlassian.net/browse/CM-XXXXX",
      "summary": "ticket summary",
      "priority": "High",
      "reason": "QA environment - requires manual review"
    }
  ],
  "errors": [
    {"ticket": "CM-XXXXX", "error": "Transition failed"}
  ],
  "workload": [
    {"person": "Yuan Yang", "current": 5, "max": 5, "status": "At limit"},
    {"person": "Ankit Kumar Sinha", "current": 8, "max": 10, "status": "OK"}
  ],
  "timestamp": "<ISO timestamp>"
}
```

## Channel 1 Send Rules

- Always send (workload is always included)
- `manual_triage` and `errors` arrays may be empty if nothing flagged

## Channel 2 Payload Format

```json
{
  "type": "daily_sync_summary",
  "assignments": [
    {
      "key": "CM-XXXXX",
      "url": "https://eptura.atlassian.net/browse/CM-XXXXX",
      "summary": "ticket summary",
      "assigned_to": "Ankit Kumar Sinha",
      "reason": "S2 High, Serraview domain"
    }
  ],
  "stale_tickets": [
    {
      "key": "CM-XXXXX",
      "url": "https://eptura.atlassian.net/browse/CM-XXXXX",
      "summary": "ticket summary",
      "assignee": "Yuan Yang",
      "days_open": 5
    }
  ],
  "timestamp": "<ISO timestamp>"
}
```

## Channel 2 Send Rules

- Send ONLY if `assignments` is not empty OR `stale_tickets` is not empty
- If both are empty → Do NOT send
- `stale_tickets`: query tickets with no status change for > 3 days (JQL: `project = CM AND assignee IS NOT EMPTY AND status NOT IN (Done, Cancelled) AND updated <= -3d`)
- **Stale tickets are sent once per day only** — include them only on the first triage run of the day (controlled by `firstRunOfDay` parameter, default: true). On subsequent runs in the same day, omit `stale_tickets` entirely.
