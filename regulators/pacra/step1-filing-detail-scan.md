---
title: "PACRA Filing Detail - Step 1 Codebase Scan"
description: "Pre-implementation scan for Step 1: Filing Detail Page Scan Date: January 8, 2026"
---

## 1. PACRA Workspace Routes

### 1.1 Filing Routes

| Route | File | Status |
|-------|------|--------|
| `/regulators/pacra/filings` | `apps/app/app/(authenticated)/(general)/regulators/pacra/filings/page.tsx` | ✅ Working |
| `/regulators/pacra/filings/[filingId]` | `apps/app/app/(authenticated)/(general)/regulators/pacra/filings/[filingId]/page.tsx` | ✅ Partial |

**Filing List Page (`page.tsx`):**
- Uses `useFilings()` hook with regulator filter
- Displays `FilingCard` components with status, dates, progress
- Status filter dropdown
- Links to filing detail page

**Filing Detail Page (`[filingId]/page.tsx`):**
- Uses `useFiling()` hook for single filing data
- Uses `useFilingReadiness()` for readiness check
- Displays header with status badge, due date, progress
- Tabs: Tasks (summary only), Documents (list), Payments
- `ReadinessProgressCard` shows blocked/pending tasks
- **Gap:** Tasks tab shows summary counts, not inline task list
- **Gap:** Documents tab shows list, not requirements checklist

### 1.2 Tasks Routes

| Route | File | Status |
|-------|------|--------|
| `/regulators/pacra/tasks` | `apps/app/app/(authenticated)/(general)/regulators/pacra/tasks/page.tsx` | ✅ Working |

**Tasks Page:**
- Uses `PacraTasksClient` component
- Full task list with filters (view, status, due, search)
- `TaskDetailsModal` for viewing/updating individual tasks
- Status transitions: todo → doing → done/blocked/skipped

---

## 2. Data Fetching for Filings

### 2.1 Filing Record

**Hook:** `useFiling(filingId)` in `apps/app/lib/queries/regulators/hooks/use-filings.ts`

**Fetcher:** `fetchFiling()` in `apps/app/lib/queries/regulators/fetchers/filings.ts`

**Backend Endpoint:** `GET /filings/:id`

**Handler:** `getFilingHandler` in `packages/backend/src/modules/compliance/filings/handlers.ts`

**Service:** `getFiling()` in `packages/api-services/src/domains/compliance/filings.service.ts`

**Returns:**
```typescript
{
  id, organizationId, regulatorId, obligationId, templateId,
  periodKey, periodLabel, periodStart, periodEnd, dueOn,
  status, tasksRequired, tasksCompleted, docsRequired, docsUploaded,
  billingTag, totalPenalty, obligation, createdAt, updatedAt,
  tasksSummary: { total, todo, doing, blocked, done, skipped },
  documents: [{ id, filename, kind, uploadedAt }],
  paymentStatus: null // TODO
}
```

**What works:** Basic filing data, task summary counts, documents list
**What's missing:** Tasks array, document requirements from template, blockers detail

### 2.2 Tasks Linked to Filing

**Hook:** `useFilingTasks(filingId)` in `apps/app/lib/queries/tasks/hooks/use-tasks.ts`

**Fetcher:** `fetchFilingTasks()` in `apps/app/lib/queries/tasks/fetchers/tasks.ts`

**Backend Endpoint:** `GET /tasks/filing/:id/tasks`

**Service:** `getTasksForFiling()` in `packages/api-services/src/domains/tasks/tasks.service.ts`

**Returns:** Paginated list of tasks for the filing

**What works:** Can fetch all tasks for a filing
**What's missing:** Not used on filing detail page currently

### 2.3 Document Requirements

**Source:** `docRequirementConfigs` JSONB field on `obligation_templates` table

**Schema:** `packages/database/src/schema/compliance/obligation-templates.ts`

```typescript
interface DocRequirementConfig {
  key: string;
  name: string;
  description?: string;
  kind: "source" | "workpaper" | "submission" | "receipt" | "certificate";
  required: boolean;
  conditions?: { entityType?: string[] };
}
```

**What works:** Template stores document requirements
**What's missing:** 
- No endpoint to fetch requirements for a filing
- No tracking of which requirement is satisfied by which document
- Filing has `docsRequired` count but no requirement details

### 2.4 Uploaded Documents

**Hook:** `useDocuments({ filingId })` in `apps/app/lib/queries/regulators/hooks/use-documents.ts`

