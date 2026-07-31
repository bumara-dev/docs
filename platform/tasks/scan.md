---
title: "Task System Scan Results"
description: "Codebase scan of the Task System across the monorepo: DB schema, creation flows, APIs, UI usage, and audit logging."
---

## 0.1 DB Schema

### `tasks` Table
**File:** packages/database/src/schema/compliance/tasks.ts

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | Primary key |
| `organization_id` | text | FK → organizations, **required for org isolation** |
| `filing_id` | uuid | FK → filings (nullable) |
| `service_request_id` | uuid | FK → service_requests (nullable) |
| `template_key` | text | Links to template task definition |
| `title` | text | Required |
| `description` | text | Optional |
| `task_type` | enum | `upload_document`, `fill_form`, `review_approve`, `payment_action`, `info_request`, `custom` |
| `required` | boolean | Default false |
| `status` | enum | `todo`, `doing`, `blocked`, `done`, `skipped` |
| `blocked_reason` | text | Required when status=blocked |
| `skip_reason` | text | Required when status=skipped |
| `payload` | jsonb | Form-type task data storage |
| `assigned_to_member_id` | text | FK → organization_members |
| `sequence` | integer | Sort order |
| `is_blocking` | boolean | Default false |
| `due_on` | timestamp | Optional due date |
| `completed_at` | timestamp | Set when done |
| `completed_by` | text | FK → organization_members |
| `action_kind` | enum | ✅ `navigate`, `form_section`, `doc_upload`, `confirmation`, `none` |
| `action_ref` | jsonb | ✅ Links task to specific UI section/form |
| `regulator_key` | text | ✅ Denormalized regulator code for efficient filtering |
| `completion_trigger` | text | ✅ Indexed trigger for auto-completion (e.g., `form:PACRA_BUSINESS_NAME:business_details`) |
| `created_at`, `updated_at`, `deleted_at` | timestamp | Standard timestamps |

**Indexes:**
- `idx_tasks_org_status` → `(organization_id, status)`
- `idx_tasks_parent` → `(filing_id, service_request_id)`
- `idx_tasks_org_due` → `(organization_id, due_on)`
- `idx_tasks_assignee` → `(assigned_to_member_id, status)`
- `idx_tasks_org_required` → `(organization_id, required)`
- `idx_tasks_template_key` → `(filing_id, template_key)`
- `idx_tasks_regulator_key` → `(organization_id, regulator_key, status)` ✅ NEW
- `idx_tasks_completion_trigger` → `(organization_id, completion_trigger)` ✅ NEW

**Unique Constraints:**
- `uq_tasks_filing_template_key` → `(filing_id, template_key)`
- `uq_tasks_service_request_template_key` → `(service_request_id, template_key)`

### Task Template Configs (JSONB)
**File:** packages/database/src/schema/compliance/obligation-templates.ts

```typescript
interface TaskTemplateConfig {
  key: string;
  title: string;
  description?: string;
  taskType: "upload_document" | "fill_form" | "review_approve" | "payment_action" | "info_request" | "custom";
  required: boolean;
  isBlocking?: boolean;
  sequence: number;
  dueDaysBeforeFiling?: number;
}
```

### Service Templates
**File:** packages/database/src/schema/compliance/service-templates.ts

- Same `taskTemplateConfigs` JSONB structure as obligation templates
- Also has `docRequirementConfigs` and `paymentRuleConfig`

---

## 0.2 Task Creation Flows

### Filing Task Creation (Activation Engine)
**File:** packages/api-services/src/domains/activation/activation.service.ts

**Function:** `generateTasksForFiling` (lines 483-533)
- Creates tasks from `taskTemplateConfigs` when a filing is generated
- Uses `onConflictDoNothing` for idempotency
- Computes task due dates based on `dueDaysBeforeFiling`
- Updates filing `tasksRequired` count

### Service Request Task Creation
**File:** packages/api-services/src/domains/compliance/service-requests.service.ts

**Function:** `createServiceRequest` (lines 583-688)
- Creates tasks from `template.taskTemplateConfigs` when a service request is created
- Inserts tasks with `templateKey`, `title`, `description`, `taskType`, `required`, `sequence` from config

### Auto-Completion Hooks
**File:** packages/api-services/src/domains/tasks/tasks.service.ts

