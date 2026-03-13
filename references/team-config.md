# Team Configuration

## Team Roster

```yaml
Leads:
  - name: Gaurav Kumar
    email: gaurav.kumar@eptura.com
    accountId: "712020:e779f9ea-49c9-4573-8a3a-61a6264cd283"
    domains: [Serraview]
    role: Lead
    maxLoad: 10
    notes: Time-constrained, handles S1/S2 ONLY for his domains

Principal:
  - name: Deevanshu Gakhar
    email: deevanshu.gakhar@eptura.com
    accountId: "639af1b47145571a7ea882d7"
    domains: [Serraview/Database related changes\updates]
    role: Principal
    maxLoad: 5
    notes: ONLY assign CLAUTO tickets or automation-fix CMs

Engineers:
  - name: Yuan Yang
    email: yuan.yang@eptura.com
    accountId: "6362e6fe59c794184bcc1a3e"
    domains: [Serraview/Database related changes\updates]
    skillWill: Average
    maxLoad: 5
    notes: Time-constrained, handles DBA tasks ONLY for his domains
    assign: database related CMs

  - name: Ankit Kumar Sinha
    email: Ankit.Sinha@eptura.com
    accountId: "712020:b217ad2b-f35c-41e1-8ec5-73d5d58952d0"
    domains: [Serraview]
    skillWill: Top Performer
    maxLoad: 10
    notes: Fire-and-forget, can explore unknowns, INCUBATE from daily CM work
    assign: Critical CMs, Exploratory work

  - name: Shobhit Mishra
    email: shobhit.mishra@eptura.com
    accountId: "712020:7f06dd3b-4d20-4e02-bb38-b3ff9ea66c64"
    domains: [Serraview]
    skillWill: Average
    maxLoad: 5
    notes: ONLY assign CLAUTO tickets
    assign: CloudOps Automation portal related work

  - name: Mashkoor Ahmad
    email: mashkoor.ahmad@eptura.com
    accountId: "64238bb20152b5f4f9f2e7f9"
    domains: [Serraview]
    skillWill: Top Performer
    maxLoad: 5
    notes: Migration related activity and infrastructure management
    assign: Non Critical CMs

  - name: Mridul Raina
    email: mridul.raina@eptura.com
    accountId: "712020:4962c34a-ee1a-427c-aced-015675053cae"
    domains: [Serraview/Database related changes\updates]
    skillWill: Average
    maxLoad: 5
    notes: Time-constrained, handles dba tasks ONLY for his domains
    assign: database related cms

  - name: Vikas Kumar
    email: vikas.kumar@eptura.com
    accountId: "62b8f3e8118b20bee2ba7228"
    domains: [Serraview]
    skillWill: Average
    maxLoad: 5
    notes: Fire-and-forget, can explore unknowns, INCUBATE from daily CM work
    assign: Critical CMs, Exploratory work
```

## Jira Filter & Transition Configuration

```yaml
Filter ID: 55922
Filter Name: Serraview_NewIssue_CM
Expected Status: "New Issue"
Exclusions: "S1 tickets handled separately"

Approve Transition ID: "31"
# Find via: GET /rest/api/3/issue/{issueKey}/transitions

Manual Triage Label: ClopsManualTriage
```

---

## Routing Rules

### Domain-Based Routing Table

| Category/Domain | Primary | Secondary | Notes |
|---|---|---|---|
| Serraview General (Critical/Exploratory) | Ankit Kumar Sinha | Vikas Kumar | Critical/exploratory work |
| Serraview BAU | Ankit Kumar Sinha | Mashkoor Ahmad | Workload-balanced; Mashkoor for non-critical overflow |
| Serraview Database | Yuan Yang | Mridul Raina | DB changes, schema updates, migrations; Deevanshu tertiary |
| Serraview Automation / CLAUTO | Deevanshu Gakhar | Shobhit Mishra | CLAUTO tickets or automation-fix CMs ONLY |
| Serraview S1/S2 | Gaurav Kumar | Ankit Kumar Sinha | Lead handles critical severity |

### Severity Rules

```yaml
S1 (Critical):
  - Route to: Gaurav Kumar (Lead) - accountId "712020:e779f9ea-49c9-4573-8a3a-61a6264cd283"
  - Action: Immediate assignment, handles S1 ONLY for Serraview domain
  - Note: Often handled by separate event-based trigger

S2 (High):
  - Route to: Ankit Kumar Sinha (senior engineer, Top Performer)
  - Check workload first before assignment
  - If Ankit is over maxLoad, escalate to Gaurav Kumar

S3/S4 (Standard):
  - Follow domain-based routing table
  - Apply workload balancing
```

### Special Routing Rules

```yaml
Rule 1: CLAUTO / Automation-Fix Routing
  IF summary/description contains CLAUTO keywords (clauto, automation-fix):
    Assign to Deevanshu Gakhar (Principal) regardless of other rules

Rule 2: Database Task Routing
  IF ticket involves database changes/updates/migrations/schema:
    Assign to Yuan Yang (primary), Mridul Raina (secondary), or Deevanshu Gakhar (tertiary)
    DO NOT assign DB tasks to general Serraview engineers
```

### Workload Balancing

```yaml
IF primary.currentLoad >= primary.maxLoad:
  1. Try secondary assignee
  2. If secondary also over capacity, find engineer with:
     - Matching domain skills
     - Most available capacity (maxLoad - currentLoad)
     - Higher skillWill rating (tiebreaker)
  3. NEVER fail to assign - always pick someone

IF no one in domain has ANY capacity:
  - Assign to person with LEAST overage in domain
  - Add note: "Assigned despite capacity - all assignees overloaded"
```

### Skill/Will Matching

```yaml
Exploration/Unknown tasks:
  - Ankit Kumar Sinha (Top Performer, can explore unknowns)
  - Vikas Kumar (can explore unknowns)
  - AVOID: Yuan Yang (Average, DB tasks only)

Known/Repetitive tasks:
  - Yuan Yang (database related CMs)
  - Mashkoor Ahmad (non-critical BAU CMs)

Database tasks:
  - Yuan Yang (primary)
  - Mridul Raina (secondary, dba tasks)
  - Deevanshu Gakhar (tertiary, automation-related DB)
```

### Protected Engineers (Incubation)

```yaml
Protected (INCUBATE from daily CM work):
  - Ankit Kumar Sinha: maxLoad 10, but prioritize Critical/Exploratory over BAU
    Exceptions: Critical CMs, Exploratory work, or Serraview specialty
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
  - "Blocked until deployment completes"
  - "On hold per manager request"
  - "Need more information from requester"
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

### Category Keywords Mapping

```yaml
Serraview: serraview, sv, serra, serraview-app, svlive
Serraview Database: database, db, schema, migration, sql, dba, rds, postgres, db-change
Serraview Automation: clauto, automation-fix, automation, auto-fix
```

### Keyword Override Rules

```yaml
IF summary/description contains CLAUTO keywords (clauto, automation-fix):
  Route to Deevanshu Gakhar (Principal) regardless of Category field

IF summary/description contains DB keywords (database, schema, migration, sql, dba):
  Route to Yuan Yang / Mridul Raina / Deevanshu Gakhar regardless of Category field
```
