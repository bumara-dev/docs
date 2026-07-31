---
title: "PACRA Company Registration — Implementation Specification"
description: "Version: 1.0.0 Last Updated: 2026-01-12"
---

## Overview

This document specifies the implementation of the PACRA Company Registration feature in the Bumara tenant application. The feature provides a dedicated multi-step wizard for registering new companies with PACRA (Patents and Companies Registration Agency).

---

## Routes

| Route | Description |
|-------|-------------|
| `/regulators/pacra/services/company-registration` | Dedicated wizard page |
| `/regulators/pacra/services/company-registration?srId=...` | Resume existing service request |
| `/regulators/pacra/services/company-registration?regId=...` | Resume existing registration |
| `/regulators/pacra/service-requests/{id}` | Service request detail page |

---

## Wizard Steps

| Step | Component | Backend Endpoint | Description |
|------|-----------|------------------|-------------|
| 1 | `StepCompanyType` | `POST /company-registration` | Select company type and enter reserved name |
| 2 | `StepCompanyDetails` | `POST /company-registration/{id}/details` | Company name, activity, address, share capital |
| 3 | `StepDirectors` | `POST /company-registration/{id}/directors` | Add directors (min 1 required) |
| 4 | `StepShareholders` | `POST /company-registration/{id}/shareholders` | Add shareholders with share allocation |
| 5 | `StepBeneficialOwners` | `POST /company-registration/{id}/beneficial-owners` | Declare beneficial owners (25%+ ownership) |
| 6 | `StepGuarantors` | `POST /company-registration/{id}/guarantors` | Add guarantors (only for companies limited by guarantee) |
| 7 | `StepSecretary` | `POST /company-registration/{id}/secretaries` | Company secretary (optional) |
| 8 | `StepDeclaration` | `POST /company-registration/{id}/declaration` | Sign compliance declaration |
| 9 | `StepDocuments` | N/A | Document upload guidance (via SR documents) |
| 10 | `StepReview` | `PUT /company-registration/{id}` | Review and submit |

### Step Visibility

- Steps 1-5, 7-10 are always shown
- Step 6 (Guarantors) is only shown for company type `private_limited_guarantee`

### Company Types

| Value | Label |
|-------|-------|
| `private_limited_shares` | Private Company Limited by Shares |
| `private_limited_guarantee` | Private Company Limited by Guarantee |
| `public_limited` | Public Limited Company |

---

## Endpoint Contracts

### Backend Endpoints Used

All endpoints are in `packages/backend/src/modules/pacra/company-registration/`:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/company-registration` | POST | Initialize new registration |
| `/company-registration` | GET | List registrations |
| `/company-registration/{id}` | GET | Get single registration |
| `/company-registration/{id}` | PUT | Update status/step |
| `/company-registration/{id}` | DELETE | Delete registration |
| `/company-registration/{id}/details` | POST/PUT/GET | Company details |
| `/company-registration/{id}/directors` | POST/GET | Directors CRUD |
| `/company-registration/{id}/directors/{directorId}` | PUT/DELETE | Director updates |
| `/company-registration/{id}/shareholders` | POST/GET | Shareholders CRUD |
| `/company-registration/{id}/shareholders/{shareholderId}` | PUT/DELETE | Shareholder updates |
| `/company-registration/{id}/beneficial-owners` | POST/GET | Beneficial owners |
| `/company-registration/{id}/guarantors` | POST/GET | Guarantors |
| `/company-registration/{id}/secretaries` | POST/GET | Company secretaries |
| `/company-registration/{id}/declaration` | POST/PUT/GET | Compliance declaration |

### Service Request Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/service-requests` | POST | Create new SR from template |
| `/service-requests/{id}/view` | GET | Get SR with tasks/docs/progress |

---

## Service Request Linkage

### SR Creation Strategy

1. User starts wizard at Step 1
2. On Step 1 completion, create `company_registration` record
3. On final submit (Step 10), create `service_request` linked to template `PACRA_COMPANY_REGISTRATION_V1`
4. Store `companyRegistrationId` in SR metadata (via name field)
5. Navigate to SR detail page on success

### Resume Strategy

- `?srId=...` - Loads existing service request (future: fetch linked registration)
- `?regId=...` - Directly loads company registration data
- Local draft store - Persists form state to localStorage (7 day expiry)

