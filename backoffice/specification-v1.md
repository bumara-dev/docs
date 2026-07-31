---
title: "Bumara Backoffice Specification"
description: "Version: v1 (MVP → V1) Audience: Product, Engineering, Ops/Compliance, Finance Applies to: Backoffice (separate internal app), shared backend services..."
---

**Version:** v1 (MVP → V1)  
**Audience:** Product, Engineering, Ops/Compliance, Finance  
**Applies to:** Backoffice (separate internal app), shared backend services (Hono RPC), shared DB (Postgres/Drizzle), shared auth (Clerk), documents storage (S3)

---

## 1) Purpose and Principles

### 1.1 Purpose
Bumara Backoffice is the internal operations application used by Bumara staff to **deliver managed compliance services** for tenant organizations. It enables staff to intake work, verify readiness (tasks/docs/payments), perform manual regulator submissions, track outcomes, and maintain full audit/evidence.

### 1.2 Product Principles
1. **Ops-first, not tenant-first:** Backoffice home is a queue/inbox—not compliance alerts.
2. **Case-centric:** Everything a staff member works on is a *Case* (filing/service request/submission job).
3. **Gated execution:** Submission cannot happen until readiness gates are met (tasks, documents, payments, authorizations).
4. **Evidence required:** Every payout and submission must have evidence.
5. **Audit by default:** All changes are logged (who/what/when/why).
6. **Least privilege:** Staff access is role-based; admin-only features are hidden and blocked.

### 1.3 Non-goals (MVP)
- Fully automated regulator submissions (APIs) — handled manually.
- Complex analytics and forecasting (can be phase 2).
- Advanced workflow automation rules engine (phase 2).
- Full internal chat/collaboration suite (notes only in MVP).

---

## 2) Scope (MVP)

### 2.1 MVP Modules
**Operations**
- Inbox / Work Queue
- Cases (unified list + detail view)
- Tickets & SLA

**Finance**
- Payments & Payouts (tenant payments + regulator payouts)

**Documents**
- Documents & Evidence (upload + tagging + linking)

**Directory**
- Organizations (tenants search + profile snapshot)

**Admin**
- System Admin (staff roles, access)
- Catalog & Rules (optional admin-only in MVP; otherwise phase 2)
- Reports & Analytics (feature-flag / hidden in MVP)

### 2.2 What “Done” means for MVP
A staff member can process a compliance request end-to-end:
- intake → assignment → request info → verify payment → pay regulator → submit → upload evidence → close → audit trail preserved

---

## 3) Personas and Roles

### 3.1 Personas
- **Ops Analyst (Member):** executes cases, requests info, uploads evidence, updates statuses.
- **Ops Manager (Manager):** assigns work, approves high-value payouts, resolves escalations, overrides with reason.
- **System Admin (Admin):** staff management, system config, sensitive operations (refunds, permissions).

### 3.2 Roles (Default)
- `admin`
- `manager`
- `member`

### 3.3 Permission Matrix (MVP)
| Capability | member | manager | admin |
|---|---:|---:|---:|
| View inbox/queues | ✅ | ✅ | ✅ |
| Claim/assign case to self | ✅ | ✅ | ✅ |
| Reassign others | ❌ | ✅ | ✅ |
| Update case status (normal transitions) | ✅ | ✅ | ✅ |
| Override gates / force status transitions | ❌ | ✅ (with reason) | ✅ (with reason) |
| Create tickets / request info from tenant | ✅ | ✅ | ✅ |
| Upload evidence | ✅ | ✅ | ✅ |
| Verify tenant payment | ✅ (if policy) | ✅ | ✅ |
| Initiate regulator payout | ✅ (create) | ✅ | ✅ |
| Approve payout above threshold | ❌ | ✅ | ✅ |
| Manage staff and roles | ❌ | ❌ | ✅ |
| Configure catalog/rules | ❌ | ❌/✅ (optional) | ✅ |

---

## 4) Navigation and Routing (Backoffice App)

Backoffice is a separate app. All routes are rooted at `/`.

### 4.1 Sidebar IA (Target)
**Operations**
- Inbox → `/inbox` (default landing)
- Cases → `/cases`
- Tickets & SLA → `/tickets`

**Finance**
- Payments & Payouts → `/payments`

**Documents**
- Documents & Evidence → `/documents`

**Directory**
- Organizations (Tenants) → `/orgs`

