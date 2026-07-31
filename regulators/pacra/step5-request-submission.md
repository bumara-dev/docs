---
title: "Step 5: Request Submission Implementation"
description: "Date: January 8, 2026 Status: Complete Module: Submission Request Flow"
---

## Overview

This document describes the **Request Submission** feature that allows tenants to request Bumara to submit their filings and service requests to regulators. The implementation is designed to be **reusable across all regulators** (PACRA, ZRA, NAPSA, etc.).

### Key Components

1. **Submission Jobs** - Tracks the lifecycle of a submission request
2. **Gate Snapshots** - Immutable record of readiness state at request time
3. **Readiness Gates** - Server-side validation of submission prerequisites
4. **Tenant UI** - Request submission button with blocker display
5. **Backoffice Queue** - Staff can view and claim submission jobs

---

## Gate Rules (What Blocks Submission)

### Required Gates (MVP)

1. **Tasks Gate**
   - All **required** tasks must have status `done`
   - No tasks can be in status `blocked`
   - Optional tasks can be `done` or `skipped`

2. **Documents Gate**
   - All **required** document requirements must be satisfied
   - A requirement is satisfied when a document with matching `requirementKey` exists

3. **Payment Gate**
   - Only applies if a `payment_request` exists for the entity
   - If payment required: status must be `paid_platform_verified`
   - If no payment request exists: gate passes

### Explicitly NOT Blocked (for now)

- **Authorization** - Not checked during submission request
- **Payouts** - Not checked during submission request

---

## API Endpoints

### Tenant Endpoints

#### `POST /submissions/request`

Request submission for a filing or service request.

**Request:**
```json
{
  "sourceType": "filing" | "service_request",
  "sourceId": "uuid"
}
```

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "ok": true,
    "submissionJobId": "uuid",
    "status": "queued",
    "isExisting": false
  }
}
```

**Blockers Response (200):**
```json
{
  "success": true,
  "data": {
    "ok": false,
    "blockers": {
      "missingTasks": [
        { "taskId": "uuid", "title": "Upload financial statement", "status": "todo" }
      ],
      "missingDocs": [
        { "requirementKey": "financial_statement", "name": "Financial Statement" }
      ],
      "payment": {
        "required": true,
        "status": "required_pending",
        "message": "Payment must be verified before submission"
      }
    }
  }
}
```

**Behavior:**
1. Validates source entity exists and belongs to org
2. Checks for existing open job (idempotent - returns existing if found)
3. Computes readiness via server-side gates
4. If not ready: returns blockers
5. If ready: creates job + snapshot, updates source status

#### `GET /submissions/source`

Get the current submission job for a source entity.

**Query Parameters:**
- `sourceType`: `filing` | `service_request`
- `sourceId`: UUID

**Response:**
```json
{
  "success": true,
  "data": {
    "job": {
      "id": "uuid",
      "status": "queued",
      "requestedAt": "2026-01-08T12:00:00Z",
      ...
    }
  }
}
```

### Backoffice Endpoints

#### `GET /submissions/queue`

List submission jobs with filters.

**Query Parameters:**
- `regulatorKey`: Filter by regulator (optional)
- `status`: Filter by status (optional)
- `unassignedOnly`: `true` to show only unassigned jobs
- `limit`, `offset`: Pagination
- `sortBy`: `createdAt` | `requestedAt` | `priority`
- `sortOrder`: `asc` | `desc`

**Response:**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "organizationId": "org_xxx",
      "regulatorKey": "pacra",
      "sourceType": "filing",
      "sourceId": "uuid",
      "status": "queued",
      "priority": "normal",
      "requestedAt": "2026-01-08T12:00:00Z",
      "organization": { "id": "org_xxx", "name": "Acme Corp" },
      "sourceName": "Annual Return 2025"
    }
  ],
  "pagination": { "limit": 20, "offset": 0, "total": 5 }
}
```

#### `POST /submissions/queue/claim`

Claim a submission job for processing.

**Request:**
```json
{
  "submissionJobId": "uuid"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "status": "assigned",
    "assignedToAgentId": "uuid",
    "assignedAt": "2026-01-08T12:05:00Z"
  }
}
```

