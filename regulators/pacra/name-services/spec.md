---
title: "PACRA Name Services - Specification"
description: "Implementation specification for PACRA Name Search, Name Clearance, and Name Reservation workflows."
---

## Overview

Name Services are the first steps in registering a business or company in Zambia:

1. **Name Search/Clearance** - Check if proposed names are available and submit for clearance
2. **Name Reservation** - Reserve a cleared name for 90 days while preparing registration documents

Both services use dedicated workflow pages with service request tracking, task auto-completion, and progress updates.

---

## Routes

| Route | Description |
|-------|-------------|
| `/regulators/pacra/services/name-clearance` | Name Search/Clearance wizard page |
| `/regulators/pacra/services/name-reservation` | Name Reservation wizard page |
| `/regulators/pacra/service-requests/:id` | Generic SR detail page (fallback) |

### Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `srId` | UUID (optional) | Link to existing service request to continue |
| `name` | string (optional) | Prefill name (reservation page only) |
| `entityType` | string (optional) | Prefill entity type (reservation page only) |
| `clearanceId` | UUID (optional) | Link to prior name clearance (reservation page only) |

---

## API Endpoints Used

### Name Search (PACRA Registry Lookup)

```
GET /pacra/check-name?name={proposedName}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "available": true,
    "suggestions": ["Similar Name 1", "Similar Name 2"],
    "pacraCount": 5
  }
}
```

### Name Clearance CRUD

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/pacra/name-clearance` | GET | List all applications |
| `/pacra/name-clearance` | POST | Create application |
| `/pacra/name-clearance/{id}` | GET | Get single application |
| `/pacra/name-clearance/{id}` | PUT | Update application data |
| `/pacra/name-clearance/{id}/status` | PUT | Update status |
| `/pacra/name-clearance/{id}` | DELETE | Delete application |

**Create Request:**
```json
{
  "businessType": "local_company",
  "businessClass": "ordinary_company",
  "businessCategory": "limited_by_shares",
  "applicationType": "new",
  "proposedName1": "My Company Limited",
  "proposedName2": "Alternative Name Ltd",
  "proposedName3": null,
  "serviceRequestId": "uuid-of-service-request"
}
```

### Name Reservation CRUD

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/pacra/name-reservation` | GET | List all applications |
| `/pacra/name-reservation` | POST | Create application |
| `/pacra/name-reservation/{id}` | GET | Get single application |
| `/pacra/name-reservation/{id}/status` | PUT | Update status |
| `/pacra/name-reservation/{id}` | DELETE | Delete application |

**Create Request:**
```json
{
  "reservedName": "My Company Limited",
  "entityType": "company",
  "nameClearanceId": "uuid-of-clearance-application",
  "serviceRequestId": "uuid-of-service-request"
}
```

### Service Requests

```
POST /service-requests
```

**Request:**
```json
{
  "templateId": "uuid-of-template",
  "name": "Name Clearance: My Company Limited"
}
```

---

## Database Schema

### name_clearance_applications

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| org_id | TEXT | FK to organizations |
| service_request_id | UUID | FK to service_requests |
| business_type | ENUM | local_company, foreign_company, business_name |
| business_class | VARCHAR | Company class |
| business_category | VARCHAR | Limited by shares, etc. |
| application_type | ENUM | new, amendment, ceasation |
| proposed_name_1 | VARCHAR(100) | Primary proposed name |
| proposed_name_2 | VARCHAR(100) | Alternative name 2 |
| proposed_name_3 | VARCHAR(100) | Alternative name 3 |
| pacra_status | ENUM | pending, approved, rejected, etc. |
| rejection_reason | TEXT | Reason if rejected |
| fee_paid | DECIMAL(10,2) | PACRA fee amount |
| created_at | TIMESTAMP | Creation timestamp |
| updated_at | TIMESTAMP | Last update timestamp |

### name_reservation_applications

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| org_id | TEXT | FK to organizations |
| service_request_id | UUID | FK to service_requests |
| name_clearance_id | UUID | FK to name_clearance_applications |
| reserved_name | VARCHAR(100) | Name being reserved |
| reservation_number | VARCHAR(50) | System-generated reference |
| pacra_reservation_id | VARCHAR(100) | PACRA reference (from backoffice) |
| reservation_status | ENUM | pending, approved, rejected, etc. |
| reservation_date | DATE | When reservation was approved |
| expiry_date | DATE | 90 days after approval |
| fee_paid | DECIMAL(10,2) | Reservation fee |
| rejection_reason | TEXT | Reason if rejected |
| created_at | TIMESTAMP | Creation timestamp |
| updated_at | TIMESTAMP | Last update timestamp |

