---
title: "PACRA Company Registration — Codebase Scan Findings"
description: "Codebase scan of the infrastructure behind PACRA company registration: endpoints, templates, tenant UI, documents, and progress model."
---

## 0.1 Backend Endpoints and Schemas

### Existing RPC Routes

**Location:** `packages/backend/src/modules/pacra/company-registration/`

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/company-registration` | POST | Initialize new company registration |
| `/company-registration` | GET | List company registrations (with pagination, status filter) |
| `/company-registration/{id}` | GET | Get single registration |
| `/company-registration/{id}` | PUT | Update registration status/tracking number |
| `/company-registration/{id}` | DELETE | Delete registration |
| `/company-registration/{id}/details` | POST/PUT/GET | Company details (Step 1) |
| `/company-registration/{id}/directors` | POST/GET | Directors list (Step 2) |
| `/company-registration/{id}/directors/{directorId}` | PUT/DELETE | Update/delete director |
| `/company-registration/{id}/shareholders` | POST/GET | Shareholders list (Step 3) |
| `/company-registration/{id}/shareholders/{shareholderId}` | PUT/DELETE | Update/delete shareholder |
| `/company-registration/{id}/beneficial-owners` | POST/GET | Beneficial owners (Step 4) |
| `/company-registration/{id}/beneficial-owners/{ownerId}` | PUT/DELETE | Update/delete beneficial owner |
| `/company-registration/{id}/guarantors` | POST/GET/PUT/DELETE | Guarantors (Step 5) |
| `/company-registration/{id}/secretaries` | POST/GET/PUT/DELETE | Company secretaries (Step 6) |
| `/company-registration/{id}/declaration` | POST/GET/PUT | Compliance declaration (Step 7) |

### Input Schemas

**Location:** `packages/api-services/src/domains/pacra/company-registration.schema.ts`

Key schemas:
- `createCompanyDetailsSchema` — company name, type, activity, address, share capital, etc.
- `createDirectorSchema` — name, NRC/passport, nationality, position, appointment date
- `createShareholderSchema` — individual or corporate, shares, identity info
- `createBeneficialOwnerSchema` — beneficial ownership with corporate sub-entities support
- `createGuarantorSchema` — for companies limited by guarantee
- `createCompanySecretarySchema` — secretary details
- `createComplianceDeclarationSchema` — declarant info and signature

### Output/Response Structure

All endpoints return:
```typescript
{
  success: boolean;
  data: T;
  message?: string;
}
```

Registration object includes:
- `id`, `userId`, `orgId`
- `registrationNumber`, `registrationDate`
- `registrationStatus` (draft, pending, submitted, approved, rejected)
- `pacraTrackingNumber`
- `step` (1-7 indicating progress)
- `createdAt`, `updatedAt`

### Processing Model

- **Synchronous** — All endpoints respond immediately
- **Draft persistence** — Company registration record persists all step data in DB
- **Status tracking** — `registrationStatus` enum in `regulatorStatusEnum`

---

## 0.2 Existing PACRA Templates

### Location
`packages/database/src/seeds/pacra-templates.ts`

### Existing Service Templates
| Template Key | Name | Status |
|--------------|------|--------|
| `PACRA_NAME_CLEARANCE_V1` | Name Clearance | ✅ Implemented |
| `PACRA_NAME_RESERVATION_V1` | Name Reservation | ✅ Implemented |
| `PACRA_CHANGE_DIRECTORS_V1` | Change of Directors | Defined, not implemented |
| `PACRA_CHANGE_REGISTERED_OFFICE_V1` | Change of Registered Office | Defined |

### Missing Template
`PACRA_COMPANY_REGISTRATION_V1` — **NOT YET DEFINED** in seeds

### UI Config Reference
`apps/app/features/regulators/config.ts` already references:
```typescript
{
  templateKey: "PACRA_COMPANY_REGISTRATION_V1",
  name: "Company Registration",
  shortDescription: "Register a new private limited company",
  icon: Building,
  color: "emerald",
  category: "registration",
  featured: true,
  estimatedDays: "5-7",
}
```

### Template Structure Pattern
From existing templates:
```typescript
interface ServiceTemplateSeed {
  templateKey: string;
  templateVersion: number;
  name: string;
  description: string;
  regulator: "pacra";
  whatIsThis: string;
  whyItMatters: string;
  consequencesOfDelay: string;
  expectedDueInDays: string;
  defaultPriority: "low" | "normal" | "high" | "urgent" | "critical";
  intakeFieldsSchema: IntakeFieldSchema[];
  activationRules: ActivationRules;
  taskTemplateConfigs: TaskTemplateConfig[];
  docRequirementConfigs: DocRequirementConfig[];
  paymentRuleConfig: PaymentRuleConfig;
}
```

---

## 0.3 Tenant UI

### Dedicated Service Pages Pattern

**Route convention:** `/regulators/pacra/services/{service-name}/page.tsx`

**Existing implementations:**
- `apps/app/app/(authenticated)/(general)/regulators/pacra/services/name-clearance/page.tsx`
- `apps/app/app/(authenticated)/(general)/regulators/pacra/services/name-reservation/page.tsx`

### Wizard Component Pattern

**Location:** `apps/app/features/pacra/components/`

Structure for each dedicated service:
```
features/pacra/components/{service-name}/
├── {service-name}-wizard.tsx    # Main wizard component
├── types.ts                      # Form data types, step type
├── step-*.tsx                    # Individual step components
└── index.ts                      # Exports
```

### Form Library Conventions
- **React Hook Form** with Zod validation
- **Draft persistence** via Zustand stores (`lib/stores/`)
- **Service request creation** on submit using `useCreateServiceRequest`

### Service Catalog Routing

**Location:** `apps/app/features/regulators/components/service-requests/service-requests-content.tsx`

```typescript
const DEDICATED_SERVICE_ROUTES: Record<string, string> = {
  PACRA_NAME_CLEARANCE_V1: "/regulators/pacra/services/name-clearance",
  PACRA_NAME_RESERVATION_V1: "/regulators/pacra/services/name-reservation",
  // PACRA_COMPANY_REGISTRATION_V1: "/regulators/pacra/services/company-registration", // TO ADD
};
```

### Existing Form Components
- Address fields, NRC validation patterns in existing wizards
- Reusable field patterns for dates, names, identity documents

---

## 0.4 Documents System

### Upload Component
`apps/app/features/compliance/components/entity-documents-section.tsx`

### Document Types (DocumentKind)
- `source` — Input documents (NRCs, IDs)
- `workpaper` — Working documents
- `submission` — Screenshots, submitted forms
- `receipt` — Payment proof
- `certificate` — Approval letters

### Document Requirements Satisfaction
- Each template defines `docRequirementConfigs[]`
- When SR is created, requirements are instantiated
- Uploading a doc with matching `requirementKey` marks requirement as satisfied
- Progress computation includes `docsDone/docsTotal`

### Upload Flow
1. User selects file
2. Frontend gets presigned URL
3. Upload to R2
4. Call `useCompleteDocumentUpload` to register in DB
5. Requirement automatically satisfied if `requirementKey` matches

---

## 0.5 Progress Model

### Service Request View Endpoint
`GET /service-requests/{id}/view` returns:
```typescript
{
  serviceRequest: ServiceRequestResponse;
  tasks: ServiceRequestViewTask[];
  docRequirements: ServiceRequestViewDocRequirement[];
  documents: ServiceRequestViewDocument[];
  progress: {
    tasksDone: number;
    tasksTotal: number;
    docsDone: number;
    docsTotal: number;
  };
  blockers: {
    isReady: boolean;
    blockedTasks: Array<{...}>;
    pendingRequiredTasks: Array<{...}>;
    missingRequiredDocs: Array<{...}>;
  };
}
```

### Task Status Flow
```
TODO → DOING → DONE | SKIPPED
         ↓
      BLOCKED