**Backend Endpoint:** `GET /documents?filingId=...`

**Service:** `listDocuments()` in `packages/api-services/src/domains/compliance/documents.service.ts`

**What works:** List documents attached to filing
**What's missing:** No `requirementKey` to map doc to requirement

---

## 3. Task Completion

### 3.1 UI Components

| Component | File | Purpose |
|-----------|------|---------|
| `PacraTasksClient` | `apps/app/features/pacra/tasks/pacra-tasks-client.tsx` | Main task list with filters |
| `TaskDetailsModal` | `apps/app/features/pacra/tasks/components/task-details-modal.tsx` | View task details, change status |
| `TaskStatusBadge` | `apps/app/features/pacra/tasks/components/task-status-badge.tsx` | Status display |
| `TaskComments` | `apps/app/features/pacra/tasks/components/task-comments.tsx` | Comments on task |
| `TaskDocuments` | `apps/app/features/pacra/tasks/components/task-documents.tsx` | Documents linked to task |
| `BlockTaskDialog` | `apps/app/features/pacra/tasks/components/block-task-dialog.tsx` | Block task with reason |
| `SkipTaskDialog` | `apps/app/features/pacra/tasks/components/skip-task-dialog.tsx` | Skip task with reason |

**What works:** Full task management in separate modal/page
**What's missing:** 
- Inline task completion on filing page
- Task type-specific UI (confirmation checkbox, form fields)
- No payload editing for form-type tasks

### 3.2 Endpoints/Mutations

| Hook | Endpoint | Purpose |
|------|----------|---------|
| `useUpdateTaskStatus()` | `PATCH /tasks/:id/status` | Update status (todo/doing/done/blocked) |
| `useSkipTask()` | `POST /tasks/:id/skip` | Skip with reason |
| `useReopenTask()` | `POST /tasks/:id/reopen` | Reopen done/skipped task |
| `useAddTaskComment()` | `POST /tasks/:id/comments` | Add comment |

**Backend Service:** `packages/api-services/src/domains/tasks/tasks.service.ts`

Functions: `updateTaskStatus()`, `skipTask()`, `reopenTask()`, `addTaskComment()`

**What works:** Status transitions, skip/reopen, comments
**What's missing:** 
- No `payload` field in task schema
- No endpoint to save task payload (form data)

### 3.3 Validation

**Schema Location:** `packages/api-services/src/domains/tasks/tasks.schema.ts`

```typescript
// Current UpdateTaskStatusInput
{
  status: "todo" | "doing" | "blocked" | "done",
  blockedReason?: string
}
```

**What works:** Status validation, blocked reason
**What's missing:** 
- No `payload` in input schema
- No validation for task-type-specific payload schemas

---

## 4. Document Upload

### 4.1 Signed Upload URL Flow

**Documented in:** `docs/modules/documents/api.md`

Expected flow:
1. `POST /documents/uploads/init` → returns `{ uploadUrl, storageKey, expiresAt }`
2. Client PUTs file to `uploadUrl`
3. `POST /documents/uploads/complete` → finalizes document

**What's implemented:** Documentation only - endpoints NOT implemented

**Current implementation:**
- `POST /documents` creates document record directly with `storageKey`
- Assumes file already uploaded externally
- Uses Vercel Blob (`packages/storage/index.ts` exports `@vercel/blob`)

### 4.2 Document Metadata Creation

**Hook:** `useCreateDocument()` in `apps/app/lib/queries/regulators/hooks/use-documents.ts`

**Endpoint:** `POST /documents`

**Input Schema:** `packages/api-services/src/domains/compliance/documents.schema.ts`

```typescript
{
  regulatorId?: string,
  filingId?: string,
  serviceRequestId?: string,
  kind: "source" | "workpaper" | "submission" | "receipt" | "certificate",
  storageKey: string,
  filename?: string,
  mimeType?: string,
  sizeBytes?: number,
  metadata?: Record<string, unknown>
}
```

**What works:** Create document record with storageKey
**What's missing:** No `requirementKey` field

### 4.3 Attach Document to Filing

**Hook:** `useAttachDocument()` in `apps/app/lib/queries/regulators/hooks/use-documents.ts`

**Endpoint:** `POST /documents/:id/attach`

**Input:** `{ filingId?: string, serviceRequestId?: string }`

**Service:** `attachDocument()` updates filingId, increments `docs_uploaded` count

