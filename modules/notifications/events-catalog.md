---
title: "Notifications Event Catalog"
description: "Single source of truth for all notification event types, payloads, and Zod schemas."
---

## Table of Contents

1. [Overview](#overview)
2. [Event Types](#event-types)
3. [Payload Schemas](#payload-schemas)
4. [Event Emission](#event-emission)
5. [Implementation](#implementation)

---

## Overview

All notification events are defined in a central catalog with:

- Unique event type identifier
- Zod schema for payload validation
- Severity level
- Default recipient rules
- Default channel configuration

---

## Event Types

### MVP Event Types

| Event Type | Severity | Description |
|------------|----------|-------------|
| `TASK_ASSIGNED` | INFO | Task assigned to a user |
| `TASK_ACTION_REQUIRED` | IMPORTANT | Task blocked/waiting on user action |
| `FILING_DUE_SOON` | IMPORTANT | Filing due in T-7, T-3, T-1 days |
| `FILING_OVERDUE` | URGENT | Filing past due date |
| `PAYMENT_REQUIRED` | IMPORTANT | Payment required for submission |
| `SUBMISSION_STATUS_CHANGED` | INFO/IMPORTANT | Filing submitted/accepted/rejected |

### Event Type Enum

```typescript
// packages/api-services/src/domains/notifications/events.ts

export const NotificationEventType = {
  // Task events
  TASK_ASSIGNED: 'TASK_ASSIGNED',
  TASK_ACTION_REQUIRED: 'TASK_ACTION_REQUIRED',
  
  // Filing events
  FILING_DUE_SOON: 'FILING_DUE_SOON',
  FILING_OVERDUE: 'FILING_OVERDUE',
  
  // Payment events
  PAYMENT_REQUIRED: 'PAYMENT_REQUIRED',
  
  // Submission events
  SUBMISSION_STATUS_CHANGED: 'SUBMISSION_STATUS_CHANGED',
} as const;

export type NotificationEventType = 
  (typeof NotificationEventType)[keyof typeof NotificationEventType];
```

---

## Payload Schemas

### Base Payload Schema

All events share a common base structure:

```typescript
import { z } from 'zod';

// Base payload fields (required for all events)
export const baseEventPayloadSchema = z.object({
  // Tenant isolation (REQUIRED)
  tenantId: z.string().min(1),
  
  // Entity reference
  entityType: z.enum([
    'task',
    'filing',
    'service_request',
    'payment_request',
    'submission_job',
  ]),
  entityId: z.string().uuid(),
  
  // Display content (safe, PII-minimal)
  title: z.string().max(200),
  message: z.string().max(500),
  
  // Navigation
  deepLink: z.string().url().or(z.string().startsWith('/')),
  
  // Classification
  severity: z.enum(['INFO', 'IMPORTANT', 'URGENT']),
  category: z.enum(['TASK', 'FILING', 'PAYMENT', 'SUBMISSION', 'SYSTEM']),
});

export type BaseEventPayload = z.infer<typeof baseEventPayloadSchema>;
```

### Task Events

```typescript
// TASK_ASSIGNED
export const taskAssignedPayloadSchema = baseEventPayloadSchema.extend({
  assignedUserId: z.string(),
  assignedByUserId: z.string().optional(),
  taskTitle: z.string(),
  dueOn: z.string().datetime().optional(),
});

// TASK_ACTION_REQUIRED
export const taskActionRequiredPayloadSchema = baseEventPayloadSchema.extend({
  assignedUserId: z.string(),
  taskTitle: z.string(),
  blockReason: z.string().optional(),
  requiredAction: z.string(),
});

export type TaskAssignedPayload = z.infer<typeof taskAssignedPayloadSchema>;
export type TaskActionRequiredPayload = z.infer<typeof taskActionRequiredPayloadSchema>;
```

### Filing Events

```typescript
// FILING_DUE_SOON
export const filingDueSoonPayloadSchema = baseEventPayloadSchema.extend({
  filingName: z.string(),
  regulatorCode: z.string(),
  dueOn: z.string().datetime(),
  daysUntilDue: z.number().int().min(0),
  periodLabel: z.string(), // e.g., "November 2025"
});

// FILING_OVERDUE
export const filingOverduePayloadSchema = baseEventPayloadSchema.extend({
  filingName: z.string(),
  regulatorCode: z.string(),
  dueOn: z.string().datetime(),
  daysPastDue: z.number().int().min(1),
  periodLabel: z.string(),
  // Optional: penalty info
  potentialPenaltyAmount: z.number().optional(),
});

export type FilingDueSoonPayload = z.infer<typeof filingDueSoonPayloadSchema>;
export type FilingOverduePayload = z.infer<typeof filingOverduePayloadSchema>;
```

### Payment Events

```typescript
// PAYMENT_REQUIRED
export const paymentRequiredPayloadSchema = baseEventPayloadSchema.extend({
  filingId: z.string().uuid().optional(),
  serviceRequestId: z.string().uuid().optional(),
  amount: z.number().int().min(0),  // Amount in minor units (e.g., ngwee for ZMW)
  currency: z.string().default('ZMW'),
  dueBy: z.string().datetime().optional(),
  paymentType: z.enum(['regulator_fee', 'service_fee', 'combined']),
});

export type PaymentRequiredPayload = z.infer<typeof paymentRequiredPayloadSchema>;
```

### Submission Events

```typescript
// SUBMISSION_STATUS_CHANGED
export const submissionStatusChangedPayloadSchema = baseEventPayloadSchema.extend({
  filingId: z.string().uuid().optional(),
  serviceRequestId: z.string().uuid().optional(),
  previousStatus: z.string(),
  newStatus: z.string(),
  regulatorCode: z.string(),
  // Optional: rejection reason
  rejectionReason: z.string().optional(),
});

export type SubmissionStatusChangedPayload = z.infer<typeof submissionStatusChangedPayloadSchema>;
```

---

## Event Emission

### How to Emit Events

Events are emitted by writing to the `event_outbox` table within the same transaction as the domain mutation.

```typescript
// Example: Emitting TASK_ASSIGNED event

import { eventOutbox } from '@repo/database';
import { taskAssignedPayloadSchema, NotificationEventType } from './events';

export async function assignTask(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  taskId: string,
  assigneeId: string
): Promise<Task> {
  return await deps.db.transaction(async (tx) => {
    // 1. Update the task
    const [task] = await tx
      .update(tasks)
      .set({ assignedToMemberId: assigneeId, updatedAt: new Date() })
      .where(
        and(
          eq(tasks.id, taskId),
          eq(tasks.organizationId, ctx.orgId) // Tenant isolation
        )
      )
      .returning();
    
    // 2. Create outbox event (same transaction)
    const payload = taskAssignedPayloadSchema.parse({
      tenantId: ctx.orgId,
      entityType: 'task',
      entityId: task.id,
      title: 'Task Assigned',
      message: `You have been assigned: ${task.title}`,
      deepLink: `/tasks/${task.id}`,
      severity: 'INFO',
      category: 'TASK',
      assignedUserId: assigneeId,
      assignedByUserId: ctx.userId,
      taskTitle: task.title,
      dueOn: task.dueOn?.toISOString(),
    });
    
    await tx.insert(eventOutbox).values({
      tenantId: ctx.orgId,
      eventType: NotificationEventType.TASK_ASSIGNED,
      payload,
      status: 'pending',
    });
    
    // 3. Create audit log
    await recordAuditLog(ctx, deps, {
      action: 'task.assigned',
      entityType: 'task',
      entityId: task.id,
      changes: { after: { assignedToMemberId: assigneeId } },
    });
    
    return task;
  });
}
```

### Cloudflare Queue Integration

After the transaction commits, a Cloudflare Queue trigger picks up the event:

```typescript
// packages/backend/src/index.ts

import { processOutboxQueue } from './queues/outbox-processor';

export default {
  fetch: app.fetch,
  
  // Queue consumer for outbox processing
  async queue(
    batch: MessageBatch<OutboxQueueMessage>,
    env: Env,
    ctx: ExecutionContext
  ) {
    await processOutboxQueue(batch, env, ctx);
  },
};
```

---

## Implementation

### Complete Events File

```typescript
// packages/api-services/src/domains/notifications/events.ts

import { z } from 'zod';

// ============================================
// Event Type Enum
// ============================================

export const NotificationEventType = {
  TASK_ASSIGNED: 'TASK_ASSIGNED',
  TASK_ACTION_REQUIRED: 'TASK_ACTION_REQUIRED',
  FILING_DUE_SOON: 'FILING_DUE_SOON',
  FILING_OVERDUE: 'FILING_OVERDUE',
  PAYMENT_REQUIRED: 'PAYMENT_REQUIRED',
  SUBMISSION_STATUS_CHANGED: 'SUBMISSION_STATUS_CHANGED',
} as const;

export type NotificationEventType =
  (typeof NotificationEventType)[keyof typeof NotificationEventType];

// ============================================
// Severity Enum
// ============================================

export const NotificationSeverity = {
  INFO: 'INFO',
  IMPORTANT: 'IMPORTANT',
  URGENT: 'URGENT',
} as const;

export type NotificationSeverity =
  (typeof NotificationSeverity)[keyof typeof NotificationSeverity];

// ============================================
// Category Enum
// ============================================

export const NotificationCategory = {
  TASK: 'TASK',
  FILING: 'FILING',
  PAYMENT: 'PAYMENT',
  SUBMISSION: 'SUBMISSION',
  SYSTEM: 'SYSTEM',
} as const;

export type NotificationCategory =
  (typeof NotificationCategory)[keyof typeof NotificationCategory];

// ============================================
// Base Payload Schema
// ============================================

export const baseEventPayloadSchema = z.object({
  tenantId: z.string().min(1),
  entityType: z.enum([
    'task',
    'filing',
    'service_request',
    'payment_request',
    'submission_job',
  ]),
  entityId: z.string().uuid(),
  title: z.string().max(200),
  message: z.string().max(500),
  deepLink: z.string(),
  severity: z.nativeEnum(NotificationSeverity),
  category: z.nativeEnum(NotificationCategory),
});

export type BaseEventPayload = z.infer<typeof baseEventPayloadSchema>;

// ============================================
// Event-Specific Payload Schemas
// ============================================

export const taskAssignedPayloadSchema = baseEventPayloadSchema.extend({
  assignedUserId: z.string(),
  assignedByUserId: z.string().optional(),
  taskTitle: z.string(),
  dueOn: z.string().datetime().optional(),
});

export const taskActionRequiredPayloadSchema = baseEventPayloadSchema.extend({
  assignedUserId: z.string(),
  taskTitle: z.string(),
  blockReason: z.string().optional(),
  requiredAction: z.string(),
});

export const filingDueSoonPayloadSchema = baseEventPayloadSchema.extend({
  filingName: z.string(),
  regulatorCode: z.string(),
  dueOn: z.string().datetime(),
  daysUntilDue: z.number().int().min(0),
  periodLabel: z.string(),
});

export const filingOverduePayloadSchema = baseEventPayloadSchema.extend({
  filingName: z.string(),
  regulatorCode: z.string(),
  dueOn: z.string().datetime(),
  daysPastDue: z.number().int().min(1),
  periodLabel: z.string(),
  potentialPenaltyAmount: z.number().optional(),
});

export const paymentRequiredPayloadSchema = baseEventPayloadSchema.extend({
  filingId: z.string().uuid().optional(),
  serviceRequestId: z.string().uuid().optional(),
  amount: z.number().int().min(0),  // Amount in minor units
  currency: z.string().default('ZMW'),
  dueBy: z.string().datetime().optional(),
  paymentType: z.enum(['regulator_fee', 'service_fee', 'combined']),
});

export const submissionStatusChangedPayloadSchema = baseEventPayloadSchema.extend({
  filingId: z.string().uuid().optional(),
  serviceRequestId: z.string().uuid().optional(),
  previousStatus: z.string(),
  newStatus: z.string(),
  regulatorCode: z.string(),
  rejectionReason: z.string().optional(),
});

// ============================================
// Payload Type Union
// ============================================

export const eventPayloadSchema = z.discriminatedUnion('category', [
  taskAssignedPayloadSchema.extend({ 
    category: z.literal(NotificationCategory.TASK) 
  }),
  filingDueSoonPayloadSchema.extend({ 
    category: z.literal(NotificationCategory.FILING) 
  }),
  paymentRequiredPayloadSchema.extend({ 
    category: z.literal(NotificationCategory.PAYMENT) 
  }),
  submissionStatusChangedPayloadSchema.extend({ 
    category: z.literal(NotificationCategory.SUBMISSION) 
  }),
]);

export type EventPayload = z.infer<typeof eventPayloadSchema>;

// ============================================
// Event Metadata (for routing decisions)
// ============================================

export interface EventTypeMetadata {
  eventType: NotificationEventType;
  defaultSeverity: NotificationSeverity;
  category: NotificationCategory;
  description: string;
}

export const EVENT_TYPE_METADATA: Record<NotificationEventType, EventTypeMetadata> = {
  [NotificationEventType.TASK_ASSIGNED]: {
    eventType: NotificationEventType.TASK_ASSIGNED,
    defaultSeverity: NotificationSeverity.INFO,
    category: NotificationCategory.TASK,
    description: 'Task assigned to a user',
  },
  [NotificationEventType.TASK_ACTION_REQUIRED]: {
    eventType: NotificationEventType.TASK_ACTION_REQUIRED,
    defaultSeverity: NotificationSeverity.IMPORTANT,
    category: NotificationCategory.TASK,
    description: 'Task requires user action',
  },
  [NotificationEventType.FILING_DUE_SOON]: {
    eventType: NotificationEventType.FILING_DUE_SOON,
    defaultSeverity: NotificationSeverity.IMPORTANT,
    category: NotificationCategory.FILING,
    description: 'Filing due within 7 days',
  },
  [NotificationEventType.FILING_OVERDUE]: {
    eventType: NotificationEventType.FILING_OVERDUE,
    defaultSeverity: NotificationSeverity.URGENT,
    category: NotificationCategory.FILING,
    description: 'Filing past due date',
  },
  [NotificationEventType.PAYMENT_REQUIRED]: {
    eventType: NotificationEventType.PAYMENT_REQUIRED,
    defaultSeverity: NotificationSeverity.IMPORTANT,
    category: NotificationCategory.PAYMENT,
    description: 'Payment required for submission',
  },
  [NotificationEventType.SUBMISSION_STATUS_CHANGED]: {
    eventType: NotificationEventType.SUBMISSION_STATUS_CHANGED,
    defaultSeverity: NotificationSeverity.INFO,
    category: NotificationCategory.SUBMISSION,
    description: 'Submission status updated',
  },
};
```

---

## Adding New Event Types

When adding a new event type:

1. Add to `NotificationEventType` enum
2. Create Zod schema extending `baseEventPayloadSchema`
3. Add to `EVENT_TYPE_METADATA` map
4. Update routing rules in `routing.ts`
5. Add email/WhatsApp templates if needed
6. Update documentation

