---
title: "PACRA Name Services - Codebase Scan"
description: "Scan document for PACRA Name Search, Name Clearance, and Name Reservation implementation."
---

## 0.1 Backend Endpoints

### Name Search (Check Name Availability)

**File:** `packages/backend/src/modules/pacra/name-clearance/name-clearance.routes.ts`

| Route | Method | Description |
|-------|--------|-------------|
| `/pacra/check-name?name={name}` | GET | Check name availability against PACRA registry |

**Input Schema:**
```typescript
z.object({
  name: z.string().min(1, 'Business name is required'),
})
```

**Output Schema:**
```typescript
{
  success: true,
  data: {
    available: boolean,
    suggestions: string[], // Similar names (up to 5)
    pacraCount: number     // Total matches found
  }
}
```

**Notes:**
- Synchronous operation - calls PACRA API directly
- No auth required for PACRA endpoint (public registry)
- Response not persisted (transient lookup)

### Name Clearance

**File:** `packages/backend/src/modules/pacra/name-clearance/name-clearance.index.ts`

| Route | Method | Description |
|-------|--------|-------------|
| `/pacra/name-clearance` | GET | List all name clearance applications |
| `/pacra/name-clearance` | POST | Create name clearance application |
| `/pacra/name-clearance/{id}` | GET | Get single application by ID |
| `/pacra/name-clearance/{id}` | PUT | Update application data |
| `/pacra/name-clearance/{id}/status` | PUT | Update application status |
| `/pacra/name-clearance/{id}` | DELETE | Delete application |

**Create Input Schema (`packages/api-services/src/domains/pacra/name-clearance.schema.ts`):**
```typescript
z.object({
  businessType: z.enum(['local_company', 'foreign_company', 'business_name']),
  businessClass: z.string(),
  businessCategory: z.string(),
  applicationType: z.enum(['new', 'amendment', 'ceasation']),
  proposedName1: z.string().min(2),
  proposedName2: z.string().optional().nullable(),
  proposedName3: z.string().optional().nullable(),
  serviceRequestId: z.string().uuid().optional().nullable(), // Links to SR
})
```

**Service Features (`packages/api-services/src/domains/pacra/name-clearance.service.ts`):**
- Auto-completes `submit_application_form` task when created with serviceRequestId
- Auto-transitions SR to `ready_for_submission` when all tasks done
- Full audit logging

### Name Reservation

**File:** `packages/backend/src/modules/pacra/name-reservation/name-reservation.index.ts`

| Route | Method | Description |
|-------|--------|-------------|
| `/pacra/name-reservation` | GET | List all name reservation applications |
| `/pacra/name-reservation` | POST | Create name reservation application |
| `/pacra/name-reservation/{id}` | GET | Get single application by ID |
| `/pacra/name-reservation/{id}/status` | PUT | Update application status |
| `/pacra/name-reservation/{id}` | DELETE | Delete application |

**Create Input Schema (`packages/api-services/src/domains/pacra/name-reservation.schema.ts`):**
```typescript
z.object({
  nameClearanceId: z.uuid().optional(),
  reservedName: z.string().min(1).max(100),
})
```

**GAP:** No `serviceRequestId` field - not linked to service request workflow!

**Service Features (`packages/api-services/src/domains/pacra/name-reservation.service.ts`):**
- Generates reservation number (`RES-{timestamp}-{random}`)
- Sets 90-day expiry when approved
- Basic audit logging
- **MISSING:** No task auto-completion, no SR linkage

---

## 0.2 Existing PACRA SR Templates

**File:** `packages/database/src/seeds/pacra-templates.ts`

### PACRA_NAME_CLEARANCE_V1 (EXISTS)

```typescript
{
  templateKey: "PACRA_NAME_CLEARANCE_V1",
  name: "Name Clearance",
  description: "Search and reserve a business or company name with PACRA.",
  regulator: "pacra",
  expectedDueInDays: "3",
  defaultPriority: "normal",
  
  intakeFieldsSchema: [
    { key: "proposedName1", label: "First Choice Name", type: "text", required: true },
    { key: "proposedName2", label: "Second Choice Name", type: "text", required: false },
    { key: "proposedName3", label: "Third Choice Name", type: "text", required: false },
    { key: "entityTypeForRegistration", label: "Type of Entity", type: "select", required: true },
    { key: "reasonForName", label: "Reason/Meaning of Name", type: "textarea", required: false },
  ],
  
  taskTemplateConfigs: [
    {
      key: "submit_application_form",
      title: "Submit Application Form",
      taskType: "fill_form",
      required: true,
      isBlocking: true,
      sequence: 1,
    },
  ],
  
  docRequirementConfigs: [], // No docs required
  
  paymentRuleConfig: {
    paymentRequired: true,
    feeKey: "PACRA_NAME_CLEARANCE",
    serviceFee: 5000, // ZMW 50.00
  },
}
```