---

## Service Templates

### PACRA_NAME_CLEARANCE_V1

- **Task**: `submit_application_form` (fill_form, required, blocking)
- **Doc Requirements**: None
- **Payment**: PACRA_NAME_CLEARANCE fee + service fee

### PACRA_NAME_RESERVATION_V1

- **Task**: `submit_reservation_form` (fill_form, required, blocking)
- **Doc Requirements**: None
- **Payment**: PACRA_NAME_RESERVATION fee + service fee

---

## Service Request Linkage

Both applications include a `service_request_id` FK that links to the service request workflow:

```
service_requests (1) ─────> tasks (many)
        │
        ├──> name_clearance_applications (1)
        │
        └──> name_reservation_applications (1)
```

Benefits:
- Progress tracking in "Your Requests" list
- Task auto-completion when form data is submitted
- Unified timeline and activity logging
- Standard submission flow via backoffice

---

## Task Auto-Completion

When an application is created with a linked service request, the service layer automatically:

1. Finds the task by template key (`submit_application_form` or `submit_reservation_form`)
2. Sets task status to 'done' with completedAt timestamp
3. Records audit log for the task completion
4. Checks if all required tasks are done
5. Auto-transitions SR to `ready_for_submission` if ready

**Code Location:** `packages/api-services/src/domains/pacra/name-*.service.ts`

---

## Progress Computation

Progress is computed server-side in `getServiceRequestView`:

```typescript
const progress = {
  tasksDone: requestTasks.filter(t => t.status === "done" || t.status === "skipped").length,
  tasksTotal: requestTasks.filter(t => t.required).length || requestTasks.length,
  docsDone: requirementsWithStatus.filter(r => r.satisfied && r.required).length,
  docsTotal: requirementsWithStatus.filter(r => r.required).length,
};
```

For Name Clearance:
- `tasksTotal`: 1 (submit_application_form)
- `tasksDone`: 1 after submission (auto-completed)

For Name Reservation:
- `tasksTotal`: 1 (submit_reservation_form)
- `tasksDone`: 1 after submission (auto-completed)

---

## Fee Calculation

### Name Clearance

| Business Type | PACRA Fee (ZMW) |
|---------------|-----------------|
| Local Company | 120.20 |
| Foreign Company | 266.67 |
| Business Name | 111.20 |

### Name Reservation

| Entity Type | PACRA Fee (ZMW) |
|-------------|-----------------|
| Private Company | 218.20 |
| Public Company | 300.00 |
| Business Name | 150.00 |

Service fee (Bumara): Varies by template configuration.

---

## Wizard Flows

### Name Clearance (3 Steps)

1. **Business Details** - Type, category, class, application type, proposed names
2. **Name Review** - Real-time PACRA registry lookup for each name
3. **Confirm & Submit** - Review summary, fee breakdown, submit

### Name Reservation (2 Steps)

1. **Details** - Name to reserve, entity type, optional clearance reference
2. **Confirm & Submit** - Review summary, fee breakdown, estimated expiry

---

## Navigation Behavior

### From Service Catalog

When user selects "Name Search & Clearance" or "Name Reservation" from:
- Popular Services cards
- Service Catalog modal

They are redirected to the dedicated page instead of opening the generic intake modal.

**Implementation:** `DEDICATED_SERVICE_ROUTES` map in:
- `service-requests-content.tsx`
- `service-catalog-modal.tsx`

### From Service Requests List

When user clicks an existing SR card:
- If templateKey is in `DEDICATED_SERVICE_ROUTES`, routes to `/services/{key}?srId={id}`
- Otherwise, routes to generic `/service-requests/{id}` detail page

**Implementation:** `getServiceRequestRoute()` in `service-request-card.tsx`

---

## Files

### Backend

| File | Description |
|------|-------------|
| `packages/database/src/schema/pacra/name-clearance.ts` | DB schema |
| `packages/database/src/schema/pacra/name-reservation.ts` | DB schema |
| `packages/database/src/seeds/pacra-templates.ts` | Service templates |
| `packages/database/drizzle/0026_*.sql` | Name clearance migration |
| `packages/database/drizzle/0027_*.sql` | Name reservation migration |
| `packages/api-services/src/domains/pacra/name-clearance.*.ts` | Schema/service |
| `packages/api-services/src/domains/pacra/name-reservation.*.ts` | Schema/service |
| `packages/backend/src/modules/pacra/name-clearance/` | API routes/handlers |
| `packages/backend/src/modules/pacra/name-reservation/` | API routes/handlers |

