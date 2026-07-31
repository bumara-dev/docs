---
title: "Progress & Readiness"
description: "This document explains how task progress is computed and what gates a filing or service request must pass before becoming ready for submission."
---

This document explains how task progress is computed and what gates a filing or service request must pass before becoming ready for submission.

## Progress Computation

### Metrics

| Metric | Description |
|--------|-------------|
| `tasksDone` | Count of tasks with status `done` |
| `tasksTotal` | Total count of all tasks |
| `tasksRequired` | Count of tasks with `required = true` |
| `tasksPendingRequired` | Required tasks not yet `done` or `skipped` |
| `tasksBlocked` | Tasks with status `blocked` |

### API Response

```typescript
interface ProgressAndReadinessResult {
  progress: {
    tasksDone: number;
    tasksTotal: number;
    tasksRequired: number;
    tasksPendingRequired: number;
    tasksBlocked: number;
  };
  isReady: boolean;
  blockers: {
    pendingRequiredTasks: Array<{ id: string; title: string; status: string }>;
    blockedTasks: Array<{ id: string; title: string; reason: string | null }>;
  };
}
```

### Helper Function

```typescript
import { recomputeParentProgressAndStatus } from "@repo/api-services/domains/tasks";

const result = await recomputeParentProgressAndStatus(ctx, deps, {
  parentType: "filing",
  parentId: filingId,
});

if (result.isReady) {
  // Filing can proceed to submission
}
```

## Readiness Gates

A parent entity is **ready for submission** when ALL of the following are true:

### 1. All Required Tasks Complete

```typescript
const pendingRequired = tasks.filter(
  (t) => t.required && t.status !== "done" && t.status !== "skipped"
);
const allRequiredDone = pendingRequired.length === 0;
```

### 2. No Blocked Tasks

```typescript
const blockedTasks = tasks.filter((t) => t.status === "blocked");
const noneBlocked = blockedTasks.length === 0;
```

### 3. Required Documents Uploaded (for service requests)

```typescript
const missingDocs = docRequirements.filter((r) => r.required && !r.satisfied);
const allDocsUploaded = missingDocs.length === 0;
```

### Combined Check

```typescript
const isReady = allRequiredDone && noneBlocked && allDocsUploaded;
```

## Auto-Transition

When a task is marked as `done`, the system automatically checks if the parent entity should transition to `ready_for_submission`.

### Eligible Statuses

Only entities in these statuses can auto-transition:
- `pending_data`
- `in_progress`

### Transition Logic

```typescript
const eligibleStatuses = ["pending_data", "in_progress"];

if (eligibleStatuses.includes(currentStatus) && isReady) {
  // Update status to ready_for_submission
  // Record audit log with trigger: "auto_task_completion"
}
```

### Audit Log

```typescript
{
  action: "update",
  entityType: "filing", // or "service_request"
  entityId: filingId,
  changes: {
    before: { status: "in_progress" },
    after: { status: "ready_for_submission" },
  },
  metadata: {
    trigger: "auto_task_completion",
    tasksCompleted: 5,
    totalRequired: 5,
  },
}
```

## API Endpoints

### Filing Readiness

```http
GET /filings/{id}/readiness
```

Response:
```json
{
  "isReady": false,
  "tasksComplete": 3,
  "tasksRequired": 5,
  "docsUploaded": 2,
  "docsRequired": 3,
  "blockers": {
    "pendingRequiredTasks": [
      { "id": "task-1", "title": "Upload Payroll Summary", "status": "todo" }
    ],
    "blockedTasks": [],
    "missingRequiredDocs": [
      { "key": "PAYROLL_SUMMARY", "name": "Payroll Summary" }
    ]
  }
}
```

### Service Request View

The `getServiceRequestView` endpoint includes readiness information:

```typescript
{
  serviceRequest: { ... },
  tasks: [ ... ],
  docRequirements: [ ... ],
  progress: {
    tasksDone: 3,
    tasksTotal: 5,
    docsDone: 2,
    docsTotal: 3,
  },
  blockers: {
    isReady: false,
    blockedTasks: [],
    pendingRequiredTasks: [ ... ],
    missingRequiredDocs: [ ... ],
  },
}
```

## UI Integration

### Progress Bar

```tsx
function TaskProgress({ progress }: { progress: Progress }) {
  const percentage = (progress.tasksDone / progress.tasksTotal) * 100;
  
  return (
    <div className="space-y-2">
      <div className="flex justify-between text-sm">
        <span>Tasks Complete</span>
        <span>{progress.tasksDone} / {progress.tasksTotal}</span>
      </div>
      <Progress value={percentage} />
    </div>
  );
}
```

### Blockers Panel

```tsx
function BlockersPanel({ blockers }: { blockers: Blockers }) {
  if (blockers.isReady) {
    return (
      <Alert variant="success">
        <CheckCircle className="h-4 w-4" />
        <AlertTitle>Ready for Submission</AlertTitle>
      </Alert>
    );
  }

  return (
    <Alert variant="warning">
      <AlertTriangle className="h-4 w-4" />
      <AlertTitle>Action Required</AlertTitle>
      <AlertDescription>
        <ul>
          {blockers.pendingRequiredTasks.map((t) => (
            <li key={t.id}>Complete: {t.title}</li>
          ))}
          {blockers.blockedTasks.map((t) => (
            <li key={t.id}>Unblock: {t.title} ({t.reason})</li>
          ))}
          {blockers.missingRequiredDocs.map((d) => (
            <li key={d.key}>Upload: {d.name}</li>
          ))}
        </ul>
      </AlertDescription>
    </Alert>
  );
}
```

## Related Documentation

- [Task System Overview](/platform/tasks/overview)
- [Task Linking Guide](/platform/tasks/linking)
- [Task System Specification](/platform/tasks/spec)
