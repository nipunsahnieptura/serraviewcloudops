# Notifications

## Teams Webhook Configuration

```yaml
Webhook URL: https://default454af8f364734bce8aa5f9403e1d12.71.environment.api.powerplatform.com:443/powerautomate/automations/direct/workflows/2f988e1527cc416284dc27d3df468b58/triggers/manual/paths/invoke?api-version=1&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=LIjoyCCexv6xBCuFMnWU6mT73Sd2bkvOSMBHN85pf1Q

Notify on:
  - Every triage run (always send)
```

## Notification Recipients

```yaml
Tag in alerts:
  - name: Hritik Chaudhary
    email: hritik.chaudhary@eptura.com
  - name: Shilpa Goyal
    email: shilpa.goyal@eptura.com
```

## Notification Payload Format

```json
{
  "type": "triage_summary",
  "notify": [
    {"name": "Hritik Chaudhary", "email": "hritik.chaudhary@eptura.com"},
    {"name": "Shilpa Goyal", "email": "shilpa.goyal@eptura.com"}
  ],
  "assigned": [
    {
      "key": "CM-XXXXX",
      "url": "https://eptura.atlassian.net/browse/CM-XXXXX",
      "summary": "ticket summary",
      "assignee": "engineer@eptura.com",
      "reason": "BAU Serraview, workload balanced"
    }
  ],
  "transitioned_only": [
    {
      "key": "CM-XXXXX",
      "url": "https://eptura.atlassian.net/browse/CM-XXXXX",
      "summary": "ticket summary",
      "assignee": "engineer@eptura.com"
    }
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
  "timestamp": "<ISO timestamp>"
}
```

## Send Rules

- **If filter 55922 had tickets**: send the full `triage_summary` payload (assigned, transitioned_only, manual_triage, errors)
- **If filter 55922 had NO tickets**: send a `workload_summary` payload (see below) — do NOT send an empty triage summary

## Workload-Only Payload Format

Use this when filter 55922 returns no tickets to process:

```json
{
  "type": "workload_summary",
  "notify": [
    {"name": "Nipun Sahni", "email": "nipun.sahni@eptura.com"},
    {"name": "Gaurav Kumar", "email": "Gaurav.Kumar@eptura.com"}
  ],
  "message": "No new Serraview CM tickets to triage. Current workload:",
  "workload": [
    {"engineer": "Ankit Kumar Sinha", "active_tickets": 0, "max_load": 10, "status": "OK"},
    {"engineer": "Yuan Yang", "active_tickets": 0, "max_load": 5, "status": "OK"},
    {"engineer": "Deevanshu Gakhar", "active_tickets": 0, "max_load": 5, "status": "OK"},
    {"engineer": "Gaurav Kumar", "active_tickets": 0, "max_load": 10, "status": "OK"}
  ],
  "timestamp": "<ISO timestamp>"
}
```
