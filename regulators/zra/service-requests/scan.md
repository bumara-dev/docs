---
title: "ZRA Service Requests - Codebase Scan"
description: "Codebase scan of existing PACRA service request patterns and ZRA infrastructure to reuse when building ZRA service requests."
---

## 0.1 Existing PACRA Service Request Implementation

### Service Template Schema

**File:** `packages/database/src/schema/compliance/service-templates.ts`

```typescript
interface ServiceTemplate {
  id: uuid
  organizationId: text | null  // NULL = global template
  regulatorId: uuid
  regulator: enum('pacra' | 'zra' | 'napsa' | 'other')
  templateKey: text (unique)
  templateVersion: integer
  name: text
  description: text
  whatIsThis: text
  whyItMatters: text
  consequencesOfDelay: text
  intakeFieldsSchema: jsonb<IntakeFieldSchema[]>
  expectedDueInDays: text
  defaultPriority: enum
  activationRules: jsonb
  taskTemplateConfigs: jsonb<TaskTemplateConfig[]>
  docRequirementConfigs: jsonb<DocRequirementConfig[]>
  paymentRuleConfig: jsonb
  billingTag: enum('included' | 'overage')
  timestamps
}
```

### Service Request Schema

**File:** `packages/database/src/schema/compliance/service-requests.ts`

```typescript
interface ServiceRequest {
  id: uuid
  organizationId: text (notNull, FK)
  regulatorId: uuid (FK)
  templateId: uuid (FK to service_templates)
  regulator: enum
  name: text
  description: text
  status: enum (pending_data, in_progress, ready_for_submission, ...)
  billingTag: enum
  dueOn: timestamp
  submittedToBackofficeBy: text (FK)
  submittedToBackofficeAt: timestamp
  submittedToBackofficeAgentId: uuid (FK)
  regulatorReferenceNumber: text
  slaDeadline: timestamp
  timestamps
}
```

**Status Values:**
- `pending_data`, `in_progress`, `ready_for_submission`
- `submission_in_progress`, `submitted`, `accepted`
- `needs_correction`, `waived`, `cancelled`

### Service Requests API Service

**File:** `packages/api-services/src/domains/compliance/service-requests.service.ts`

Key functions:
- `listServiceRequests()` - paginated list with filters
- `getServiceRequest()` - single SR by ID
- `getServiceRequestView()` - comprehensive view with tasks/docs/requirements
- `listServiceTemplates()` - catalog for a regulator
- `createServiceRequest()` - create from template, generates tasks + doc requirements

### Service Requests Schema (Zod)

**File:** `packages/api-services/src/domains/compliance/service-requests.schema.ts`

Exports:
- `listServiceRequestsParamsSchema`
- `createServiceRequestInputSchema`
- `serviceRequestResponseSchema`
- `serviceRequestViewResponseSchema`
- `docRequirementWithStatusSchema`
- `serviceRequestViewTaskSchema`

---

## Creation Flow

### 1. Template-Based Creation

**Flow:**
1. User selects service from catalog
2. Call `createServiceRequest({ templateId, name?, description?, intakeData? })`
3. Service function:
   - Fetches template
   - Creates service_request record
   - Generates tasks from `taskTemplateConfigs`
   - Generates doc requirements from `docRequirementConfigs`
   - Records audit log

### 2. Dedicated Pages Pattern

**PACRA Services with Dedicated Pages:**
| Template Key | Route |
|---|---|
| `PACRA_NAME_CLEARANCE_V1` | `/regulators/pacra/services/name-clearance` |
| `PACRA_NAME_RESERVATION_V1` | `/regulators/pacra/services/name-reservation` |
| `PACRA_COMPANY_REGISTRATION_V1` | `/regulators/pacra/services/company-registration` |
| `PACRA_BUSINESS_NAME_REGISTRATION_V1` | `/regulators/pacra/services/business-name-registration` |

**Dedicated Page Structure:**
```
apps/app/app/(authenticated)/(general)/regulators/pacra/services/
├── name-clearance/page.tsx
├── name-reservation/page.tsx
├── company-registration/page.tsx
└── business-name-registration/page.tsx
```

**Dedicated Page Pattern (Example: name-clearance):**
- Loads templates, finds matching template by `templateKey`
- Renders wizard component with `templateId` and optional `srId` from URL
- Wizard manages multi-step form state
- Creates SR on first submit via `createServiceRequest({ templateId, name })`
- Custom domain logic via specialized endpoints (e.g., `pacra.name-clearance.$post`)

### 3. Resume Behavior (`srId`)

Pages accept `?srId=...` query param to resume existing service request:
```typescript
const existingServiceRequestId = searchParams.get("srId") ?? undefined;
// Passed to wizard as prop
```

