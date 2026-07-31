---
title: "Tasks Module - Jobs & Notifications"
description: "Background jobs, scheduled tasks, event emission, and notification integration for the Tasks module."
---

## Table of Contents

1. [Event Emission](#event-emission)
2. [Notification Triggers](#notification-triggers)
3. [Scheduled Jobs](#scheduled-jobs)
4. [Integration Points](#integration-points)

---

## Event Emission

### Task Events

The Tasks module emits events for significant state changes. These events can trigger notifications, analytics, and integrations.

| Event | Trigger | Payload |
|-------|---------|---------|
| `task.created` | New task inserted | taskId, filingId/serviceRequestId, taskType, orgId |
| `task.assigned` | Assignee changed | taskId, previousAssignee, newAssignee, orgId |
| `task.status_changed` | Status transition | taskId, previousStatus, newStatus, orgId, actorId |
| `task.completed` | Status → done | taskId, completedBy, completedAt, orgId |
| `task.blocked` | Status → blocked | taskId, blockedReason, orgId |
| `task.skipped` | Status → skipped | taskId, skipReason, orgId |
| `task.reopened` | done/skipped → todo | taskId, reopenedBy, orgId |
| `task.comment_added` | Comment created | taskId, commentId, authorId, orgId |
| `task.due_soon` | 7 days until due | taskId, dueOn, assigneeId, orgId |
| `task.overdue` | Due date passed | taskId, dueOn, assigneeId, orgId, daysPastDue |

### Event Structure

```typescript
interface TaskEvent {
  type: string;
  timestamp: string;
  data: {
    taskId: string;
    organizationId: string;
    filingId?: string;
    serviceRequestId?: string;
    actorId?: string;
    actorType?: 'tenant' | 'backoffice' | 'system';
    [key: string]: unknown;
  };
}

// Example: task.completed
{
  type: 'task.completed',
  timestamp: '2025-12-15T10:30:00.000Z',
  data: {
    taskId: 'uuid-123',
    organizationId: 'org_abc',
    filingId: 'uuid-456',
    completedBy: 'user_xyz',
    completedAt: '2025-12-15T10:30:00.000Z',
    taskTitle: 'Upload Annual Return Documents',
    wasRequired: true,
  }
}
```

### Event Emission Implementation

```typescript
// packages/api-services/src/domains/tasks/tasks.events.ts

import type { ServiceDependencies } from "../../core/context";

export type TaskEventType =
  | "task.created"
  | "task.assigned"
  | "task.status_changed"
  | "task.completed"
  | "task.blocked"
  | "task.skipped"
  | "task.reopened"
  | "task.comment_added"
  | "task.due_soon"
  | "task.overdue";

export interface TaskEventPayload {
  taskId: string;
  organizationId: string;
  filingId?: string;
  serviceRequestId?: string;
  [key: string]: unknown;
}

export async function emitTaskEvent(
  deps: ServiceDependencies,
  eventType: TaskEventType,
  payload: TaskEventPayload
): Promise<void> {
  const event = {
    type: eventType,
    timestamp: new Date().toISOString(),
    data: payload,
  };

  // Log event for debugging
  deps.logger?.info?.(event, `Task event: ${eventType}`);

  // Future: publish to message queue (e.g., Cloudflare Queues, Redis)
  // await deps.eventBus.publish(eventType, event);

  // For now: direct notification dispatch
  await dispatchNotification(deps, event);
}

async function dispatchNotification(
  deps: ServiceDependencies,
  event: { type: string; data: TaskEventPayload }
): Promise<void> {
  // Notification dispatch logic
  // This will be integrated with the notifications package
}
```

---

## Notification Triggers

### In-App Notifications

| Event | Recipients | Message Template |
|-------|------------|------------------|
| `task.created` (info_request) | Task assignee or all org admins | "New info request: &#123;title&#125;" |
| `task.assigned` | New assignee | "Task assigned to you: &#123;title&#125;" |
| `task.completed` | Task creator (if backoffice) | "Tenant completed: &#123;title&#125;" |
| `task.blocked` | Org admins | "Task blocked: &#123;title&#125; — &#123;reason&#125;" |
| `task.due_soon` | Assignee | "Task due in &#123;days&#125; days: &#123;title&#125;" |
| `task.overdue` | Assignee + org admins | "Task overdue: &#123;title&#125;" |

### Email Notifications

| Event | Trigger Condition | Email Type |
|-------|-------------------|------------|
| `task.created` | taskType = 'info_request' | Immediate: "Action Required" |
| `task.due_soon` | 7 days before due | Digest or immediate |
| `task.overdue` | Every day past due (max 3) | Immediate: "Overdue Alert" |
| `task.assigned` | User preference enabled | Immediate: "New Assignment" |

### Email Templates

```typescript
// packages/email/templates/task-info-request.tsx

interface TaskInfoRequestEmailProps {
  recipientName: string;
  taskTitle: string;
  taskDescription?: string;
  dueDate: string;
  filingName: string;
  actionUrl: string;
}

// Subject: "Action Required: {taskTitle}"
// Body: Explains what's needed, links to task
```

### Notification Preferences

Users can configure notification preferences:

```typescript
interface TaskNotificationPreferences {
  email: {
    taskAssigned: boolean;      // Default: true
    taskDueSoon: boolean;       // Default: true
    taskOverdue: boolean;       // Default: true
    infoRequestReceived: boolean; // Default: true
  };
  inApp: {
    allTaskEvents: boolean;     // Default: true
  };
  digest: {
    enabled: boolean;           // Default: false
    frequency: 'daily' | 'weekly';
  };
}
```

---

## Scheduled Jobs

### 1. Due Soon Check

**Purpose:** Identify tasks due within 7 days and emit `task.due_soon` events.

**Schedule:** Daily at 08:00 UTC (10:00 CAT for Zambia)

**Logic:**

```typescript
// packages/backend/src/jobs/task-due-soon.ts

export async function checkTasksDueSoon(
  db: DrizzleClient,
  deps: ServiceDependencies
): Promise<{ notified: number }> {
  const sevenDaysFromNow = addDays(new Date(), 7);
  const today = startOfDay(new Date());

  // Find tasks due within 7 days that haven't been notified today
  const tasksDueSoon = await db
    .select()
    .from(tasks)
    .where(
      and(
        gte(tasks.dueOn, today),
        lte(tasks.dueOn, sevenDaysFromNow),
        notInArray(tasks.status, ['done', 'skipped']),
        isNull(tasks.deletedAt)
      )
    );

  let notified = 0;

  for (const task of tasksDueSoon) {
    await emitTaskEvent(deps, 'task.due_soon', {
      taskId: task.id,
      organizationId: task.organizationId,
      filingId: task.filingId ?? undefined,
      serviceRequestId: task.serviceRequestId ?? undefined,
      dueOn: task.dueOn?.toISOString(),
      assigneeId: task.assignedToMemberId ?? undefined,
      daysUntilDue: differenceInDays(task.dueOn!, today),
    });
    notified++;
  }

  deps.logger?.info?.({ notified }, 'Due soon check complete');
  return { notified };
}
```

### 2. Overdue Check

**Purpose:** Identify overdue tasks and emit `task.overdue` events.

**Schedule:** Daily at 09:00 UTC (11:00 CAT)

**Logic:**

```typescript
// packages/backend/src/jobs/task-overdue.ts

export async function checkTasksOverdue(
  db: DrizzleClient,
  deps: ServiceDependencies
): Promise<{ notified: number }> {
  const today = startOfDay(new Date());

  // Find overdue tasks
  const overdueTasks = await db
    .select()
    .from(tasks)
    .where(
      and(
        lt(tasks.dueOn, today),
        notInArray(tasks.status, ['done', 'skipped']),
        isNull(tasks.deletedAt)
      )
    );

  let notified = 0;

  for (const task of overdueTasks) {
    const daysPastDue = differenceInDays(today, task.dueOn!);

    // Only notify for first 3 days, then weekly
    const shouldNotify = daysPastDue <= 3 || daysPastDue % 7 === 0;

    if (shouldNotify) {
      await emitTaskEvent(deps, 'task.overdue', {
        taskId: task.id,
        organizationId: task.organizationId,
        filingId: task.filingId ?? undefined,
        serviceRequestId: task.serviceRequestId ?? undefined,
        dueOn: task.dueOn?.toISOString(),
        assigneeId: task.assignedToMemberId ?? undefined,
        daysPastDue,
      });
      notified++;
    }
  }

  deps.logger?.info?.({ notified, total: overdueTasks.length }, 'Overdue check complete');
  return { notified };
}
```

### 3. Stale Task Cleanup (Future)

**Purpose:** Archive tasks that have been in `done` status for extended periods.

**Schedule:** Weekly

**Logic:** Mark tasks as archived after 90 days in `done` status (soft delete).

---

## Integration Points

### Notifications Package

The Tasks module integrates with `packages/notifications` for delivery:

```typescript
// Integration with notifications package
import { sendInAppNotification, sendEmail } from '@repo/notifications';

async function dispatchTaskNotification(
  event: TaskEvent
): Promise<void> {
  const { type, data } = event;

  switch (type) {
    case 'task.created':
      if (data.taskType === 'info_request') {
        await sendInAppNotification({
          userId: data.assigneeId,
          title: 'New info request',
          body: `Please complete: ${data.taskTitle}`,
          actionUrl: `/tasks/${data.taskId}`,
        });

        await sendEmail({
          to: data.assigneeEmail,
          template: 'task-info-request',
          data: { ... },
        });
      }
      break;

    case 'task.overdue':
      await sendInAppNotification({
        userId: data.assigneeId,
        title: 'Task overdue',
        body: `${data.taskTitle} was due ${data.daysPastDue} day(s) ago`,
        actionUrl: `/tasks/${data.taskId}`,
        priority: 'high',
      });
      break;

    // ... other cases
  }
}
```

### Analytics Integration

Task events feed into analytics for:

- Task completion rate by org
- Average time to complete by task type
- Overdue rate trends
- Info request turnaround time

```typescript
// Future: Analytics event tracking
await trackAnalyticsEvent('task_completed', {
  taskId: task.id,
  taskType: task.taskType,
  timeToComplete: differenceInHours(task.completedAt, task.createdAt),
  wasOverdue: task.dueOn < task.completedAt,
});
```

### Audit Log Integration

All task state changes are recorded in audit logs:

```typescript
// Called from tasks.service.ts on every mutation
await recordAuditLog(ctx, deps, {
  action: 'update',
  entityType: 'task',
  entityId: taskId,
  changes: {
    before: { status: previousStatus },
    after: { status: newStatus },
  },
  metadata: {
    reason: skipReason || blockedReason,
  },
});
```

---

## Cron Configuration

### Cloudflare Workers Scheduled Triggers

```typescript
// packages/backend/src/index.ts

export default {
  async scheduled(event: ScheduledEvent, env: Env, ctx: ExecutionContext) {
    const db = createDb(env);
    const deps = createDependencies(env);

    switch (event.cron) {
      case '0 8 * * *': // Daily 08:00 UTC
        await checkTasksDueSoon(db, deps);
        break;

      case '0 9 * * *': // Daily 09:00 UTC
        await checkTasksOverdue(db, deps);
        break;
    }
  },
};
```

### Wrangler Configuration

```jsonc
// packages/backend/wrangler.jsonc
{
  "triggers": {
    "crons": [
      "0 8 * * *",  // Due soon check
      "0 9 * * *"   // Overdue check
    ]
  }
}
```

---

## Monitoring

### Key Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `tasks.due_soon.count` | Tasks due within 7 days | Info only |
| `tasks.overdue.count` | Currently overdue tasks | > 100 org-wide |
| `tasks.notification.sent` | Notifications dispatched | - |
| `tasks.notification.failed` | Failed notification attempts | > 5% |
| `jobs.due_soon.duration_ms` | Job execution time | > 30s |
| `jobs.overdue.duration_ms` | Job execution time | > 30s |

### Logging

```typescript
// Standard log format for task jobs
deps.logger?.info?.({
  job: 'task_due_soon',
  tasksChecked: total,
  notificationsSent: notified,
  durationMs: endTime - startTime,
}, 'Task due soon job completed');
```