```

### Progress Computation
- **Tasks:** Count of tasks with status `done` or `skipped` / total tasks
- **Docs:** Count of satisfied required doc requirements / total required docs
- Computed server-side in view endpoint

### Task Auto-Completion
- Tasks linked from template when SR is created
- Task payload can store references (e.g., `companyRegistrationId`)
- Manual task completion via task update endpoints

---

## 0.6 Database Schema

### Location
`packages/database/src/schema/pacra/company-registration/`

### Tables
| Table | Description |
|-------|-------------|
| `company_registration` | Main registration record |
| `company_details` | Step 1 data (name, type, address, capital) |
| `directors` | Director records |
| `shareholders` | Shareholder records |
| `beneficial_owners` | Beneficial ownership records |
| `guarantors` | Guarantor records |
| `company_secretaries` | Secretary records |
| `compliance_declarations` | Declaration records |
| `applicants` | Applicant info |

### Key Columns in company_registration
- `id` (UUID)
- `user_id`, `organization_id` — tenant isolation
- `registration_number`, `registration_date` — PACRA assigned
- `registration_status` — enum (draft, pending, submitted, etc.)
- `pacra_tracking_number` — PACRA reference
- `step` — current wizard step (1-7)

### Missing: Service Request Linkage
Currently no `service_request_id` column on `company_registration`.

Options:
1. Add column to table (migration required)
2. Store `companyRegistrationId` in SR metadata/task payload (preferred for MVP)

---

## Summary: What Needs Implementation

### Required New Files
1. Service template seed (`PACRA_COMPANY_REGISTRATION_V1`)
2. Frontend API hooks (`use-company-registration.ts`, `company-registration.ts` fetchers)
3. Draft store (`company-registration-draft-store.ts`)
4. Wizard page (`/regulators/pacra/services/company-registration/page.tsx`)
5. Wizard components (`features/pacra/components/company-registration/`)

### Required Modifications
1. `DEDICATED_SERVICE_ROUTES` — add company registration route
2. `features/pacra/components/index.ts` — export new components

### Backend Changes
**None required** — all endpoints already exist