---

## Persistence

### Draft Persistence (Client-Side)

**Store:** `apps/app/lib/stores/company-registration-draft-store.ts`

```typescript
interface CompanyRegistrationDraft {
  formData: Partial<CompanyRegistrationFormData>;
  currentStep: WizardStep;
  lastUpdated: number;
  templateId?: string;
  registrationId?: string;
  serviceRequestId?: string;
}
```

**Behavior:**
- Auto-saves on every form data change
- Prompts user to resume on page load if draft exists
- Clears on successful submission
- Expires after 7 days

### Backend Persistence

**Database Tables:**

| Table | Description |
|-------|-------------|
| `company_registration` | Main registration record |
| `company_details` | Step 2 data |
| `directors` | Step 3 data |
| `shareholders` | Step 4 data |
| `beneficial_owners` | Step 5 data |
| `guarantors` | Step 6 data |
| `company_secretaries` | Step 7 data |
| `compliance_declarations` | Step 8 data |

**Status Flow:**
```
draft → pending → submitted → approved
                           → rejected
```

---

## Tasks & Document Requirements

### Template Tasks

| Task Key | Title | Type | Required |
|----------|-------|------|----------|
| `provide_company_details` | Provide Company Details | fill_form | Yes |
| `add_directors` | Add Directors | fill_form | Yes |
| `add_shareholders` | Add Shareholders | fill_form | Yes |
| `add_beneficial_owners` | Declare Beneficial Owners | fill_form | Yes |
| `add_secretary` | Add Company Secretary | fill_form | No |
| `upload_documents` | Upload Required Documents | upload_document | Yes |
| `sign_declaration` | Sign Compliance Declaration | review_approve | Yes |
| `confirm_submission` | Review & Confirm Submission | review_approve | Yes |

### Document Requirements

| Key | Name | Required |
|-----|------|----------|
| `director_nrc_copies` | Director NRC/ID Copies | Yes |
| `shareholder_nrc_copies` | Shareholder NRC/ID Copies | Yes |
| `director_consent_forms` | Director Consent Forms | Yes |
| `registered_office_proof` | Proof of Registered Office | Yes |
| `beneficial_owner_declaration` | Beneficial Ownership Declaration | No |
| `corporate_shareholder_docs` | Corporate Shareholder Documents | No |

### Progress Computation

Progress is computed server-side via `GET /service-requests/{id}/view`:

```typescript
{
  progress: {
    tasksDone: number;
    tasksTotal: number;
    docsDone: number;
    docsTotal: number;
  }
}
```

---

## UI Components

### File Structure

```
apps/app/features/pacra/components/company-registration/
├── company-registration-wizard.tsx  # Main orchestrator
├── types.ts                          # Type definitions
├── step-company-type.tsx            # Step 1
├── step-company-details.tsx         # Step 2
├── step-directors.tsx               # Step 3
├── step-shareholders.tsx            # Step 4
├── step-beneficial-owners.tsx       # Step 5
├── step-guarantors.tsx              # Step 6
├── step-secretary.tsx               # Step 7
├── step-declaration.tsx             # Step 8
├── step-documents.tsx               # Step 9
├── step-review.tsx                  # Step 10
└── index.ts                          # Exports
```

### Page Location

```
apps/app/app/(authenticated)/(general)/regulators/pacra/services/company-registration/page.tsx
```

---

## API Hooks

### Location

- **Fetchers:** `apps/app/lib/queries/pacra/fetchers/company-registration.ts`
- **Hooks:** `apps/app/lib/queries/pacra/hooks/use-company-registration.ts`

### Available Hooks

