---
title: "PACRA Timeline - Codebase Scan Results"
description: "Scan Date: January 2026 Purpose: Document existing event/audit infrastructure before implementing PACRA Timeline"
---

## 0.1 Existing Event Storage

### Primary Table: `audit_logs`

**Location:** `packages/database/src/schema/system/audit-logs.ts`

```typescript
export const auditLogs = pgTable("audit_logs", {
  id: uuid("id").primaryKey().defaultRandom(),
  organizationId: text("organization_id"),
  userId: text("user_id"),
  actorType: text("actor_type"), // "STAFF" | "TENANT" | "SYSTEM"
  ipAddress: text("ip_address"),
  userAgent: text("user_agent"),
  action: text("action").notNull(),
  entityType: text("entity_type").notNull(),
  entityId: text("entity_id"),
  changes: jsonb("changes"), // { before?: Record, after?: Record }
  metadata: jsonb("metadata"),
  timestamp: timestamp("timestamp").defaultNow().notNull(),
});
```

**Indexes:**
- `(organizationId, timestamp)` - for org-scoped timeline queries
- `(entityType, entityId)` - for entity-specific lookups
- `(userId, timestamp)` - for user activity
- `(actorType, timestamp)` - for actor type filtering

**Assessment:**
- ✅ Supports org-scoped queries
- ✅ Has entityType/entityId for linking to entities
- ✅ Has metadata JSONB for flexible data
- ⚠️ No native `regulatorId` column - relies on metadata or entity lookups

### Secondary Table: `timeline_events` (UNUSED)

**Location:** `packages/database/src/schema/compliance/timeline-events.ts`

This table exists but is **not currently used**. The timeline service queries `audit_logs` instead.

---

## 0.2 Existing Event Emission Points

### ✅ PACRA Connect (`pacra-connect.service.ts`)

**File:** `packages/api-services/src/domains/pacra/pacra-connect.service.ts`

Events emitted:
| Action | EntityType | Metadata |
|--------|------------|----------|
| `create` | `pacra_profile` | `{ entityType, entityName }` |
| `create` | `regulator_connection` | `{ regulatorKey: "pacra", entityType }` |

### ✅ Activation Engine (`activation.service.ts`)

**File:** `packages/api-services/src/domains/activation/activation.service.ts`

Events emitted:
| Action | EntityType | Metadata |
|--------|------------|----------|
| `create` | `org_obligation` | `{ templateKey, name }` |
| `create` | `filing` | `{ periodKey, dueOn }` |
| `create` | `regulator_activation` | `{ regulatorKey, activatedObligations[], availableServices[] }` |

**Gap:** Missing `regulatorId` in obligation/filing metadata.

### ✅ Task Service (`tasks.service.ts`)

**File:** `packages/api-services/src/domains/tasks/tasks.service.ts`

Events emitted:
| Action | EntityType | Metadata |
|--------|------------|----------|
| `create` | `task` | `{ title, taskType, required }` |
| `update` | `task` | `{ reason }` (status changes) |

**Gap:** Missing `regulatorId` and `filingId`/`serviceRequestId` in metadata.

### ✅ Document Service (`documents.service.ts`)

**File:** `packages/api-services/src/domains/compliance/documents.service.ts`

Events emitted:
| Action | EntityType | Metadata |
|--------|------------|----------|
| `create` | `document` | `{ filename, kind, filingId, serviceRequestId }` |
| `update` | `document` | `{ filingId, serviceRequestId }` (attachments) |

**Gap:** Missing `regulatorId` in metadata.

### ✅ Service Request Service (`service-requests.service.ts`)

**File:** `packages/api-services/src/domains/compliance/service-requests.service.ts`

Events emitted:
| Action | EntityType | Metadata |
|--------|------------|----------|
| `create` | `service_request` | `{ templateKey, name }` |

**Gap:** Missing `regulatorId` in metadata.

### ✅ Filing Service (`filings.service.ts`)