**Admin (admin-only)**
- System Admin → `/admin`
- Catalog & Rules → `/catalog` (optional / admin)
- Reports & Analytics → `/reports` (feature-flag / later)

### 4.2 Redirect Rules
- `/` → redirect to `/inbox`
- Legacy pages (if they exist) should redirect:
  - `/filings` → `/cases?type=filing`
  - `/service-requests` → `/cases?type=service_request`
  - `/submission-jobs` → `/cases?type=submission_job`

---

## 5) Core Concepts and Domain Model

### 5.1 Tenant Organization
A business using Bumara. Tenants have:
- profile (name, TPIN, contacts)
- regulator connections (ZRA/PACRA/NAPSA)
- obligations and filing calendar
- documents and authorizations

### 5.2 Obligation
A recurring or ad-hoc compliance requirement (e.g., VAT return, PAYE, NAPSA contribution). Obligations generate filings and tasks.

### 5.3 Filing
A time-bound compliance submission generated from an obligation (monthly/quarterly/annual).

### 5.4 Service Request
A non-recurring request initiated by tenant or staff (e.g., PACRA name clearance, business registration support).

### 5.5 Task
Work required from tenant or internal team to complete a filing/service request (documents, data, approvals).

### 5.6 Case (Backoffice abstraction)
A *Case* is the unit of work shown to staff. It may represent:
- Filing case
- Service Request case
- Submission Job case

**Backoffice UI treats them uniformly**:
- list them in `/cases`
- open a single detail page `/cases/:caseId`
- show checklist, timeline, evidence, tickets, payments

### 5.7 Submission Job
Created when tenant clicks **Request Submission** (or staff triggers submission). Tracks the execution lifecycle and assignment.

### 5.8 Ticket (Request for Info)
A structured request sent to tenant to provide missing data/docs or clarify issues.

### 5.9 Evidence
Documents proving:
- tenant payment received
- regulator payout made
- submission completed (screenshots, PDFs)
- acceptance/rejection outcomes

### 5.10 SLA
Policy + timers tracking service delivery:
- time-to-first-response
- time-to-submission
- time-to-resolution
- escalation thresholds

---

## 6) State Machines and Gates

### 6.1 Filing / Service Request Status (Tenant-visible)
Typical statuses (adapt to your existing enums):
- `DRAFT` (optional)
- `PENDING_DATA` (waiting on tenant tasks/docs)
- `IN_PROGRESS` (staff working)
- `READY_FOR_SUBMISSION` (gates passed)
- `SUBMISSION_IN_PROGRESS`
- `SUBMITTED`
- `ACCEPTED`
- `NEEDS_CORRECTION`
- `REJECTED`
- `CANCELLED` / `WAIVED`

### 6.2 Submission Job Status (Backoffice Work Tracking)
- `QUEUED`
- `ASSIGNED`
- `IN_PROGRESS`
- `WAITING_ON_CLIENT`
- `WAITING_ON_PAYMENT`
- `READY_TO_SUBMIT`
- `SUBMITTED`
- `ACCEPTED`
- `NEEDS_CORRECTION`
- `REJECTED`
- `CLOSED`

### 6.3 Payment Status (Tenant → Bumara)
- `NOT_REQUIRED`
- `REQUIRED_PENDING`
- `PAID_UNVERIFIED`
- `PAID_VERIFIED`
- `REFUNDED` (later)

### 6.4 Payout Status (Bumara → Regulator)
- `NOT_REQUIRED`
- `QUEUED`
- `SENT_UNVERIFIED`
- `PAID_VERIFIED`

### 6.5 Readiness Gates (Hard Rules)
A Case cannot move to `READY_TO_SUBMIT` unless:
1. Required tenant tasks are `DONE`
2. Required documents/evidence are uploaded
3. Tenant payment is `PAID_VERIFIED` (if required)
4. Regulator payout is `PAID_VERIFIED` (if required)
5. Authorization is valid (if required)

Overrides:
- Only `manager/admin`, must supply `overrideReason`, logged in audit.

---

## 7) User Journeys (Operational Flows)

### 7.1 Happy Path: Filing Submission (Managed Service)
1. Filing generated (scheduled or created)
2. Tenant completes tasks → clicks **Request Submission**
3. System creates Submission Job → status `QUEUED`
4. Analyst opens `/inbox` → claims job (atomic)
5. Analyst checks gates:
   - if missing: create Ticket → job becomes `WAITING_ON_CLIENT`