**Behavior:**
- Atomic operation: only succeeds if job is currently `queued`
- Sets `assignedToAgentId`, `assignedAt`, and status to `assigned`
- Returns CONFLICT error if job is not in `queued` status

---

## Database Schema

### `submission_jobs` Table

```sql
CREATE TABLE submission_jobs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id TEXT NOT NULL REFERENCES organizations(id),
  
  -- Source entity (one must be set)
  filing_id UUID REFERENCES filings(id),
  service_request_id UUID REFERENCES service_requests(id),
  
  -- Context
  regulator_key TEXT,
  template_id UUID,
  
  -- Status and priority
  status submission_job_status NOT NULL DEFAULT 'queued',
  priority task_priority NOT NULL DEFAULT 'normal',
  
  -- Requester
  requested_by_user_id TEXT REFERENCES organization_members(id),
  requested_at TIMESTAMP,
  
  -- Assignment
  assigned_to_agent_id UUID REFERENCES back_office_agents(id),
  assigned_at TIMESTAMP,
  
  -- Completion
  submitted_at TIMESTAMP,
  closed_at TIMESTAMP,
  
  -- Flexible metadata
  metadata JSONB,
  
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_submission_jobs_org_status ON submission_jobs(organization_id, status);
CREATE INDEX idx_submission_jobs_regulator ON submission_jobs(regulator_key, status);
CREATE INDEX idx_submission_jobs_filing ON submission_jobs(filing_id);
CREATE INDEX idx_submission_jobs_service_request ON submission_jobs(service_request_id);

-- Idempotency: unique open job per source
CREATE UNIQUE INDEX idx_submission_jobs_open_filing_unique
  ON submission_jobs(organization_id, filing_id)
  WHERE filing_id IS NOT NULL AND status IN ('queued', 'assigned', 'in_progress');

CREATE UNIQUE INDEX idx_submission_jobs_open_service_request_unique
  ON submission_jobs(organization_id, service_request_id)
  WHERE service_request_id IS NOT NULL AND status IN ('queued', 'assigned', 'in_progress');
```

### `submission_gate_snapshots` Table

```sql
CREATE TABLE submission_gate_snapshots (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  submission_job_id UUID NOT NULL UNIQUE REFERENCES submission_jobs(id) ON DELETE CASCADE,
  organization_id TEXT NOT NULL REFERENCES organizations(id),
  
  -- Context (denormalized for querying)
  regulator_key TEXT NOT NULL,
  source_type TEXT NOT NULL,  -- 'filing' | 'service_request'
  source_id UUID NOT NULL,
  
  -- Schema version for backwards compatibility
  snapshot_version INTEGER NOT NULL DEFAULT 1,
  
  -- Gate snapshots (JSONB)
  tasks_snapshot JSONB NOT NULL,
  documents_snapshot JSONB NOT NULL,
  payment_snapshot JSONB NOT NULL,
  
  -- Result (should always be true when snapshot is created)
  computed_ready BOOLEAN NOT NULL DEFAULT TRUE,
  
  -- Immutable: only created_at
  created_at TIMESTAMP DEFAULT NOW() NOT NULL
);
```

### Snapshot JSON Structures

**tasks_snapshot:**
```json
{
  "total": 5,
  "done": 5,
  "required": 3,
  "requiredDone": 3,
  "missingTaskIds": [],
  "blockedTaskIds": []
}
```

**documents_snapshot:**
```json
{
  "required": 2,
  "satisfied": 2,
  "missingRequirementKeys": []
}
```

**payment_snapshot:**
```json
{
  "required": true,
  "status": "paid_platform_verified",
  "paymentRequestId": "uuid"
}
```

---

## UI Behavior States

### Tenant Filing/Service Request Detail Page

The `<RequestSubmissionPanel>` component shows different states:

| State | Condition | Display |
|-------|-----------|---------|
| **Already Submitted** | Job exists with status `submitted` or `accepted` | Green alert with "Submission Complete" |
| **Submission Requested** | Job exists with status `queued`, `assigned`, `in_progress` | Blue alert with status badge and "Our team is processing..." |
| **Ready for Submission** | `isReady=true` AND `status=ready_for_submission` | Green alert with "Request Bumara Submission" button |
| **Not Ready** | API returns blockers | Orange card listing missing tasks/docs/payment |
| **Hidden** | None of the above | Panel not rendered |