**File:** `packages/api-services/src/domains/compliance/filings.service.ts`

Events emitted:
| Action | EntityType | Metadata |
|--------|------------|----------|
| `update` | `filing` | Status changes with before/after |

**Gap:** Missing `regulatorId` in metadata.

---

## 0.3 Existing Timeline Infrastructure

### Timeline Service

**File:** `packages/api-services/src/domains/compliance/timeline.service.ts`

**Current implementation:**
- Queries `audit_logs` table (not `timeline_events`)
- Supports org-scoped queries
- Has regulator filtering via:
  1. Lookup regulator by `regulatorKey`
  2. Find all filings/service requests for that regulator
  3. Filter audit logs by entity IDs OR metadata regulatorId
- Generates human-readable descriptions from action/entityType
- Returns paginated results with actor info

**Regulator filtering logic (complex):**
```typescript
// Gets filing IDs and service request IDs for regulator
// Then filters: entityId IN [...ids] OR metadata->>'regulatorId' = regulatorId
```

### Timeline Schema

**File:** `packages/api-services/src/domains/compliance/timeline.schema.ts`

```typescript
export const listTimelineParamsSchema = z.object({
  regulatorId: z.string().uuid().optional(),
  regulatorKey: z.string().optional(),
  entityType: z.string().optional(),
  entityId: z.string().optional(),
  filingId: z.string().uuid().optional(),
  serviceRequestId: z.string().uuid().optional(),
  limit: z.coerce.number().int().min(1).max(100).default(50),
  offset: z.coerce.number().int().min(0).default(0),
});
```

### Timeline API Route

**File:** `packages/backend/src/modules/compliance/timeline/routes.ts`

- Route: `GET /timeline`
- Auth: `requireAuth`, `requireOrg`
- Query params: `regulatorId`, `regulatorKey`, `entityType`, `entityId`, `filingId`, `serviceRequestId`, `limit`, `offset`

### Frontend Hooks & Fetchers

**Hook:** `apps/app/lib/queries/regulators/hooks/use-timeline.ts`
```typescript
export function useTimelineEvents(params: ListTimelineParams = {}) {
  return useQuery({
    queryKey: ["timeline", params],
    queryFn: () => fetchTimelineEvents(getToken, params),
  });
}
```

**Fetcher:** `apps/app/lib/queries/regulators/fetchers/timeline.ts`
- Calls `client.timeline.$get()`
- **Gap:** Does not pass `regulatorKey` param, only `regulatorId`

### PACRA Timeline Page

**File:** `apps/app/app/(authenticated)/(general)/regulators/pacra/timeline/page.tsx`

**Current features:**
- Uses `RegulatorWorkspaceLayout`
- Fetches timeline events with `regulatorId` filter
- Displays events in a vertical timeline with icons
- Shows entity type badge, timestamp, actor info
- Shows status change details (before → after)

**Gaps:**
- No date grouping (Today/Yesterday/etc.)
- No filter tabs (All/Filings/Tasks/Documents)
- No clickable links to entities
- No pagination UI (load more)
- Uses `regulatorId` but not `regulatorKey`

---

## Summary of Gaps

| Area | Gap | Priority |
|------|-----|----------|
| Metadata | Missing `regulatorId` in task/document/SR audit events | High |
| Filtering | Complex entity lookup for regulator filtering | Medium |
| Fetcher | Missing `regulatorKey` param in frontend fetcher | Medium |
| UI | Missing date grouping | Medium |
| UI | Missing filter tabs | Medium |
| UI | Missing entity links | Medium |
| UI | Missing pagination controls | Low |

---

## Recommended Approach

1. **Enrich metadata** - Add `regulatorId` to all compliance audit events
2. **Optimize filtering** - Primary filter via `metadata->>'regulatorId'`
3. **Fix fetcher** - Pass `regulatorKey` parameter
4. **Enhance UI** - Add grouping, filters, links
5. **Document** - Create implementation docs

No new tables needed - `audit_logs` is sufficient.
