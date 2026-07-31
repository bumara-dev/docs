---
title: "PACRA Service Requests - Implementation Scan"
description: "Codebase scan of the existing service request infrastructure - routes, schema, services, hooks, and seeded PACRA templates."
---

## 1. Tenant App Routes/Pages

### 1.1 Service Requests List Page
**Path:** `apps/app/app/(authenticated)/(general)/regulators/pacra/service-requests/page.tsx`

**Status:** ✅ Exists and Working

**Features:**
- Lists existing service requests with status, progress
- Shows template catalog with featured services
- "Start" button to create new request from template
- Tabs for active/completed requests
- Uses `useServiceRequests`, `useServiceTemplates`, `useCreateServiceRequest` hooks

**Components Used:**
- `ServiceRequestsContent` - main container
- `FeaturedServicesSection` - featured template cards
- `ServiceCatalogModal` - full catalog browser
- `ServiceRequestsList` - list of existing requests

### 1.2 Service Request Detail Page
**Path:** `apps/app/app/(authenticated)/(general)/regulators/pacra/service-requests/[id]/page.tsx`

**Status:** ⚠️ Exists but Basic

**Current Features:**
- Header with name, status badge
- Info cards (Created date, Progress)
- Tabs: Tasks, Documents, Payments
- Tasks tab shows summary counts only (not individual tasks)
- Documents tab shows uploaded docs (no requirements)
- Payments tab placeholder

**Missing:**
- Inline task completion UI
- Document requirements from template
- Blockers panel
- Progress chips in header
- Uses basic `getServiceRequest` not aggregated view

---

## 2. Backend Schema

### 2.1 Service Requests Table
**Path:** `packages/database/src/schema/compliance/service-requests.ts`

```typescript
serviceRequests = pgTable("service_requests", {
  id: uuid().primaryKey().defaultRandom(),
  organizationId: text().notNull().references(organizations.id),
  regulatorId: uuid().references(regulators.id),
  templateId: uuid().references(serviceTemplates.id),
  regulator: regulatorEnum(),
  name: text().notNull(),
  description: text(),
  status: serviceRequestStatusEnum().default("pending_data"),
  billingTag: billingTagEnum().default("included"),
  dueOn: timestamp(),
  submittedToBackofficeBy: text(),
  submittedToBackofficeAt: timestamp(),
  regulatorReferenceNumber: text(),
  slaDeadline: timestamp(),
  ...timestamps
})
```

**Status:** ✅ Complete

**Missing Column:** `intakeData: jsonb()` for storing intake form responses

### 2.2 Service Templates Table
**Path:** `packages/database/src/schema/compliance/service-templates.ts`

```typescript
serviceTemplates = pgTable("service_templates", {
  id: uuid().primaryKey(),
  organizationId: text(), // NULL = global template
  regulatorId: uuid(),
  regulator: regulatorEnum(),
  templateKey: text().unique(),
  templateVersion: integer().default(1),
  name: text().notNull(),
  description: text(),
  whatIsThis: text(),
  whyItMatters: text(),
  consequencesOfDelay: text(),
  intakeFieldsSchema: jsonb<IntakeFieldSchema[]>(),
  expectedDueInDays: text(),
  defaultPriority: taskPriorityEnum(),
  activationRules: jsonb(),
  taskTemplateConfigs: jsonb<TaskTemplateConfig[]>(),
  docRequirementConfigs: jsonb<DocRequirementConfig[]>(),
  paymentRuleConfig: jsonb(),
  billingTag: billingTagEnum(),
  ...timestamps
})
```

**Status:** ✅ Complete with all needed fields

### 2.3 Tasks Table Support
**Path:** `packages/database/src/schema/compliance/tasks.ts`

- `serviceRequestId: uuid()` - ✅ Exists
- `templateKey: text()` - ✅ Exists
- Unique constraint: `uq_tasks_service_request_template_key` - ✅ Exists

