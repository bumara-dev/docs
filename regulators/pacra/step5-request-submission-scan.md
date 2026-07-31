---
title: "Step 5: Request Submission - Pre-Implementation Scan"
description: "Date: January 8, 2026 Status: Complete Purpose: Document existing entities and gaps before implementing Request Submission feature"
---

## 0.1 Existing Submission-Related Entities

### `submission_jobs` Table

**Location:** `packages/database/src/schema/compliance/submission-jobs.ts`

**Status:** EXISTS - needs enhancement

**Current Fields:**
- `id` (uuid, PK)
- `organizationId` (text, FK to organizations)
- `filingId` (uuid, nullable, FK to filings)
- `serviceRequestId` (uuid, nullable, FK to serviceRequests)
- `status` (submission_job_status enum)
- `priority` (task_priority enum, default: normal)
- `estimatedHours` (numeric)
- `actualHours` (numeric)
- `assignedToAgentId` (uuid, nullable, FK to backOfficeAgents)
- `assignedAt` (timestamp)
- `submittedAt` (timestamp)
- `createdAt`, `updatedAt`

**Missing Fields (per spec):**
- `requestedByUserId` - who requested submission
- `requestedAt` - when submission was requested
- `regulatorKey` - which regulator (e.g., "PACRA")
- `templateId` - template reference for runbooks
- `metadata` - JSON for additional context

**Indexes:**
- `idx_submission_jobs_org_status` (organizationId, status)
- `idx_submission_jobs_created` (createdAt)
- `idx_submission_jobs_assigned_staff` (assignedToAgentId, priority)

### Status Enums

**Location:** `packages/database/src/schema/enums.ts`

| Enum | Values |
|------|--------|
| `submissionJobStatusEnum` | `queued`, `assigned`, `in_progress`, `submitted`, `accepted`, `rejected`, `closed` |
| `filingStatusEnum` | `pending_data`, `in_progress`, `ready_for_submission`, `submission_in_progress`, `submitted`, `accepted`, `needs_correction`, `waived`, `cancelled` |
| `serviceRequestStatusEnum` | Same as filing status |
| `paymentRequestStatusEnum` | `required_pending`, `pending_gateway`, `paid_platform_unverified`, `paid_platform_verified`, `refunded`, `cancelled` |

### Existing Endpoints

**No "request submission" endpoint exists yet.** The submission_jobs table has no corresponding service or handlers in:
- `packages/backend/src/modules/` - no submissions module
- `packages/api-services/src/domains/` - no submissions domain

---

## 0.2 Existing Readiness/Gates Logic

### Task Readiness

**Location:** `packages/api-services/src/domains/tasks/tasks.service.ts`

**Function:** `checkFilingReadiness(ctx, deps, filingId)`

Returns:
```typescript
{
  isReady: boolean,
  totalTasks: number,
  completedTasks: number,
  totalRequired: number,
  completedRequired: number,
  blockedTasks: { id, title, status }[],
  pendingRequired: { id, title, status }[]
}
```

**Logic:**
- Ready if `pendingRequired.length === 0 && blockedTasks.length === 0`
- Required tasks must be status `done`
- Optional tasks can be `done` or `skipped`

### Filing View with Blockers

**Location:** `packages/api-services/src/domains/compliance/filings.service.ts`

**Function:** `getFilingView(ctx, deps, filingId)`

Returns comprehensive view including `blockers` object:
```typescript
blockers: {
  isReady: boolean,
  blockedTasks: { id, title, reason }[],
  pendingRequiredTasks: { id, title, status }[],
  missingRequiredDocs: { key, name }[]
}
```

### Service Request View with Blockers

**Location:** `packages/api-services/src/domains/compliance/service-requests.service.ts`

**Function:** `getServiceRequestView(ctx, deps, requestId)`

Returns same blockers structure as filing view.

### Document Requirements

**Representation:**
- Templates store `docRequirementConfigs` as JSON array
- Each requirement has: `key`, `name`, `description`, `kind`, `required`, `conditions`
- Documents table has `requirementKey` to link uploaded docs to requirements
- Satisfaction: requirement is satisfied when a document exists with matching `requirementKey`

**Location:** Document requirements are in `obligationTemplates.docRequirementConfigs` and `serviceTemplates.docRequirementConfigs`

### Payments

**Location:** `packages/database/src/schema/compliance/payment-requests.ts`

**Table:** `payment_requests`

**Key Fields:**
- `filingId` / `serviceRequestId` (nullable)
- `status` (payment_request_status enum)
- `regulatorFee`, `serviceFee`, `totalAmount` (integers in minor units)
- `verifiedAt`, `verifiedByAgentId`

**Verified Status:** `paid_platform_verified` indicates payment is confirmed

**Gap:** Payment status NOT currently integrated into readiness checks.

---

## 0.3 Existing Backoffice Queue Endpoints/UI

### Inbox Page

