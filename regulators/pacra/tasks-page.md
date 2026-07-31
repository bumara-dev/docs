---
title: "Regulator Tasks Page"
description: "Tenant-facing tasks page for viewing and completing compliance tasks across filings and service requests."
---

## Overview

The tasks page provides tenants with a consolidated view of all tasks for a specific regulator (PACRA, ZRA, NAPSA, etc.). Tasks are generated from filings and service requests and must be completed to enable submission.

## Routes

| Route | Component | Description |
|-------|-----------|-------------|
| `/regulators/pacra/tasks` | `PacraTasksClient` | PACRA-specific tasks page |
| `/regulators/[key]/tasks` | `RegulatorTasksClient` | Generic tasks page (future) |

## Component Architecture

### RegulatorTasksClient

The main tasks page component is **regulator-agnostic** and accepts props:

```typescript
interface RegulatorTasksClientProps {
  /** Regulator key (e.g., "PACRA", "ZRA", "NAPSA") */
  regulatorKey: string;
  /** Display name for the regulator */
  regulatorName: string;
}
```

### PacraTasksClient

A backward-compatible wrapper that passes PACRA-specific props:

```typescript
export function PacraTasksClient() {
  return <RegulatorTasksClient regulatorKey="PACRA" regulatorName="PACRA" />;
}
```

## Endpoint Contract

### GET /tasks

List tasks with regulator filtering.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `regulatorKey` | string | Filter by regulator (e.g., "PACRA") |
| `status` | enum | Task status: todo, doing, blocked, done, skipped |
| `assignedToMe` | boolean | Filter tasks assigned to current user |
| `overdue` | boolean | Filter overdue tasks |
| `dueSoon` | boolean | Filter tasks due within 7 days |
| `sortBy` | enum | Sort field: dueOn, createdAt, updatedAt |
| `sortOrder` | enum | Sort direction: asc, desc |
| `limit` | number | Page size (default: 20, max: 100) |
| `offset` | number | Pagination offset |

**Response:**

```typescript
interface TaskListResponse {
  data: TaskResponse[];
  pagination: {
    limit: number;
    offset: number;
    total: number;
  };
}

interface TaskResponse {
  id: string;
  organizationId: string;
  filingId: string | null;
  serviceRequestId: string | null;
  title: string;
  description: string | null;
  taskType: TaskType;
  required: boolean;
  status: TaskStatus;
  blockedReason: string | null;
  skipReason: string | null;
  payload: Record<string, unknown> | null;
  dueOn: string | null;
  createdAt: string;
  updatedAt: string;
  // Enriched fields (populated when listing)
  parentLabel: string | null;  // e.g., "Annual Return FY 2025"
  regulatorKey: string | null; // e.g., "PACRA"
}
```

## Filters & Grouping

### Available Filters

| Filter | Options | Description |
|--------|---------|-------------|
| View | My Tasks, All Tasks, Blocked | Filter by assignment/status |
| Parent Type | All, Filings, Requests | Filter by source entity |
| Status | All, Todo, In Progress, Blocked, Done, Skipped | Filter by task status |
| Due Date | Any, Due Soon, Overdue | Filter by due date |
| Search | Free text | Search by title/description |

### Summary Stats

The page header displays real-time summary chips:

- **Open tasks: X** - Tasks in todo, doing, or blocked status
- **Overdue: Y** - Open tasks with due date in the past
- **Due soon: Z** - Open tasks due within 7 days

### Client-side Filtering

Some filters are applied client-side for responsiveness:

- Search (filters by title/description)
- Parent type (filings vs. service requests)

## Quick Completion Rules

Tasks can be completed from the tasks list page or via the detail modal.

### From List Page

- All tasks support status changes via the TaskDetailsModal
- Open task detail → click "Mark Complete" or "Start Progress"

### Task Type Behaviors

| Task Type | Completion Behavior |
|-----------|---------------------|
| `review_approve` | Simple confirmation (checkbox-style) |
| `info_request` | Simple confirmation |
| `fill_form` | Open detail page, fill form, submit |
| `upload_document` | Completed via document upload |
| `payment_action` | Completed via payment flow |
| `custom` | Manual completion |

## Navigation

### Deep Links to Parent Entities

Tasks link to their parent entity detail pages:

```typescript
// Filing task
/regulators/${regulatorKey}/filings/${filingId}

// Service request task
/regulators/${regulatorKey}/service-requests/${serviceRequestId}
```

### URL Helper

```typescript
function getEntityUrl(
  sourceType: "FILING" | "SERVICE_REQUEST",
  sourceId: string,
  regulatorKey: string
): string {
  const base = `/regulators/${regulatorKey.toLowerCase()}`;
  return sourceType === "FILING"
    ? `${base}/filings/${sourceId}`
    : `${base}/service-requests/${sourceId}`;
}
```

## Reusability for Other Regulators

### Adding a New Regulator Tasks Page

1. Create the route file:

```typescript
// apps/app/app/(authenticated)/(general)/regulators/zra/tasks/page.tsx
import { RegulatorTasksClient } from "@/features/pacra/tasks/pacra-tasks-client";

export default function ZRATasksPage() {
  return <RegulatorTasksClient regulatorKey="ZRA" regulatorName="ZRA" />;
}
```

2. The component handles:
   - Filtering tasks by regulator
   - Correct navigation links
   - Regulator-specific empty states

### Configuration Points

| Config | Location | Description |
|--------|----------|-------------|
| `regulatorKey` | Component prop | API filter key |
| `regulatorName` | Component prop | Display name in UI |
| Navigation paths | `task-details-modal.tsx` | Auto-generated from regulatorKey |

## Files Reference

### Frontend

| File | Purpose |
|------|---------|
| `apps/app/features/pacra/tasks/pacra-tasks-client.tsx` | Main tasks page component |
| `apps/app/features/pacra/tasks/components/task-details-modal.tsx` | Task detail dialog |
| `apps/app/features/pacra/tasks/components/task-status-badge.tsx` | Status badge component |
| `apps/app/features/pacra/tasks/search-params.ts` | URL search params utilities |
| `apps/app/lib/queries/tasks/hooks/use-tasks.ts` | React Query hooks |
| `apps/app/lib/queries/tasks/fetchers/tasks.ts` | API fetcher functions |
| `apps/app/lib/queries/tasks/types.ts` | TypeScript types |

### Backend

| File | Purpose |
|------|---------|
| `packages/api-services/src/domains/tasks/tasks.schema.ts` | Zod schemas |
| `packages/api-services/src/domains/tasks/tasks.service.ts` | Business logic |
| `packages/backend/src/modules/tasks/` | API routes and handlers |

## Error States

| State | UI |
|-------|-----|
| Loading | Skeleton list with 6 placeholder rows |
| Empty (no filters) | "You're all caught up!" message |
| Empty (with filters) | "No tasks found. Try adjusting your filters." |
| Error | Red error banner with retry suggestion |
| Load more | "Load More" button at list bottom |

## Future Enhancements

- [ ] Task grouping by parent entity (filings/service requests)
- [ ] Quick completion checkbox for confirmation-type tasks in list view
- [ ] Bulk task operations (mark multiple as done)
- [ ] Task assignment to team members
- [ ] Task notifications and reminders