| Hook | Purpose |
|------|---------|
| `useCompanyRegistrations()` | List registrations |
| `useCompanyRegistration(id)` | Get single registration |
| `useCompanyRegistrationView(id)` | Get full registration with all related data |
| `useCreateCompanyRegistration()` | Initialize new registration |
| `useUpdateCompanyRegistration()` | Update status/step |
| `useDeleteCompanyRegistration()` | Delete registration |
| `useCreateCompanyDetails()` | Save company details (Step 2) |
| `useUpdateCompanyDetails()` | Update company details |
| `useCreateDirectors()` | Add directors (Step 3) |
| `useUpdateDirector()` | Update director |
| `useDeleteDirector()` | Remove director |
| `useCreateShareholders()` | Add shareholders (Step 4) |
| `useUpdateShareholder()` | Update shareholder |
| `useDeleteShareholder()` | Remove shareholder |
| `useCreateBeneficialOwners()` | Add beneficial owners (Step 5) |
| `useCreateGuarantors()` | Add guarantors (Step 6) |
| `useCreateCompanySecretaries()` | Add secretaries (Step 7) |
| `useCreateComplianceDeclaration()` | Save declaration (Step 8) |
| `useUpdateComplianceDeclaration()` | Update declaration |

---

## Extensibility

### Adding New Fields

1. Update Zod schema in `packages/api-services/src/domains/pacra/company-registration.schema.ts`
2. Update corresponding step component
3. Update types in `apps/app/features/pacra/components/company-registration/types.ts`

### Adding New Document Requirements

1. Add to `docRequirementConfigs` in `packages/database/src/seeds/pacra-templates.ts`
2. Re-seed the database
3. Document list in `step-documents.tsx` updates automatically via SR view

### Future: Request Submission Handoff

When ready to implement backoffice submission:

1. Create `submission_jobs` record from SR detail page
2. Create `payment_requests` record if regulator payment required
3. Backoffice picks up job and processes
4. Update SR status on completion

---

## Files Changed/Created

### New Files

| Path | Description |
|------|-------------|
| `docs/regulators/pacra/company-registration/scan.md` | Codebase scan findings |
| `docs/regulators/pacra/company-registration/spec.md` | This specification |
| `apps/app/lib/queries/pacra/fetchers/company-registration.ts` | API fetchers |
| `apps/app/lib/queries/pacra/hooks/use-company-registration.ts` | React Query hooks |
| `apps/app/lib/stores/company-registration-draft-store.ts` | Zustand draft store |
| `apps/app/features/pacra/components/company-registration/types.ts` | Type definitions |
| `apps/app/features/pacra/components/company-registration/step-*.tsx` | 10 step components |
| `apps/app/features/pacra/components/company-registration/company-registration-wizard.tsx` | Main wizard |
| `apps/app/features/pacra/components/company-registration/index.ts` | Exports |
| `apps/app/app/(authenticated)/(general)/regulators/pacra/services/company-registration/page.tsx` | Dedicated page |

### Modified Files

| Path | Change |
|------|--------|
| `packages/database/src/seeds/pacra-templates.ts` | Added `PACRA_COMPANY_REGISTRATION_V1` template |
| `apps/app/features/regulators/components/service-requests/service-requests-content.tsx` | Added to `DEDICATED_SERVICE_ROUTES` |

---

## Manual QA Steps

### Happy Path Test

1. Navigate to `/regulators/pacra/service-requests`
2. Click "New Request" → "Company Registration" (or navigate directly)
3. **Step 1:** Select "Private Company Limited by Shares", enter company name
4. **Step 2:** Fill company details (activity, address, share capital)
5. **Step 3:** Add at least one director
6. **Step 4:** Add shareholders with share allocation
7. **Step 5:** Declare at least one beneficial owner
8. **Step 6:** (Skipped for shares companies)
9. **Step 7:** Optionally add company secretary
10. **Step 8:** Sign compliance declaration
11. **Step 9:** Review document requirements
12. **Step 10:** Review all data and submit
13. Verify redirect to service request detail page
14. Verify SR appears in list with correct status

### Draft Resume Test

1. Start registration, complete steps 1-3
2. Close browser tab
3. Navigate back to company registration page
4. Verify prompt to resume draft
5. Click "Resume Draft"
6. Verify form data and step restored

### Guarantee Company Test

1. Start new registration
2. Select "Private Company Limited by Guarantee"
3. Verify Step 6 (Guarantors) appears in wizard
4. Complete all steps with guarantors instead of shareholders
5. Submit and verify

---

## Known Limitations

1. **Document Upload:** Actual upload is handled via SR documents section, not in wizard
2. **Payment:** Display-only; actual payment flow not implemented
3. **SR Linkage:** Registration ID stored in SR name field (not dedicated column)
4. **Async Processing:** Backend endpoints are synchronous; no polling needed