### Tenant App

| File | Description |
|------|-------------|
| `apps/app/lib/queries/pacra/fetchers/name-clearance.ts` | API client |
| `apps/app/lib/queries/pacra/fetchers/name-reservation.ts` | API client |
| `apps/app/lib/queries/pacra/hooks/use-name-clearance.ts` | React Query hooks |
| `apps/app/lib/queries/pacra/hooks/use-name-reservation.ts` | React Query hooks |
| `apps/app/features/pacra/components/name-clearance/` | Wizard components |
| `apps/app/features/pacra/components/name-reservation/` | Wizard components |
| `apps/app/app/(authenticated)/(general)/regulators/pacra/services/name-clearance/page.tsx` | Page |
| `apps/app/app/(authenticated)/(general)/regulators/pacra/services/name-reservation/page.tsx` | Page |
| `apps/app/features/regulators/components/service-requests/service-request-card.tsx` | SR card with routing |
| `apps/app/features/regulators/components/service-requests/service-requests-content.tsx` | Service routing |
| `apps/app/features/regulators/components/service-requests/service-catalog-modal.tsx` | Catalog routing |

---

## Extending to Company Registration

To add another dedicated service (e.g., Company Registration):

1. **Create template** in `pacra-templates.ts`:
   - `PACRA_COMPANY_REGISTRATION_V1`
   - Define intake fields, tasks, doc requirements, payment rules

2. **Update DB schema** (if needed):
   - Add `service_request_id` FK to domain table
   - Create migration

3. **Update service layer**:
   - Add `serviceRequestId` to create schema
   - Implement `autoCompleteTaskByTemplateKey` for relevant tasks
   - Call `maybeAutoTransitionServiceRequest` after task completion

4. **Create hooks/fetchers**:
   - `apps/app/lib/queries/pacra/fetchers/company-registration.ts`
   - `apps/app/lib/queries/pacra/hooks/use-company-registration.ts`

5. **Create wizard page**:
   - `apps/app/features/pacra/components/company-registration/`
   - `apps/app/app/(authenticated)/(general)/regulators/pacra/services/company-registration/page.tsx`

6. **Add route mapping**:
   ```typescript
   const DEDICATED_SERVICE_ROUTES = {
     // ...existing
     PACRA_COMPANY_REGISTRATION_V1: "/regulators/pacra/services/company-registration",
   };
   ```

---

## Manual QA Steps

### Name Search/Clearance

1. Navigate to PACRA → Service Requests
2. Click "Name Search & Clearance" in Popular Services
3. Verify redirect to `/regulators/pacra/services/name-clearance`
4. Fill Step 1: Business Details + Proposed Names
5. Click "Continue" to Step 2
6. Verify name availability check results displayed
7. Click "Continue to Submit" to Step 3
8. Verify summary with correct fee breakdown
9. Click "Submit Application"
10. Verify:
    - Toast success message
    - Redirect to Service Request detail page
    - `submit_application_form` task shows as Done
    - Progress shows 1/1 tasks complete
    - SR appears in "Your Requests" list
11. Refresh page - verify data persists

### Name Reservation

1. Navigate to PACRA → Service Requests
2. Click "Name Reservation" in Popular Services
3. Verify redirect to `/regulators/pacra/services/name-reservation`
4. Fill Step 1: Name to Reserve + Entity Type
5. Click "Continue to Review" to Step 2
6. Verify summary with fee breakdown and expiry estimate
7. Click "Submit Reservation"
8. Verify:
    - Toast success message
    - Redirect to Service Request detail page
    - `submit_reservation_form` task shows as Done
    - Progress shows 1/1 tasks complete
    - SR appears in "Your Requests" list
9. Refresh page - verify data persists

### From Name Clearance to Reservation

1. Complete a Name Clearance (follow steps above)
2. From the SR detail page, navigate to Name Reservation
   - (Future: Add "Proceed to Reservation" CTA button)
3. Verify name can be prefilled via query params:
   - `/regulators/pacra/services/name-reservation?name=MyCompany&entityType=company`

### Service Request List Navigation

1. Navigate to PACRA → Service Requests
2. Find an existing Name Clearance SR
3. Click to open
4. Verify it routes to `/regulators/pacra/services/name-clearance?srId={id}`
5. Find an existing Name Reservation SR
6. Click to open
7. Verify it routes to `/regulators/pacra/services/name-reservation?srId={id}`
