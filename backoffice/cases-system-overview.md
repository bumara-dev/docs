---
title: "Cases System: End-to-End Overview"
description: "Comprehensive analysis of the Bumara Backoffice Cases system covering business requirements, technical architecture, and implementation details."
---

**Last Updated:** 2026-02-05
**Status:** Living Document
**Audience:** Product, Engineering, Operations

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Business Context](#2-business-context)
3. [Business Requirements](#3-business-requirements)
4. [Technical Architecture](#4-technical-architecture)
5. [Data Model & State Machines](#5-data-model--state-machines)
6. [User Workflows](#6-user-workflows)
7. [API Architecture](#7-api-architecture)
8. [Frontend Implementation](#8-frontend-implementation)
9. [Security & Compliance](#9-security--compliance)
10. [Integration Points](#10-integration-points)
11. [Operational Considerations](#11-operational-considerations)

---

## 1. Executive Summary

### What is a "Case"?

A **Case** is the unified abstraction used by Bumara's backoffice staff to manage all compliance work. It represents any work item requiring staff intervention, regardless of whether it originates from:

- **Filing** (time-bound regulatory submission like VAT returns, PAYE)
- **Service Request** (one-off tasks like PACRA name clearance, business registration)
- **Submission Job** (execution-level tracking of the actual submission process)

### Why Cases Matter

Cases are central to Bumara's **managed compliance service model**:

1. **Tenants** prepare data and documents, then request submission
2. **Backoffice staff** verify readiness, pay regulators, and manually submit
3. **Evidence** is captured for every action
4. **Full audit trail** is maintained for compliance and accountability

### Key Characteristics

```
┌─────────────────────────────────────────────────────────────┐
│ CASE LIFECYCLE                                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. INTAKE       → Case enters backoffice queue            │
│  2. ASSIGNMENT   → Staff claims or is assigned             │
│  3. VERIFICATION → Check readiness gates (payment, docs)   │
│  4. EXECUTION    → Manual submission to regulator          │
│  5. EVIDENCE     → Upload proof of submission              │
│  6. OUTCOME      → Record acceptance/rejection             │
│  7. CLOSURE      → Complete audit trail                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Business Context

### 2.1 The Bumara Service Model

Bumara operates as a **managed compliance service**, not a self-serve SaaS:

| Aspect | Bumara Model | Typical SaaS |
|--------|--------------|--------------|
| Tenant Role | Prepares data & requests submission | Does everything themselves |
| Staff Role | Verifies, pays, submits manually | Minimal involvement |
| Automation | Regulated workflows, manual submission | Full automation |
| Evidence | Required for every action | Optional audit logs |
| Responsibility | Bumara liable for compliance | Tenant liable |

**Why Manual Submission?**
- Zambian regulators (ZRA, PACRA, NAPSA, NHIMA) lack full API access
- Many submissions require physical presence or portal access
- Regulatory environment requires human verification
- Compliance risk is borne by Bumara, not the tenant

### 2.2 Stakeholders

```
┌──────────────────────────────────────────────────────────────┐
│                      STAKEHOLDERS                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  TENANT ORGANIZATION                                         │
│  ├── Prepares financial data                                │
│  ├── Uploads required documents                             │
│  ├── Makes payment to Bumara                                │
│  └── Requests submission when ready                         │
│                                                              │
│  BACKOFFICE STAFF                                            │
│  ├── Ops Analyst: Claims cases, executes submissions        │
│  ├── Ops Manager: Assigns work, approves high-value payouts │
│  └── System Admin: Staff management, configuration          │
│                                                              │
│  REGULATORS                                                  │
│  ├── ZRA (Tax), PACRA (Business Registration)               │
│  ├── NAPSA (Social Security), NHIMA (Healthcare)            │
│  └── WCFCB (Workers Comp), LCC (Liquor Control)             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 2.3 Business Objectives

1. **Compliance Certainty**: 100% on-time, accurate regulatory submissions
2. **Evidence Trail**: Complete audit trail for every action
3. **Operational Efficiency**: Staff can handle high volume with minimal errors
4. **Tenant Trust**: Transparent status, clear communication
5. **Risk Management**: Prevent submissions that violate readiness gates

---

## 3. Business Requirements

### 3.1 Core Functional Requirements

#### FR-1: Work Queue (Inbox)
- Staff must see all actionable work in priority order
- Cases must be filterable by status, regulator, due date, assignment
- Queue tabs organize work by context (unassigned, due soon, waiting on payment, etc.)
- Real-time counts for each queue tab

#### FR-2: Case Assignment
- Unassigned cases can be claimed atomically (prevent double-claiming)
- Managers can reassign cases to specific staff members
- Assignment history tracked in audit log

#### FR-3: Readiness Verification (Gates)
A case cannot proceed to submission unless all gates pass:

| Gate | Requirement |
|------|-------------|
| **Tasks** | All required tenant tasks are complete |
| **Documents** | All required documents uploaded |
| **Payment** | Tenant payment verified (if required) |
| **Payout** | Regulator payout made & verified (if required) |
| **Authorization** | Valid regulator credentials/authorization |

#### FR-4: Status Transitions
- Cases follow strict state machine rules
- Invalid transitions are blocked server-side
- Managers can override with reason (logged in audit)

#### FR-5: Evidence Management
Every critical action requires evidence:
- Payment verification → upload payment proof
- Regulator payout → upload transaction proof
- Submission → upload screenshot/PDF from regulator portal
- Outcome → upload acceptance/rejection document

#### FR-6: Communication (Tickets)
- Staff can create tickets requesting information from tenants
- Tickets have SLA timers (time-to-response, time-to-resolution)
- Ticket responses captured in timeline
- Tenants notified via in-app, email, SMS, WhatsApp

#### FR-7: Audit Trail
- Every mutation creates a timeline event
- Events show: who, what, when, why
- Timeline visible on case detail page
- Immutable after creation

### 3.2 Non-Functional Requirements

#### NFR-1: Performance
- Cases list loads in &lt; 2 seconds
- Search/filtering responds in &lt; 500ms
- Pagination required (no "load all")
- Queue counts cached for 30 seconds

#### NFR-2: Security
- All endpoints require backoffice authentication
- Role-based access control (member, manager, admin)
- Sensitive actions require manager+ role
- Multi-tenancy: backoffice can view all organizations

#### NFR-3: Data Integrity
- Atomic claim operations (database-level locking)
- State transitions validated server-side
- Idempotent operations where possible
- Audit logs written in same transaction as mutations

#### NFR-4: Availability
- System must be available during business hours (08:00-18:00 CAT)
- Graceful degradation if regulators are down
- Offline mode not required (desktop web app)

#### NFR-5: Usability
- Mobile-responsive (tablets used in field)
- Accessible (keyboard navigation, screen readers)
- Clear loading/error/empty states
- Inline validation with helpful error messages

---

## 4. Technical Architecture

### 4.1 System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    BUMARA PLATFORM                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐              ┌─────────────────┐          │
│  │  TENANT APP     │              │  BACKOFFICE     │          │
│  │  (Next.js)      │              │  (Next.js)      │          │
│  │                 │              │                 │          │
│  │  • Dashboard    │              │  • Inbox/Queue  │          │
│  │  • Tasks        │ Request      │  • Cases        │          │
│  │  • Documents    │ Submission   │  • Tickets      │          │
│  └────────┬────────┘ ────────────►│  • Payments     │          │
│           │                        │  • Documents    │          │
│           │                        └────────┬────────┘          │
│           │                                 │                   │
│           │      ┌──────────────────────────┘                   │
│           │      │                                              │
│           ▼      ▼                                              │
│  ┌─────────────────────────────────────────────────────┐       │
│  │           SHARED BACKEND (Hono RPC)                 │       │
│  │  ┌─────────────────────────────────────────────┐    │       │
│  │  │  API Layer (packages/backend)               │    │       │
│  │  │  • Auth middleware (Clerk integration)      │    │       │
│  │  │  • Route handlers (cases, queue, tickets)   │    │       │
│  │  │  • Validation (Zod schemas)                 │    │       │
│  │  └──────────────────┬──────────────────────────┘    │       │
│  │                     ▼                                │       │
│  │  ┌─────────────────────────────────────────────┐    │       │
│  │  │  Business Logic (packages/api-services)     │    │       │
│  │  │  • State machines                           │    │       │
│  │  │  • Readiness gates checker                  │    │       │
│  │  │  • Validation rules                         │    │       │
│  │  └──────────────────┬──────────────────────────┘    │       │
│  │                     ▼                                │       │
│  │  ┌─────────────────────────────────────────────┐    │       │
│  │  │  Data Layer (packages/database)             │    │       │
│  │  │  • Drizzle ORM                              │    │       │
│  │  │  • Schema definitions                       │    │       │
│  │  │  • Repositories (query logic)               │    │       │
│  │  └──────────────────┬──────────────────────────┘    │       │
│  └────────────────────┼─────────────────────────────────┘       │
│                       ▼                                         │
│  ┌─────────────────────────────────────────────────────┐       │
│  │           PostgreSQL DATABASE                        │       │
│  │  • organizations                                     │       │
│  │  • filings, service_requests, submission_jobs        │       │
│  │  • tasks, tickets, documents                         │       │
│  │  • payment_requests, regulator_payouts               │       │
│  │  • timeline_events (audit log)                       │       │
│  │  • back_office_agents                                │       │
│  └─────────────────────────────────────────────────────┘       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    EXTERNAL SERVICES                            │
├─────────────────────────────────────────────────────────────────┤
│  • Clerk (Authentication & Authorization)                       │
│  • AWS S3 (Document storage)                                    │
│  • Regulators (Manual submission via portals)                   │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Technology Stack

#### Frontend (apps/backoffice)
- **Framework**: Next.js 16 (App Router, React Server Components)
- **Styling**: Tailwind CSS + shadcn/ui design system
- **State Management**:
  - React Query (server state)
  - URL search params (filter state)
  - Local component state (UI state)
- **Forms**: react-hook-form + Zod validation
- **API Client**: Hono RPC client (type-safe)
- **Auth**: Clerk (organization-based auth)

#### Backend (packages/backend)
- **Framework**: Hono (lightweight, fast, type-safe)
- **Validation**: Zod schemas (shared with frontend)
- **ORM**: Drizzle (type-safe, SQL-first)
- **Database**: PostgreSQL (with indexes for performance)
- **API Style**: RPC (not REST) for type safety

#### Shared Packages
- **packages/api-services**: Business logic (state machines, gates, rules)
- **packages/database**: Schema + repositories
- **packages/design-system**: UI components (shadcn/ui)
- **packages/auth**: Clerk utilities

### 4.3 Deployment Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  MONOREPO (Turborepo)                                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  apps/backoffice/     →  Deployed to Vercel                │
│  packages/backend/    →  Deployed to Cloudflare Workers    │
│  packages/database/   →  Connects to managed PostgreSQL    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Data Model & State Machines

### 5.1 Core Entities

```
┌──────────────────────────────────────────────────────────────┐
│                    ENTITY RELATIONSHIPS                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Organization (Tenant)                                       │
│      ├── has many → Filings                                 │
│      ├── has many → Service Requests                        │
│      └── has one → Profile (TPIN, contacts, etc.)           │
│                                                              │
│  Filing (recurring compliance)                               │
│      ├── belongs to → Organization                          │
│      ├── belongs to → Obligation (template)                 │
│      ├── has many → Tasks                                   │
│      ├── has many → Documents                               │
│      ├── has one → Payment Request                          │
│      ├── has one → Regulator Payout                         │
│      ├── generates → Submission Job                         │
│      └── tracks → Timeline Events                           │
│                                                              │
│  Service Request (one-off request)                           │
│      ├── belongs to → Organization                          │
│      ├── has many → Tasks                                   │
│      ├── has many → Documents                               │
│      ├── has one → Payment Request                          │
│      ├── has one → Regulator Payout                         │
│      ├── generates → Submission Job                         │
│      └── tracks → Timeline Events                           │
│                                                              │
│  Submission Job (execution tracking)                         │
│      ├── references → Filing OR Service Request             │
│      ├── belongs to → Organization                          │
│      ├── assigned to → Back Office Agent                    │
│      ├── has status (queued, assigned, submitted, etc.)     │
│      └── tracks priority, due date, SLA                     │
│                                                              │
│  Task (work item for tenant)                                 │
│      ├── belongs to → Filing OR Service Request             │
│      ├── has status (todo, doing, blocked, done, skipped)   │
│      └── required flag (blocks submission if incomplete)    │
│                                                              │
│  Ticket (communication with tenant)                          │
│      ├── belongs to → Filing OR Service Request             │
│      ├── has type (data_request, clarification, etc.)       │
│      ├── tracks SLA (time-to-response, time-to-resolution)  │
│      └── has responses (staff and tenant messages)          │
│                                                              │
│  Document (evidence or source)                               │
│      ├── belongs to → Organization                          │
│      ├── optionally links to → Filing OR Service Request    │
│      ├── has kind (source, payment_proof, outcome, etc.)    │
│      └── stored in S3 with reference in DB                  │
│                                                              │
│  Payment Request (tenant → Bumara)                           │
│      ├── belongs to → Filing OR Service Request             │
│      ├── has status (required, paid_unverified, verified)   │
│      ├── verified by → Back Office Agent                    │
│      └── links to → Document (payment proof)                │
│                                                              │
│  Regulator Payout (Bumara → regulator)                      │
│      ├── belongs to → Filing OR Service Request             │
│      ├── has status (queued, sent, verified)                │
│      ├── requires approval if > threshold                   │
│      ├── paid by → Back Office Agent                        │
│      └── links to → Document (payout proof)                 │
│                                                              │
│  Timeline Event (audit log)                                  │
│      ├── belongs to → Organization                          │
│      ├── links to → Filing OR Service Request               │
│      ├── has actor (staff or tenant user)                   │
│      ├── tracks action (status_changed, assigned, etc.)     │
│      └── immutable (append-only)                            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 5.2 State Machines

#### Submission Job Status Flow

```
┌─────────────────────────────────────────────────────────────┐
│  SUBMISSION JOB STATE MACHINE                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  QUEUED (initial state)                                     │
│    │                                                        │
│    ├─► Staff clicks "Claim" ────────────────────────────┐   │
│    │                                                     │   │
│    ▼                                                     ▼   │
│  ASSIGNED                                      [CANCELLED]   │
│    │                                                         │
│    ├─► Staff starts work ──────────────────────────────┐    │
│    │                                                    │    │
│    ▼                                                    │    │
│  IN_PROGRESS                                            │    │
│    │                                                    │    │
│    ├─► Missing info? ────────► WAITING_ON_CLIENT ──────┘    │
│    │                               │                        │
│    │                               │ Tenant responds        │
│    │                               └──────────────────┐     │
│    │                                                  │     │
│    ├─► Payment pending? ───────► WAITING_ON_PAYMENT ─┘     │
│    │                                                        │
│    ├─► Gates pass? ────────────────────────────────┐       │
│    │                                                │       │
│    ▼                                                │       │
│  READY_TO_SUBMIT                                    │       │
│    │                                                │       │
│    ├─► Staff clicks "Submit" ──────────────────┐   │       │
│    │                                            │   │       │
│    ▼                                            │   │       │
│  SUBMITTED                                      │   │       │
│    │                                            │   │       │
│    ├─► Regulator approves ──────────┐          │   │       │
│    │                                 │          │   │       │
│    │                                 ▼          │   │       │
│    │                            ACCEPTED        │   │       │
│    │                                 │          │   │       │
│    │                                 ▼          │   │       │
│    │                            CLOSED          │   │       │
│    │                                            │   │       │
│    ├─► Regulator rejects ───► NEEDS_CORRECTION ┘   │       │
│    │                               │                │       │
│    │                               └── loops back ──┘       │
│    │                                                        │
│    └─► Regulator denies ────────────► [REJECTED]           │
│                                                             │
│  Terminal states: CANCELLED, REJECTED, CLOSED              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### Filing/Service Request Status Flow

```
┌─────────────────────────────────────────────────────────────┐
│  FILING / SERVICE REQUEST STATE MACHINE                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  PENDING_DATA (tenant preparing)                            │
│    │                                                        │
│    ▼                                                        │
│  IN_PROGRESS (tenant working on tasks)                      │
│    │                                                        │
│    ├─► Tasks complete + docs uploaded ──────────┐          │
│    │                                             │          │
│    ▼                                             │          │
│  READY_FOR_SUBMISSION                            │          │
│    │                                             │          │
│    ├─► Tenant clicks "Request Submission" ──────┘          │
│    │      (creates Submission Job)                         │
│    │                                                        │
│    ▼                                                        │
│  SUBMISSION_IN_PROGRESS                                     │
│    │                                                        │
│    ├─► Staff submits ───────────────────────────┐          │
│    │                                             │          │
│    ▼                                             │          │
│  SUBMITTED                                       │          │
│    │                                             │          │
│    ├─► Outcome received ─────────┐              │          │
│    │                              │              │          │
│    ▼                              ▼              │          │
│  ACCEPTED                    NEEDS_CORRECTION    │          │
│                                   │              │          │
│                                   └── loops back ┘          │
│                                                             │
│  Other transitions:                                         │
│  • Any → WAIVED (manager marks as not required)            │
│  • Any → CANCELLED (tenant or manager cancels)             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 Readiness Gates Logic

```typescript
// Pseudo-code for gate checking

interface Gate {
  gate: string;
  label: string;
  passed: boolean;
  required: boolean;
  blockers: string[];
}

function checkReadinessGates(
  organizationId: string,
  caseId: string,
  caseType: "filing" | "service_request"
): { gates: Gate[]; ready: boolean } {

  const gates: Gate[] = [];

  // Gate 1: Tasks
  const tasksGate = await checkTasksGate(caseId, caseType);
  gates.push(tasksGate);

  // Gate 2: Documents
  const docsGate = await checkDocumentsGate(caseId, caseType);
  gates.push(docsGate);

  // Gate 3: Payment (if required)
  const paymentGate = await checkPaymentGate(caseId, caseType);
  gates.push(paymentGate);

  // Gate 4: Payout (if required)
  const payoutGate = await checkPayoutGate(caseId, caseType);
  gates.push(payoutGate);

  // Gate 5: Authorization (if required)
  const authGate = await checkAuthorizationGate(organizationId, regulatorKey);
  gates.push(authGate);

  // All required gates must pass
  const ready = gates
    .filter(g => g.required)
    .every(g => g.passed);

  return { gates, ready };
}

// Example gate checker
async function checkTasksGate(caseId, caseType): Promise<Gate> {
  const tasks = await db.tasks.findMany({
    where: { [caseType + "Id"]: caseId }
  });

  const requiredTasks = tasks.filter(t => t.required);
  const completedRequired = requiredTasks.filter(t => t.status === "done");

  const passed = completedRequired.length === requiredTasks.length;

  return {
    gate: "tasks",
    label: "Required Tasks",
    passed,
    required: true,
    blockers: passed ? [] : [
      `${requiredTasks.length - completedRequired.length} required tasks not complete`
    ],
  };
}
```

---

## 6. User Workflows

### 6.1 Happy Path: Complete Filing Submission

```
┌─────────────────────────────────────────────────────────────┐
│  END-TO-END WORKFLOW: VAT RETURN SUBMISSION                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  TENANT SIDE:                                               │
│  ──────────                                                 │
│  1. Tenant logs into tenant app                             │
│  2. Sees "VAT Return - Q4 2025" with status "In Progress"   │
│  3. Completes required tasks:                               │
│     ├─ Upload sales ledger                                  │
│     ├─ Upload purchase ledger                               │
│     ├─ Review VAT calculations                              │
│     └─ Mark data entry complete                             │
│  4. System auto-generates payment request                   │
│  5. Tenant pays ZMW 2,450 to Bumara (service fee + VAT)     │
│  6. Tenant clicks "Request Submission"                      │
│     └─> Creates Submission Job in QUEUED state             │
│                                                             │
│  ──────────────────────────────────────────────────────────│
│                                                             │
│  BACKOFFICE SIDE:                                           │
│  ────────────────                                           │
│  7. Staff (Jane) opens /inbox                               │
│  8. Sees "Unassigned" tab with new VAT return              │
│  9. Clicks "Claim" → atomically assigns to self            │
│     └─> Status: QUEUED → ASSIGNED                          │
│  10. Opens case detail page                                 │
│  11. Reviews readiness checklist:                           │
│      ✓ Tasks complete (3/3)                                 │
│      ✓ Documents uploaded (2/2)                             │
│      ✗ Payment verified (pending)                           │
│      ✓ Payout N/A (ZRA pays Bumara, not reverse)           │
│      ✓ Authorization valid (ZRA credentials active)         │
│  12. Clicks "Verify Payment" on Payments tab                │
│      ├─ Checks bank statement                               │
│      ├─ Uploads payment proof                               │
│      └─ Marks as verified                                   │
│          └─> Payment Request: PAID_UNVERIFIED → VERIFIED    │
│  13. Returns to checklist - now ALL gates pass ✓            │
│  14. Clicks "Mark Ready to Submit"                          │
│      └─> Status: ASSIGNED → READY_TO_SUBMIT                │
│  15. Clicks "Start Submission"                              │
│      └─> Status: READY_TO_SUBMIT → SUBMISSION_IN_PROGRESS  │
│  16. Logs into ZRA portal manually                          │
│  17. Enters VAT data from tenant's ledgers                  │
│  18. Submits return on ZRA portal                           │
│  19. Takes screenshot of confirmation page                  │
│  20. Returns to backoffice, uploads submission proof        │
│  21. Clicks "Mark as Submitted"                             │
│      └─> Status: SUBMISSION_IN_PROGRESS → SUBMITTED        │
│  22. Waits for ZRA to process (typically 1-3 days)          │
│  23. Receives email from ZRA: "Return Accepted"             │
│  24. Downloads acceptance PDF from ZRA portal               │
│  25. Uploads outcome proof to backoffice                    │
│  26. Clicks "Mark as Accepted"                              │
│      └─> Status: SUBMITTED → ACCEPTED                      │
│  27. System auto-transitions Submission Job to CLOSED       │
│                                                             │
│  ──────────────────────────────────────────────────────────│
│                                                             │
│  TENANT NOTIFICATION:                                       │
│  ────────────────────                                       │
│  28. Tenant receives notification: "VAT return accepted"    │
│  29. Can download acceptance certificate from tenant app    │
│                                                             │
│  ✓ Complete audit trail preserved                           │
│  ✓ Evidence documents stored (payment, submission, outcome) │
│  ✓ Timeline shows all actions with timestamps & actors      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 Alternative Flow: Needs Correction

```
┌─────────────────────────────────────────────────────────────┐
│  CORRECTION WORKFLOW                                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ... (steps 1-21 same as above) ...                         │
│                                                             │
│  22. ZRA responds: "Error in purchase ledger totals"        │
│  23. Staff creates ticket:                                  │
│      ├─ Type: CORRECTION                                    │
│      ├─ Subject: "VAT return needs correction"              │
│      ├─ Description: "ZRA rejected due to discrepancy..."   │
│      └─ Attaches ZRA rejection notice                       │
│  24. Clicks "Needs Correction"                              │
│      └─> Status: SUBMITTED → NEEDS_CORRECTION              │
│  25. Submission Job status: SUBMITTED → WAITING_ON_CLIENT   │
│                                                             │
│  ──────────────────────────────────────────────────────────│
│                                                             │
│  26. Tenant receives notification + ticket                  │
│  27. Tenant corrects purchase ledger                        │
│  28. Re-uploads corrected document                          │
│  29. Responds to ticket: "Corrected and re-uploaded"        │
│      └─> Ticket: AWAITING_TENANT → TENANT_RESPONDED        │
│  30. Clicks "Request Re-Submission"                         │
│                                                             │
│  ──────────────────────────────────────────────────────────│
│                                                             │
│  31. Staff reviews tenant's correction                      │
│  32. Marks ticket as resolved                               │
│  33. Re-submits to ZRA with corrected data                  │
│  34. ... (continues as normal submission flow) ...          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6.3 Edge Cases & Error Handling

#### Case 1: Payment Not Received
```
Problem: Tenant requests submission but hasn't paid
Gate: Payment verification fails
Action:
  1. Staff creates ticket: "Payment Required"
  2. Submission Job → WAITING_ON_PAYMENT
  3. Cannot proceed until payment verified
```

#### Case 2: Missing Required Documents
```
Problem: Tenant forgot to upload authorization letter
Gate: Documents gate fails
Action:
  1. Staff creates ticket: "Missing Authorization"
  2. Lists required documents
  3. Submission Job → WAITING_ON_CLIENT
  4. Tenant uploads, responds to ticket
  5. Staff verifies and proceeds
```

#### Case 3: Regulator Portal Down
```
Problem: ZRA portal unavailable during submission
Action:
  1. Staff adds note to case
  2. Keeps status as READY_TO_SUBMIT
  3. Monitors regulator status
  4. Submits when portal available
  5. May need to override SLA if regulator outage causes delay
```

#### Case 4: Double-Claim Race Condition
```
Problem: Two staff try to claim same case simultaneously
Solution:
  Database: UPDATE submission_jobs
            SET assigned_to = 'agent-1'
            WHERE id = 'case-123'
              AND assigned_to IS NULL
  Result: Only one UPDATE succeeds (returns 1 row)
          Other staff gets conflict error
```

#### Case 5: Manager Override Required
```
Problem: Payment delayed, but filing due today
Action:
  1. Staff attempts transition, blocked by payment gate
  2. Manager reviews situation
  3. Manager clicks "Override Gate: Payment"
  4. Enters reason: "Payment confirmed via phone, proof pending"
  5. System logs override in audit trail
  6. Submission proceeds with override flag
```

---

## 7. API Architecture

### 7.1 Endpoint Structure

```
BASE_URL: https://api.bumara.com/api/v1

BACKOFFICE ENDPOINTS:
└── /backoffice/
    ├── /cases
    │   ├── GET    /                    List cases
    │   ├── GET    /{id}                Get case detail
    │   ├── POST   /{id}/claim          Claim case
    │   ├── POST   /{id}/assign         Assign to agent
    │   ├── POST   /{id}/transition     Change status
    │   └── POST   /{id}/override-gate  Override readiness gate
    │
    ├── /queue
    │   ├── GET    /                    List queue items
    │   └── GET    /counts              Get queue tab counts
    │
    ├── /tickets
    │   ├── GET    /                    List tickets
    │   ├── POST   /                    Create ticket
    │   ├── GET    /{id}                Get ticket detail
    │   ├── POST   /{id}/notes          Add internal note
    │   ├── POST   /{id}/reply          Reply to tenant
    │   ├── POST   /{id}/resolve        Mark resolved
    │   └── POST   /{id}/close          Close ticket
    │
    ├── /payments
    │   ├── GET    /                    List tenant payments
    │   └── POST   /{id}/verify         Verify payment
    │
    ├── /payouts
    │   ├── GET    /                    List regulator payouts
    │   ├── POST   /                    Create payout
    │   ├── POST   /{id}/pay            Mark as paid
    │   ├── POST   /{id}/approve        Approve payout
    │   └── POST   /{id}/verify         Verify completion
    │
    └── /documents
        ├── GET    /                    List documents
        ├── POST   /upload-url          Get presigned S3 URL
        └── POST   /{id}/attach         Link doc to case
```

### 7.2 Request/Response Formats

#### List Cases
```http
GET /backoffice/cases?type=filing&status=queued&page=1&pageSize=20

Response:
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "caseType": "submission_job",
      "organizationId": "uuid",
      "organizationName": "Acme Corp Ltd",
      "regulatorKey": "zra",
      "obligationName": "VAT Return - Q4 2025",
      "serviceName": null,
      "periodLabel": "Q4 2025",
      "status": "queued",
      "priority": "normal",
      "dueOn": "2025-12-31T00:00:00Z",
      "assignedToAgentId": null,
      "assignedToAgentName": null,
      "requestedAt": "2025-12-15T10:30:00Z",
      "createdAt": "2025-12-15T10:30:00Z",
      "updatedAt": "2025-12-15T10:30:00Z"
    }
  ],
  "meta": {
    "page": 1,
    "pageSize": 20,
    "total": 45,
    "totalPages": 3
  }
}
```

#### Get Case Detail
```http
GET /backoffice/cases/{id}

Response:
{
  "success": true,
  "data": {
    "id": "uuid",
    "caseType": "submission_job",
    "organizationId": "uuid",
    "organizationName": "Acme Corp Ltd",
    "regulatorKey": "zra",
    "obligationName": "VAT Return - Q4 2025",
    "status": "assigned",
    "priority": "high",
    "dueOn": "2025-12-31T00:00:00Z",
    "assignedToAgentId": "agent-uuid",
    "sourceEntityId": "filing-uuid",
    "submissionJobId": "uuid",
    "tasksTotal": 3,
    "tasksDone": 3,
    "docsRequired": 2,
    "docsUploaded": 2,
    "gates": [
      {
        "gate": "tasks",
        "label": "Required Tasks",
        "passed": true,
        "required": true,
        "blockers": []
      },
      {
        "gate": "documents",
        "label": "Required Documents",
        "passed": true,
        "required": true,
        "blockers": []
      },
      {
        "gate": "payment",
        "label": "Payment Verified",
        "passed": false,
        "required": true,
        "blockers": ["Payment not yet verified"]
      },
      {
        "gate": "payout",
        "label": "Regulator Payout",
        "passed": true,
        "required": false,
        "blockers": []
      }
    ],
    "isReady": false
  }
}
```

#### Claim Case
```http
POST /backoffice/cases/{id}/claim

Response (Success):
{
  "success": true,
  "data": {
    "id": "uuid",
    "status": "assigned",
    "assignedToAgentId": "current-agent-uuid",
    ...
  }
}

Response (Conflict):
{
  "success": false,
  "error": "CONFLICT",
  "message": "Case already assigned or not in queued state"
}
```

#### Transition Status
```http
POST /backoffice/cases/{id}/transition
Content-Type: application/json

{
  "status": "submitted",
  "reason": "optional reason for override"
}

Response (Success):
{
  "success": true,
  "data": {
    "id": "uuid",
    "status": "submitted",
    ...
  }
}

Response (Blocked):
{
  "success": false,
  "error": "BLOCKED",
  "message": "Submission is blocked by readiness gates",
  "blockers": [
    "payment: Payment not yet verified"
  ]
}
```

### 7.3 Authentication & Authorization

Every backoffice request requires:

```typescript
Headers:
  Authorization: Bearer <clerk-jwt-token>

Middleware Stack:
  1. requireAuth         // Validates Clerk session
  2. requireBackofficeOrg // Checks backoffice org membership
  3. requireActiveStaff  // Verifies staff record in DB
  4. requireBackoffice   // Final backoffice access check

For sensitive operations (assign, override, approve):
  5. requireBackofficeApprover // Checks manager/admin role
```

### 7.4 Error Responses

Standard error format:
```json
{
  "success": false,
  "error": "ERROR_CODE",
  "message": "Human-readable message",
  "hint": "Optional hint for resolution",
  "fieldErrors": {  // For validation errors
    "field": "error message"
  }
}
```

HTTP Status Codes:
- `200` OK - Success
- `201` Created - Resource created
- `400` Bad Request - Validation error
- `401` Unauthorized - Not authenticated
- `403` Forbidden - Not authorized (role check failed)
- `404` Not Found - Resource not found
- `409` Conflict - State machine violation, already assigned, etc.
- `422` Unprocessable Entity - Request format invalid
- `500` Internal Server Error - Server error

---

## 8. Frontend Implementation

### 8.1 Page Structure

```
apps/backoffice/app/(authenticated)/(home)/(general)/
├── inbox/
│   ├── page.tsx                          Server component (wrapper)
│   └── _components/
│       ├── inbox-client.tsx              Main client orchestrator
│       ├── inbox-page-header.tsx         Stats + "New Request" button
│       ├── inbox-filter-bar.tsx          Queue tabs + filters
│       ├── inbox-table.tsx               Responsive table/cards
│       └── inbox-pagination.tsx          Page navigation
│
└── cases/
    ├── page.tsx                          Server component (wrapper)
    ├── _components/
    │   ├── cases-client.tsx              Main client orchestrator
    │   ├── cases-page-header.tsx         Stats cards
    │   ├── cases-filter-bar.tsx          Type tabs + filters + search
    │   ├── cases-table.tsx               Responsive table/cards
    │   ├── cases-pagination.tsx          Page navigation
    │   └── status-badge.tsx              Colored status badges
    │
    └── [id]/
        ├── page.tsx                      Server component (wrapper)
        └── _components/
            ├── case-detail-client.tsx    Main orchestrator
            ├── case-header.tsx           Breadcrumb + actions
            ├── readiness-checklist.tsx   Gates with progress bar
            ├── case-timeline.tsx         Audit log viewer
            └── tabs/
                ├── overview-tab.tsx      Org + dates + metadata
                ├── tasks-tab.tsx         Task list with status
                ├── documents-tab.tsx     Document upload + list
                ├── payments-tab.tsx      Payment verification
                └── tickets-tab.tsx       Ticket list + create
```

### 8.2 State Management Strategy

```typescript
// URL STATE (persisted in search params)
// For list pages: filters, sorting, pagination
const searchParams = useSearchParams();
const type = searchParams.get('type') || 'all';
const status = searchParams.get('status');
const page = parseInt(searchParams.get('page') || '1');
// Updated via: router.push(`?type=${type}&page=${page}`)

// SERVER STATE (React Query)
// For API data with caching
const { data, isLoading } = useCases({
  type, status, page, pageSize: 20
});
// Automatic refetching, caching, deduplication

// LOCAL COMPONENT STATE (useState)
// For UI-only state (modals, form inputs, etc.)
const [isDialogOpen, setIsDialogOpen] = useState(false);
const [searchInput, setSearchInput] = useState('');
// Debounced before updating URL state

// FORM STATE (react-hook-form)
// For forms with validation
const form = useForm({
  resolver: zodResolver(schema),
  defaultValues: { ... }
});
```

### 8.3 Data Fetching Pattern

```typescript
// Query Hook (apps/backoffice/lib/queries/cases/index.ts)

export function useCases(params?: ListCasesParams) {
  const { getToken } = useAuth();

  return useQuery({
    queryKey: casesQueryKeys.list(params),
    queryFn: () => fetchCases(getToken, params),
    staleTime: 30_000, // 30 seconds
    retry: 1,
  });
}

// Usage in Component
function CasesClient() {
  const searchParams = useSearchParams();
  const router = useRouter();

  const [filters, setFilters] = useState({
    type: searchParams.get('type') || 'all',
    status: searchParams.get('status'),
    page: parseInt(searchParams.get('page') || '1'),
  });

  // Sync URL with filters
  useEffect(() => {
    const params = new URLSearchParams();
    if (filters.type !== 'all') params.set('type', filters.type);
    if (filters.status) params.set('status', filters.status);
    params.set('page', filters.page.toString());
    router.push(`?${params.toString()}`);
  }, [filters]);

  // Fetch data
  const { data, isLoading, error } = useCases(filters);

  return (
    <div>
      <CasesFilterBar
        filters={filters}
        onFiltersChange={setFilters}
      />
      <CasesTable
        data={data?.data}
        isLoading={isLoading}
      />
    </div>
  );
}
```

### 8.4 Mutation Pattern

```typescript
// Mutation Hook
export function useClaimCase() {
  const { getToken } = useAuth();
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (caseId: string) =>
      claimCaseRequest(getToken, caseId),

    onSuccess: () => {
      // Invalidate lists to refetch
      queryClient.invalidateQueries({
        queryKey: casesQueryKeys.lists()
      });
      queryClient.invalidateQueries({
        queryKey: queueQueryKeys.all
      });
    },
  });
}

// Usage in Component
function CaseActions({ caseId }) {
  const claimCase = useClaimCase();
  const { toast } = useToast();

  const handleClaim = async () => {
    try {
      await claimCase.mutateAsync(caseId);
      toast.success("Case claimed successfully");
    } catch (error) {
      toast.error("Failed to claim case", {
        description: error.message
      });
    }
  };

  return (
    <Button
      onClick={handleClaim}
      disabled={claimCase.isPending}
    >
      {claimCase.isPending ? "Claiming..." : "Claim"}
    </Button>
  );
}
```

### 8.5 UI Patterns

#### Responsive Table/Card Layout
```tsx
// Desktop: Full table
// Mobile: Stacked cards

<div className="hidden sm:block">
  <Table>
    <TableHeader>...</TableHeader>
    <TableBody>
      {cases.map(case => (
        <TableRow key={case.id}>
          <TableCell>{case.organizationName}</TableCell>
          <TableCell><StatusBadge status={case.status} /></TableCell>
          ...
        </TableRow>
      ))}
    </TableBody>
  </Table>
</div>

<div className="sm:hidden space-y-3">
  {cases.map(case => (
    <Card key={case.id}>
      <CardHeader>
        <CardTitle>{case.organizationName}</CardTitle>
        <StatusBadge status={case.status} />
      </CardHeader>
      <CardContent>...</CardContent>
    </Card>
  ))}
</div>
```

#### Loading States
```tsx
{isLoading ? (
  <div className="hidden sm:block">
    <TableSkeleton rows={10} columns={8} />
  </div>
  <div className="sm:hidden">
    <CardSkeleton count={5} />
  </div>
) : (
  <CasesTable data={data} />
)}
```

#### Empty States
```tsx
{data?.data.length === 0 ? (
  <EmptyState
    icon={Inbox}
    title="No cases found"
    description="There are no cases matching your filters."
    action={
      <Button onClick={clearFilters}>Clear Filters</Button>
    }
  />
) : (
  <CasesTable data={data.data} />
)}
```

#### Error States
```tsx
{error ? (
  <Alert variant="destructive">
    <AlertCircle className="h-4 w-4" />
    <AlertTitle>Error loading cases</AlertTitle>
    <AlertDescription>
      {error.message}
      <Button onClick={() => refetch()}>Try Again</Button>
    </AlertDescription>
  </Alert>
) : (
  ...
)}
```

---

## 9. Security & Compliance

### 9.1 Authentication Flow

```
┌─────────────────────────────────────────────────────────────┐
│  AUTHENTICATION FLOW                                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. User visits backoffice.bumara.com                       │
│     └─> Redirected to Clerk sign-in                        │
│                                                             │
│  2. User signs in with email/password or SSO                │
│     └─> Clerk validates credentials                        │
│                                                             │
│  3. Clerk checks organization membership                    │
│     ├─> User must be member of CLERK_INTERNAL_ORG_ID       │
│     └─> User email must match ALLOWED_EMAIL_DOMAINS        │
│                                                             │
│  4. Clerk issues JWT token with org claims                  │
│                                                             │
│  5. Next.js middleware validates token on every request     │
│     ├─> Checks session is active                           │
│     ├─> Checks backoffice org membership                   │
│     └─> Checks email domain                                │
│                                                             │
│  6. API middleware validates token again                    │
│     ├─> Extracts user_id and org_id from JWT               │
│     ├─> Queries back_office_agents table                   │
│     ├─> Checks agent.is_active = true                      │
│     └─> Extracts role (member, manager, admin)             │
│                                                             │
│  7. Request proceeds with context:                          │
│     ├─> c.get('userId')                                     │
│     ├─> c.get('staffAgentId')                              │
│     └─> c.get('staffRole')                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 9.2 Authorization Matrix

| Action | Member | Manager | Admin |
|--------|:------:|:-------:|:-----:|
| **Cases** |
| View cases | ✅ | ✅ | ✅ |
| Claim unassigned case | ✅ | ✅ | ✅ |
| Work on assigned case | ✅ (own) | ✅ (any) | ✅ (any) |
| Reassign case | ❌ | ✅ | ✅ |
| Override readiness gate | ❌ | ✅ | ✅ |
| Cancel case | ❌ | ✅ | ✅ |
| **Tickets** |
| Create ticket | ✅ | ✅ | ✅ |
| Reply to ticket | ✅ | ✅ | ✅ |
| Close ticket | ✅ | ✅ | ✅ |
| **Payments** |
| Verify tenant payment | ✅ | ✅ | ✅ |
| Create regulator payout | ✅ | ✅ | ✅ |
| Approve payout &lt; ZMW 5,000 | Auto | Auto | Auto |
| Approve payout ZMW 5,000-50,000 | ❌ | ✅ | ✅ |
| Approve payout > ZMW 50,000 | ❌ | ✅ | ✅ |
| **Documents** |
| View documents | ✅ | ✅ | ✅ |
| Upload evidence | ✅ | ✅ | ✅ |
| Delete document | ❌ | ✅ | ✅ |
| **Admin** |
| View staff list | ❌ | ✅ | ✅ |
| Invite staff | ❌ | ❌ | ✅ |
| Change staff role | ❌ | ❌ | ✅ |
| Deactivate staff | ❌ | ❌ | ✅ |
| Configure system | ❌ | ❌ | ✅ |

### 9.3 Data Isolation

**Multi-Tenancy Model:**
- Backoffice can view ALL organizations
- Each organization's data is isolated from other orgs
- Queries always filter by `organizationId` where applicable
- Audit logs track which org's data was accessed

**Key Principles:**
```sql
-- ✅ CORRECT: Always filter by organizationId
SELECT * FROM filings
WHERE organization_id = $orgId
  AND status = 'queued';

-- ❌ WRONG: Never query without org filter
SELECT * FROM filings
WHERE status = 'queued';
```

### 9.4 Audit Logging

Every mutation creates an audit event:

```typescript
interface AuditEvent {
  id: string;
  organizationId: string;      // Tenant org (not backoffice)
  actorId: string;             // Clerk user ID
  actorType: 'STAFF' | 'TENANT' | 'SYSTEM';
  resourceType: string;        // 'filing', 'submission_job', etc.
  resourceId: string;
  action: string;              // 'status_changed', 'claimed', etc.
  beforeState: JSON;           // Previous state
  afterState: JSON;            // New state
  reason?: string;             // For overrides
  metadata?: JSON;
  occurredAt: timestamp;
}
```

**Audit Actions:**
- `create`, `update`, `delete`
- `status_changed`, `assigned`, `claimed`
- `payment_verified`, `payout_approved`
- `evidence_uploaded`, `ticket_created`
- `gate_overridden`, `submit`, `approve`, `reject`

### 9.5 Sensitive Operations

**Payment Approval (Dual Control):**
- Payout creator cannot approve own payout
- Approval thresholds enforced server-side
- Approval history logged in audit

**Gate Override:**
- Requires manager/admin role
- Requires reason (min 10 characters)
- Blockers logged before override
- Override logged in audit with full context

**Case Cancellation:**
- Requires manager+ role
- Requires reason
- Cannot cancel if already submitted
- Logged in audit

---

## 10. Integration Points

### 10.1 Clerk (Authentication)

```
Purpose: Manage backoffice staff authentication and organization membership

Integration:
  - Frontend: @clerk/nextjs (middleware + components)
  - Backend: @clerk/backend (token verification)

Key Features:
  - Organization-based auth (CLERK_INTERNAL_ORG_ID)
  - Role assignment (member, manager, admin)
  - Email domain restriction (ALLOWED_EMAIL_DOMAINS)
  - MFA support (can be enforced)

Webhook Events (if needed):
  - user.created → create back_office_agents record
  - organizationMembership.deleted → deactivate agent
```

### 10.2 AWS S3 (Document Storage)

```
Purpose: Store all documents (source, evidence, proof)

Integration:
  - Backend generates presigned URLs for upload
  - Frontend uploads directly to S3 (no file through backend)
  - DB stores only metadata + S3 key

Upload Flow:
  1. Frontend requests presigned URL from backend
  2. Backend generates URL with expiry (15 minutes)
  3. Frontend uploads file directly to S3
  4. Frontend confirms upload to backend
  5. Backend creates document record in DB

Security:
  - Presigned URLs expire after 15 minutes
  - Files stored with UUID-based keys (not original filenames)
  - Access controlled via presigned download URLs
```

### 10.3 Regulators (Manual Portals)

```
Supported Regulators:
  - ZRA (tax.gov.zm) - VAT, PAYE, CIT returns
  - PACRA (pacra.org.zm) - Business registration, name search
  - NAPSA (napsa.co.zm) - Social security contributions
  - NHIMA (nhima.co.zm) - Healthcare contributions
  - WCFCB - Workers compensation
  - LCC - Liquor licenses

Integration Type: MANUAL
  - No API access
  - Staff logs into portal with credentials
  - Manual data entry from tenant's prepared data
  - Screenshot/PDF download as proof
  - Status checked manually

Future: API integration when regulators provide it
```

### 10.4 Notifications (Future)

```
Channels (priority order):
  1. In-app notification (immediate)
  2. Email (asynchronous)
  3. SMS (critical only)
  4. WhatsApp (if enabled)

Events:
  - Case status changed (submitted, accepted)
  - Ticket created/responded
  - Payment verified
  - Evidence uploaded
  - SLA at risk
```

---

## 11. Operational Considerations

### 11.1 SLA Management

```
SLA Timers:
  - Time-to-first-response (ticket): 4 hours
  - Time-to-submission: Based on due date
  - Time-to-resolution (ticket): 24 hours

SLA Status:
  - OK: > 24 hours remaining
  - AT_RISK: 1-24 hours remaining
  - BREACHED: Past deadline

Actions on Breach:
  - Visual indicator (red dot)
  - Manager notification
  - Escalation queue
  - Logged in audit trail
```

### 11.2 Capacity Planning

```
Staff Capacity:
  - Each analyst: ~15-20 cases/day
  - Manager: oversees 5-10 analysts
  - Workload dashboard shows:
    - Cases per agent
    - Completion rates
    - Average time per case type
    - Overdue cases by agent

Load Balancing:
  - Unassigned cases in priority order
  - Managers can reassign if imbalanced
  - Automatic priority escalation if past due
```

### 11.3 Performance Targets

```
Database:
  - List queries: < 500ms
  - Detail queries: < 300ms
  - Mutations: < 200ms

Frontend:
  - List page load: < 2s
  - Detail page load: < 1.5s
  - Action response: < 500ms

Indexes Required:
  - submission_jobs (status, assigned_to, due_on)
  - filings (organization_id, status, due_on)
  - timeline_events (organization_id, filing_id, created_at)
```

### 11.4 Monitoring & Alerts

```
Key Metrics:
  - Cases in queue (by status)
  - Average time-to-claim
  - Average time-to-submission
  - SLA breach rate
  - Error rate (by endpoint)
  - API latency (p50, p95, p99)

Alerts:
  - Queue growing faster than processing
  - SLA breach rate > 5%
  - API error rate > 1%
  - Database connection pool exhausted
```

---

## Appendix A: Database Schema (Key Tables)

### submission_jobs
```sql
CREATE TABLE submission_jobs (
  id UUID PRIMARY KEY,
  organization_id UUID NOT NULL REFERENCES organizations(id),
  filing_id UUID REFERENCES filings(id),
  service_request_id UUID REFERENCES service_requests(id),
  regulator_key TEXT NOT NULL,
  status TEXT NOT NULL, -- queued, assigned, in_progress, etc.
  priority TEXT NOT NULL DEFAULT 'normal',
  assigned_to_agent_id UUID REFERENCES back_office_agents(id),
  assigned_at TIMESTAMPTZ,
  requested_at TIMESTAMPTZ,
  submitted_at TIMESTAMPTZ,
  closed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  CONSTRAINT filing_or_service_req CHECK (
    (filing_id IS NOT NULL AND service_request_id IS NULL) OR
    (filing_id IS NULL AND service_request_id IS NOT NULL)
  )
);

CREATE INDEX idx_jobs_status_assigned
  ON submission_jobs(status, assigned_to_agent_id);
CREATE INDEX idx_jobs_org
  ON submission_jobs(organization_id);
```

### timeline_events
```sql
CREATE TABLE timeline_events (
  id UUID PRIMARY KEY,
  organization_id UUID NOT NULL REFERENCES organizations(id),
  filing_id UUID REFERENCES filings(id),
  service_request_id UUID REFERENCES service_requests(id),
  event_type TEXT NOT NULL,
  title TEXT NOT NULL,
  description TEXT,
  actor_id TEXT, -- Clerk user ID
  actor_type TEXT NOT NULL, -- 'agent', 'tenant', 'system'
  metadata JSONB,
  occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_timeline_filing
  ON timeline_events(filing_id, occurred_at DESC);
CREATE INDEX idx_timeline_org
  ON timeline_events(organization_id, occurred_at DESC);
```

---

## Appendix B: State Machine Validation

```typescript
// State transition validation logic

const SUBMISSION_JOB_TRANSITIONS: Record<string, string[]> = {
  queued: ['assigned', 'cancelled'],
  assigned: ['in_progress', 'waiting_on_client', 'waiting_on_payment', 'cancelled'],
  in_progress: ['waiting_on_client', 'waiting_on_payment', 'ready_to_submit', 'cancelled'],
  waiting_on_client: ['in_progress', 'cancelled'],
  waiting_on_payment: ['in_progress', 'cancelled'],
  ready_to_submit: ['submission_in_progress', 'cancelled'],
  submission_in_progress: ['submitted', 'waiting_on_client', 'cancelled'],
  submitted: ['accepted', 'needs_correction', 'rejected'],
  needs_correction: ['in_progress'],
  accepted: ['closed'],
  rejected: [], // terminal
  closed: [], // terminal
  cancelled: [], // terminal
};

function validateTransition(
  currentStatus: string,
  targetStatus: string,
  hasOverridePermission: boolean,
  reason?: string
): { valid: boolean; error?: string } {

  const allowedTransitions = SUBMISSION_JOB_TRANSITIONS[currentStatus];

  if (!allowedTransitions) {
    return { valid: false, error: 'Invalid current status' };
  }

  if (allowedTransitions.includes(targetStatus)) {
    return { valid: true };
  }

  // Check if manager can override
  if (hasOverridePermission && reason && reason.length >= 10) {
    return { valid: true };
  }

  return {
    valid: false,
    error: `Cannot transition from ${currentStatus} to ${targetStatus}`
  };
}
```

---

## Appendix C: Quick Reference

### Key Files
```
Backend:
  packages/backend/src/modules/compliance/cases/
    ├── routes.ts       (OpenAPI route definitions)
    ├── handlers.ts     (Request handlers)
    └── index.ts        (Hono app registration)

Frontend:
  apps/backoffice/app/(authenticated)/(home)/(general)/cases/
    ├── page.tsx                (Server wrapper)
    ├── [id]/page.tsx           (Detail page)
    └── _components/            (Client components)

  apps/backoffice/lib/queries/cases/
    └── index.ts                (React Query hooks)

State Machine:
  packages/api-services/src/domains/compliance/
    ├── state-machine.ts        (Transition validation)
    └── gates.ts                (Readiness gates checker)

Database:
  packages/database/src/schema/compliance/
    ├── submission-jobs.ts
    ├── filings.ts
    ├── service-requests.ts
    └── timeline-events.ts
```

### Common Commands
```bash
# Run backoffice dev server
cd apps/backoffice && pnpm dev

# Run backend API
cd packages/backend && pnpm dev

# Database migrations
pnpm db:migrate

# Type checking
pnpm typecheck

# Linting
pnpm lint
```

---

**End of Document**

*For questions or updates, contact the engineering team or update this document directly.*