6. If payment required:
   - verify tenant payment → `PAID_VERIFIED`
7. If regulator payout required:
   - initiate payout → upload proof → manager approval (threshold) → `PAID_VERIFIED`
8. Analyst submits manually (portal/office)
9. Upload submission evidence (PDF/screenshot/receipt)
10. Update status → `SUBMITTED`
11. When outcome confirmed:
   - `ACCEPTED` (upload proof) → `CLOSED`
   - or `NEEDS_CORRECTION` (create Ticket) → returns to `WAITING_ON_CLIENT`

### 7.2 Service Request Flow (e.g., PACRA Name Clearance)
1. Tenant creates service request
2. Case appears in Inbox
3. Staff requests documents/choices → Ticket
4. Payment verified (if required)
5. Submit to PACRA (manual)
6. Upload evidence + outcome
7. Close

### 7.3 Ticket Lifecycle
- `OPEN` → `AWAITING_TENANT` → `TENANT_RESPONDED` → `RESOLVED` / `CLOSED`
Rules:
- Tickets must link to a Case
- Tenant responses attach files/comments
- SLA timers apply (response time)

---

## 8) Backoffice UI Specification

### 8.1 Global Layout
- Top bar: search, quick actions, notifications, profile
- Sidebar: sections grouped by workflow; role-aware visibility
- Main content: table/list pages and detail pages
- Dark theme compatible (ShadCN)

### 8.2 Inbox Page (`/inbox`)
Purpose: show actionable work requiring staff action.

**Default queue tabs (MVP)**
- Unassigned
- Assigned to me
- Due soon (next N days)
- Overdue
- Waiting on client
- Waiting on payment
- Ready to submit
- Submitted awaiting outcome

**Table columns**
- Case ID
- Type (Filing / Service / Job)
- Organization
- Regulator
- Obligation (if filing)
- Status
- Due date + SLA indicator
- Assigned to
- Priority

**Actions**
- Claim / Assign to me (atomic)
- Reassign (manager/admin)
- Change priority
- Open case

**Filtering**
- status, type, regulator, due date range, assigned, organization
- search by org name, case id, TPIN

### 8.3 Cases List (`/cases`)
Unified list with tabs/filters:
- Tabs: All | Filings | Service Requests | Submission Jobs
- Query param `type=` drives tab selection.

Same table/filters as Inbox but not queue-specific.

### 8.4 Case Detail (`/cases/:caseId`)
This is the most important screen.

**Header**
- Case title (org + obligation/service)
- Status badge
- Due date + SLA
- Assigned to + reassign
- Priority
- Quick actions:
  - Create ticket
  - Mark ready
  - Mark submitted
  - Mark accepted / needs correction / rejected
  - Upload evidence

**Sections/Tabs**
1. **Overview**
   - org info, regulator, obligation metadata
   - key dates (period, due, created)
2. **Checklist (Readiness Gates)**
   - task completion
   - payment verified
   - payout verified
   - required docs present
   - authorization valid
   - show blockers + buttons to resolve
3. **Tasks**
   - tenant tasks list + status
   - internal tasks list (optional MVP)
4. **Payments**
   - tenant payment requests (amount, method, proofs, verify)
   - regulator payout (amount, proof, approval)
5. **Documents & Evidence**
   - list of attached docs with tags
   - upload evidence with type + notes
6. **Tickets**
   - open/closed tickets linked to this case
7. **Timeline (Audit Log View)**
   - append-only events: status changes, assignments, payment verifications, uploads
8. **Notes**
   - internal-only notes with author + timestamp

### 8.5 Tickets & SLA (`/tickets`)
- View all tickets with filters (open/awaiting tenant/overdue)
- SLA timers displayed
- Bulk reminders (phase 2)

### 8.6 Payments & Payouts (`/payments`)
- Two main views (tabs):
  - Tenant Payments (incoming)
  - Regulator Payouts (outgoing)
- Filters: status, org, regulator, amount range, date range
- Actions:
  - verify payment (requires evidence)
  - initiate payout (create record)
  - approve payout (manager/admin)
  - attach payout evidence
- Export CSV (phase 2)

