---
title: "PACRA Workspace Documentation"
description: "Tenant-facing UI for managing PACRA compliance"
---

This document describes the PACRA workspace implementation, routes, components, and patterns used.

---

## Overview

The PACRA workspace provides tenants with a complete interface for managing their PACRA (Patents and Companies Registration Agency) compliance obligations. It follows a reusable architecture that can be extended to other regulators (ZRA, NAPSA, NHIMA).

---

## Routes

| Route | Page | Description |
|-------|------|-------------|
| `/regulators/pacra` | Overview | Dashboard with compliance summary and quick actions |
| `/regulators/pacra/obligations` | Obligations List | Recurring compliance requirements |
| `/regulators/pacra/filings` | Filings List | Period-specific submissions |
| `/regulators/pacra/filings/[filingId]` | Filing Detail | Filing with tasks, docs, payments |
| `/regulators/pacra/service-requests` | Service Requests List | One-off workflows |
| `/regulators/pacra/service-requests/[id]` | Service Request Detail | Request with tasks, docs |
| `/regulators/pacra/tasks` | Tasks List | Task management |
| `/regulators/pacra/documents` | Documents List | Document management |
| `/regulators/pacra/timeline` | Timeline | Activity history |
| `/regulators/pacra/authorization` | Authorization | Representative management |
| `/regulators/pacra/payments` | Payments (Placeholder) | Coming soon |
| `/regulators/pacra/reports` | Reports (Placeholder) | Coming soon |

---

## API Endpoints

### Obligations

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/obligations` | List obligations with filters |
| GET | `/obligations/:id` | Get obligation detail with filings |
| GET | `/compliance/summary` | Get compliance summary stats |

### Filings

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/filings` | List filings with filters |
| GET | `/filings/:id` | Get filing detail |

### Service Requests

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/service-requests` | List service requests |
| GET | `/service-requests/:id` | Get service request detail |
| POST | `/service-requests` | Create new service request |
| GET | `/service-templates` | List available templates |

### Documents

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/documents` | List documents with filters |
| GET | `/documents/:id` | Get document |
| POST | `/documents` | Create document record |
| POST | `/documents/:id/attach` | Attach to filing/request |
| DELETE | `/documents/:id` | Delete document |

### Timeline

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/timeline` | List timeline events |

### Authorization

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/authorization/:regulatorId` | Get authorization |
| GET | `/authorization/by-key/:key` | Get by regulator key |
| POST | `/authorization/:regulatorId` | Create/update authorization |
| PATCH | `/authorization/:regulatorId/status` | Update status |
| DELETE | `/authorization/:regulatorId` | Delete authorization |

---

## Reusable Components

### Config System

**File:** `apps/app/features/regulators/config.ts`

Provides regulator configuration:

```typescript
import { PACRA_CONFIG, getRegulatorConfig } from "@/features/regulators/config";

// Get config by key
const config = getRegulatorConfig("pacra");

// Access sections
const sections = config.sections.filter(s => s.enabled);
```

### Workspace Layout

**File:** `apps/app/features/regulators/components/workspace-layout.tsx`

Wraps workspace pages with connection check:

```tsx
<RegulatorWorkspaceLayout
  regulatorKey="pacra"
  regulatorId={connection?.regulatorId}
>
  {/* Page content */}
</RegulatorWorkspaceLayout>
```

Shows `ConnectCta` component if not connected.

### Connect CTA

**File:** `apps/app/features/regulators/components/connect-cta.tsx`

Displayed when tenant hasn't connected to a regulator.

### Overview Stats

**File:** `apps/app/features/regulators/components/overview-stats.tsx`

Dashboard statistics cards showing:
- Active obligations count
- Open filings count
- Next due date
- Task progress

---

## Data Hooks

All hooks are in `apps/app/features/regulators/hooks/`:

| Hook | Purpose |
|------|---------|
| `useObligations(params)` | List obligations |
| `useObligation(id)` | Get single obligation |
| `useComplianceSummary(regulatorId)` | Get compliance stats |
| `useFilings(params)` | List filings |
| `useFiling(id)` | Get filing detail |
| `useServiceRequests(params)` | List service requests |
| `useServiceRequest(id)` | Get service request detail |
| `useServiceTemplates(regulatorId)` | List available templates |
| `useCreateServiceRequest()` | Create new request |
| `useDocuments(params)` | List documents |
| `useCreateDocument()` | Create document record |
| `useAttachDocument()` | Attach to entity |
| `useDeleteDocument()` | Delete document |
| `useTimelineEvents(params)` | List timeline events |
| `useAuthorizationByKey(key)` | Get authorization |
| `useUpsertAuthorization()` | Create/update authorization |

---

## Database Schema

### regulator_authorizations

New table for storing authorization representatives:

```sql
CREATE TABLE regulator_authorizations (
  id UUID PRIMARY KEY,
  organization_id TEXT NOT NULL,
  regulator_id UUID NOT NULL,
  representative_name TEXT NOT NULL,
  representative_role TEXT,
  representative_email TEXT,
  representative_phone TEXT,
  document_id UUID,
  valid_from TIMESTAMP,
  valid_until TIMESTAMP,
  status authorization_status DEFAULT 'pending',
  notes TEXT,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL,
  deleted_at TIMESTAMP
);
```

---

## Known Gaps (Future Work)

### 1. Request Submission Flow

The "Request Submission" button and backoffice workflow are **not implemented**. This requires:
- Creating `submission_jobs` record
- Creating `payment_requests` if payment required
- Creating `tickets` for tenant action items
- Integration with backoffice for assignment and processing

### 2. Payment Integration

Payments page shows placeholder. Full integration requires:
- Payment gateway (PSP) webhook handling
- `payment_requests` status management
- Receipt/evidence display
- Fee breakdown display

### 3. Document Upload

The Documents page has an Upload button, but the actual file upload to R2 is not wired up. Requires:
- Presigned URL generation endpoint
- Client-side upload to R2
- Document record creation after upload

### 4. Form-type Tasks

Tasks of type `fill_form` currently show "Not yet supported". Full implementation requires:
- Dynamic form rendering from template schema
- Form data storage and validation
- Integration with filing/request data

### 5. Real-time Updates

No WebSocket/polling for status changes. Users must refresh to see updates.

### 6. Compliance Health Score

The `complianceScore` field in summary returns `null`. Algorithm needs implementation.

---

## Extending to Other Regulators

The workspace is designed to be regulator-agnostic. To add a new regulator:

1. **Add config** in `apps/app/features/regulators/config.ts`:

```typescript
export const NEW_REGULATOR_CONFIG: RegulatorConfig = {
  key: "new-regulator",
  displayName: "New Regulator Name",
  shortName: "NR",
  // ... other config
};
```

2. **Create routes** in `apps/app/app/(authenticated)/(general)/regulators/new-regulator/`

3. **Update sidebar** in `apps/app/config/secondary-sidebar/regulator-sidebar.ts`

4. **Reuse components**:
   - `RegulatorWorkspaceLayout`
   - `OverviewStats`
   - All data hooks (pass `regulatorId` or `regulatorKey`)

---

## Testing Recommendations

### Manual QA Steps

1. **Overview Page**
   - Verify stats cards show correct counts
   - Check quick action links work
   - Verify connection state detection

2. **Obligations**
   - List obligations by status
   - Check next filing display
   - Verify "View Filings" navigation

3. **Filings**
   - Filter by status
   - Open filing detail
   - Check tasks summary
   - Verify document list

4. **Service Requests**
   - Create new request from catalog
   - View request detail
   - Check task progress

5. **Documents**
   - Filter by type
   - Filter by linked entity
   - Delete document

6. **Timeline**
   - Verify events display
   - Check status change formatting
   - Verify actor attribution

7. **Authorization**
   - Create new authorization
   - Edit existing authorization
   - Verify form validation

---

## File Structure

```
apps/app/
├── app/(authenticated)/(general)/regulators/pacra/
│   ├── layout.tsx
│   ├── page.tsx                    # Overview
│   ├── obligations/page.tsx
│   ├── filings/
│   │   ├── page.tsx
│   │   └── [filingId]/page.tsx
│   ├── service-requests/
│   │   ├── page.tsx
│   │   └── [id]/page.tsx
│   ├── tasks/page.tsx
│   ├── documents/page.tsx
│   ├── timeline/page.tsx
│   ├── authorization/page.tsx
│   ├── payments/page.tsx           # Placeholder
│   └── reports/page.tsx            # Placeholder
├── features/regulators/
│   ├── config.ts
│   ├── hooks/
│   │   ├── index.ts
│   │   ├── use-obligations.ts
│   │   ├── use-filings.ts
│   │   ├── use-service-requests.ts
│   │   ├── use-documents.ts
│   │   ├── use-timeline.ts
│   │   └── use-authorization.ts
│   └── components/
│       ├── index.ts
│       ├── workspace-layout.tsx
│       ├── connect-cta.tsx
│       └── overview-stats.tsx

packages/api-services/src/domains/compliance/
├── index.ts
├── obligations.schema.ts
├── obligations.service.ts
├── filings.schema.ts
├── filings.service.ts
├── service-requests.schema.ts
├── service-requests.service.ts
├── documents.schema.ts
├── documents.service.ts
├── timeline.schema.ts
├── timeline.service.ts
├── authorization.schema.ts
└── authorization.service.ts

packages/backend/src/modules/compliance/
├── index.ts
├── obligations/
├── filings/
├── service-requests/
├── documents/
├── timeline/
└── authorization/

packages/database/src/schema/compliance/
└── regulator-authorizations.ts
```