### 4. Catalog + "Your Requests" List

**Component:** `ServiceRequestsContent`

Features:
- Featured services grid (from config)
- Full catalog modal (`ServiceCatalogModal`)
- Filtered requests list
- Support for dedicated routes via `DEDICATED_SERVICE_ROUTES` map

**Route Mapping:**
```typescript
const DEDICATED_SERVICE_ROUTES: Record<string, string> = {
  PACRA_NAME_CLEARANCE_V1: "/regulators/pacra/services/name-clearance",
  // ...
};
```

When clicking service:
1. Check if `templateKey` has dedicated route
2. If yes → `router.push(dedicatedRoute)`
3. If no → Open intake modal

---

## 0.2 Existing ZRA API Endpoints

### ZRA Connect Service

**File:** `packages/api-services/src/domains/zra/zra-connect.service.ts`

- `connectZraWithActivation()` - connects org to ZRA and activates selected obligations
- Uses activation engine to generate filings from templates

### ZRA Obligation Templates (Seeded)

**File:** `packages/database/src/seeds/zra-templates.ts`

Templates:
- `ZRA_PAYE_MONTHLY_V1` - PAYE monthly filing
- `ZRA_TURNOVER_TAX_MONTHLY_V1` - Turnover tax monthly
- `ZRA_WHT_MONTHLY_V1` - Withholding tax monthly

**Service Templates:** Empty (`services: []`)

### ZRA UI Routes

**Directory:** `apps/app/app/(authenticated)/(general)/regulators/zra/`
- `page.tsx` - ZRA overview
- `filings/` - Filings list and detail
- `obligations/` - Obligations list and detail
- `tasks/` - Tasks page

**Missing:** No `/regulators/zra/services/` routes exist yet.

### ZRA Config

**File:** `apps/app/features/regulators/config.ts`

```typescript
const ZRA_SERVICES: ServiceDisplayConfig[] = [
  // ZRA services will be added when templates are created
];

export const ZRA_CONFIG: RegulatorConfig = {
  key: "zra",
  basePath: "/regulators/zra",
  services: ZRA_SERVICES,  // Empty
  // ... other config
};
```

---

## 0.3 Tenant UI Patterns to Reuse

### Shared Components

| Component | Path | Usage |
|---|---|---|
| `RegulatorWorkspaceLayout` | `features/regulators/components/workspace-layout.tsx` | Page wrapper with sidebar |
| `ServiceRequestsContent` | `features/regulators/components/service-requests/service-requests-content.tsx` | Main list + catalog |
| `ServiceCatalogModal` | `features/regulators/components/service-requests/service-catalog-modal.tsx` | Full catalog modal |
| `FeaturedServicesSection` | `features/regulators/components/service-requests/featured-services-grid.tsx` | Quick start cards |
| `IntakeModal` | `features/service-requests/components/` | Generic intake form modal |
| `ServiceRequestCard` | `features/regulators/components/service-requests/service-request-card.tsx` | SR list item |

### Wizard Pattern

**Example:** `NameClearanceWizard`

Features:
- Multi-step with progress
- Draft persistence (Zustand + localStorage)
- Resume dialog
- Creates SR on first submit
- Mobile-friendly step navigation

### Hooks

- `useServiceTemplates(regulatorId)` - fetch available templates
- `useServiceRequests(params)` - fetch SR list
- `useServiceRequestView(srId)` - fetch SR detail with tasks/docs
- `useCreateServiceRequest()` - create SR mutation

---

## Summary

### What Exists
1. ✅ Full `service_templates` and `service_requests` schema
2. ✅ API service with CRUD + view operations
3. ✅ PACRA dedicated page pattern working
4. ✅ Catalog + list UI components
5. ✅ Template seeding infrastructure
6. ✅ ZRA obligation templates (PAYE, ToT, WHT)
7. ✅ ZRA connect service with activation engine

### What's Missing for ZRA Service Requests
1. ❌ ZRA service templates (need to create seed data)
2. ❌ ZRA service display config (empty `ZRA_SERVICES` array)
3. ❌ ZRA service-requests page (`/regulators/zra/service-requests`)
4. ❌ ZRA dedicated service pages (e.g., `/regulators/zra/services/tax-clearance`)
5. ❌ ZRA-specific intake forms/wizards
6. ❌ Submission request integration for ZRA services

### Implementation Path
1. Create ZRA service template seeds
2. Add ZRA service display configs
3. Create ZRA service-requests page (reuse existing components)
4. Create first dedicated page (Tax Clearance Certificate)
5. Wire up DEDICATED_SERVICE_ROUTES for ZRA
6. Test end-to-end flow