### 8.7 Documents & Evidence (`/documents`)
- Search documents by org, case, tag, date
- Tags:
  - `SOURCE` (tenant uploaded)
  - `WORKPAPER` (internal)
  - `PAYMENT_PROOF`
  - `PAYOUT_PROOF`
  - `SUBMISSION_PROOF`
  - `OUTCOME_PROOF`
- Permissions:
  - evidence immutable after case closed (unless admin override)

### 8.8 Organizations (`/orgs`)
- Search-first: org name, TPIN, reg number
- Org profile:
  - compliance snapshot (open cases, overdue, upcoming)
  - regulator connections
  - documents
  - recent activity timeline
  - contacts (for notifications)

### 8.9 Admin (`/admin`, `/catalog`)
- Staff management (invite, role change, disable)
- Role policy configuration (optional)
- Catalog:
  - obligations templates
  - task templates
  - SLA policies
  - fee schedules (phase 2 if complex)

---

## 9) Backend / API Specification (Hono RPC)

> Naming is illustrative—use your actual RPC conventions. Backoffice endpoints must be distinct and protected.

### 9.1 AuthZ Middleware
- `requireStaff()` – blocks non-staff entirely
- `requireRole(['admin'])` – admin pages
- `requireRole(['manager','admin'])` – approvals/overrides

### 9.2 Core Backoffice RPC Groups

#### `backoffice.queue`
- `listQueues(filters)` → returns queue counts + list
- `listItems({queue, filters, pagination})`

#### `backoffice.cases`
- `list({type, filters, pagination, sort})`
- `get({caseId})` → includes gates summary, linked entities, evidence, tickets
- `assign({caseId, assigneeId})` (manager/admin)
- `claim({caseId})` (atomic)
- `setPriority({caseId, priority})`
- `transition({caseId, toStatus, reason?})` → validates transitions
- `overrideGate({caseId, gateKey, reason})` (manager/admin)

#### `backoffice.tickets`
- `create({caseId, title, message, requiredItems[], dueAt?})`
- `list({filters, pagination})`
- `resolve({ticketId})`
- `addInternalNote({ticketId, note})`

#### `backoffice.payments`
- `listTenantPayments(...)`
- `verifyTenantPayment({paymentId, evidenceDocId, note})`
- `createPayout({caseId, amount, method, reference?})`
- `submitPayoutEvidence({payoutId, evidenceDocId})`
- `approvePayout({payoutId, decision, reason?})` (manager/admin)

#### `backoffice.documents`
- `createUploadUrl({orgId, caseId?, tag, filename, mimeType})`
- `attachToCase({docId, caseId, tag})`
- `list({filters})`

#### `backoffice.orgs`
- `search({query, pagination})`
- `getProfile({orgId})`

#### `backoffice.audit`
- `listEvents({caseId})`
- `listEventsGlobal({filters})` (admin)

---

## 10) Data Model (Conceptual)

### 10.1 Tables (MVP)
- `organizations`
- `organization_members` (tenant side)
- `staff_users` (or staff membership derived from Clerk)
- `regulators`
- `regulator_connections`
- `obligations`
- `filings`
- `service_requests`
- `tasks`
- `cases` (optional unified table) **OR** computed “case view” from filing/service_request/job
- `submission_jobs`
- `tickets`
- `payments` (tenant payments)
- `payouts` (regulator payouts)
- `documents`
- `case_documents` (link docs to cases with tags)
- `audit_events`
- `sla_timers` (or fields on case/job)

### 10.2 Audit Event (Append-only)
Fields:
- `id`
- `entityType` (case/job/payment/payout/ticket)
- `entityId`
- `action` (ASSIGNED, STATUS_CHANGED, EVIDENCE_UPLOADED, PAYMENT_VERIFIED, etc.)
- `actorStaffId`
- `before` JSON
- `after` JSON
- `reason` (required for overrides)
- `createdAt`

### 10.3 Indexing (Performance)
- cases/jobs: `(status, dueAt)`, `(assignedTo)`, `(orgId)`
- tickets: `(status, dueAt)`
- audit: `(entityType, entityId, createdAt)`
- documents: `(orgId, tag, createdAt)`

---

## 11) Documents & Evidence (S3)

### 11.1 Storage
- Store documents in S3 with signed uploads.
- DB stores metadata:
  - `key`, `bucket`, `size`, `mimeType`, `checksum`, `uploadedBy`, `orgId`, `createdAt`
- Link to case via `case_documents` with `tag` + `note`.

