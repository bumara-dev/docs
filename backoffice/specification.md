---
title: "Bumara Backoffice Specification"
description: "Complete product specification for the internal operations platform."
---

**Version:** 1.0  
**Status:** Authoritative  
**Audience:** Product, Engineering, Operations, Finance

---

## Table of Contents

1. [Purpose and Principles](#1-purpose-and-principles)
2. [Scope (MVP)](#2-scope-mvp)
3. [Personas and Roles](#3-personas-and-roles)
4. [Navigation and Routing](#4-navigation-and-routing)
5. [Core Concepts](#5-core-concepts)
6. [State Machines and Gates](#6-state-machines-and-gates)
7. [User Journeys](#7-user-journeys)
8. [UI Specification](#8-ui-specification)
9. [Security Requirements](#9-security-requirements)
10. [Acceptance Criteria](#10-acceptance-criteria)

---

## 1. Purpose and Principles

### 1.1 Purpose

Bumara Backoffice is the **internal operations application** used by Bumara staff to deliver managed compliance services for tenant organizations. It enables staff to:

- Intake work from the submission queue
- Verify readiness (tasks, documents, payments, authorizations)
- Perform manual regulator submissions
- Track outcomes and upload evidence
- Maintain full audit trail

### 1.2 Product Principles

| # | Principle | Description |
|---|-----------|-------------|
| 1 | **Ops-first** | Home is a work queue/inbox—not compliance alerts |
| 2 | **Case-centric** | Everything staff works on is a *Case* |
| 3 | **Gated execution** | Submission requires all readiness gates to pass |
| 4 | **Evidence required** | Every payout and submission must have evidence |
| 5 | **Audit by default** | All changes are logged (who/what/when/why) |
| 6 | **Least privilege** | Role-based access; admin features are hidden and server-blocked |

### 1.3 Non-goals (MVP) - future implementations

- ❌ Fully automated regulator API submissions (handled manually)
- ❌ Complex analytics and forecasting (phase 2)
- ❌ Advanced workflow automation rules engine (phase 2)
- ❌ Full internal chat/collaboration suite (notes only in MVP)

---

## 2. Scope (MVP)

### 2.1 MVP Modules

| Section | Module | Route |
|---------|--------|-------|
| **Operations** | Work Queue (unclaimed only) | `/backoffice/queue` |
| | Workspace (claimed/assigned) | `/backoffice/workspace` |
| | Tickets & SLA | `/tickets` |
| **Finance** | Payments & Payouts | `/payments` |
| **Documents** | Documents & Evidence | `/documents` |
| **Directory** | Organizations (tenants or organizations) | `/orgs` |
| **Admin** | System Admin | `/admin` |
| | Catalog & Rules | `/catalog` (optional) |
| | Reports & Analytics | `/reports` (feature-flagged) |

### 2.2 MVP Definition of Done

A staff member can process a compliance request **end-to-end**:

```
intake → assignment → request info → verify payment → pay regulator 
      → submit → upload evidence → close → audit trail preserved
```

---

## 3. Personas and Roles

### 3.1 Personas

| Persona | Description |
|---------|-------------|
| **Ops Analyst** | Executes cases, requests info, uploads evidence, updates statuses |
| **Ops Manager** | Assigns work, approves high-value payouts, resolves escalations |
| **System Admin** | Staff management, system config, sensitive operations |

### 3.2 Roles

| Role | Clerk Mapping | Level |
|------|---------------|-------|
| `admin` | `org:admin`, `org:backoffice_admin` | 3 |
| `manager` | `org:manager`, `org:backoffice_manager` | 2 |
| `member` | `org:member`, `org:backoffice_member` | 1 |

### 3.3 Permission Matrix (MVP)

| Capability | member | manager | admin |
|------------|:------:|:-------:|:-----:|
| View inbox/queues | ✅ | ✅ | ✅ |
| Claim case to self | ✅ | ✅ | ✅ |
| Reassign others | ❌ | ✅ | ✅ |
| Update case status (normal) | ✅ | ✅ | ✅ |
| Override gates / force status | ❌ | ✅ (reason) | ✅ (reason) |
| Create tickets | ✅ | ✅ | ✅ |
| Upload evidence | ✅ | ✅ | ✅ |
| Verify tenant payment | ✅ | ✅ | ✅ |
| Initiate regulator payout | ✅ | ✅ | ✅ |
| Approve payout above threshold | ❌ | ✅ | ✅ |
| Manage staff and roles | ❌ | ❌ | ✅ |
| Configure catalog/rules | ❌ | ❌/✅ | ✅ |

---

## 4. Navigation and Routing

### 4.1 Sidebar Information Architecture

```
OPERATIONS
├── Inbox                    → /inbox (default landing)
├── Cases                    → /cases
└── Tickets & SLA            → /tickets

FINANCE
└── Payments & Payouts       → /payments

DOCUMENTS
└── Documents & Evidence     → /documents

DIRECTORY
└── Organizations            → /orgs

ADMIN (admin-only)
├── System Admin             → /admin
├── Catalog & Rules          → /catalog
└── Reports & Analytics      → /reports (manager+)
```

### 4.2 Route Map

| Route | Description | Access |
|-------|-------------|--------|
| `/` | Redirects to `/backoffice/queue` | All staff |
| `/backoffice/queue` | Unclaimed work queue | All staff |
| `/backoffice/workspace` | My claimed/assigned cases | All staff |
| `/backoffice/workspace/all` | All claimed cases | Manager+ |
| `/backoffice/workspace/staff/:userId` | Claimed cases by assignee | Manager+ |
| `/cases` | Legacy workspace wrapper (migration window) | All staff |
| `/inbox` | Legacy queue wrapper (migration window) | All staff |
| `/tickets` | Tickets with SLA tracking | All staff |
| `/payments` | Payments + payouts | All staff |
| `/documents` | Documents browser | All staff |
| `/orgs` | Organizations directory | All staff |
| `/orgs/:id` | Organization profile | All staff |
| `/admin` | System administration | Admin only |
| `/catalog` | Templates and rules | Admin only |
| `/reports` | Reports dashboard | Manager+ |

### 4.3 Legacy Route Deprecation

During migration, legacy inbox APIs remain functional with deprecation metadata:

- `Deprecation: true`
- `Sunset: Tue, 30 Jun 2026 23:59:59 GMT`
- `Link: <successor-route>; rel="successor-version"`

Successor APIs:

- Queue: `/backoffice/queue/*`
- Workspace: `/backoffice/workspace/*`

## 5. Core Concepts

### 5.1 Tenant Organization

A business using Bumara's managed compliance services.

**Attributes:**
- Profile (name, TPIN, contacts)
- Regulator connections (ZRA, PACRA, NAPSA, NHIMA)
- Obligations and filing calendar
- Documents and authorizations
- Plan and billing status

### 5.2 Obligation

A recurring or ad-hoc compliance requirement.

**Examples:** VAT return, PAYE submission, NAPSA contribution, annual returns.

### 5.3 Filing

A time-bound compliance submission generated from an obligation (monthly/quarterly/annual period).

### 5.4 Service Request

A non-recurring request initiated by tenant or staff.

**Examples:** PACRA name clearance, business registration, director changes.

### 5.5 Case (Backoffice Abstraction)

A **Case** is the unit of work shown to staff. It may represent:

- Filing case
- Service Request case
- Submission Job case

**Backoffice UI treats them uniformly:**
- Listed in `/cases`
- Single detail page at `/cases/:caseId`
- Shows checklist, timeline, evidence, tickets, payments

### 5.6 Submission Job

Created when tenant clicks **Request Submission** (or staff triggers). Tracks execution lifecycle and assignment.

### 5.7 Task

Work required from tenant or internal team to complete a filing/service request.

**Types:** Document upload, data entry, authorization, review.

### 5.8 Ticket

A structured request sent to tenant to provide missing data/docs or clarify issues.

**Lifecycle:** `OPEN` → `AWAITING_TENANT` → `TENANT_RESPONDED` → `RESOLVED` → `CLOSED`

### 5.9 Evidence

Documents proving:

| Evidence Type | Purpose |
|---------------|---------|
| `PAYMENT_PROOF` | Tenant payment received |
| `PAYOUT_PROOF` | Regulator payout made |
| `SUBMISSION_PROOF` | Submission completed (screenshot/PDF) |
| `OUTCOME_PROOF` | Acceptance/rejection from regulator |

### 5.10 SLA (Service Level Agreement)

Timers tracking service delivery:

- Time-to-first-response
- Time-to-submission
- Time-to-resolution
- Escalation thresholds

---

## 6. State Machines and Gates

### 6.1 Filing / Service Request Status

```
PENDING_DATA
    │
    ▼
IN_PROGRESS ◄───────────────────────────────────────┐
    │                                                │
    ▼                                                │
READY_FOR_SUBMISSION                                 │
    │                                                │
    ▼                                                │
SUBMISSION_IN_PROGRESS                               │
    │                                                │
    ▼                                                │
SUBMITTED ──────────► NEEDS_CORRECTION ──────────────┘
    │
    ▼
ACCEPTED

Terminal States: WAIVED | CANCELLED | REJECTED
```

### 6.2 Submission Job Status

```
QUEUED
    │
    ▼
ASSIGNED ◄──────────────────────────────────────────┐
    │                                                │
    ▼                                                │
IN_PROGRESS ──────► WAITING_ON_CLIENT ──────────────┤
    │                                                │
    ├─────────────► WAITING_ON_PAYMENT ─────────────┤
    │                                                │
    ▼                                                │
READY_TO_SUBMIT                                      │
    │                                                │
    ▼                                                │
SUBMITTED ──────────► NEEDS_CORRECTION ─────────────┘
    │
    ▼
ACCEPTED
    │
    ▼
CLOSED

Terminal: REJECTED | CANCELLED
```

### 6.3 Payment Request Status (Tenant → Bumara)

```
NOT_REQUIRED
    │ (if payment needed)
    ▼
REQUIRED_PENDING
    │
    ▼ (tenant pays)
PAID_UNVERIFIED
    │
    ▼ (staff verifies with evidence)
PAID_VERIFIED ──► (gates pass for submission)
    │
    └──► REFUNDED (if applicable)
```

### 6.4 Regulator Payout Status (Bumara → Regulator)

```
NOT_REQUIRED
    │ (if payout needed)
    ▼
QUEUED
    │
    ▼ (staff initiates payment)
SENT_UNVERIFIED
    │
    ▼ (evidence uploaded + approved)
PAID_VERIFIED ──► (gates pass for submission)
```

### 6.5 Readiness Gates

A Case **cannot move to `READY_TO_SUBMIT`** unless:

| Gate | Rule |
|------|------|
| Tasks | All required tenant tasks are `DONE` |
| Documents | All required documents are uploaded |
| Payment | Tenant payment is `PAID_VERIFIED` (if required) |
| Payout | Regulator payout is `PAID_VERIFIED` (if required) |
| Authorization | Valid authorization exists (if required) |

### 6.6 Override Rules

- Only `manager` or `admin` can override gates
- **Must supply `overrideReason`**
- Override is logged in audit with before/after state

---

## 7. User Journeys

### 7.1 Happy Path: Filing Submission

```
1. Filing generated (scheduled or created)
          │
          ▼
2. Tenant completes tasks → clicks "Request Submission"
          │
          ▼
3. System creates Submission Job → status QUEUED
          │
          ▼
4. Analyst opens /inbox → claims job (atomic or assigned by manager)
          │
          ▼
5. Analyst checks gates:
   ├── Missing? → Create Ticket → WAITING_ON_CLIENT
   └── All pass? → Continue
          │
          ▼
6. Payment required?
   └── Verify tenant payment → PAID_VERIFIED
          │
          ▼
7. Regulator payout required?
   └── Initiate payout → Upload proof → Manager approval → PAID_VERIFIED
          │
          ▼
8. Submit to regulator (manual via portal/office)
          │
          ▼
9. Upload submission evidence (PDF/screenshot)
          │
          ▼
10. Update status → SUBMITTED
          │
          ▼
11. Outcome confirmed:
    ├── ACCEPTED → Upload proof → CLOSED
    └── NEEDS_CORRECTION → Create Ticket → WAITING_ON_CLIENT
```

### 7.2 Service Request Flow

```
1. Tenant creates service request
          │
          ▼
2. Case appears in Inbox
          │
          ▼
3. Staff requests documents/choices → Ticket
          │
          ▼
4. Payment verified (if required)
          │
          ▼
5. Submit to regulator (manual)
          │
          ▼
6. Upload evidence + outcome
          │
          ▼
7. Close
```

### 7.3 Ticket Lifecycle

```
Staff creates ticket (missing info/docs)
          │
          ▼
       OPEN
          │
          ▼ (sent to tenant)
    AWAITING_TENANT
          │
          ▼ (tenant responds)
   TENANT_RESPONDED
          │
          ▼ (staff reviews)
    RESOLVED / CLOSED
```

**Rules:**
- Tickets must link to a Case
- Tenant responses can attach files/comments
- SLA timers apply (response time)

---

## 8. UI Specification

### 8.1 Global Layout

```
┌──────────────────────────────────────────────────────────────────┐
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ TOPBAR: Search │ Quick Actions │ Notifications │ Profile   │ │
│  └─────────────────────────────────────────────────────────────┘ │
│  ┌───────────┬─────────────────────────────────────────────────┐ │
│  │           │                                                 │ │
│  │  SIDEBAR  │              MAIN CONTENT                       │ │
│  │           │                                                 │ │
│  │  Ops      │   ┌─────────────────────────────────────────┐   │ │
│  │  - Inbox  │   │                                         │   │ │
│  │  - Cases  │   │     Table / List / Detail View          │   │ │
│  │  - Tickets│   │                                         │   │ │
│  │           │   │                                         │   │ │
│  │  Finance  │   │                                         │   │ │
│  │  - Payments│  │                                         │   │ │
│  │           │   │                                         │   │ │
│  │  ...      │   └─────────────────────────────────────────┘   │ │
│  │           │                                                 │ │
│  └───────────┴─────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

### 8.2 Inbox Page (`/inbox`)

**Purpose:** Show actionable work requiring staff action.

**Queue Tabs (MVP):**
- Unassigned
- Assigned to me
- Due soon (next N days)
- Overdue
- Waiting on client
- Waiting on payment
- Ready to submit
- Submitted awaiting outcome

**Table Columns:**
| Column | Description |
|--------|-------------|
| Case ID | Unique identifier |
| Type | Filing / Service / Job |
| Organization | Tenant name |
| Regulator | ZRA, PACRA, etc. |
| Obligation | If filing |
| Status | Current status |
| Due Date | With SLA indicator |
| Assigned To | Staff member |
| Priority | LOW / NORMAL / HIGH / URGENT |

**Actions:**
- Claim / Assign to me (atomic)
- Reassign (manager/admin only)
- Change priority
- Open case

### 8.3 Cases List (`/cases`)

Unified list with type tabs:

```
[All] [Filings] [Service Requests] [Submission Jobs]
```

Query param `?type=` controls active tab.

Same table/filters as Inbox but not queue-specific.

### 8.4 Case Detail (`/cases/:id`)

**This is the most important screen.**

**Header:**
- Case title (org + obligation/service)
- Status badge
- Due date + SLA indicator
- Assigned to + reassign button
- Priority selector
- Quick action buttons

**Sections/Tabs:**

| Tab | Content |
|-----|---------|
| **Overview** | Org info, regulator, obligation, key dates |
| **Checklist** | Readiness gates with blockers + resolve buttons |
| **Tasks** | Tenant tasks list + status |
| **Payments** | Tenant payments + regulator payouts |
| **Documents** | Attached docs with tags + upload |
| **Tickets** | Open/closed tickets for this case |
| **Timeline** | Append-only audit events |
| **Notes** | Internal-only notes |

### 8.5 Tickets Page (`/tickets`)

- View all tickets with filters
- SLA timers displayed
- Filter by: status, org, case, overdue

### 8.6 Payments Page (`/payments`)

**Two tabs:**
- Tenant Payments (incoming)
- Regulator Payouts (outgoing)

**Actions:**
- Verify payment (requires evidence)
- Initiate payout
- Approve payout (manager/admin)
- Attach evidence

### 8.7 Documents Page (`/documents`)

- Search by org, case, tag, date
- Upload with tagging
- Link to cases

**Document Tags:**
| Tag | Purpose |
|-----|---------|
| `SOURCE` | Tenant uploaded |
| `WORKPAPER` | Internal working document |
| `PAYMENT_PROOF` | Payment evidence |
| `PAYOUT_PROOF` | Payout evidence |
| `SUBMISSION_PROOF` | Submission screenshot/PDF |
| `OUTCOME_PROOF` | Acceptance/rejection |

### 8.8 Organizations Page (`/orgs`)

- Search-first interface
- Organization profile with:
  - Compliance snapshot (open cases, overdue, upcoming)
  - Regulator connections
  - Documents
  - Recent activity timeline
  - Contacts

### 8.9 Admin Page (`/admin`)

- Staff management (invite, role change, disable)
- Role policy configuration
- Catalog management (templates, SLA policies)

---

## 9. Security Requirements

### 9.1 Access Control

| Requirement | Implementation |
|-------------|----------------|
| Staff-only access | Clerk backoffice org membership |
| Tenant blocked | Middleware rejects non-staff |
| Admin routes | Server-side role check + UI hidden |
| Data isolation | Backoffice can view all orgs |

### 9.2 Defense in Depth

```
Layer 1: Next.js Middleware
   └── Check Clerk session
   └── Check backoffice org membership
   └── Check email domain

Layer 2: Server Component Guards
   └── AuthGuard
   └── ServerOrganizationGuard
   └── BackofficeOrgGuard
   └── MfaGuard

Layer 3: Route Guards
   └── requireAdmin()
   └── requireManagerOrAbove()

Layer 4: API Middleware
   └── requireBackofficeOrg()
   └── requireActiveStaff()
   └── requireRole(['admin'])
```

### 9.3 Operational Safety

| Requirement | Implementation |
|-------------|----------------|
| Atomic assignment | Prevent double-claim with DB transaction |
| Server-side transitions | Validate state machine rules |
| Override reason | Required for gate bypasses |
| Idempotency | Keys for critical operations |
| Audit logging | Every mutation creates audit event |

### 9.4 Data Protection

- Minimum necessary data shown
- Evidence documents may contain sensitive info
- Role-based visibility for sensitive fields
- Log all downloads

---

## 10. Acceptance Criteria

### 10.1 Inbox

- [ ] Staff can filter and claim cases
- [ ] Unassigned cases can only be claimed once (atomic)
- [ ] SLA indicators visible
- [ ] Queue counts accurate

### 10.2 Case Detail

- [ ] Checklist displays all gates + blockers
- [ ] Staff can create tickets
- [ ] Status changes require valid transition
- [ ] Evidence upload works and links to case
- [ ] Timeline shows all events

### 10.3 Tickets

- [ ] Ticket creation notifies tenant
- [ ] Ticket status updates tracked in timeline
- [ ] SLA timers displayed

### 10.4 Payments & Payouts

- [ ] Payment cannot be verified without proof
- [ ] Payout approval required above threshold
- [ ] All actions logged

### 10.5 Security

- [ ] Non-staff cannot access app/routes/endpoints
- [ ] Non-admin cannot access admin routes
- [ ] All sensitive actions create audit events
- [ ] Overrides require reason

---

## Appendix: Open Questions

1. Do we implement a physical `cases` table, or compute "case view" from filings/service requests/jobs?
2. What payout approval threshold(s) require manager approval?
3. Which payment methods are supported (mobile money, bank transfer, card)?
4. Do we require two-person review for certain regulators?
5. Which SLA timers matter first: first response vs submission vs resolution?