### PACRA_NAME_RESERVATION_V1 (DOES NOT EXIST)

**GAP:** Template referenced in UI config (`apps/app/features/regulators/config.ts`) but not defined in seeds!

```typescript
// Referenced in PACRA_SERVICES array:
{
  templateKey: "PACRA_NAME_RESERVATION_V1",
  name: "Name Reservation",
  shortDescription: "Reserve an approved name for 90 days",
  icon: FileSignature,
  // ...
}
```

---

## 0.3 Tenant UI Current State

### PACRA Service Requests List

**File:** `apps/app/app/(authenticated)/(general)/regulators/pacra/service-requests/page.tsx`

- Uses `ServiceRequestsContent` component
- Shows service catalog with popular services
- Filters: status, due, search

### Dedicated Service Routes

**File:** `apps/app/features/regulators/components/service-requests/service-requests-content.tsx`

```typescript
const DEDICATED_SERVICE_ROUTES: Record<string, string> = {
  PACRA_NAME_CLEARANCE_V1: "/regulators/pacra/services/name-clearance",
  // PACRA_NAME_RESERVATION_V1: "/regulators/pacra/services/name-reservation", // Commented out!
};
```

### Name Clearance Page (EXISTS)

**Route:** `/regulators/pacra/services/name-clearance`
**Files:**
- `apps/app/app/(authenticated)/(general)/regulators/pacra/services/name-clearance/page.tsx`
- `apps/app/features/pacra/components/name-clearance/` (5 files)

Features:
- 3-step wizard (Business Details → Name Review → Confirm)
- Real-time PACRA registry lookup
- Creates SR on submit
- Auto-completes task
- Draft persistence via Zustand store

### Name Reservation Page (DOES NOT EXIST)

**GAP:** No dedicated page. Route is commented out in `DEDICATED_SERVICE_ROUTES`.

### Service Request Detail

**File:** `apps/app/app/(authenticated)/(general)/regulators/pacra/service-requests/[id]/page.tsx`

Shows:
- SR header with status/progress
- Tasks section (inline completion)
- Documents section
- Blockers panel
- Edit modal for name clearance applications

---

## 0.4 Progress Model

### Progress Computation (Server-Side)

**File:** `packages/api-services/src/domains/compliance/service-requests.service.ts`

Progress is computed in `getServiceRequestView`:

```typescript
const progress = {
  tasksDone: requestTasks.filter(t => t.status === "done" || t.status === "skipped").length,
  tasksTotal: requestTasks.filter(t => t.required).length || requestTasks.length,
  docsDone: requirementsWithStatus.filter(r => r.satisfied && r.required).length,
  docsTotal: requirementsWithStatus.filter(r => r.required).length,
};
```

### Task Update Flow

**Hook:** `apps/app/lib/queries/tasks/hooks/use-tasks.ts`

```typescript
export function useUpdateTaskStatus() {
  // Calls PATCH /tasks/:id/status
  // Invalidates task lists and filing queries on success
}
```

### Auto-Completion (Service Layer)

**Function:** `autoCompleteTaskByTemplateKey` in `name-clearance.service.ts`

- Finds task by template key + SR ID
- Sets status to 'done', records completedBy
- Triggers `maybeAutoTransitionServiceRequest`

---

## Summary of Gaps

| Component | Name Clearance | Name Reservation |
|-----------|----------------|------------------|
| Template | PACRA_NAME_CLEARANCE_V1 | MISSING |
| DB Schema | Has `service_request_id` | MISSING `service_request_id` |
| Service SR Linkage | Yes | No |
| Task Auto-Complete | Yes | No |
| Dedicated Route | Yes | Commented out |
| Wizard Page | Yes | MISSING |
| Hooks | Yes | MISSING |
| Fetchers | Yes | MISSING |

---

## Implementation Required

1. **Add PACRA_NAME_RESERVATION_V1 template** to `pacra-templates.ts`
2. **Add service_request_id FK** to `name_reservation_applications` table
3. **Update name-reservation.service.ts** with SR linkage + task auto-complete
4. **Create hooks/fetchers** in `apps/app/lib/queries/pacra/`
5. **Create dedicated page** at `/regulators/pacra/services/name-reservation`
6. **Uncomment route** in `DEDICATED_SERVICE_ROUTES`