### 2.4 Documents Table Support
**Path:** `packages/database/src/schema/compliance/documents.ts`

- `serviceRequestId: uuid()` - ✅ Exists
- `requirementKey: text()` - ✅ Exists
- Index: `idx_documents_parents` on (org, filingId, serviceRequestId) - ✅ Exists

**Missing:** Index on `(serviceRequestId, requirementKey)` for requirement lookups

---

## 3. Backend Services

### 3.1 Service Requests Service
**Path:** `packages/api-services/src/domains/compliance/service-requests.service.ts`

| Function | Status | Notes |
|----------|--------|-------|
| `listServiceRequests` | ✅ Working | Pagination, filters, task counts |
| `getServiceRequest` | ⚠️ Basic | Returns task summary, docs, but no doc requirements |
| `listServiceTemplates` | ✅ Working | Returns catalog with intake schema |
| `createServiceRequest` | ⚠️ Partial | Creates SR + tasks, but no intake data storage |

**Missing:**
- `getServiceRequestView` - aggregated view with full tasks, doc requirements, blockers

### 3.2 Documents Service
**Path:** `packages/api-services/src/domains/compliance/documents.service.ts`

- `createDocument` - ✅ Supports `serviceRequestId`
- `completeDocumentUpload` - ✅ Supports `serviceRequestId`
- `attachDocument` - ✅ Supports `serviceRequestId`

### 3.3 Tasks Service
**Path:** `packages/api-services/src/domains/tasks/tasks.service.ts`

- `createTask` - ✅ Supports `serviceRequestId`
- `updateTaskStatus` - ✅ Works for SR tasks
- `updateTaskPayload` - ✅ Works for SR tasks

---

## 4. Frontend Hooks & Fetchers

### 4.1 Service Request Hooks
**Path:** `apps/app/lib/queries/regulators/hooks/use-service-requests.ts`

| Hook | Status |
|------|--------|
| `useServiceRequests` | ✅ Working |
| `useServiceRequest` | ✅ Working (basic) |
| `useServiceTemplates` | ✅ Working |
| `useCreateServiceRequest` | ✅ Working |

**Missing:** `useServiceRequestView` for aggregated detail

### 4.2 Fetchers
**Path:** `apps/app/lib/queries/regulators/fetchers/service-requests.ts`

All basic fetchers exist. Missing `fetchServiceRequestView`.

---

## 5. PACRA Service Templates (Seeded)

**Path:** `packages/database/src/seeds/pacra-templates.ts`

### 5.1 Name Clearance (`PACRA_NAME_CLEARANCE_V1`)
**Intake Fields:**
- `proposedName1` (text, required) - First choice name
- `proposedName2` (text, optional) - Second choice
- `proposedName3` (text, optional) - Third choice
- `entityTypeForRegistration` (select, required) - company/company_public/business_name
- `reasonForName` (textarea, optional)

**Tasks:**
1. Review Name Choices (review_approve, required)
2. Confirm Entity Type (review_approve, required)

**Doc Requirements:** None

### 5.2 Change Directors (`PACRA_CHANGE_DIRECTORS_V1`)
**Intake Fields:**
- `changeType` (select, required) - appointment/resignation/removal/secretary_change/multiple
- `effectiveDate` (date, required)
- `changeDetails` (textarea, required)

**Tasks:**
1. Upload Board Resolution (upload_document, required)
2. Upload Consent Letter (upload_document, optional)
3. Upload Resignation Letter (upload_document, optional)
4. Provide Director Details (fill_form, required)
5. Review & Confirm (review_approve, required)

**Doc Requirements:**
- Board Resolution (required)
- Consent Letter (optional)
- Resignation Letter (optional)
- NRC/ID Copy (optional)

### 5.3 Change Registered Office (`PACRA_CHANGE_REGISTERED_OFFICE_V1`)
**Intake Fields:**
- `newAddress` (textarea, required)
- `effectiveDate` (date, required)
- `city` (text, required)
- `province` (select, required) - Zambian provinces