### Backoffice Inbox Page

The inbox shows three tabs:

1. **Unassigned Queue** - Jobs with `status=queued` and no assignment
   - Each job card has a "Claim" button
   - Claiming moves job to "Assigned" tab

2. **Assigned** - Jobs with `status=assigned`
   - Shows who claimed and when
   - "View" button to see details

3. **In Progress** - Jobs with `status=in_progress`
   - Active work in progress

---

## Audit Events

The following events are recorded to the audit log:

| Event | Entity Type | Action | Metadata |
|-------|-------------|--------|----------|
| Submission Requested | `submission_job` | `submit` | `regulatorKey`, `sourceType`, `sourceId`, `templateId` |
| Job Claimed | `submission_job` | `update` | `claimedByAgentId`, status change |

---

## Reusability for Other Regulators

The implementation is designed to be **regulator-agnostic**:

1. **Endpoints** don't hardcode any regulator
2. **Schema** uses `regulator_key` field
3. **UI components** render based on `regulatorKey` from data
4. **Gate logic** uses templates from the source entity

### To add support for a new regulator:

1. Ensure templates have `docRequirementConfigs` defined
2. Ensure tasks are generated with `required` flag set correctly
3. Ensure regulator has a `code` in the `regulators` table
4. Filing/Service Request pages should use `<RequestSubmissionPanel>`

---

## Files Created/Modified

### Created

| File | Description |
|------|-------------|
| `packages/database/src/schema/compliance/submission-gate-snapshots.ts` | Gate snapshot schema |
| `packages/database/drizzle/0025_submission_request_schema.sql` | Migration file |
| `packages/api-services/src/domains/submissions/submissions.schema.ts` | Zod schemas |
| `packages/api-services/src/domains/submissions/submissions.service.ts` | Service functions |
| `packages/api-services/src/domains/submissions/index.ts` | Domain exports |
| `packages/backend/src/modules/submissions/routes.ts` | OpenAPI routes |
| `packages/backend/src/modules/submissions/handlers.ts` | Request handlers |
| `packages/backend/src/modules/submissions/index.ts` | Module registration |
| `apps/app/lib/queries/submissions/fetchers.ts` | Tenant API fetchers |
| `apps/app/lib/queries/submissions/hooks.ts` | Tenant React Query hooks |
| `apps/app/features/submissions/components/request-submission-panel.tsx` | UI component |
| `apps/backoffice/lib/queries/submissions/fetchers.ts` | Backoffice API fetchers |
| `apps/backoffice/lib/queries/submissions/hooks.ts` | Backoffice React Query hooks |
| `apps/backoffice/app/.../inbox/_components/inbox-content.tsx` | Queue UI |
| `docs/regulators/pacra/step5-request-submission-scan.md` | Scan findings |
| `docs/regulators/pacra/step5-request-submission.md` | This document |

### Modified

| File | Change |
|------|--------|
| `packages/database/src/schema/compliance/submission-jobs.ts` | Added new fields |
| `packages/database/src/schema/compliance/compliance-relations.ts` | Added relations |
| `packages/database/src/schema/compliance/index.ts` | Export snapshots |
| `packages/api-services/src/domains/audit/audit-log.service.ts` | Added `submission_job` entity type |
| `packages/api-services/src/index.ts` | Export submissions domain |
| `packages/backend/src/modules/index.ts` | Register submissions router |
| `apps/app/.../pacra/filings/[filingId]/page.tsx` | Use RequestSubmissionPanel |
| `apps/app/.../pacra/service-requests/[id]/page.tsx` | Use RequestSubmissionPanel |
| `apps/backoffice/.../inbox/page.tsx` | Use InboxContent component |

---

## Manual QA Steps

1. **Connect PACRA** (if not already connected)
2. **Create a service request** with required tasks
3. **Complete all tasks** and upload required documents
4. **Verify "Ready for Submission"** alert appears
5. **Click "Request Bumara Submission"** button
6. **Verify** success message and status changes to "Submission Requested"
7. **Open Backoffice Inbox** → verify job appears in "Unassigned Queue"
8. **Click "Claim"** on the job
9. **Verify** job moves to "Assigned" tab
10. **Request again** (on tenant side) → verify returns same job (idempotent)
11. **Create incomplete entity** → verify blockers are displayed
