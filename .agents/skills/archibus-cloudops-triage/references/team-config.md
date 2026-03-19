# Team Configuration — Archibus CloudOps

## Team Roster

```yaml
Lead:
  - name: Maninder Singh
    email: maninder.singh@eptura.com
    accountId: LOOKUP_REQUIRED
    domains: [All Archibus, GitHub Access, PROD Deployments, Customer Security]
    role: Lead
    maxLoad: 8
    notes: Team lead, handles GitHub access, security questionnaires, debug config, PROD deployments
    assign: GitHub/repo access, security follow-ups, debug mode changes, overflow PROD deployments

Senior Engineers:
  - name: Santosh Prasad
    email: santosh.prasad@eptura.com
    accountId: LOOKUP_REQUIRED
    domains: [SaaS PROD Deployments, Infrastructure, Monitoring, Customer Config, OnSite Upgrades, DB Operations]
    role: Senior Engineer
    maxLoad: 15
    notes: Highest throughput, handles bulk of PROD deployments, infra alerts, heap issues, customer config
    assign: SaaS PROD PRs, infra/monitoring alerts, heap restarts, whitelist/config, DB restore, upgrades

  - name: Dany Silva
    email: dany.silva@eptura.com
    accountId: LOOKUP_REQUIRED
    domains: [WAR Deployments, Auth/Login, Complex PROD Deployments, DB Backups, OnSite Troubleshooting]
    role: Senior Engineer
    maxLoad: 8
    notes: US timezone, handles WAR file deploys, auth/login issues, complex deployments
    assign: WAR deploys, login/auth troubleshooting, complex PROD deployments, DB backup pulls

  - name: Alex Plotkin
    email: alex.plotkin@eptura.com
    accountId: LOOKUP_REQUIRED
    domains: [Apollo/GraphQL, Logs, Patches, Mobile Builds, Releases]
    role: Senior Engineer
    maxLoad: 8
    notes: US timezone, Apollo/logs specialist, handles patches, mobile builds, Allbound posts
    assign: Apollo logs retrieval, GraphQL deployments, cumulative patches, mobile APK/IPA, Allbound

Engineers:
  - name: Muskan Chaudhary
    email: muskan.chaudhary@eptura.com
    accountId: LOOKUP_REQUIRED
    domains: [SaaS UAT Deployments, File Operations, Suspensions, Customer Config]
    role: Engineer
    maxLoad: 10
    notes: Primary UAT deployment handler, good with file operations and config changes
    assign: UAT PRs/deployments, drawing file copies, schemaCompiled deletes, suspensions, UAT config

  - name: Arpit Singh
    email: arpit.singh@eptura.com
    accountId: LOOKUP_REQUIRED
    domains: [SaaS UAT/PROD PRs, Automation Portal Access, Debug/Config Changes]
    role: Engineer
    maxLoad: 8
    notes: Handles PR deployments across UAT and PROD, automation portal access
    assign: UAT/PROD pull request deployments, automation portal access, debug mode changes

  - name: Shobhit Mishra
    email: shobhit.mishra@eptura.com
    accountId: LOOKUP_REQUIRED
    domains: [Patches, GitHub Access, Artifactory, Reporting, UAT Deployments]
    role: Engineer
    maxLoad: 8
    notes: Handles patches, GitHub repo access, Artifactory uploads, uptime reports
    assign: Patch builds/uploads, GitHub access provisioning, Artifactory, uptime/SLA reports, UAT PRs
```

## Jira Filter & Transition Configuration

```yaml
Filter ID: 55937
Filter Name: Archibus_NewIssue_CM
Expected Status: "New Issue"
Category JQL: '"Category and Sub-category[Select List (cascading)]" = Archibus'

Approve Transition ID: "31"
# Find via: GET /rest/api/3/issue/{issueKey}/transitions

Manual Triage Label: ClopsManualTriage
```

---

## Routing Rules

### Domain-Based Routing Table