**Location:** `apps/backoffice/app/(authenticated)/(home)/(general)/inbox/page.tsx`

**Status:** Static/mocked UI

- Shows hardcoded stats (Assigned: 12, Due Today: 4, etc.)
- Displays mock work queue items
- "Unassigned" tab shows empty state
- No actual API calls to backend

### Cases Page

**Location:** `apps/backoffice/app/(authenticated)/(home)/(general)/cases/page.tsx`

**Status:** Exists but uses client component wrapper

- Supports query params: `type`, `status`, `search`
- Type can be: `filing`, `service_request`, `submission_job`, `all`
- Client component at `_components/cases-client.tsx` (not found - may need creation)

### Submission Jobs Redirect

**Location:** `apps/backoffice/app/(authenticated)/(home)/(general)/submission-jobs/page.tsx`

Redirects to `/cases?type=submission_job`

### Backend Queue Endpoints

**Status:** DO NOT EXIST

No handlers or routes for:
- Listing submission jobs
- Claiming/assigning jobs
- Updating job status

---

## 0.4 Existing Audit/Activity System

### Audit Log Service

**Location:** `packages/api-services/src/domains/audit/audit-log.service.ts`

**Function:** `recordAuditLog(ctx, deps, payload)`

**Payload Structure:**
```typescript
{
  action: AuditAction, // create, update, submit, approve, etc.
  entityType: AuditEntityType, // filing, service_request, task, etc.
  entityId?: string,
  changes?: { before?: Record, after?: Record },
  metadata?: Record<string, unknown>,
  actorType?: "STAFF" | "TENANT" | "SYSTEM"
}
```

**Note:** `submission_job` is NOT in `AuditEntityType` yet - needs to be added.

### Timeline Events Table

**Location:** `packages/database/src/schema/compliance/timeline-events.ts`

**Fields:**
- `id`, `organizationId`
- `filingId`, `serviceRequestId` (nullable)
- `eventType` (timeline_event_type enum)
- `title`, `description`
- `actorId`, `actorType`
- `metadata`, `occurredAt`

### Timeline Service

**Location:** `packages/api-services/src/domains/compliance/timeline.service.ts`

**Function:** `listTimelineEvents(ctx, deps, params)`

- Queries `audit_logs` table
- Filters by entityType, entityId, regulatorId, filingId, serviceRequestId
- Generates human-readable descriptions
- Supports pagination

**Note:** Already supports filtering by `filingId` and `serviceRequestId` via metadata lookup.

---

## Summary: Gaps & Recommended Alignments

### Database Schema Gaps

| Gap | Recommendation |
|-----|----------------|
| Missing `submission_gate_snapshots` table | Create new table to store immutable readiness snapshot |
| Missing fields on `submission_jobs` | Add `requestedByUserId`, `requestedAt`, `regulatorKey`, `templateId`, `metadata` |
| No unique constraint for idempotent jobs | Add partial unique index on (org, filing/serviceRequest) where status is open |

### Service Layer Gaps

| Gap | Recommendation |
|-----|----------------|
| No unified readiness check | Create `computeSubmissionReadiness()` combining tasks + docs + payments |
| No submission request endpoint | Create `tenant.submissions.request()` |
| No submission query endpoint | Create `tenant.submissions.getForSource()` |
| `submission_job` not in audit types | Add to `AuditEntityType` union |

### Backoffice Gaps

| Gap | Recommendation |
|-----|----------------|
| No queue list endpoint | Create `backoffice.queue.list()` |
| No claim endpoint | Create `backoffice.queue.claim()` |
| Inbox page is static | Wire to real API endpoints |

### UI Gaps

| Gap | Recommendation |
|-----|----------------|
| Request Submission button not wired | Create mutation hook, wire button |
| No submission status display | Show chip when job exists |
| No blockers display on request failure | Render blockers from API response |

---

## File Paths Referenced

### Existing (to modify)
- `packages/database/src/schema/compliance/submission-jobs.ts`
- `packages/database/src/schema/enums.ts`
- `packages/api-services/src/domains/audit/audit-log.service.ts`
- `apps/backoffice/app/(authenticated)/(home)/(general)/inbox/page.tsx`
- `apps/app/app/(authenticated)/(general)/regulators/pacra/filings/[filingId]/page.tsx`
- `apps/app/app/(authenticated)/(general)/regulators/pacra/service-requests/[id]/page.tsx`

### To Create
- `packages/database/src/schema/compliance/submission-gate-snapshots.ts`
- `packages/database/drizzle/XXXX_submission_request_schema.sql`
- `packages/api-services/src/domains/submissions/submissions.service.ts`
- `packages/api-services/src/domains/submissions/submissions.schema.ts`
- `packages/backend/src/modules/submissions/` (routes, handlers, index)
- `apps/app/lib/queries/submissions/` (hooks, fetchers)
- `apps/app/features/submissions/components/request-submission-panel.tsx`