**Functions:**
- `maybeAutoTransitionFiling` (lines 1068-1173) — Auto-transitions filing to `ready_for_submission` when all required tasks are done
- `maybeAutoTransitionServiceRequest` (lines 1179+) — Same for service requests
- Called from `updateTaskStatus` when a task is marked done

---

## 0.3 APIs (Hono RPC)

**Routes File:** packages/backend/src/modules/tasks/tasks.routes.ts

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/tasks` | GET | List tasks (paginated, filters: status, taskType, required, assignedToMe, regulatorKey, filingId, serviceRequestId, overdue, dueSoon) |
| `/tasks/{id}` | GET | Get single task |
| `/tasks` | POST | Create task |
| `/tasks/{id}/status` | PATCH | Update task status |
| `/tasks/{id}/payload` | PATCH | Update task payload |
| `/tasks/{id}/assign` | PATCH | Assign task |
| `/tasks/{id}/skip` | POST | Skip task (reason required) |
| `/tasks/{id}/reopen` | POST | Reopen done/skipped task |
| `/tasks/{id}/comments` | GET | List task comments |
| `/tasks/{id}/comments` | POST | Add task comment |
| `/filings/{id}/readiness` | GET | Check filing readiness |
| `/filings/{id}/tasks` | GET | Get filing tasks |
| `/service-requests/{id}/tasks` | GET | Get SR tasks |

**Service File:** packages/api-services/src/domains/tasks/tasks.service.ts (1395 lines)

**Key Functions:**
- `listTasks` — With regulator filtering via subquery
- `getTask` — Includes comment count
- `createTask` — With permission check
- `updateTaskStatus` — With transition validation and auto-transition hooks
- `assignTask`, `skipTask`, `reopenTask`
- `checkFilingReadiness`, `checkServiceRequestReadiness`
- `getEntityTasks`, `getFilingTasks`, `getServiceRequestTasks`

---

## 0.4 UI Usage

### Tasks Page Content
**File:** apps/app/features/regulators/components/tasks-page-content.tsx

- 865 lines, comprehensive tasks listing
- Groups tasks by entity (filing/service request)
- Filters: status, search
- Quick complete via checkbox
- Task details modal
- Deep links to parent entity

### Entity Tasks Section
**File:** apps/app/features/compliance/components/entity-tasks-section.tsx

- 685 lines, inline task completion UI
- Handles different task types: confirmation, form, upload_document, payment
- External task detection via `payload.sourceUrl`
- Form notes with save/complete actions
- Task details modal integration

### PACRA Tasks Components
**Directory:** `apps/app/features/pacra/tasks/components/`
- `task-details-modal.tsx` — Full task details with actions
- `task-status-badge.tsx` — Status display
- `task-comments.tsx` — Comment thread
- `task-documents.tsx` — Document attachments
- `block-task-dialog.tsx`, `skip-task-dialog.tsx`

### Regulator Tasks Components
**Directory:** `apps/app/features/regulators/components/tasks/`
- Mirror of PACRA components for generic regulator use

---

## 0.5 Audit Logging

**File:** packages/api-services/src/domains/tasks/tasks.service.ts

**Audit log calls (9 total):**
- Task creation (line 535)
- Task status update (line 644)
- Task assignment (line 731)
- Task skip (line 809)
- Task reopen (line 882)
- Filing auto-transition (line 1152)
- Service request auto-transition (line 1266)
- Filing manual transition (line 1337)

**Metadata includes:** `regulatorId`, `filingId`, `serviceRequestId`, `title`, `reason`, `trigger`

---

## Gap Analysis

### ✅ Resolved: Columns / Features

| Item | Status | Notes |
|------|--------|-------|
| `regulator_key` column | ✅ Added | Denormalized on task creation |
| `action_kind` enum | ✅ Added | `navigate`, `form_section`, `doc_upload`, `confirmation`, `none` |
| `action_ref` jsonb | ✅ Added | Links task to specific UI section/form |
| `completion_trigger` column | ✅ Added | Indexed trigger for efficient auto-completion |

### ✅ Resolved: Task Action Anchors

Tasks now deep-link to specific form sections or pages:
- ✅ `action_kind` and `action_ref` in task schema
- ✅ Templates include `actionKind` and `actionRefTemplate`
- ✅ Placeholder replacement (`{filingId}`, `{serviceRequestId}`) on task creation

### ✅ Resolved: Task Generation

| Scenario | Status | Notes |
|----------|--------|-------|
| Filing tasks | ✅ Working | Generated via activation engine with action anchors |
| Service request tasks | ✅ Working | Generated in `createServiceRequest` with action anchors |
| Task `action_ref` | ✅ Populated | From template `actionRefTemplate` |
| Task `regulator_key` | ✅ Populated | Denormalized from parent entity's regulator |
| Task `completion_trigger` | ✅ Populated | Built from `actionKind` + `actionRefTemplate` |

### ✅ Resolved: Automation

**Implemented helpers:**
- `completeTasksForAction` — Original JSONB-based matching (existing)
- `completeTasksByTrigger` — New indexed trigger-based completion
- `completeTasksOnFormSave` — Convenience wrapper for form saves
- `completeTasksOnDocUpload` — Convenience wrapper for doc uploads
- `recomputeParentProgressAndStatus` — Centralized readiness computation

**Document upload automation:**
- ✅ `createDocument` calls `completeTasksForAction` when `requirementKey` is provided

**Form save automation:**
- ⚠️ Not yet wired — Form save handlers need to call `completeTasksOnFormSave`

### ✅ Resolved: Progress/Readiness Recomputation

- ✅ `recomputeParentProgressAndStatus` helper exists
- ✅ Returns progress stats and readiness result with blockers
- ✅ Called after task completion in automation helpers

### ✅ Resolved: Org Isolation Checks

| API | org_id Check |
|-----|--------------|
| `listTasks` | ✅ Yes |
| `getTask` | ✅ Yes |
| `createTask` | ✅ Yes |
| `updateTaskStatus` | ✅ Yes |
| `assignTask` | ✅ Yes |
| `skipTask` | ✅ Yes |
| `reopenTask` | ✅ Yes |
| `completeTasksByTrigger` | ✅ Yes |
| `completeTasksOnFormSave` | ✅ Yes |
| `completeTasksOnDocUpload` | ✅ Yes |
| Task activation | ✅ Yes |

**No gaps in org isolation** — All task operations properly filter by `organization_id`.

### ⚠️ Remaining: Tests

| Test Type | Status | Notes |
|-----------|--------|-------|
| Unit tests for task transitions | ✅ Basic exists | `tasks.service.test.ts` |
| Integration tests for org scoping | ⚠️ Missing | Need cross-org isolation tests |
| Automation tests | ⚠️ Missing | Need trigger-based completion tests |

### ✅ Resolved: Documentation

| Doc | Status |
|-----|--------|
| `docs/platform/tasks/spec.md` | ✅ Created |
| `docs/platform/tasks/linking.md` | ✅ Created |
| `docs/platform/tasks/scan.md` | ✅ Updated |

---

## Summary of Findings

### ✅ What Works Well

1. **DB schema** — Tasks table with proper columns, indexes, and unique constraints
2. **Action anchors** — ✅ `action_kind`, `action_ref`, `completion_trigger` columns
3. **API coverage** — Comprehensive endpoints for CRUD, status, assignment, comments
4. **Audit logging** — All task transitions are audited with metadata
5. **Org isolation** — Properly enforced in all service functions
6. **Auto-transition** — Filings and SRs auto-transition to `ready_for_submission` when tasks complete
7. **UI components** — Rich inline completion UI with different task type handling
8. **Task generation** — Works for both filings (activation) and service requests with action anchors
9. **Automation helpers** — `completeTasksByTrigger`, `completeTasksOnFormSave`, `completeTasksOnDocUpload`
10. **Template coverage** — All PACRA and ZRA service templates have action anchors configured

### ⚠️ What Needs Work

1. **Form Save Wiring** — Form save handlers need to call `completeTasksOnFormSave`
2. **Integration Tests** — Need tests for trigger-based automation and org isolation
3. **UI Action Buttons** — Verify UI uses `actionRef` for deep linking

---

## Recommended Next Steps

1. ~~Add action anchor columns to `tasks` schema~~ ✅ Done
2. ~~Update template configs to include action refs~~ ✅ Done
3. ~~Implement `completeTasksByTrigger` helper~~ ✅ Done
4. ~~Implement `recomputeParentProgressAndStatus` helper~~ ✅ Done
5. **Wire form save handlers** to call `completeTasksOnFormSave`
6. **Add integration tests** for trigger-based automation
7. ~~Create documentation~~ ✅ Done