**Tasks:**
1. Upload Board Resolution (upload_document, required)
2. Provide Complete Address (fill_form, required)
3. Review & Confirm (review_approve, required)

**Doc Requirements:**
- Board Resolution (required)
- Proof of Address (optional)

---

## 6. Existing Reusable Components

### 6.1 Filing Components (to be generalized)
**Path:** `apps/app/features/filings/components/`

| Component | Props | Reusability |
|-----------|-------|-------------|
| `FilingBlockersPanel` | `blockers` | ✅ Can be generalized (no filing-specific logic) |
| `FilingTasksSection` | `filingId, tasks, progress` | ⚠️ Uses `filingId` in cache invalidation |
| `FilingDocumentsSection` | `filingId, regulatorId, docRequirements, documents, progress` | ⚠️ Uses `filingId` in upload |

### 6.2 Service Request Components
**Path:** `apps/app/features/regulators/components/service-requests/`

| Component | Status |
|-----------|--------|
| `ServiceRequestsContent` | ✅ Working |
| `FeaturedServicesGrid` | ✅ Working |
| `ServiceCatalogModal` | ✅ Working |
| `ServiceRequestCard` | ✅ Working |

---

## 7. Gap Summary

### Must Implement

1. **Backend: `getServiceRequestView`** - Aggregated endpoint returning:
   - Full task objects (not just summary)
   - Document requirements from template with satisfaction status
   - Progress computation
   - Blockers computation

2. **Schema: `intakeData` column** - Store intake form responses on service_requests

3. **Frontend: `useServiceRequestView` hook** - Query the aggregated endpoint

4. **UI: Intake Form Modal** - Dynamic form based on template's `intakeFieldsSchema`

5. **UI: Generalize Filing Components** - Accept `entityType`/`entityId` instead of `filingId`

6. **UI: Upgrade SR Detail Page** - Use generalized components, show inline tasks/docs

### Nice to Have (Not Required for This Step)

- Request submission flow (explicitly excluded)
- Payment handling (display-only OK)
- Timeline/audit log display

---

## 8. Risks & Considerations

### 8.1 Component Generalization
The filing components use `filingId` in:
- Query cache keys
- Upload completion calls
- Query invalidation

**Mitigation:** Introduce `entityType` and `entityId` props, update all usages.

### 8.2 Intake Data Schema Validation
Templates define `intakeFieldsSchema` with validation rules, but there's no runtime Zod schema generation.

**Mitigation:** Build Zod schema dynamically from `intakeFieldsSchema` or validate server-side only.

### 8.3 Doc Requirement Satisfaction
Documents need `requirementKey` to match template's `docRequirementConfigs.key`.

**Mitigation:** Pass `requirementKey` through upload flow (already supported).

---

## 9. Files to Create/Modify

### New Files
- `docs/regulators/pacra/step2-service-requests-scan.md` ← This file
- `apps/app/features/compliance/components/entity-tasks-section.tsx`
- `apps/app/features/compliance/components/entity-documents-section.tsx`
- `apps/app/features/compliance/components/blockers-panel.tsx`
- `apps/app/features/service-requests/components/intake-form.tsx`
- `apps/app/features/service-requests/components/intake-modal.tsx`

### Modified Files
- `packages/database/src/schema/compliance/service-requests.ts` (add intakeData)
- `packages/api-services/src/domains/compliance/service-requests.service.ts`
- `packages/api-services/src/domains/compliance/service-requests.schema.ts`
- `packages/backend/src/modules/compliance/service-requests/routes.ts`
- `packages/backend/src/modules/compliance/service-requests/handlers.ts`
- `apps/app/lib/queries/regulators/hooks/use-service-requests.ts`
- `apps/app/lib/queries/regulators/fetchers/service-requests.ts`
- `apps/app/lib/queries/regulators/types.ts`
- `apps/app/app/.../pacra/service-requests/[id]/page.tsx`