### 11.2 Evidence Rules
- Payment verification requires:
  - `PAYMENT_PROOF` doc attached
- Payout verification requires:
  - `PAYOUT_PROOF`
- Submission requires:
  - `SUBMISSION_PROOF`
- Outcome requires:
  - `OUTCOME_PROOF` (acceptance/rejection)

### 11.3 Immutability
- When case is `CLOSED`, evidence attachments become immutable except admin override.

---

## 12) Notifications (Backoffice ↔ Tenant)

### 12.1 Events that notify tenant (MVP)
- Ticket created / updated
- Payment required / payment verified
- Submission submitted
- Needs correction / rejected (with required actions)
- Upcoming due dates (optional if already in tenant app)

### 12.2 Events that notify staff
- New case queued
- SLA breach risk
- Ticket overdue
- Payout approval needed

(Actual delivery channels: email, SMS, WhatsApp can be implemented by your notification service; backoffice just emits events.)

---

## 13) SLA & Priority

### 13.1 SLA Timers (MVP)
Per case/job:
- `dueAt`
- `firstResponseDueAt` (optional)
- `submissionDueAt` (optional)
- `breachedAt` (if breached)
- `slaStatus`: `OK | AT_RISK | BREACHED`

### 13.2 Priority Model (MVP)
Priority can be manual:
- `LOW | NORMAL | HIGH | URGENT`

Sorting:
- breached first
- at risk next
- then due date

Escalations:
- Manager sees a queue of `BREACHED` items
- optional automated escalation notifications

---

## 14) Security & Compliance

### 14.1 Access Control
- Backoffice requires staff membership.
- Tenant users must never access backoffice routes or endpoints.
- Admin-only pages must be hidden and server-blocked.

### 14.2 Data Protection
- Minimum necessary data shown.
- Evidence documents may contain sensitive info; restrict by role and org.
- Log all downloads (phase 2) if needed.

### 14.3 Operational Safety
- Assignment is atomic (avoid double-claim).
- Status transitions validated server-side.
- Overrides require reason + manager/admin role.
- Idempotency keys for critical transitions (submit/verify) to avoid duplicates.

---

## 15) Non-functional Requirements

### 15.1 Performance
- Pagination everywhere (lists)
- Server-side filtering
- Avoid loading whole timelines by default; paginate audit events

### 15.2 Reliability
- “Retry” patterns for RPC failures
- Clear empty/error states (not tenant widgets)
- Graceful degradation if documents service is down

### 15.3 Observability
- structured logs for:
  - auth failures
  - transitions
  - payment/payout verifications
- dashboards/alerts (phase 2)

---

## 16) MVP Delivery Plan (Engineering)

### Phase 1: Foundations
- Backoffice layout + sidebar (config-driven + role-aware)
- Auth guard + staff RBAC middleware
- `/` → `/inbox`

### Phase 2: Operations Core
- Inbox queues (list + filters + claim/assign)
- Cases list + case detail skeleton

### Phase 3: Execution Features
- Tickets
- Readiness checklist gates
- Status transitions with validation
- Timeline (audit event viewer) + internal notes

### Phase 4: Finance + Evidence
- Tenant payment verification
- Regulator payouts + approvals
- S3 evidence upload + tagging

### Phase 5: Org Directory
- Org search + profile view

---

## 17) Acceptance Criteria (MVP)

### Inbox
- Staff can filter and claim cases
- Unassigned cases can only be claimed once
- SLA indicators visible

### Case Detail
- Checklist displays all gates + blockers
- Staff can create tickets
- Status changes require valid transition
- Evidence upload works and links to case

### Tickets
- Ticket creation notifies tenant
- Ticket status updates tracked in timeline

### Payments & Payouts
- Payment cannot be verified without proof
- Payout approval required above threshold
- All actions logged

### Security
- Non-staff cannot access app/routes/endpoints
- Non-admin cannot access admin routes (even via direct URL)
- All sensitive actions create audit events

---

## 18) Open Questions (Decide before implementation hardens)
1. Do we implement a physical `cases` table, or compute “case view” from filings/service requests/jobs?
2. What payout approval threshold(s) should require manager approval?
3. Which payment methods are supported in MVP (mobile money, bank transfer, card) and what evidence is acceptable?
4. Do we require two-person review for certain regulators/services?
5. Which SLA timers matter first: first response vs submission vs resolution?

---
