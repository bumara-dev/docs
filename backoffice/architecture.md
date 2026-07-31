---
title: "Bumara Backoffice Architecture"
description: "System design, data model, and integration patterns for the internal operations platform."
---

## Table of Contents

1. [Monorepo Integration](#1-monorepo-integration)
2. [Authentication Model](#2-authentication-model)
3. [Backend Boundary](#3-backend-boundary)
4. [Data Model](#4-data-model)
5. [Audit and Eventing](#5-audit-and-eventing)
6. [API Contract](#6-api-contract)

---

## 1. Monorepo Integration

### 1.1 Repository Structure

```
bumara/
├── apps/
│   ├── app/              # Tenant-facing Next.js app
│   ├── backoffice/       # Internal ops Next.js app ◄── THIS
│   ├── api/              # API deployment wrapper
│   └── ...
│
├── packages/
│   ├── backend/          # Hono RPC server (shared)
│   │   └── src/
│   │       ├── core/
│   │       │   └── middleware/
│   │       │       └── auth.ts     # Auth + RBAC middleware
│   │       └── modules/
│   │           ├── backoffice/     # Backoffice-specific endpoints
│   │           ├── tasks/          # Compliance tasks
│   │           └── ...
│   │
│   ├── api-services/     # Business logic layer
│   │   └── src/domains/
│   │       ├── backoffice/        # Staff management service
│   │       ├── tasks/             # Tasks service
│   │       └── compliance/        # Filings, cases, etc.
│   │
│   ├── database/         # Drizzle schema + repositories
│   │   └── src/
│   │       ├── schema/
│   │       │   ├── core/          # Organizations, staff
│   │       │   ├── compliance/    # Filings, tasks, payments
│   │       │   └── system/        # Audit logs
│   │       └── repositories/
│   │
│   ├── auth/             # Clerk utilities
│   ├── design-system/    # ShadCN components
│   └── ...
│
└── docs/
    └── backoffice/       # This documentation
```

### 1.2 Backoffice App Structure

```
apps/backoffice/
├── app/
│   ├── (authenticated)/
│   │   ├── layout.tsx              # Auth guards stack
│   │   └── (home)/
│   │       ├── layout.tsx          # Sidebar + topbar
│   │       ├── page.tsx            # Redirects to /inbox
│   │       └── (general)/
│   │           ├── inbox/page.tsx
│   │           ├── cases/page.tsx
│   │           ├── cases/[id]/page.tsx
│   │           ├── tickets/page.tsx
│   │           ├── payments/page.tsx
│   │           ├── documents/page.tsx
│   │           ├── orgs/page.tsx
│   │           ├── orgs/[id]/page.tsx
│   │           ├── admin/page.tsx
│   │           ├── catalog/page.tsx
│   │           └── reports/page.tsx
│   │
│   └── (unauthenticated)/
│       ├── sign-in/page.tsx
│       └── access-denied/page.tsx
│
├── components/
│   └── layout/
│       └── sidebar/
│           └── app-sidebar.tsx
│
├── config/
│   └── nav.ts                      # Navigation config
│
├── data/
│   └── access.ts                   # Role resolution
│
├── lib/
│   ├── guards/
│   │   └── require-role.ts         # Route guards
│   └── api/
│       └── client.ts               # RPC client
│
└── docs/
    ├── ARCHITECTURE.md             # App-level overview
    └── AUTH.md                     # Security details
```

### 1.3 Package Dependencies

```
apps/backoffice
    │
    ├── @repo/design-system        # UI components
    ├── @repo/auth                 # Clerk utilities
    ├── @repo/api-client           # RPC client (hono-client)
    └── @repo/database             # Types only (no direct DB)

packages/backend
    │
    ├── @repo/database             # Schema + repositories
    ├── @repo/api-services         # Business logic
    └── @repo/auth                 # Server-side auth
```

---

## 2. Authentication Model

### 2.1 Clerk Organization Model

Bumara uses Clerk's organization feature with a dedicated **backoffice organization**:

```
Clerk Organizations
│
├── Tenant Orgs (org_xxx, org_yyy, ...)
│   └── Members: Tenant users
│   └── Roles: org:admin, org:manager, org:member
│
└── Backoffice Org (CLERK_INTERNAL_ORG_ID)
    └── Members: Bumara staff
    └── Roles: org:backoffice_admin, org:backoffice_manager, org:backoffice_member
```

### 2.2 Staff Access Determination

A user is considered backoffice staff if:

1. They are authenticated via Clerk
2. They are a member of the backoffice organization (`CLERK_INTERNAL_ORG_ID`)
3. Their email domain matches allowed domains (`ALLOWED_EMAIL_DOMAINS`)
4. They have an active `back_office_agents` record in the database

### 2.3 Role Mapping

| Clerk Org Role | Staff Role | Level |
|----------------|------------|-------|
| `org:admin` | `admin` | 3 |
| `org:backoffice_admin` | `admin` | 3 |
| `org:manager` | `manager` | 2 |
| `org:backoffice_manager` | `manager` | 2 |
| `org:member` | `member` | 1 |
| `org:backoffice_member` | `member` | 1 |

### 2.4 Defense-in-Depth Layers

```
┌─────────────────────────────────────────────────────────────────┐
│ Layer 1: Next.js Middleware (apps/backoffice/middleware.ts)     │
│ ─────────────────────────────────────────────────────────────── │
│ • Validates Clerk session                                       │
│ • Checks backoffice org membership                              │
│ • Validates company email domain                                │
│ • Redirects to /sign-in or /access-denied                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Layer 2: Server Component Guards (authenticated layout)         │
│ ─────────────────────────────────────────────────────────────── │
│ • AuthGuard - Ensures user is authenticated                     │
│ • ServerOrganizationGuard - Ensures org context                 │
│ • BackofficeOrgGuard - Ensures correct org                      │
│ • MfaGuard - Ensures MFA enabled (when required)                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Layer 3: Route Guards (in page components)                      │
│ ─────────────────────────────────────────────────────────────── │
│ • requireAdmin() - Admin-only pages                             │
│ • requireManagerOrAbove() - Manager+ pages                      │
│ • Redirects to /access-denied if insufficient role              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Layer 4: API Middleware (packages/backend)                      │
│ ─────────────────────────────────────────────────────────────── │
│ • requireBackofficeOrg() - Validates caller org                 │
│ • requireActiveStaff() - Validates staff record                 │
│ • requireRole(['admin']) - Role-based access                    │
│ • All return proper HTTP errors                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.5 Key Implementation Files

| Purpose | File |
|---------|------|
| Next.js middleware | `apps/backoffice/middleware.ts` |
| Auth guards | `apps/backoffice/components/guards/` |
| Role resolution | `apps/backoffice/data/access.ts` |
| Route guards | `apps/backoffice/lib/guards/require-role.ts` |
| API auth middleware | `packages/backend/src/core/middleware/auth.ts` |
| Backoffice service | `packages/api-services/src/domains/backoffice/service.ts` |

---

## 3. Backend Boundary

### 3.1 Endpoint Separation

Backoffice and tenant apps share the backend but have separate endpoint groups:

```
packages/backend/src/modules/
│
├── backoffice/           # Backoffice-only endpoints
│   └── routes.ts         # Staff operations
│
├── tasks/                # Shared (with role checks)
│   └── routes.ts         # Task management
│
├── organizations/        # Tenant-focused
│   └── routes.ts         # Org management
│
└── ...
```

### 3.2 Backoffice vs Tenant Endpoints

| Pattern | Audience | Auth Check |
|---------|----------|------------|
| `/api/backoffice/*` | Backoffice only | `requireBackofficeOrg()` |
| `/api/tasks/*` | Both (role-aware) | `requireAuth()` + role check |
| `/api/orgs/*` | Tenant primary | `requireOrg()` |

### 3.3 API Middleware Stack

```typescript
// Backoffice-only endpoint
app.use('/backoffice/*', 
  clerkAuth(),
  attachAuthToContext(),
  requireAuth(),
  requireBackofficeOrg(),   // ◄── Blocks non-backoffice
  requireActiveStaff()       // ◄── Blocks inactive staff
);

// Role-restricted endpoint
app.post('/backoffice/staff/:id/role',
  requireRole(['admin'])     // ◄── Admin only
);
```

### 3.4 RPC Client Usage

Backoffice pages use the shared RPC client:

```typescript
// apps/backoffice/lib/api/client.ts
import { createClient } from '@repo/api-client';

export const api = createClient({
  baseUrl: process.env.NEXT_PUBLIC_API_URL,
  // Auth token attached automatically
});
```

---

## 4. Data Model

### 4.1 Core Tables Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CORE ENTITIES                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────┐         ┌──────────────────┐                 │
│  │  organizations   │◄────────│ back_office_agents│                 │
│  │  ──────────────  │         │ ────────────────  │                 │
│  │  id              │         │ id                │                 │
│  │  name            │         │ clerk_user_id     │                 │
│  │  tpin            │         │ role              │                 │
│  │  plan_id         │         │ department        │                 │
│  └────────┬─────────┘         │ is_active         │                 │
│           │                   └──────────────────┘                 │
│           │                                                         │
│           ▼                                                         │
│  ┌──────────────────┐                                              │
│  │    regulators    │                                              │
│  │  ──────────────  │                                              │
│  │  id              │                                              │
│  │  name (ZRA, etc) │                                              │
│  └──────────────────┘                                              │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                      COMPLIANCE ENTITIES                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────┐         ┌──────────────────┐                 │
│  │   obligations    │────────►│     filings      │                 │
│  │  ──────────────  │         │  ──────────────  │                 │
│  │  id              │         │  id              │                 │
│  │  organization_id │         │  organization_id │                 │
│  │  regulator_id    │         │  obligation_id   │                 │
│  │  template_id     │         │  regulator_id    │                 │
│  │  frequency       │         │  status          │◄───── State     │
│  └──────────────────┘         │  period_start    │       Machine   │
│                               │  period_end      │                 │
│                               │  due_on          │                 │
│  ┌──────────────────┐         │  sla_deadline    │                 │
│  │ service_requests │         └────────┬─────────┘                 │
│  │  ──────────────  │                  │                           │
│  │  id              │                  │                           │
│  │  organization_id │                  │                           │
│  │  regulator_id    │                  │                           │
│  │  template_id     │                  ▼                           │
│  │  status          │◄───┐    ┌──────────────────┐                 │
│  └──────────────────┘    │    │ submission_jobs  │                 │
│                          │    │  ──────────────  │                 │
│                          │    │  id              │                 │
│                          ├────│  filing_id       │                 │
│                          │    │  service_req_id  │                 │
│                          │    │  status          │◄───── State     │
│                          │    │  assigned_to_id  │       Machine   │
│                          │    │  priority        │                 │
│                          │    └──────────────────┘                 │
│                          │                                         │
│  ┌──────────────────┐    │    ┌──────────────────┐                 │
│  │      tasks       │    │    │     tickets      │                 │
│  │  ──────────────  │    │    │  ──────────────  │                 │
│  │  id              │    │    │  id              │                 │
│  │  filing_id       │────┤    │  filing_id       │─────────────────┤
│  │  service_req_id  │────┘    │  service_req_id  │─────────────────┘
│  │  status          │         │  status          │                 │
│  │  task_type       │         │  type            │                 │
│  │  required        │         │  subject         │                 │
│  └──────────────────┘         └──────────────────┘                 │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                       FINANCIAL ENTITIES                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────┐         ┌──────────────────┐                 │
│  │ payment_requests │         │ regulator_payouts│                 │
│  │  ──────────────  │         │  ──────────────  │                 │
│  │  id              │         │  id              │                 │
│  │  organization_id │         │  organization_id │                 │
│  │  filing_id       │         │  filing_id       │                 │
│  │  service_req_id  │         │  service_req_id  │                 │
│  │  status          │◄─────   │  status          │◄───── State     │
│  │  total_amount    │  State  │  amount          │       Machine   │
│  │  verified_at     │  Machine│  paid_by_id      │                 │
│  │  verified_by_id  │         │  verified_by_id  │                 │
│  └──────────────────┘         │  evidence_doc_id │                 │
│                               └──────────────────┘                 │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                      DOCUMENT & AUDIT                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────┐         ┌──────────────────┐                 │
│  │    documents     │         │    audit_logs    │                 │
│  │  ──────────────  │         │  ──────────────  │                 │
│  │  id              │         │  id              │                 │
│  │  organization_id │         │  organization_id │                 │
│  │  filing_id       │         │  actor_id        │                 │
│  │  service_req_id  │         │  actor_type      │ (STAFF/TENANT)  │
│  │  kind            │         │  resource_type   │                 │
│  │  storage_key     │         │  resource_id     │                 │
│  │  filename        │         │  action          │                 │
│  │  mime_type       │         │  before_state    │ (JSON)          │
│  └──────────────────┘         │  after_state     │ (JSON)          │
│                               │  reason          │                 │
│                               │  created_at      │                 │
│                               └──────────────────┘                 │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 Key Status Enums

```typescript
// packages/database/src/schema/enums.ts

// Filing / Service Request statuses
export const filingStatusEnum = pgEnum('filing_status', [
  'DRAFT',
  'PENDING_DATA',
  'IN_PROGRESS',
  'READY_FOR_SUBMISSION',
  'SUBMISSION_IN_PROGRESS',
  'SUBMITTED',
  'NEEDS_CORRECTION',
  'ACCEPTED',
  'REJECTED',
  'WAIVED',
  'CANCELLED'
]);

// Submission Job statuses
export const submissionJobStatusEnum = pgEnum('submission_job_status', [
  'QUEUED',
  'ASSIGNED',
  'IN_PROGRESS',
  'WAITING_ON_CLIENT',
  'WAITING_ON_PAYMENT',
  'READY_TO_SUBMIT',
  'SUBMITTED',
  'NEEDS_CORRECTION',
  'ACCEPTED',
  'REJECTED',
  'CLOSED',
  'CANCELLED'
]);

// Payment Request statuses (Tenant → Bumara)
export const paymentRequestStatusEnum = pgEnum('payment_request_status', [
  'NOT_REQUIRED',
  'REQUIRED_PENDING',
  'PAID_UNVERIFIED',
  'PAID_VERIFIED',
  'REFUNDED',
  'CANCELLED'
]);

// Regulator Payout statuses (Bumara → Regulator)
export const regulatorPayoutStatusEnum = pgEnum('regulator_payout_status', [
  'NOT_REQUIRED',
  'QUEUED',
  'SENT_UNVERIFIED',
  'PAID_VERIFIED',
  'FAILED'
]);

// Task statuses
export const complianceTaskStatusEnum = pgEnum('compliance_task_status', [
  'TODO',
  'DOING',
  'BLOCKED',
  'DONE',
  'SKIPPED'
]);

// Ticket statuses
export const ticketStatusEnum = pgEnum('ticket_status', [
  'OPEN',
  'AWAITING_TENANT',
  'TENANT_RESPONDED',
  'ESCALATED',
  'RESOLVED',
  'CLOSED'
]);

// Document kinds
export const documentKindEnum = pgEnum('document_kind', [
  'SOURCE',
  'WORKPAPER',
  'SUBMISSION',
  'RECEIPT',
  'CERTIFICATE',
  'PAYMENT_PROOF',
  'PAYOUT_PROOF',
  'SUBMISSION_PROOF',
  'OUTCOME_PROOF'
]);
```

### 4.3 Table Locations

| Table | Schema File |
|-------|-------------|
| `organizations` | `packages/database/src/schema/core/organizations.ts` |
| `back_office_agents` | `packages/database/src/schema/core/back-office-agents.ts` |
| `filings` | `packages/database/src/schema/compliance/filings.ts` |
| `service_requests` | `packages/database/src/schema/compliance/service-requests.ts` |
| `submission_jobs` | `packages/database/src/schema/compliance/submission-jobs.ts` |
| `tasks` | `packages/database/src/schema/compliance/tasks.ts` |
| `tickets` | `packages/database/src/schema/compliance/tickets.ts` |
| `payment_requests` | `packages/database/src/schema/compliance/payment-requests.ts` |
| `regulator_payouts` | `packages/database/src/schema/compliance/regulator-payouts.ts` |
| `documents` | `packages/database/src/schema/compliance/documents.ts` |
| `audit_logs` | `packages/database/src/schema/system/audit-logs.ts` |

### 4.4 Required Indexes

All list queries must use appropriate indexes:

```sql
-- Case listing
CREATE INDEX idx_filings_org_status ON filings(organization_id, status);
CREATE INDEX idx_filings_org_due ON filings(organization_id, due_on);
CREATE INDEX idx_filings_regulator ON filings(regulator_id);

-- Job assignment
CREATE INDEX idx_jobs_status_assigned ON submission_jobs(status, assigned_to_agent_id);
CREATE INDEX idx_jobs_org ON submission_jobs(organization_id);

-- Tickets
CREATE INDEX idx_tickets_status ON tickets(status);
CREATE INDEX idx_tickets_case ON tickets(filing_id, service_request_id);

-- Audit
CREATE INDEX idx_audit_entity ON audit_logs(resource_type, resource_id, created_at);
CREATE INDEX idx_audit_org ON audit_logs(organization_id, created_at);
```

---

## 5. Audit and Eventing

### 5.1 Audit Log Structure

Every mutation creates an audit event:

```typescript
interface AuditEvent {
  id: string;
  organizationId: string;           // Tenant org (not backoffice)
  actorId: string;                  // Clerk user ID
  actorType: 'STAFF' | 'TENANT' | 'SYSTEM';
  resourceType: string;             // 'filing', 'submission_job', etc.
  resourceId: string;
  action: string;                   // 'status_changed', 'assigned', etc.
  beforeState: Record<string, any>; // Previous state (JSON)
  afterState: Record<string, any>;  // New state (JSON)
  reason?: string;                  // Required for overrides
  metadata?: Record<string, any>;   // Additional context
  createdAt: Date;
}
```

### 5.2 Audit Actions

| Action | When |
|--------|------|
| `create` | Entity created |
| `update` | Entity updated |
| `status_changed` | Status transition |
| `assigned` | Case assigned to staff |
| `claimed` | Staff claimed case |
| `payment_verified` | Payment marked verified |
| `payout_approved` | Payout approved by manager |
| `evidence_uploaded` | Document attached as evidence |
| `ticket_created` | Ticket created |
| `gate_overridden` | Readiness gate bypassed |
| `submit` | Submitted to regulator |
| `approve` | Accepted outcome |
| `reject` | Rejected outcome |

### 5.3 Audit Repository

```typescript
// packages/database/src/repositories/audit-logs.ts

export async function createAuditLog(data: {
  organizationId: string;
  actorId: string;
  actorType: 'STAFF' | 'TENANT' | 'SYSTEM';
  resourceType: string;
  resourceId: string;
  action: string;
  beforeState?: Record<string, any>;
  afterState?: Record<string, any>;
  reason?: string;
  metadata?: Record<string, any>;
}) {
  // Insert into audit_logs table
}

export async function getAuditLogsForEntity(
  resourceType: string,
  resourceId: string,
  pagination: { limit: number; offset: number }
) {
  // Query with pagination
}
```

### 5.4 Timeline Component

The case detail page shows audit events in a timeline:

```
Timeline
────────────────────────────────────────────────────────
◯ Today, 2:30 PM
│ Payment verified by jane@bumara.com
│ Evidence: payment_receipt.pdf
│
◯ Today, 11:00 AM
│ Claimed by john@bumara.com
│
◯ Yesterday, 4:15 PM
│ Submission requested by tenant
│ Status: QUEUED
│
◯ Dec 28, 2025
│ Filing created
│ Status: PENDING_DATA
────────────────────────────────────────────────────────
```

---

## 6. API Contract

### 6.1 Backoffice RPC Groups

```typescript
// Proposed endpoint structure

backoffice.queue
  .list()                    // List queue items with filters
  .getCounts()               // Queue counts by status

backoffice.cases
  .list()                    // List all cases
  .get(id)                   // Case detail with gates
  .assign(id, agentId)       // Assign to staff
  .claim(id)                 // Atomic self-assign
  .transition(id, status, reason?)  // Status change
  .overrideGate(id, gate, reason)   // Bypass gate

backoffice.tickets
  .create(caseId, data)      // Create ticket
  .list(filters)             // List tickets
  .resolve(id)               // Mark resolved
  .addNote(id, note)         // Internal note

backoffice.payments
  .listTenantPayments()      // Pending verifications
  .verify(id, evidenceDocId) // Verify with proof
  .listPayouts()             // Regulator payouts
  .createPayout(caseId, data) // Initiate payout
  .approvePayout(id, decision) // Manager approval

backoffice.documents
  .getUploadUrl(data)        // Presigned S3 URL
  .attach(docId, caseId, tag) // Link to case
  .list(filters)             // Search documents

backoffice.orgs
  .search(query)             // Search tenants
  .getProfile(id)            // Org details

backoffice.audit
  .listForCase(caseId)       // Case timeline
  .listGlobal(filters)       // Admin audit view
```

### 6.2 Response Format

All API responses follow consistent structure:

```typescript
// Success
{
  data: T,
  meta?: {
    total: number;
    page: number;
    pageSize: number;
  }
}

// Error
{
  error: {
    code: 'BAD_REQUEST' | 'UNAUTHORIZED' | 'FORBIDDEN' | 'NOT_FOUND' | 'CONFLICT';
    message: string;
    hint?: string;
    fieldErrors?: Record<string, string>;
  }
}
```

### 6.3 Status Codes

| Code | Usage |
|------|-------|
| 200 | Success |
| 201 | Created |
| 400 | Validation error |
| 401 | Not authenticated |
| 403 | Not authorized (role) |
| 404 | Entity not found |
| 409 | Conflict (e.g., already claimed, invalid transition) |
| 500 | Internal error |

---

## Related Documentation

- [Specification](/backoffice/specification) - Full product spec
- Gap Analysis - Current vs target
- [Implementation Plan](/backoffice/implementation-plan) - Build steps

