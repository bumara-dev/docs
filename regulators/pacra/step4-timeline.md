---
title: "PACRA Timeline - Implementation Documentation"
description: "Implemented: January 2026 Purpose: Tenant-facing activity feed for PACRA compliance events"
---

## Overview

The PACRA Timeline provides tenants with a chronological view of all compliance-related activity for their organization's PACRA connection. Events are sourced from the existing `audit_logs` table, enriched with regulator context, and displayed in a filterable, grouped timeline UI.

---

## Architecture

### Event Model

Events are stored in the existing `audit_logs` table:

```sql
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY,
  organization_id TEXT REFERENCES organizations(id),
  user_id TEXT,
  actor_type TEXT, -- 'STAFF' | 'TENANT' | 'SYSTEM'
  ip_address TEXT,
  user_agent TEXT,
  action TEXT NOT NULL, -- 'create' | 'update' | 'delete'
  entity_type TEXT NOT NULL,
  entity_id TEXT,
  changes JSONB, -- { before?: Record, after?: Record }
  metadata JSONB, -- Contains regulatorId for filtering
  timestamp TIMESTAMP DEFAULT NOW()
);
```

**Key indexes:**
- `(organization_id, timestamp)` - For org-scoped timeline queries
- `(entity_type, entity_id)` - For entity-specific lookups

### Metadata Enrichment

All compliance-related audit events now include `regulatorId` in metadata for efficient filtering:

```typescript
await recordAuditLog(ctx, deps, {
  action: "create",
  entityType: "filing",
  entityId: filing.id,
  changes: { ... },
  metadata: {
    regulatorId,      // Primary filter key
    filingId,         // For linking related events
    serviceRequestId, // For service request events
    title,            // For display
    filename,         // For document events
  },
});
```

---

## Events Emitted

### Entity Types

| Entity Type | Description |
|-------------|-------------|
| `regulator_connection` | PACRA connected to organization |
| `regulator_activation` | Templates activated |
| `org_obligation` | Obligation instance created |
| `filing` | Filing created or status changed |
| `service_request` | Service request created or status changed |
| `task` | Task created, completed, blocked, etc. |
| `document` | Document uploaded, attached, or deleted |
| `pacra_profile` | PACRA profile created/updated |

### Event Actions

| Action | When Emitted |
|--------|--------------|
| `create` | Entity created |
| `update` | Entity modified (status change, assignment, etc.) |
| `delete` | Entity deleted (soft delete) |

### Emission Points

| File | Events |
|------|--------|
| `pacra-connect.service.ts` | `regulator_connection`, `pacra_profile` |
| `activation.service.ts` | `regulator_activation`, `org_obligation`, `filing` |
| `tasks.service.ts` | `task` (create, status update, assign, skip, reopen) |
| `documents.service.ts` | `document` (create, attach, delete) |
| `service-requests.service.ts` | `service_request` |
| `filings.service.ts` | `filing` (status updates) |

---

## API Endpoint

### GET /timeline

Lists timeline events with optional filters.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `regulatorId` | UUID | Filter by regulator ID |
| `regulatorKey` | string | Filter by regulator code (e.g., "pacra") |
| `entityType` | string | Filter by entity type |
| `entityId` | string | Filter by specific entity |
| `filingId` | UUID | Filter events related to a filing |
| `serviceRequestId` | UUID | Filter events related to a service request |
| `limit` | number | Max events to return (default: 50, max: 100) |
| `offset` | number | Pagination offset (default: 0) |

**Response:**

```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "organizationId": "org_xxx",
      "userId": "user_xxx",
      "actorType": "TENANT",
      "action": "update",
      "entityType": "task",
      "entityId": "uuid",
      "changes": {
        "before": { "status": "todo" },
        "after": { "status": "done" }
      },
      "metadata": {
        "regulatorId": "uuid",
        "filingId": "uuid",
        "title": "Upload Annual Returns"
      },
      "timestamp": "2026-01-08T12:00:00.000Z",
      "description": "Completed task \"Upload Annual Returns\"",
      "actor": {
        "id": "user_xxx",
        "name": null,
        "type": "TENANT"
      }
    }
  ],
  "pagination": {
    "limit": 50,
    "offset": 0,
    "total": 123
  }
}
```

---

## UI Components

### Route

`/regulators/pacra/timeline`

### RegulatorTimeline Component

Reusable component for any regulator:

```tsx
<RegulatorTimeline
  regulatorKey="pacra"
  regulatorId={pacraConnection?.regulatorId}
/>
```

