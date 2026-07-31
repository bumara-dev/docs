---
title: "Operations Module"
description: "Inbox, Cases, and Tickets & SLA documentation."
---

## Table of Contents

1. [Inbox (Work Queue)](#1-inbox-work-queue)
2. [Cases](#2-cases)
3. [Tickets & SLA](#3-tickets--sla)
4. [Migration Status (Queue/Workspace)](#4-migration-status-queueworkspace)

---

## 1. Inbox (Work Queue)

**Route:** `/inbox`  
**Purpose:** Show actionable work requiring staff attention.

### 1.1 Queue Tabs

| Tab | Filter | Description |
|-----|--------|-------------|
| Unassigned | `assigned_to IS NULL` | Cases not yet claimed |
| Assigned to me | `assigned_to = current_staff` | My active work |
| Due soon | `due_on <= NOW + 7 days` | Upcoming deadlines |
| Overdue | `due_on < NOW AND status NOT IN terminal` | Past due |
| Waiting on client | `status = WAITING_ON_CLIENT` | Blocked by tenant |
| Waiting on payment | `status = WAITING_ON_PAYMENT` | Payment pending |
| Ready to submit | `status = READY_TO_SUBMIT` | Gates passed |
| Submitted | `status = SUBMITTED` | Awaiting outcome |

### 1.2 Queue Counts

Display counts for each queue in tab badges:

```typescript
interface QueueCounts {
  unassigned: number;
  assignedToMe: number;
  dueSoon: number;
  overdue: number;
  waitingOnClient: number;
  waitingOnPayment: number;
  readyToSubmit: number;
  submitted: number;
}
```

### 1.3 Table Columns

| Column | Source | Notes |
|--------|--------|-------|
| Case ID | `id` | Link to detail |
| Type | `filing` / `service_request` / `submission_job` | Badge |
| Organization | `organization.name` | Link to org profile |
| Regulator | `regulator.name` | ZRA, PACRA, etc. |
| Obligation | `obligation.name` (if filing) | - |
| Status | `status` | Colored badge |
| Due Date | `due_on` | With SLA indicator |
| Assigned To | `assigned_to.name` | - |
| Priority | `priority` | LOW / NORMAL / HIGH / URGENT |

### 1.4 SLA Indicators

Visual indicators for time sensitivity:

| Status | Condition | Color |
|--------|-----------|-------|
| OK | > 24 hours to due | Green |
| AT_RISK | 1-24 hours to due | Yellow |
| BREACHED | Past due | Red |

```tsx
<SlaIndicator status={calculateSlaStatus(dueOn)} />
```

### 1.5 Actions

| Action | Permission | Description |
|--------|------------|-------------|
| Claim | All staff | Atomic self-assignment |
| Reassign | Manager+ | Assign to another staff |
| Change Priority | All staff | Update priority level |
| Open Case | All staff | Navigate to detail |

### 1.6 Claim Flow (Atomic)

Prevent double-claiming with database transaction:

```sql
UPDATE submission_jobs 
SET assigned_to_agent_id = $staffId, 
    assigned_at = NOW(),
    status = 'ASSIGNED'
WHERE id = $jobId 
  AND assigned_to_agent_id IS NULL
RETURNING *;
```

If no rows returned → already claimed → show error.

### 1.7 Filtering

| Filter | Type | Options |
|--------|------|---------|
| Status | Multi-select | All statuses |
| Type | Radio | Filing / Service Request / Job |
| Regulator | Multi-select | ZRA, PACRA, NAPSA, NHIMA |
| Due Date | Date range | From/To |
| Organization | Search | Name/TPIN autocomplete |
| Assigned | Radio | All / Assigned / Unassigned |

### 1.8 API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `GET /compliance/queue` | GET | List queue items with filters |
| `GET /compliance/queue/counts` | GET | Queue counts by tab |
| `POST /compliance/cases/:id/claim` | POST | Atomic claim |

---

## 2. Cases

**Route:** `/cases` (list), `/cases/:id` (detail)  
**Purpose:** Unified view of all compliance work items.

### 2.1 Case Types

A "Case" is a backoffice abstraction representing:

| Type | Source Table | Description |
|------|--------------|-------------|
| Filing | `filings` | Time-bound compliance submission |
| Service Request | `service_requests` | One-off request |
| Submission Job | `submission_jobs` | Execution tracking |

### 2.2 List View

**Tabs:**
- All (default)
- Filings (`?type=filing`)
- Service Requests (`?type=service_request`)
- Submission Jobs (`?type=submission_job`)

Same table structure as Inbox.

### 2.3 Detail Page (`/cases/:id`)

**Header Section:**

```
┌─────────────────────────────────────────────────────────────────────┐
│ Case Title: [Org Name] - [Obligation/Service Name]                  │
│                                                                     │
│ ┌──────────┐  Due: Dec 31, 2025 (5 days)  Assigned: Jane Doe       │
│ │IN_PROGRESS│  Priority: [HIGH ▼]         [Reassign]               │
│ └──────────┘                                                        │
│                                                                     │
│ [Create Ticket] [Upload Evidence] [Mark Ready] [▼ More Actions]    │
└─────────────────────────────────────────────────────────────────────┘
```

**Main Content (Tabs):**

| Tab | Content |
|-----|---------|
| Overview | Org info, regulator, dates, metadata |
| Tasks | Tenant task list with status |
| Payments | Tenant payment + regulator payout |
| Documents | Attached files with tags |
| Tickets | Related tickets |

**Sidebar:**

| Section | Content |
|---------|---------|
| Readiness Checklist | Gates with pass/fail |
| Timeline | Audit events (most recent) |

### 2.4 Readiness Checklist

Display all gates with status:

```
Readiness Checklist
────────────────────────────────────────
✓ Required tasks completed (5/5)
✓ Required documents uploaded (3/3)
✗ Payment verified
    └─ Blocker: Payment pending verification
✓ Payout verified
✓ Authorization valid
────────────────────────────────────────
[Override Gates] (manager+ only)
```

### 2.5 Status Transitions

**Valid Transitions from Case Detail:**

| Current Status | Available Actions |
|----------------|-------------------|
| IN_PROGRESS | Mark Ready (if gates pass) |
| READY_FOR_SUBMISSION | Start Submission |
| SUBMISSION_IN_PROGRESS | Mark Submitted |
| SUBMITTED | Mark Accepted / Needs Correction |
| NEEDS_CORRECTION | Return to In Progress |

**Override Flow (Manager+):**

1. Staff attempts transition blocked by gate
2. If manager/admin: "Override" button appears
3. Click opens modal requiring reason
4. Reason logged in audit

### 2.6 Case Actions

| Action | Trigger | Result |
|--------|---------|--------|
| Create Ticket | Button | Opens ticket modal |
| Upload Evidence | Button | Opens upload modal |
| Mark Ready | Button (gates pass) | Transition to READY_FOR_SUBMISSION |
| Start Submission | Button | Transition to SUBMISSION_IN_PROGRESS |
| Mark Submitted | Button + evidence required | Transition to SUBMITTED |
| Mark Accepted | Button + evidence required | Transition to ACCEPTED |
| Needs Correction | Button | Creates ticket + transitions |

### 2.7 Timeline Component

Append-only audit log viewer:

```
Timeline
────────────────────────────────────────
◯ Jan 3, 2:30 PM - Jane Doe
│ Payment verified
│ Evidence: payment_receipt.pdf
│
◯ Jan 3, 11:00 AM - John Smith
│ Case claimed
│
◯ Jan 2, 4:15 PM - System
│ Submission requested by tenant
│ Status changed: PENDING_DATA → QUEUED
│
[Load more...]
```

### 2.8 API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `GET /compliance/cases` | GET | List with filters |
| `GET /compliance/cases/:id` | GET | Detail with gates |
| `POST /compliance/cases/:id/claim` | POST | Atomic claim |
| `POST /compliance/cases/:id/assign` | POST | Reassign (manager+) |
| `POST /compliance/cases/:id/transition` | POST | Status change |
| `POST /compliance/cases/:id/override-gate` | POST | Bypass gate (manager+) |

---

## 3. Tickets & SLA

**Route:** `/tickets`  
**Purpose:** Track information requests and SLA compliance.

### 3.1 Ticket Types

| Type | Purpose |
|------|---------|
| `DATA_REQUEST` | Request missing data from tenant |
| `PAYMENT_REQUEST` | Request payment from tenant |
| `CLARIFICATION` | Request clarification |
| `CORRECTION` | Request correction of submitted data |
| `LIMIT_REACHED` | Plan limit notification |

### 3.2 Ticket Lifecycle

```
OPEN
  │
  ▼ (sent to tenant)
AWAITING_TENANT ◄───────────────────┐
  │                                  │
  ▼ (tenant responds)                │
TENANT_RESPONDED ───────────────────►│ (need more info)
  │                                  │
  ▼ (staff reviews)                  │
RESOLVED ─────────────────────────► ESCALATED (if SLA breached)
  │
  ▼
CLOSED
```

### 3.3 Ticket Table

| Column | Description |
|--------|-------------|
| ID | Ticket reference |
| Subject | Brief description |
| Type | DATA_REQUEST, etc. |
| Case | Linked case (clickable) |
| Organization | Tenant name |
| Status | Current status |
| Created | When created |
| Updated | Last activity |
| SLA | Time remaining / breached |

### 3.4 Ticket Filters

| Filter | Options |
|--------|---------|
| Status | Open / Awaiting Tenant / Responded / Resolved / All |
| Type | All types |
| Organization | Search |
| SLA | All / At Risk / Breached |
| Assigned | All / Mine |

---

## 4. Migration Status (Queue/Workspace)

The legacy `/inbox` + `/cases` model remains available for compatibility, but the canonical MVP model is now:

- Queue:
  - `GET /backoffice/queue`
  - `POST /backoffice/queue/{entityType}/{id}/claim`
  - `POST /backoffice/queue/{entityType}/{id}/unclaim` (manager/admin)
  - `POST /backoffice/queue/{entityType}/{id}/assign` (manager/admin, reason required)
- Workspace:
  - `GET /backoffice/workspace/my`
  - `GET /backoffice/workspace/all` (manager/admin)
  - `GET /backoffice/workspace/staff/{userId}` (manager/admin)
  - `POST /backoffice/workspace/{entityType}/{id}/revoke` (manager/admin, reason required)
  - `POST /backoffice/workspace/{entityType}/{id}/reassign` (manager/admin, reason required)

Entity-aware operations:

- Queue/workspace entity endpoints accept `entityType` of `filing | service_request | submission_job`.
- Backend resolves filing/service-request IDs to canonical `submission_job` ownership records before claim/revoke/reassign.

Legacy inbox deprecation:

- `/backoffice/inbox/*` is still active during migration.
- Responses include deprecation metadata headers:
  - `Deprecation: true`
  - `Sunset: Tue, 30 Jun 2026 23:59:59 GMT`
  - `Link: <successor-route>; rel="successor-version"`

Credential access update:

- Credential sessions are removed.
- Current flow is claim-gated direct access:
  - `GET /backoffice/regulator-connections/{id}/masked`
  - `POST /backoffice/regulator-connections/{id}/reveal` (step-up + reason)
  - `POST /backoffice/regulator-connections/{id}/rotate` (manager/admin)
  - `POST /backoffice/regulator-connections/{id}/verify`

### 3.5 Create Ticket Modal

**Required Fields:**
- Subject (text)
- Type (select)
- Description (rich text)
- Required items (list of what's needed)
- Due date (optional)

**Linked automatically:**
- Case ID
- Organization ID
- Created by (staff)

### 3.6 Ticket Actions

| Action | Description |
|--------|-------------|
| View Details | Open ticket panel |
| Add Note | Internal note (not visible to tenant) |
| Reply | Add response (visible to tenant) |
| Resolve | Mark as resolved |
| Escalate | Escalate to manager |
| Close | Close ticket |

### 3.7 SLA Timers

| Timer | Description | Default |
|-------|-------------|---------|
| First Response | Time to first staff response | 4 hours |
| Resolution | Time to resolve ticket | 24 hours |
| Tenant Response | Time for tenant to respond | 48 hours |

**SLA Status Calculation:**

```typescript
function getTicketSlaStatus(ticket: Ticket): SlaStatus {
  const targetTime = getTargetTime(ticket);
  const elapsed = Date.now() - ticket.createdAt.getTime();
  const remaining = targetTime - elapsed;
  
  if (remaining < 0) return 'BREACHED';
  if (remaining < 4 * 60 * 60 * 1000) return 'AT_RISK'; // 4 hours
  return 'OK';
}
```

### 3.8 Tenant Notifications

When ticket is created or updated, notify tenant via:
- In-app notification
- Email
- SMS (if configured)
- WhatsApp (if configured)

**Notification Events:**
- Ticket created
- Staff replied
- Ticket resolved
- SLA at risk (tenant reminder)

### 3.9 API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `GET /tickets` | GET | List with filters |
| `POST /tickets` | POST | Create ticket |
| `GET /tickets/:id` | GET | Get detail |
| `POST /tickets/:id/notes` | POST | Add internal note |
| `POST /tickets/:id/reply` | POST | Reply to tenant |
| `POST /tickets/:id/resolve` | POST | Mark resolved |
| `POST /tickets/:id/escalate` | POST | Escalate |
| `POST /tickets/:id/close` | POST | Close ticket |

---

## File Locations

**Current Implementation:**

| Component | Location |
|-----------|----------|
| Inbox page | `apps/backoffice/app/(authenticated)/(home)/(general)/inbox/page.tsx` |
| Cases page | `apps/backoffice/app/(authenticated)/(home)/(general)/cases/page.tsx` |
| Tickets page | `apps/backoffice/app/(authenticated)/(home)/(general)/tickets/page.tsx` |
| Nav config | `apps/backoffice/config/nav.ts` |

**To Create:**

| Component | Location |
|-----------|----------|
| Case detail | `apps/backoffice/app/(authenticated)/(home)/(general)/cases/[id]/page.tsx` |
| API hooks | `apps/backoffice/lib/api/hooks/use-cases.ts` |
| Case components | `apps/backoffice/components/cases/` |
| Ticket components | `apps/backoffice/components/tickets/` |

---

## Related Documentation

- [Finance Module](/backoffice/modules/finance) - Payments & Payouts
- [Documents Module](/backoffice/modules/documents) - Evidence
- [Implementation Plan](/backoffice/implementation-plan) - Build steps

