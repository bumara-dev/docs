---
title: "PACRA Business Name Registration — Scan Document"
description: "This document captures scan findings for implementing the PACRA Business Name Registration dedicated service page in the tenant app."
---

## Overview
This document captures scan findings for implementing the PACRA Business Name Registration dedicated service page in the tenant app.

---

## 0.1 Backend Endpoints and Schemas

### Location
- **Routes**: `packages/backend/src/modules/pacra/business-name-registration/business-name-registration.routes.ts`
- **Handlers**: `packages/backend/src/modules/pacra/business-name-registration/business-name-registration.handlers.ts`
- **Schema**: `packages/api-services/src/domains/pacra/business-name-registration.schema.ts`
- **Service**: `packages/api-services/src/domains/pacra/business-name-registration.service.ts`

### RPC Routes (under `/pacra/business-name-registration`)
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/` | List all registrations for org |
| GET | `/:id` | Get registration by ID |
| GET | `/reference/:referenceNumber` | Get by reference number |
| POST | `/` | Create registration |
| PATCH | `/:id` | Update registration |
| PATCH | `/:id/status` | Update status |
| DELETE | `/:id` | Delete registration |
| GET | `/:registrationId/owners` | List business owners |
| GET | `/owners/:id` | Get owner by ID |
| POST | `/:registrationId/owners` | Create owner |
| PUT | `/owners/:id` | Update owner |
| DELETE | `/owners/:id` | Delete owner |

### Input Schema (`createBusinessNameRegistrationSchema`)
```typescript
{
  // Corporate applicant (optional)
  isCorporateBody: boolean,
  companyName: string?,
  companyNumber: string?,
  businessAddress: string?,
  postalAddress: string?,
  corporateEmail: string?,

  // Business details (required)
  approvedName: string?,
  reservedName: string,          // REQUIRED
  incorporationType: string,      // REQUIRED
  natureOfBusiness: string,       // REQUIRED

  // Premises
  plotNumber: string?,
  premisesOwnershipType: string?,
  premisesOwnerName: string?,

  // Address (required)
  physicalAddress: string,        // REQUIRED
  province: string,               // REQUIRED
  email: string,                  // REQUIRED
  contactNumber: string,          // REQUIRED

  // Commencement
  proposedCommencementDate: Date?,
  financialYearEnd: string?,

  // Prior history
  hadPreviousCertificate: boolean,
  previousCertificateDetails: string?,
  previousCertificateReason: string?,
  hasOtherBusiness: boolean,
  otherBusinessDetails: string?,
  hasOtherInterests: boolean,
  otherInterestsDetails: string?,

  // Declaration
  declarationConfirmed: boolean,

  // Owners (optional array for creation)
  owners: CreateBusinessOwnerInput[]?
}
```

### Owner Schema (`createBusinessOwnerSchema`)
```typescript
{
  fullName: string,
  nrcNumber: string,
  dob: Date,
  nationality: string,
  shareholding: number (0-100, 2 decimal places)
}
```

### Processing Mode
- **Synchronous**: Registration is created immediately with `referenceNumber` auto-generated (`BN-YYYY-NNN`)
- Backend persists to `business_name_registrations` table
- Status enum: `pending | in_review | approved | rejected | submitted`
- Audit logs are recorded via `recordAuditLog()`

---

## 0.2 Existing PACRA Templates

### Service Config Location
- **File**: `apps/app/features/regulators/config.ts`

### Template Definition
```typescript
{
  templateKey: "PACRA_BUSINESS_NAME_REGISTRATION_V1",
  name: "Business Name Registration",
  shortDescription: "Register a sole proprietorship or partnership",
  icon: Briefcase,
  color: "emerald",
  category: "registration",
  featured: false,
  estimatedDays: "3-5",
}
```

### Template System
- Templates are fetched via `useServiceTemplates(regulatorId)` hook
- Service requests are created via `useCreateServiceRequest()` hook with `{ templateId, name }`
- **No SR template table with tasks/docs definition found** — tasks/docs must be created when SR is created

### SR Creation Flow (from Company Registration pattern)
1. User starts wizard → backend creates registration record
2. User completes steps → data persisted to domain-specific tables
3. On final submit → `createServiceRequest.mutateAsync({ templateId, name })`
4. Redirect to SR detail page `/regulators/pacra/service-requests/{srId}`

---

## 0.3 Tenant UI Patterns

### Existing Service Pages
- `/regulators/pacra/services/company-registration` — `CompanyRegistrationWizard`
- `/regulators/pacra/services/name-clearance` — Name clearance page
- `/regulators/pacra/services/name-reservation` — Name reservation page

### Company Registration Wizard Pattern
```
apps/app/features/pacra/components/company-registration/
├── company-registration-wizard.tsx    # Main wizard component
├── step-company-type.tsx              # Step 1
├── step-company-details.tsx           # Step 2
├── ...                                # More steps
├── step-review.tsx                    # Final review
└── types.ts                           # TypeScript types
```

### Key Patterns Used
1. **Multi-step wizard** with progress bar
2. **Zustand draft store** for local persistence (`lib/stores/company-registration-draft-store.ts`)
3. **React Query hooks** for API calls (`lib/queries/pacra/hooks/use-company-registration.ts`)
4. **Resume via query params**: `?srId=...` or `?regId=...`
5. **Draft dialog** on mount if draft exists
6. **Service request creation** on final submit
7. **Error boundary** wrapping

---

## 0.4 Documents System

### Document Upload Pattern
- Documents attached to service requests via documents endpoints
- R2 storage for bytes, metadata in DB
- Signed URLs for download
- No explicit `case_document_requirements` table found
- Company Registration uses a `StepDocuments` component

### Current State
- Documents can be uploaded and attached to SRs
- **No template-driven doc requirements** discovered
- Doc satisfaction would need manual tracking if implemented

---

## 0.5 Progress + Tasks

### Progress Computation
- No centralized `computeEntityProgress()` helper found
- SR view includes tasks and can compute progress client-side
- Company registration tracks `step` number in registration record

### Task System
- Tasks are associated with service requests
- Standard task states: `TODO | DOING | BLOCKED | DONE | SKIPPED`
- **No auto-completion based on form steps** in company registration
- Tasks primarily managed through backoffice

---

## Summary: What Exists vs. What's Needed

| Component | Exists | Needs Implementation |
|-----------|--------|---------------------|
| Backend CRUD endpoints | ✅ Complete | — |
| Database schema | ✅ Complete | — |
| API schemas/types | ✅ Complete | — |
| Audit logging | ✅ In service | — |
| Frontend fetchers | ❌ Missing | Create matching hooks/fetchers |
| Zustand draft store | ❌ Missing | Create for business name reg |
| Wizard component | ❌ Missing | Create 5-step wizard |
| Page route | ❌ Missing | Create `/services/business-name-registration` |
| Service request linkage | ❌ Missing | Add SR creation on submit |
| Tasks/doc requirements | ⚠️ No template system | Opt to skip or implement minimal |
| Progress tracking | ⚠️ Server-side step only | Use step for progress |

---

## Recommended Approach

1. **Reuse existing patterns** from company-registration
2. **Create simpler wizard** (4-5 steps):
   - Step 1: Business Type (individual vs corporate)
   - Step 2: Business Details (name, nature, address)
   - Step 3: Owner(s) Details
   - Step 4: Documents Upload
   - Step 5: Review & Submit
3. **Use existing backend endpoints** directly
4. **Create SR on final submit** using `PACRA_BUSINESS_NAME_REGISTRATION_V1` template
5. **Skip template-driven tasks** for MVP — let backoffice manage tasks manually