**Location:** `apps/app/features/regulators/components/timeline/regulator-timeline.tsx`

### Features

1. **Date Grouping**
   - Today
   - Yesterday
   - This Week
   - Specific dates for older events

2. **Filter Tabs**
   - All
   - Filings
   - Service Requests
   - Tasks
   - Documents

3. **Entity Links**
   - Filing events link to `/regulators/pacra/filings/{id}`
   - Service request events link to `/regulators/pacra/service-requests/{id}`
   - Task/document events link to their parent filing or service request

4. **Visual Indicators**
   - Color-coded icons by entity type
   - Status change badges (before → after)
   - Relative timestamps
   - Actor information

5. **Pagination**
   - Initial load: 50 events
   - "Load more" button for additional events
   - Total count display

### States

- **Loading:** Skeleton placeholders
- **Empty:** Contextual message with icon
- **Error:** Error message with retry button
- **Filtered empty:** Message suggesting to clear filter

---

## Generalization for Other Regulators

The implementation is designed to support any regulator:

1. **Pass `regulatorKey` prop:**
   ```tsx
   <RegulatorTimeline regulatorKey="zra" regulatorId={zraRegulatorId} />
   ```

2. **Events automatically filtered** by `metadata.regulatorId` or entity relationships

3. **Links dynamically generated** based on `regulatorKey`:
   ```
   /regulators/{regulatorKey}/filings/{id}
   /regulators/{regulatorKey}/service-requests/{id}
   ```

4. **No PACRA-specific hardcoding** in the timeline component

### Future Regulators

When implementing ZRA/NAPSA/NHIMA timelines:

1. Add audit metadata enrichment to their service files
2. Create route: `/regulators/{key}/timeline/page.tsx`
3. Use `<RegulatorTimeline regulatorKey="{key}" />`

---

## Files Changed

### New Files

| File | Purpose |
|------|---------|
| `docs/regulators/pacra/step4-timeline-scan.md` | Codebase scan results |
| `docs/regulators/pacra/step4-timeline.md` | This documentation |
| `apps/app/features/regulators/components/timeline/regulator-timeline.tsx` | Reusable timeline component |
| `apps/app/features/regulators/components/timeline/index.ts` | Export barrel |

### Modified Files

| File | Change |
|------|--------|
| `packages/api-services/src/domains/activation/activation.service.ts` | Add regulatorId to metadata |
| `packages/api-services/src/domains/tasks/tasks.service.ts` | Add regulatorId/filingId to metadata |
| `packages/api-services/src/domains/compliance/documents.service.ts` | Add regulatorId to metadata |
| `packages/api-services/src/domains/compliance/service-requests.service.ts` | Add regulatorId to metadata |
| `packages/api-services/src/domains/compliance/filings.service.ts` | Add regulatorId to metadata |
| `packages/api-services/src/domains/compliance/timeline.service.ts` | Improved descriptions |
| `apps/app/lib/queries/regulators/fetchers/timeline.ts` | Add regulatorKey param |
| `apps/app/app/(authenticated)/(general)/regulators/pacra/timeline/page.tsx` | Use RegulatorTimeline component |

---

## Testing

### Manual QA Steps

1. **Connect PACRA**
   - Navigate to PACRA workspace
   - Complete connection flow
   - Open Timeline → Verify "Connected to regulator" and obligation activation events

2. **Complete a Task**
   - Open a filing
   - Complete a task
   - Open Timeline → Verify "Completed task" event appears

3. **Upload a Document**
   - Upload a document to a filing
   - Open Timeline → Verify "Uploaded document" event with filename

4. **Create Service Request**
   - Create a new service request
   - Open Timeline → Verify "Started service request" event

5. **Filter Events**
   - Use filter tabs to filter by type
   - Verify correct events shown for each filter

6. **Load More**
   - If >50 events exist, click "Load more"
   - Verify additional events load

7. **Entity Links**
   - Click on a filing event
   - Verify navigation to filing detail page

---

## Acceptance Criteria Verification

| Criteria | Status |
|----------|--------|
| PACRA Timeline page exists and loads org-scoped PACRA events | ✅ |
| Connecting PACRA results in timeline entries | ✅ |
| Completing a task creates a timeline event | ✅ |
| Uploading/attaching a document creates a timeline event | ✅ |
| Creating a service request creates a timeline event | ✅ |
| Pagination works and UI remains responsive | ✅ |
| No Authorization gating introduced | ✅ |