**What works:** Attach existing doc to filing, update counts
**What's missing:** No `requirementKey` to track satisfaction

---

## 5. Progress Computation

### 5.1 Task Progress

**Source:** `filings` table columns `tasks_required`, `tasks_completed`

**Updated by:** Task status change triggers in `tasks.service.ts`
- When task → done, filing `tasksCompleted` incremented
- When task reopened, filing `tasksCompleted` decremented

**UI:** `Progress` component in filing detail header

**What works:** Basic counts updated on status change
**What's missing:** Real-time UI update (requires query invalidation)

### 5.2 Document Progress

**Source:** `filings` table columns `docs_required`, `docs_uploaded`

**Updated by:** `documents.service.ts`
- `createDocument()` increments `docs_uploaded` if `filingId` set
- `attachDocument()` increments on new attachment
- `deleteDocument()` decrements

**What works:** Count tracking
**What's missing:**
- `docs_required` set at filing creation from template, but not verified
- No per-requirement satisfaction tracking

### 5.3 Readiness Computation

**Hook:** `useFilingReadiness(filingId)`

**Endpoint:** `GET /filings/:id/readiness`

**Service:** `checkFilingReadinessInternal()` in `filings.service.ts`

**Returns:**
```typescript
{
  isReady: boolean,
  totalTasks: number,
  completedTasks: number,
  totalRequired: number,
  completedRequired: number,
  blockedTasks: [{ id, title, status }],
  pendingRequired: [{ id, title, status }]
}
```

**What works:** Task-based readiness check, blockers list
**What's missing:** Document requirement readiness not included

---

## 6. Summary: What's Missing/Broken

### 6.1 Database Schema Gaps

| Table | Missing Column | Purpose |
|-------|----------------|---------|
| `tasks` | `payload` (JSONB) | Store form-type task data |
| `documents` | `requirement_key` (text) | Track which requirement doc satisfies |

### 6.2 Backend Gaps

| Gap | Impact |
|-----|--------|
| No `getFilingView()` aggregated endpoint | Multiple queries needed for filing detail |
| No task payload save | Form tasks can't persist data |
| No upload init/complete endpoints | Can't do proper presigned upload flow |
| No doc requirement satisfaction tracking | Can't show which requirements are met |

### 6.3 Frontend Gaps

| Gap | Impact |
|-----|--------|
| No inline task list on filing page | Must navigate away to complete tasks |
| No task type-specific UI | Can't show confirmation/form/upload UIs |
| No document requirements checklist | Only shows uploaded docs, not what's needed |
| No integrated upload | Links to separate page |

### 6.4 Risks

1. **Duplication:** Task data exists on filing (counts) and separate tasks table
2. **Missing constraints:** No FK from documents to requirements (just string key)
3. **Stale counts:** `docs_required` set at creation, may not match current template
4. **Template not populated:** Some templates may lack `docRequirementConfigs`

---

## 7. File Reference

### Key Files to Modify

```
# Database
packages/database/src/schema/compliance/tasks.ts
packages/database/src/schema/compliance/documents.ts

# Backend Services
packages/api-services/src/domains/tasks/tasks.schema.ts
packages/api-services/src/domains/tasks/tasks.service.ts
packages/api-services/src/domains/compliance/filings.service.ts
packages/api-services/src/domains/compliance/documents.service.ts

# Backend Routes
packages/backend/src/modules/compliance/documents/routes.ts
packages/backend/src/modules/compliance/documents/handlers.ts

# Frontend Page
apps/app/app/(authenticated)/(general)/regulators/pacra/filings/[filingId]/page.tsx

# Frontend Hooks
apps/app/lib/queries/regulators/hooks/use-filings.ts
apps/app/lib/queries/regulators/hooks/use-documents.ts
apps/app/lib/queries/tasks/hooks/use-tasks.ts
```

### Key Files to Create

```
# Documentation
docs/regulators/pacra/step1-filing-detail.md

# Frontend Components
apps/app/features/filings/components/filing-tasks-section.tsx
apps/app/features/filings/components/filing-documents-section.tsx
apps/app/features/filings/components/filing-blockers-panel.tsx
apps/app/components/documents/document-upload-dropzone.tsx

# Frontend Hooks
apps/app/lib/queries/regulators/hooks/use-filing-view.ts
apps/app/lib/queries/regulators/hooks/use-document-upload.ts
```
