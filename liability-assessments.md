---
title: "Liability Assessments — Comprehensive Reference"
description: "How Bumara audits an organization's outstanding statutory obligations across every regulator it is registered with, and scores the result."
---

## Table of Contents

1. [Overview](#1-overview)
2. [Supported Regulators](#2-supported-regulators)
3. [Assessment Lifecycle](#3-assessment-lifecycle)
4. [Assessment Status & Outcome Rules](#4-assessment-status--outcome-rules)
5. [Health Score Computation](#5-health-score-computation)
6. [Per-Regulator Checks](#6-per-regulator-checks)
7. [Assessment Jobs (Backoffice Execution)](#7-assessment-jobs-backoffice-execution)
8. [Findings — Detection & Reporting](#8-findings--detection--reporting)
9. [Findings — Verification Status Flow](#9-findings--verification-status-flow)
10. [Finding Categories by Regulator](#10-finding-categories-by-regulator)
11. [Findings — Aggregation & Totals](#11-findings--aggregation--totals)
12. [Evidence Links](#12-evidence-links)
13. [Recommendations — Generation Logic](#13-recommendations--generation-logic)
14. [Recommendations — Types & Mapping](#14-recommendations--types--mapping)
15. [Recommendations — Backoffice Review & Execution](#15-recommendations--backoffice-review--execution)
16. [Actions — Recommendation Execution Records](#16-actions--recommendation-execution-records)
17. [Findings — Dispute & Acknowledgement by Tenant](#17-findings--dispute--acknowledgement-by-tenant)
18. [Backoffice ↔ Tenant Communication](#18-backoffice--tenant-communication)
19. [Tenant-Facing Features During Assessment](#19-tenant-facing-features-during-assessment)
20. [Credential Management](#20-credential-management)
21. [Regulator Authorizations](#21-regulator-authorizations)
22. [Financial Precision & Fee Calculation](#22-financial-precision--fee-calculation)
23. [Access Control](#23-access-control)
24. [Reassessment & History](#24-reassessment--history)
25. [Key Design Constraints](#25-key-design-constraints)
26. [Data Model Summary](#26-data-model-summary)

---

## 1. Overview

A **Liability Assessment** is a structured compliance audit performed on behalf of an organization (tenant) across every statutory regulator to which that organization is registered. It determines outstanding tax, pension, or statutory obligations and produces a final outcome, verified health score, and per-regulator finding breakdown.

Assessments are created from an **Intake** (the organization's onboarding questionnaire). Execution is carried out by backoffice agents who work through per-regulator **Assessment Jobs**, recording **Findings**, generating **Recommendations**, and communicating with the tenant via **Tickets** and **Timeline Events**.

The tenant has a live, read-only view into the assessment as it progresses — they can see confirmed findings, dispute them, review recommendations, and act on them.

---

## 2. Supported Regulators

| Code | Full Name | Job Priority | Notes |
|------|-----------|-------------|-------|
| `zra` | Zambia Revenue Authority | **Urgent** | Tax compliance — highest risk, highest impact |
| `napsa` | National Pension Scheme Authority | Normal | Employee pension contributions |
| `pacra` | Patents and Companies Registration Agency | Normal | Company registration, annual returns |
| `nhima` | National Health Insurance Management Authority | Normal | Employee health insurance contributions |
| `wcf` | Workers' Compensation Fund | Normal | Workplace injury levy |
| `local_council` / `ncc` | Local Council / Lusaka City Council | Normal | Business licences, levies |

Each regulator is a global registry entry (`regulators` table). Organizations are linked via `organization_regulator_connections`. Only regulators where `isRegistered = true` in the intake snapshot receive a check during assessment.

---

## 3. Assessment Lifecycle

```
Intake submitted by organization
         │
         ▼
createAssessmentFromIntake()
         │
         ├── Supersedes any existing in_progress assessment (status → superseded)
         ├── Creates LiabilityAssessment (status: in_progress, SLA: +72h)
         ├── Creates one LiabilityRegulatorCheck per registered regulator
         ├── Creates one AssessmentJob per check (ZRA → urgent, others → normal)
         └── Transitions org onboarding phase → assessment
                  │
                  ▼
         Backoffice agents work jobs:
           queued → assigned → in_progress
                  │
                  ├── Agent logs Findings (detected → under_review → confirmed / dismissed)
                  ├── Evidence attached to findings (screenshots, statements, notices)
                  ├── Tickets raised to tenant for clarification / document requests
                  ├── Tenant can see confirmed findings and dispute them
                  └── Job completes → check status → completed (or blocked / skipped)
                  │
                  ▼
         evaluateAssessmentCompletion()  ← triggered after every check status update
         (fires only when ALL checks are terminal: completed | blocked | skipped)
                  │
                  ├── Aggregates totals (principal, penalties, interest, finding counts)
                  ├── Computes verifiedHealthScore (0–100)
                  ├── Determines outcome (clean | minor | material | critical)
                  └── Assessment status → completed
                  │
                  ▼
         Recommendations generated per confirmed finding
                  │
                  ├── Backoffice reviews / approves / executes recommendations
                  ├── Tenant reviews approved recommendations → Proceed / Defer / Dismiss
                  └── Org phase advanced:
                        clean   → active
                        others  → provisionally_active
```

---

## 4. Assessment Status & Outcome Rules

### Status Values

| Status | Meaning |
|--------|---------|
| `in_progress` | Active; agents are working regulator checks |
| `completed` | All checks terminal; outcome and health score finalized |
| `superseded` | Replaced by a newer assessment from re-intake |

### Outcome Determination

| Outcome | Trigger Conditions |
|---------|--------------------|
| `clean` | No confirmed findings across all regulators |
| `critical` | At least one `critical`-severity finding **or** any regulator account is `suspended` / `strike_off` |
| `material` | 3 or more `warning` findings **or** total financial exposure > ZMW 50,000 |
| `minor` | Findings exist but do not meet material or critical thresholds |

---

## 5. Health Score Computation

The **verifiedHealthScore** is an integer 0–100 computed at assessment completion by `computeHealthScoreSeed()`. A **provisionalHealthScore** may be set earlier as an estimate visible to the tenant before all checks are done.

| Deduction Trigger | Points Deducted |
|-------------------|----------------|
| Critical finding (confirmed) | −15 per finding |
| Warning finding (confirmed) | −8 per finding |
| Informational finding (confirmed) | −2 per finding |
| Any finding where `accruing = true` | −5 per finding |
| Regulator account status: `strike_off_notice` | −25 |
| Regulator account status: `suspended` | −20 |
| Each blocked regulator check | −5 |

Score is floored at 0 and capped at 100.

---

## 6. Per-Regulator Checks

One `LiabilityRegulatorCheck` row is created per registered regulator for each assessment.

### Check Status Flow

```
pending → in_progress → completed
                     └─ blocked
                     └─ skipped
```

| Status | Meaning |
|--------|---------|
| `pending` | Not yet started by agent |
| `in_progress` | Agent is actively reviewing this regulator's portal/records |
| `completed` | Review finished; all findings recorded |
| `blocked` | Cannot proceed — missing credentials, portal outage, or other blocker |
| `skipped` | Organization is not registered with this regulator |

### Coverage Status

Tracks granularity of portal coverage within a check:

| Value | Meaning |
|-------|---------|
| `not_started` | No portal review begun |
| `in_progress` | Partial areas reviewed |
| `complete` | All available portal sections reviewed |
| `blocked` | Coverage halted due to access or data issues |

### Credential Access Methods

| Method | Description |
|--------|-------------|
| `not_provided` | No credentials supplied by tenant |
| `direct_login` | Username/password provided by tenant |
| `api_key` | API key provided |
| `oauth` | OAuth token obtained |

### Per-Check Stored Totals

Incrementally maintained as findings are added/dismissed:

| Field | Description |
|-------|-------------|
| `totalPrincipal` | Sum of `amountPrincipal` across confirmed findings |
| `totalPenalties` | Sum of `amountPenalty` across confirmed findings |
| `totalInterest` | Sum of `amountInterest` across confirmed findings |
| `findingCount` | Count of all non-dismissed findings |
| `regulatorAccountStatus` | Current account standing at regulator |
| `lastFilingDate` | Last verified filing date observed on portal |

---

## 7. Assessment Jobs (Backoffice Execution)

Each `LiabilityRegulatorCheck` has a corresponding `AssessmentJob` assigned to a backoffice agent.

### Job Status Flow

```
queued → assigned → in_progress → completed
                              └─ blocked
                              └─ cancelled
```

### Priority

| Regulator | Priority |
|-----------|----------|
| ZRA | `urgent` |
| All others | `normal` |

### SLA

All jobs inherit the assessment's 72-hour SLA (`dueAt = assessment.startedAt + 72 hours`).

### Agent Actions

| Action | Service Method | Status Transition | Effect |
|--------|---------------|-------------------|--------|
| Assign | `assignAssessmentJob()` | → `assigned` | Sets `assignedTo`, `assignedAt` |
| Start | `startAssessmentJob()` | → `in_progress` | Sets `startedAt`; mirrors on regulator check |
| Complete | `completeAssessmentJob()` | → `completed` | Sets `completedAt`, saves `notes` |
| Block | — | → `blocked` | Sets `blockedReason`, `blockedAt` |
| Cancel | — | → `cancelled` | Job voided |

Completing a job updates the corresponding `LiabilityRegulatorCheck` status. When all checks become terminal, `evaluateAssessmentCompletion()` fires automatically.

---

## 8. Findings — Detection & Reporting

Findings are the core output of an assessment. An agent adds a finding to a regulator check when they observe a compliance issue on the regulator's portal.

### Finding Fields

| Field | Type | Description |
|-------|------|-------------|
| `assessmentId` | UUID | Parent assessment |
| `regulatorCheckId` | UUID | Parent check (links to regulator) |
| `orgId` | text | Denormalized org reference |
| `regulator` | enum | `zra` \| `napsa` \| `pacra` \| `nhima` \| `wcf` \| `local_council` \| `other` |
| `category` | text | Regulator-specific category (e.g., `unpaid_vat`, `late_annual_return`) |
| `severity` | enum | `info` \| `warning` \| `critical` |
| `description` | text | Human-readable description of the finding |
| `amountPrincipal` | integer | Outstanding principal in minor units (ngwee) |
| `amountPenalty` | integer | Accumulated penalty in minor units |
| `amountInterest` | integer | Accrued interest in minor units |
| `currency` | text | Default `ZMW` |
| `accruing` | boolean | `true` if interest/penalties continue to grow over time |
| `periodStart` / `periodEnd` | date | Tax/levy period the finding covers |
| `verificationStatus` | enum | See §9 |
| `remediable` | boolean | Whether the issue can be resolved |
| `estimatedRemediationDays` | integer | Agent's estimate to resolve |
| `sourceType` | text | Default `portal_check`; may be `document_review`, `agent_manual` |
| `sourceReference` | text | Portal URL, document ref, or other pointer |
| `findingFingerprint` | text | Composite dedup key: `{orgId}:{regulator}:{category}:{periodStart}:{periodEnd}` |
| `detectedBy` | text | Actor ID of agent who added the finding |
| `confirmedBy` | text | Actor ID of agent who confirmed the finding |
| `metadata` | jsonb | Regulator-specific extras; also stores dispute reason + `disputedAt` |

### Service Methods

| Method | Description |
|--------|-------------|
| `addFinding()` | Creates a finding; increments `findingCount` on the check |
| `updateFinding()` | Updates fields; blocked if status is `confirmed` or `dismissed` |
| `confirmFinding()` | Sets `verificationStatus → confirmed`; triggers `recomputeAssessmentTotals()` |
| `dismissFinding()` | Sets `verificationStatus → dismissed`; decrements counts; cancels linked recommendation |
| `listFindings()` | Query findings with filters (assessment, regulator, severity, status) |

---

## 9. Findings — Verification Status Flow

```
detected → under_review → confirmed ──────────────────── dismissed
                       └─ evidence_attached → confirmed
                                           └─ dismissed
confirmed → tenant_disputed → (backoffice reviews) → dismissed
                                                   → confirmed (dispute rejected)
```

| Status | Visible to Tenant? | Description |
|--------|--------------------|-------------|
| `detected` | No | Agent has flagged the issue; not yet reviewed |
| `under_review` | No | Agent is gathering more data / awaiting portal confirmation |
| `evidence_attached` | No | Supporting documents uploaded; awaiting final confirmation |
| `confirmed` | **Yes** | Finding is verified; appears in tenant's findings list |
| `tenant_disputed` | **Yes** | Tenant has filed a dispute; under backoffice review |
| `dismissed` | No | Finding rejected or resolved; excluded from totals |

**Rule:** Tenants can only see findings with `verificationStatus IN ('confirmed', 'tenant_disputed')`. All internal statuses (`detected`, `under_review`, `evidence_attached`) are hidden from the tenant portal.

---

## 10. Finding Categories by Regulator

### ZRA (Zambia Revenue Authority)

| Category | Description |
|----------|-------------|
| `unpaid_income_tax` | Outstanding income tax |
| `unpaid_vat` | Unpaid VAT |
| `unpaid_withholding_tax` | Withholding tax arrears |
| `unpaid_turnover_tax` | Turnover tax arrears |
| `unpaid_paye` | PAYE arrears |
| `unpaid_property_transfer_tax` | Property transfer tax |
| `tax_interest_accrued` | Accrued interest on tax debt |
| `tax_penalty_assessed` | Penalty levied by ZRA |
| `tcc_expired` | Tax Clearance Certificate expired |
| `tcc_not_issued` | TCC not yet issued |
| `zra_audit_flag` | Organization flagged for audit |
| `zra_enforcement_action` | Enforcement action initiated |
| `unfiled_tax_returns` | Missing tax return filings |

### NAPSA (National Pension Scheme Authority)

| Category | Description |
|----------|-------------|
| `contribution_arrears` | Unpaid NAPSA contributions |
| `napsa_penalty` | Penalty assessed by NAPSA |
| `unregistered_employees` | Employees not registered with NAPSA |
| `employer_registration_inactive` | Employer registration lapsed |

### PACRA (Patents and Companies Registration Agency)

| Category | Description |
|----------|-------------|
| `late_annual_return` | Annual return filed late or not filed |
| `strike_off_notice` | Company has received strike-off notice |
| `restoration_required` | Company requires restoration from strike-off |
| `pending_director_change` | Director change not yet filed |
| `pending_share_transfer` | Share transfer not yet registered |
| `pending_address_change` | Registered address change not filed |
| `outstanding_pacra_penalty` | Penalty owed to PACRA |
| `missing_compliance_certificate` | Required compliance certificate absent |

### NHIMA (National Health Insurance Management Authority)

| Category | Description |
|----------|-------------|
| `nhima_contribution_arrears` | Unpaid NHIMA contributions |
| `nhima_penalty` | NHIMA penalty assessed |
| `nhima_registration_inactive` | Employer NHIMA registration inactive |

### WCF (Workers' Compensation Fund)

| Category | Description |
|----------|-------------|
| `wcf_premium_arrears` | Unpaid WCF premiums |
| `wcf_assessment_outstanding` | Outstanding WCF levy assessment |
| `wcf_registration_inactive` | WCF registration inactive |

### Local Council / NCC

| Category | Description |
|----------|-------------|
| `trading_license_expired` | Business trading licence expired |
| `business_levy_arrears` | Business levy unpaid |
| `fire_certificate_expired` | Fire safety certificate expired |
| `council_penalty_outstanding` | Council penalty outstanding |

---

## 11. Findings — Aggregation & Totals

### Real-Time Incremental Updates (during assessment)

`addFinding()` → increments `regulatorCheck.findingCount`
`dismissFinding()` → decrements `regulatorCheck.findingCount`
`confirmFinding()` / `dismissFinding()` → triggers `recomputeAssessmentTotals()`

### `recomputeAssessmentTotals()` (called after each confirm/dismiss)

Recalculates from all findings where `verificationStatus = 'confirmed'`:
- `assessment.totalPrincipal`
- `assessment.totalPenalties`
- `assessment.totalInterest`

### `evaluateAssessmentCompletion()` (called when all checks are terminal)

Fires a final authoritative aggregation:

```
findingCount          = all non-dismissed findings
criticalFindingCount  = findings where severity = 'critical'
totalPrincipal        = SUM(amountPrincipal) of confirmed findings
totalPenalties        = SUM(amountPenalty) of confirmed findings
totalInterest         = SUM(amountInterest) of confirmed findings
verifiedHealthScore   = computeHealthScoreSeed() → clamped 0–100
outcome               = determineOutcome(findings, checkStatuses)
```

After computation:
- Assessment `status → completed`
- Org onboarding phase: `clean → active`, others `→ provisionally_active`

### Report Aggregation

`aggregateLiabilityReportData()` packages the completed assessment, all checks, all confirmed findings, and all recommendations into a structure used for HTML report generation. The report is downloadable by the tenant once the assessment is `completed`.

---

## 12. Evidence Links

Agents attach supporting documents to findings (or to a regulator check as a whole) using `evidence_links`.

| Field | Description |
|-------|-------------|
| `findingId` | Links evidence to a specific finding (nullable if check-level) |
| `regulatorCheckId` | Links evidence to the entire regulator check |
| `documentId` | Reference to uploaded file in R2 storage |
| `label` | Document type tag: `portal_screenshot` \| `tax_statement` \| `penalty_notice` \| `loa_signed` \| etc. |
| `uploadedBy` | Actor ID of the agent who uploaded |

Evidence links are **immutable** (no `updatedAt`). A new link must be created to replace evidence.

---

## 13. Recommendations — Generation Logic

Recommendations are generated after findings are confirmed. They prescribe the action the organization should take to remediate each finding.

### Generation Trigger

`generateRecommendations(deps, regulatorCheckId, generatedBy)` runs inside a transaction after an agent confirms one or more findings on a check. It:
1. Reads all `confirmed` findings for the check
2. Maps each finding `category` to a recommendation `type` and `description` via a hardcoded 170+ entry rule table (spec §7.1)
3. Applies **bundle rules** (spec §7.2) to merge related findings into a single recommendation
4. Sets `requiresManagerReview = true` for critical findings and certain escalation types
5. Creates one `Recommendation` row per finding (or per bundle group)
6. `generatedBy` is set to `'system'` for automatic generation or to the agent's user ID if manually triggered

### Bundle Rules

Related findings for the same regulator are merged into a single bundled recommendation to avoid overwhelming the tenant with redundant actions:

| Bundle Group | Regulators / Finding Categories Merged |
|-------------|----------------------------------------|
| `pacra_regularization` | `late_annual_return` + `outstanding_pacra_penalty` |
| `zra_tax_clearance` | `unpaid_*` taxes + `tax_penalty_assessed` + `tax_interest_accrued` |
| `napsa_regularization` | `contribution_arrears` + `napsa_penalty` + `unregistered_employees` |

Bundled recommendations share a `bundleGroup` tag and reference a `serviceCatalogId` for a matching bundled service offering.

---

## 14. Recommendations — Types & Mapping

| Type | Description | Example Trigger |
|------|-------------|----------------|
| `create_service_request` | Create a billable service to remediate the issue | `strike_off_notice`, `restoration_required` |
| `create_obligation` | Set up a recurring compliance obligation | `unpaid_vat`, `contribution_arrears` |
| `backfill_filings` | Catch up on missed period filings | `unfiled_tax_returns` |
| `create_payment_request` | Initiate a payment for outstanding liability | `tax_penalty_assessed`, `wcf_premium_arrears` |
| `request_tenant_input` | Request information or documents from tenant | Missing registration details |
| `escalate_to_manager` | Requires manager-level review before action | `zra_audit_flag`, `zra_enforcement_action` |
| `accepted_risk` | Organization acknowledges but accepts the risk | Minor info-level findings |
| `not_remediable` | Issue cannot be fixed (e.g., expired period) | Historical penalties beyond statute |
| `bundle_with` | Finding is grouped into a bundle recommendation | See bundle rules above |

### Additional Recommendation Fields

| Field | Description |
|-------|-------------|
| `description` | System-generated action text (e.g., "Ensure VAT obligation exists. Backfill missed periods.") |
| `findingContext` | Original finding description shown to tenant for context |
| `estimatedDays` | Estimated days to resolve |
| `estimatedCost` | Estimated cost in minor units |
| `obligationTemplateId` | Pre-selected obligation template for `create_obligation` type |
| `backfillPeriods` | Array of `{periodStart, periodEnd}` for backfill types |
| `serviceCatalogId` | Linked service catalog entry for `create_service_request` |
| `requiresManagerReview` | `true` → tenant cannot act until backoffice approves |
| `bundleGroup` | Groups findings that share a single bundled recommendation |

---

## 15. Recommendations — Backoffice Review & Execution

### Review Workflow

Recommendations where `requiresManagerReview = true` must be reviewed by a backoffice manager before the tenant can act on them.

```
generated (pending)
    │
    ├─ requiresManagerReview = true
    │       │
    │       ▼
    │  Backoffice reviews:
    │  POST /assessments/{id}/recommendations/{recId}
    │  { approved: true/false, reviewNotes: string }
    │       │
    │       ├── Approved → reviewNotes = "[APPROVED]", reviewedBy, reviewedAt set
    │       └── Rejected → reviewNotes = "[REJECTED]"
    │
    └─ requiresManagerReview = false → tenant can act immediately
```

### Execution

After approval, backoffice executes the recommendation:

```
POST /assessments/recommendations/{recId}/execute
```

`executeRecommendation()`:
1. Creates an `Action` record with `status: executed`
2. Creates the target entity (service request, obligation, payment request, etc.)
3. Sets `reviewNotes = [EXECUTED]`, `reviewedBy`, `reviewedAt` on recommendation

### `reviewNotes` Status Tags

`reviewNotes` is a text field used as a lightweight state machine. Tags are inserted as prefixes:

| Tag | Meaning |
|----|---------|
| `[APPROVED]` | Backoffice approved; ready for tenant action or execution |
| `[REJECTED]` | Backoffice rejected the recommendation |
| `[EXECUTED]` | Backoffice has executed the recommendation (created service/obligation/payment) |
| `[TENANT_APPROVED]` | Tenant chose to proceed |
| `[TENANT_DEFERRED]` | Tenant deferred (with optional reason appended) |
| `[CANCELLED]` | Recommendation voided (e.g., parent finding was dismissed) |

---

## 16. Actions — Recommendation Execution Records

An `Action` row is created each time a recommendation is executed. It records what was created and who decided.

| Field | Description |
|-------|-------------|
| `recommendationId` | Parent recommendation |
| `assessmentId` | Parent assessment |
| `orgId` | Tenant organization |
| `status` | `pending_review` \| `approved` \| `executed` \| `rejected` \| `deferred` |
| `targetType` | What was created: `service_request` \| `obligation` \| `filing_backfill` \| `payment_request` \| `ticket` \| `accepted_risk` \| `not_remediable` |
| `targetId` / `targetIds` | ID(s) of the created entity; `null` for `accepted_risk` / `not_remediable` |
| `decidedBy` / `decidedAt` | Backoffice agent who approved/rejected |
| `executedBy` / `executedAt` | Agent who performed execution |
| `reason` | Notes on decision |

---

## 17. Findings — Dispute & Acknowledgement by Tenant

### Disputing a Finding

A tenant can dispute any finding with `verificationStatus = 'confirmed'`.

**Endpoint:** `POST /liability/assessment/findings/{findingId}/dispute`

**Request body:**
```json
{ "reason": "We have proof of payment for this period." }
```

**Effect:**
- `verificationStatus → tenant_disputed`
- `metadata.disputeReason = reason`
- `metadata.disputedAt = <timestamp>`
- The Dispute button is disabled once a finding is already `tenant_disputed`

### Dispute Resolution by Backoffice

Backoffice reviews the dispute against the evidence on record:

| Resolution | Action |
|-----------|--------|
| Dispute accepted | `dismissFinding()` → `verificationStatus → dismissed`; linked recommendation is cancelled |
| Dispute rejected | Finding reverts to `confirmed`; tenant is notified via ticket/timeline event |

### Acknowledging a Recommendation (Acting on a Finding)

Rather than a formal "acknowledge" action, the tenant acts on the linked recommendation:

| Tenant Action | API | Effect |
|--------------|-----|--------|
| **Proceed** | `POST /recommendations/{id}/approve` | `reviewNotes = [TENANT_APPROVED]`, `reviewedAt = now` |
| **Defer** | `POST /recommendations/{id}/defer` with optional reason | `reviewNotes = [TENANT_DEFERRED] {reason}`, `reviewedAt = now` |
| **Dismiss** | `DELETE /recommendations/{id}` | Soft-deletes; removed from tenant view |

Proceeding on a `create_service_request` type recommendation initiates the service request flow. Proceeding on `create_obligation` triggers obligation setup.

---

## 18. Backoffice ↔ Tenant Communication

### Tickets

Tickets are the primary structured communication channel. A ticket can be raised by backoffice against a filing, service request, payment request, or task.

| Field | Values / Description |
|-------|---------------------|
| `type` | `data_request` \| `payment_request` \| `clarification` \| `correction` \| `limit_reached` |
| `status` | `open` → `waiting_on_tenant` → `waiting_on_backoffice` → `escalated` → `resolved` → `closed` |
| `priority` | `low` \| `normal` \| `high` \| `urgent` \| `critical` |
| `subject` | Short description of what is needed |
| `description` | Full detail of the request |
| `createdByAgentId` | Backoffice agent who opened the ticket |
| `assignedToAgentId` | Backoffice agent currently responsible |
| `slaDeadline` | Expected resolution time |
| `resolvedAt` | Timestamp when resolved |

### Ticket Notes

Notes are threaded replies on a ticket, visible to both parties unless marked internal.

| Field | Description |
|-------|-------------|
| `content` | Message content |
| `isInternal` | `true` → backoffice-only (hidden from tenant); `false` → visible to tenant |
| `createdByAgentId` | If written by a backoffice agent |
| `createdByUserId` | If written by a tenant user |

Notes are immutable once created (no `updatedAt`).

### Timeline Events

Timeline events provide an audit trail of all significant actions on a filing or service request.

| `eventType` | When it fires |
|------------|---------------|
| `status_change` | Any entity status transition |
| `submission` | Document or filing submitted |
| `payment` | Payment recorded |
| `document_upload` | Document attached |
| `comment` | A note or comment added |
| `assignment` | Task or ticket assigned to an agent |
| `bulk_operation` | Batch update applied |

Each event stores `actorId`, `actorType`, `title`, `description`, and `metadata` (jsonb for event-specific data).

### Task Comments

Tasks (within filings or service requests) can have threaded comments from tenant organization members.

| Field | Description |
|-------|-------------|
| `taskId` | Parent task |
| `authorId` | `organizationMembers` ID |
| `content` | Comment text |

Task comments are immutable once created.

### Communication Flow During Assessment

```
Backoffice agent
    │
    ├── Raises ticket (type: data_request / clarification)
    │       │
    │       ▼
    │   Tenant sees ticket in portal → replies via ticket note
    │       │
    │       ▼
    │   Backoffice receives reply → resolves or escalates
    │
    ├── Confirms finding → finding becomes visible to tenant (confirmed)
    │
    ├── Tenant disputes finding → verificationStatus → tenant_disputed
    │       │
    │       ▼
    │   Backoffice reviews dispute → dismisses or reconfirms
    │
    └── All timeline events logged for full audit trail
```

---

## 19. Tenant-Facing Features During Assessment

While the assessment is `in_progress`, the tenant has a live portal view that auto-refreshes every **2 minutes** via `refetchInterval`.

### Assessment Overview Panel (`assessment-overview.tsx`)

| Element | Details |
|---------|---------|
| **Status banner** | "In progress" with assessment start date |
| **Provisional health score** | Indicative score updated as findings are confirmed |
| **Total liability exposure** | Sum of confirmed finding amounts across all regulators |
| **Regulator checks table** | One row per regulator showing: coverage status, finding count, liability exposure, check status |
| **Report download** | Not available during `in_progress`; appears once `completed` |

### Findings List (`findings-list.tsx`)

Only shows findings with `verificationStatus IN ('confirmed', 'tenant_disputed')`.

| Column | Description |
|--------|-------------|
| Severity badge | `Critical` / `Warning` / `Info` |
| Description | Human-readable finding text |
| Regulator | Which regulator the finding is from |
| Period | `periodStart` – `periodEnd` |
| Principal | Outstanding principal amount |
| Penalties | Assessed penalties |
| Interest | Accrued interest |
| Status | `Confirmed` or `Disputed` badge |
| Dispute button | Enabled only if `verificationStatus = 'confirmed'`; opens dispute dialog |

**Dispute Dialog:** Tenant enters a free-text reason. On submit, `verificationStatus → tenant_disputed` and the button becomes disabled.

### Recommendations List (`recommendations-list.tsx`)

Shows all recommendations linked to confirmed findings. Hides recommendations with `[CANCELLED]` in `reviewNotes`.

| Element | Condition |
|---------|-----------|
| **Proceed / Defer** buttons | Shown if `requiresManagerReview = false` and recommendation is pending |
| **"Awaiting Review" badge** | Shown if `requiresManagerReview = true` and not yet reviewed by manager |
| **"In Progress" badge** | Shown if `[EXECUTED]` in reviewNotes |
| **"Approved" badge** | Shown if `[TENANT_APPROVED]` in reviewNotes |
| **"Deferred" badge** | Shown if `[TENANT_DEFERRED]` in reviewNotes |
| **Dismiss button** | Always visible; removes recommendation from tenant view |

Each recommendation card also shows:
- Type badge (e.g., "Service Request", "New Obligation", "Payment Required")
- Regulator badge
- System-generated action description
- Original finding description (`findingContext`) for reference
- Estimated days and cost (if set)
- `serviceCatalogId` (links to the relevant service in the catalog, if applicable)

### Ticket Inbox

Tenants can view and reply to open tickets raised by backoffice during the assessment. Ticket notes with `isInternal = true` are never shown.

### What Tenants **Cannot** Do During Assessment

| Action | Restriction |
|--------|-------------|
| See `detected` / `under_review` / `evidence_attached` findings | Hidden — backoffice-internal only |
| Modify intake data while assessment is in progress | Locked; a new intake supersedes the current assessment |
| Download the assessment report | Only available once `completed` |
| Act on recommendations requiring manager review | Blocked until `[APPROVED]` tag is set |

---

## 20. Credential Management

Regulator credentials supplied by the tenant are stored in `regulator_credentials` using **envelope encryption**:

| Field | Description |
|-------|-------------|
| `encryptedDek` | Data Encryption Key (DEK) encrypted with master key |
| `encryptedPayload` | Actual credentials encrypted with the DEK |
| `keyVersion` | Supports key rotation |
| `version` | Record version for optimistic concurrency |
| `lastRotatedAt` | Last key rotation timestamp |
| `rotatedByUserId` | Agent who rotated the key |

The `credentialVaultRef` on a `LiabilityRegulatorCheck` points to the relevant credential record. If `credentialAccessMethod = 'not_provided'`, the check may be blocked immediately.

---

## 21. Regulator Authorizations

Some regulators (notably PACRA) require a named authorized representative. Authorization records track:

| Field | Description |
|-------|-------------|
| `representativeName` | Authorized person's full name |
| `representativeRole` | Their role at the organization |
| `representativeEmail` / `Phone` | Contact details |
| `documentId` | Uploaded authorization letter (LOA) |
| `validFrom` / `validUntil` | Authorization validity period |
| `status` | `pending` → `approved` → `expired` / `revoked` |

An expired or missing authorization for a regulator that requires one will block the corresponding regulator check.

---

## 22. Financial Precision & Fee Calculation

All monetary amounts are stored in **minor units (ngwee for ZMW)**. Divide by 100 for display.

Example: ZMW 1,250.00 is stored as `125000`.

### Regulator Fee Lookup

Fees are stored in `regulator_fees` and looked up by `feeKey`:

```
Format:  {REGULATOR}_{SERVICE_TYPE}_{ENTITY_TYPE}
Example: PACRA_ANNUAL_RETURN_COMPANY
```

Only fees with `effectiveUntil = NULL` are **active**. Historical records are retained for audit.

### Fee Calculation

```
handlingFee  = regulatorFee × 10%   (default rate)
totalAmount  = regulatorFee + handlingFee
```

Constraint: `totalAmount = regulatorFee + handlingFee` is enforced at the database level.

---

## 23. Access Control

| Field | Description |
|-------|-------------|
| `minimumPlanRequired` | Plan tier to access this regulator (`start`, `plus`, `pro`) |
| `isPublicAccess` | `true` → no plan restriction |
| `trialFeatureLimits` | JSONB overrides for feature gating during trial |

Backoffice routes operate without org-scope restriction. Tenant routes are scoped to the authenticated organization. All assessment and finding reads are filtered by `orgId`.

---

## 24. Reassessment & History

Organizations can submit a new intake at any time:

1. `createAssessmentFromIntake()` detects existing `in_progress` assessment
2. Marks it `superseded`
3. Creates new assessment with:
   - `assessmentNumber` incremented
   - `previousAssessmentId` linking back to the superseded record
4. Full chain of historical assessments is preserved

The `intakeSnapshot` (JSONB) on each assessment freezes the organization's self-reported state at intake time and is never mutated — providing a reliable historical record even if org data changes later.

---

## 25. Key Design Constraints

| Constraint | Details |
|-----------|---------|
| Single active assessment per org | New intake auto-supersedes any `in_progress` assessment |
| All checks must be terminal before completion | `completed`, `blocked`, or `skipped` — no partial completions |
| Blocked checks still score | −5 health score deduction per blocked check; may also trigger `critical` if account status warrants |
| ZRA always urgent | Bypasses normal queue priority due to tax compliance centrality |
| Amounts never nullable | All financial totals default to `0`; no NULL arithmetic in aggregations |
| Finding dedup via fingerprint | `findingFingerprint` prevents duplicate findings for the same org/regulator/category/period |
| Recommendations are 1-to-1 per finding | Unique index on `(assessmentId, findingId)` unless bundled |
| Dismissing a finding auto-cancels its recommendation | `reviewNotes = [CANCELLED]`; removed from tenant view |
| Manager review required for critical findings | `requiresManagerReview = true` blocks tenant action until backoffice approves |
| Evidence links are immutable | No updates; new document = new evidence link record |
| Ticket notes are immutable | No edits after creation; `isInternal` controls tenant visibility |

---

## 26. Data Model Summary

```
LiabilityAssessment
  ├── status: in_progress | completed | superseded
  ├── outcome: clean | minor | material | critical
  ├── intakeSnapshot: jsonb (frozen at intake)
  ├── totalPrincipal, totalPenalties, totalInterest (minor units)
  ├── findingCount, criticalFindingCount
  ├── provisionalHealthScore, verifiedHealthScore (0–100)
  ├── startedAt, dueAt (+72h SLA), completedAt
  ├── previousAssessmentId, assessmentNumber
  │
  └── LiabilityRegulatorCheck  [1 per registered regulator]
        ├── regulator: zra | napsa | pacra | nhima | wcf | local_council
        ├── status: pending | in_progress | completed | blocked | skipped
        ├── coverageStatus: not_started | in_progress | complete | blocked
        ├── credentialAccessMethod, credentialVaultRef
        ├── regulatorAccountStatus, lastFilingDate
        ├── totalPrincipal, totalPenalties, totalInterest, findingCount
        ├── blockedReason, blockedAt
        │
        ├── AssessmentJob  [1 per check]
        │     ├── status: queued | assigned | in_progress | completed | blocked | cancelled
        │     ├── priority: urgent | normal | low
        │     ├── dueAt (inherited from assessment SLA)
        │     ├── assignedTo, assignedAt, startedAt, completedAt
        │     └── notes, blockedReason
        │
        ├── Finding  [0..n per check]
        │     ├── severity: info | warning | critical
        │     ├── category: (regulator-specific, see §10)
        │     ├── verificationStatus: detected | under_review | evidence_attached
        │     │                     | confirmed | tenant_disputed | dismissed
        │     ├── amountPrincipal, amountPenalty, amountInterest (minor units)
        │     ├── accruing: boolean
        │     ├── periodStart, periodEnd
        │     ├── remediable, estimatedRemediationDays
        │     ├── findingFingerprint (dedup key)
        │     ├── metadata: jsonb (incl. disputeReason, disputedAt)
        │     │
        │     ├── EvidenceLink  [0..n per finding]
        │     │     ├── documentId, label
        │     │     └── uploadedBy
        │     │
        │     └── Recommendation  [0..1 per finding]
        │           ├── type: create_service_request | create_obligation | backfill_filings
        │           │       | create_payment_request | request_tenant_input
        │           │       | escalate_to_manager | accepted_risk | not_remediable | bundle_with
        │           ├── description (system-generated action text)
        │           ├── findingContext (original finding description for tenant)
        │           ├── requiresManagerReview: boolean
        │           ├── reviewNotes: [APPROVED|REJECTED|EXECUTED|TENANT_APPROVED
        │           │                |TENANT_DEFERRED|CANCELLED]
        │           ├── reviewedBy, reviewedAt
        │           ├── bundleGroup, serviceCatalogId
        │           ├── estimatedDays, estimatedCost
        │           │
        │           └── Action  [0..1 per recommendation]
        │                 ├── status: pending_review | approved | executed | rejected | deferred
        │                 ├── targetType: service_request | obligation | filing_backfill
        │                 │            | payment_request | ticket | accepted_risk | not_remediable
        │                 ├── targetId / targetIds (created entity references)
        │                 ├── decidedBy, decidedAt
        │                 └── executedBy, executedAt
        │
        └── Ticket  [0..n per filing / service request]
              ├── type: data_request | payment_request | clarification | correction | limit_reached
              ├── status: open | waiting_on_tenant | waiting_on_backoffice | escalated | resolved | closed
              ├── priority: low | normal | high | urgent | critical
              ├── slaDeadline, resolvedAt
              │
              └── TicketNote  [0..n per ticket]
                    ├── content
                    ├── isInternal: boolean (false = visible to tenant)
                    └── createdByAgentId | createdByUserId
```