| Domain | Primary | Secondary | Notes |
|--------|---------|-----------|-------|
| SaaS PROD Deployments (PRs) | Santosh Prasad | Dany Silva | PR #, deploy to PROD |
| SaaS UAT Deployments (PRs) | Muskan Chaudhary | Arpit Singh | PR #, deploy to UAT |
| Infrastructure / Monitoring Alerts | Santosh Prasad | Maninder Singh | Heap, LB, LogicMonitor, uptime |
| WAR File Deployments | Dany Silva | Santosh Prasad | Deploy WAR, test WAR |
| Auth / Login / SSO Issues | Dany Silva | Santosh Prasad | Login failures, SSO, SAML |
| Apollo / GraphQL / Logs | Alex Plotkin | Santosh Prasad | Apollo logs, GraphQL deploy |
| Patches / Hotfixes | Alex Plotkin | Shobhit Mishra | Cumulative patches, hotfix branches |
| Mobile Builds (APK/IPA) | Alex Plotkin | - | Allbound posts, mobile client |
| OnSite Upgrades | Santosh Prasad | Dany Silva | Version upgrades, v2025.xx |
| DB Operations (Backup/Restore) | Santosh Prasad | Dany Silva | DB backup, restore, refresh |
| Customer Config Changes | Santosh Prasad | Muskan Chaudhary | Whitelist IP, restart, mail config |
| File Operations (Drawings/Schema) | Muskan Chaudhary | Arpit Singh | Copy files, delete schemaCompiled |
| GitHub / Repo Access | Shobhit Mishra | Maninder Singh | Repo access provisioning |
| Automation Portal Access | Arpit Singh | Shobhit Mishra | Portal access provisioning |
| Artifactory Uploads | Shobhit Mishra | Alex Plotkin | JAR uploads, library management |
| Suspensions / Decommissions | Muskan Chaudhary | Santosh Prasad | Client suspension |
| Git Merges / Branch Ops | Santosh Prasad | Shobhit Mishra | Branch merges, healthfirst-staging |
| Security Questionnaires | Maninder Singh | - | SEC questions, customer follow-ups |
| Uptime / SLA Reports | Shobhit Mishra | Maninder Singh | SGS reports, uptime reports |
| Debug Mode Changes | Arpit Singh | Maninder Singh | WebCentral debug on/off |
| S3 / Storage Operations | Santosh Prasad | - | S3 buckets, storage config |

### Severity Rules

```yaml
S1 (Critical):
  - Route to: Santosh Prasad (primary), Dany Silva (secondary)
  - Notify: Maninder Singh
  - Action: Immediate assignment

S2 (High):
  - Route to: Senior engineers (Santosh, Dany, Alex) based on domain
  - Check workload first

S3/S4 (Standard):
  - Follow domain-based routing table
  - Apply workload balancing
```

### Special Routing Rules

```yaml
Rule 1: Apollo/GraphQL Keyword Override
  IF summary/description contains: apollo, graphql, onsite api
    Assign to Alex Plotkin regardless of other rules

Rule 2: US Timezone Routing
  IF ticket requires US-hours coordination AND created outside IST business hours:
    Prefer: Alex Plotkin or Dany Silva (US timezone)

Rule 3: Urgency Signals
  IF summary/description contains: URGENT, ASAP, customer escalated, blocking, down, outage, 503
    Route to Santosh Prasad (primary) or Dany Silva (secondary)
```

### Workload Balancing

```yaml
IF primary.currentLoad >= primary.maxLoad:
  1. Try secondary assignee
  2. If secondary also over capacity, find engineer with:
     - Matching domain skills
     - Most available capacity (maxLoad - currentLoad)
  3. NEVER fail to assign - always pick someone

IF no one in domain has ANY capacity:
  - Assign to person with LEAST overage in domain
  - Add note: "Assigned despite capacity - all assignees overloaded"
```

---

## Ticket Analysis

### Blocked Ticket Signals

```yaml
Blocked signals (flag for ManualTriage):
  - "waiting on"
  - "blocked by"
  - "depends on CM-XXXXX"
  - "pending"
  - "on hold"
  - "waiting for customer response"
  - "need more information"
  - "I'll take this"
  - "working on it"
  - "assigned to myself"
```

### Environment Detection

```yaml
Flag for Manual Triage (DO NOT auto-assign):
  - "dev tenant", "dev environment", "dev-", "DEV-"
  - "QA environment", "QA tenant", "qa-", "QA-"
  - "sandbox", "pre-prod", "PQA"

Valid Environments (OK to auto-assign):
  - "staging", "stage", "STAGE"
  - "production", "prod", "PROD"
```

**IMPORTANT**: Stage/Staging and Prod/Production are VALID environments — do NOT flag these. Only flag Dev and QA environments.

### Exploration Detection

```yaml
Route to senior engineers only (Santosh / Dany):
  - "investigate", "unknown", "root cause", "why is", "debug", "troubleshoot"

Well-defined tasks (can go to Muskan / Arpit / Shobhit):
  - Clear steps: "please deploy", "execute script", "whitelist", "restart", "PR #"
```

### Category Keywords Mapping

```yaml
SaaS Deployments: deploy, deployment, PR #, pull request, release, staging, production
Infrastructure: heap, load balancer, monitoring, LogicMonitor, uptime, unhealthy, alert
WAR/Auth: WAR, war file, login, SSO, SAML, authentication, unable to login
Apollo/GraphQL: apollo, graphql, onsite api, api server
Patches: patch, hotfix, cumulative, fix, branch
Mobile: mobile, APK, IPA, allbound, mobile client
OnSite: onsite, on-site, upgrade, v2025, v2024, webcentral
Database: database, backup, restore, refresh, DB, schema
Config: config, restart, whitelist, debug, mail, property
File Operations: drawing, enterprise graphics, schemaCompiled, files, copy
Access: github, repo access, automation portal, artifactory
Suspend: suspend, decommission, decom
Security: security, SEC-, questionnaire, follow up questions
Reports: uptime, report, SLA
S3/Storage: s3, bucket, storage
```
