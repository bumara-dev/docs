---
title: "PACRA Name Clearance - Specification"
description: "Implementation specification for PACRA Name Clearance workflow."
---

## Overview

Name Clearance is the first step in registering a business or company in Zambia. This feature provides a dedicated workflow page that integrates with the service request system for proper task tracking and progress updates.

## Routes

| Route | Description |
|-------|-------------|
| `/regulators/pacra/services/name-clearance` | Dedicated Name Clearance workflow page |
| `/regulators/pacra/service-requests/:id` | Service Request detail page (after submission) |

### Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `srId` | UUID (optional) | Link to existing service request to continue |

## API Endpoints Used

### Name Check (PACRA Registry Lookup)
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

### Create Name Clearance Application
```
POST /pacra/name-clearance
```

**Request:**
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

### Create Service Request
```
POST /service-requests
```

**Request:**
```json
{
  "templateId": "uuid-of-PACRA_NAME_CLEARANCE_V1-template",
  "name": "Name Clearance: My Company Limited"
}
```

## Database Schema

### name_clearance_applications (Updated)

```sql
CREATE TABLE name_clearance_applications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id TEXT NOT NULL REFERENCES organizations(id),
  service_request_id UUID REFERENCES service_requests(id),  -- NEW
  business_type pacra_business_type NOT NULL,
  business_class VARCHAR NOT NULL,
  business_category VARCHAR NOT NULL,
  application_type application_type NOT NULL,
  proposed_name_1 VARCHAR(100) NOT NULL,
  proposed_name_2 VARCHAR(100),
  proposed_name_3 VARCHAR(100),
  pacra_status regulator_status DEFAULT 'pending' NOT NULL,
  rejection_reason TEXT,
  fee_paid DECIMAL(10,2) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  deleted_at TIMESTAMP
);
```

### Service Request Linkage

The `service_request_id` FK links the name clearance application to the service request workflow, enabling:
- Progress tracking in the "Your Requests" list
- Task auto-completion when form data is submitted
- Unified timeline and activity logging

## Task Auto-Completion

When a name clearance application is created with a linked service request, the following tasks are auto-completed:

| Task Template Key | Auto-Complete Trigger |
|-------------------|----------------------|
| `review_name_choices` | When form data is submitted |
| `confirm_entity_type` | User must manually confirm |

## Wizard Flow

### Step 1: Business Details
- Business Type (local_company, foreign_company, business_name)
- Business Category (limited_by_shares, limited_by_guarantee, unlimited)
- Business Class (ordinary_company, local_bank, etc.)
- Application Type (new, amendment, ceasation)
- Proposed Name 1 (required)
- Proposed Name 2 (optional)
- Proposed Name 3 (optional)

### Step 2: Name Review
- Real-time availability check against PACRA registry
- Shows availability status for each proposed name
- Displays similar registered names if unavailable
- User can refresh individual name checks

### Step 3: Confirm & Submit
- Summary of all entered data
- Fee calculation based on business type
- What happens next explanation
- Submit creates:
  1. Service Request (if not existing)
  2. Name Clearance Application (linked to SR)
  3. Auto-completes "review_name_choices" task
- Redirects to Service Request detail page

## Progress Computation

Progress is computed in `getServiceRequestView` service:

```typescript
const progress = {
  tasksDone: requestTasks.filter(t => t.status === "done").length,
  tasksTotal: requestTasks.filter(t => t.required).length || requestTasks.length,
  docsDone: requirementsWithStatus.filter(r => r.satisfied && r.required).length,
  docsTotal: requirementsWithStatus.filter(r => r.required).length
};
```

For Name Clearance:
- `tasksTotal`: 2 (review_name_choices, confirm_entity_type)
- `tasksDone`: 1 after submission (auto-completed task)
- `docsDone/docsTotal`: 0/0 (no document requirements)

## Fee Calculation

| Business Type | Fee (ZMW) |
|---------------|-----------|
| Local Company | 120.20 |
| Foreign Company | 266.67 |
| Business Name | 111.20 |

## Files Created/Modified

### New Files
- `packages/database/drizzle/0026_add_service_request_id_to_name_clearance.sql`
- `apps/app/lib/queries/pacra/fetchers/name-clearance.ts`
- `apps/app/lib/queries/pacra/hooks/use-name-clearance.ts`
- `apps/app/features/pacra/components/name-clearance/` (5 files)
- `apps/app/app/(authenticated)/(general)/regulators/pacra/services/name-clearance/page.tsx`

### Modified Files
- `packages/database/src/schema/pacra/name-clearance.ts` - Added serviceRequestId FK
- `packages/api-services/src/domains/pacra/name-clearance.schema.ts` - Added serviceRequestId
- `packages/api-services/src/domains/pacra/name-clearance.service.ts` - SR linkage + task auto-completion
- `apps/app/features/regulators/components/service-requests/service-requests-content.tsx` - Navigation routing
- `apps/app/features/regulators/components/service-requests/service-catalog-modal.tsx` - Navigation routing

## Replicating for Other PACRA Services

To add another dedicated service page (e.g., Name Reservation):

1. **Create dedicated page**:
   - `apps/app/app/(authenticated)/(general)/regulators/pacra/services/name-reservation/page.tsx`

2. **Create wizard components**:
   - `apps/app/features/pacra/components/name-reservation/`

3. **Add route mapping**:
   ```typescript
   // In service-requests-content.tsx and service-catalog-modal.tsx
   const DEDICATED_SERVICE_ROUTES = {
     PACRA_NAME_CLEARANCE_V1: "/regulators/pacra/services/name-clearance",
     PACRA_NAME_RESERVATION_V1: "/regulators/pacra/services/name-reservation",
   };
   ```

4. **Link to existing DB table** (if applicable):
   - Add `service_request_id` FK to the domain table
   - Update service to auto-complete relevant tasks

## Manual QA Steps

1. Navigate to PACRA → Service Requests
2. Click "Name Search & Clearance" in Popular Services
3. Verify redirect to `/regulators/pacra/services/name-clearance`
4. Fill Step 1: Business Details + Proposed Names
5. Click "Check Name Availability"
6. Verify Step 2: Name check results displayed
7. Click "Continue to Submit"
8. Verify Step 3: Summary displayed with correct fee
9. Click "Submit Application"
10. Verify:
    - Toast success message
    - Redirect to Service Request detail page
    - "review_name_choices" task shows as Done
    - Progress shows 1/2 tasks complete
    - Service Request appears in "Your Requests" list
11. Refresh page - verify data persists
