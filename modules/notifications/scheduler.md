---
title: "Notifications Scheduler"
description: "Cron jobs for scheduled notifications: filing reminders, task deadlines, and SLA alerts."
---

## Table of Contents

1. [Overview](#overview)
2. [Cron Configuration](#cron-configuration)
3. [Filing Reminders](#filing-reminders)
4. [Task Reminders](#task-reminders)
5. [Deduplication](#deduplication)
6. [Implementation](#implementation)

---

## Overview

The scheduler creates outbox events for time-based notifications:

| Job | Schedule | Purpose |
|-----|----------|---------|
| Filing Due Soon | Daily 06:00 UTC | Remind at T-7, T-3, T-1 days |
| Filing Overdue | Daily 07:00 UTC | Alert at T+1, T+3, then weekly |
| Task Deadline | Daily 08:00 UTC | Tasks due within 3 days |
| Outbox Poll | Every 5 min | Backup for failed queue triggers |

### Scheduler Flow

```
Cron Trigger (Cloudflare)
        │
        ▼
┌───────────────────────────┐
│   Scheduler Job           │
│   1. Query eligible items │
│   2. Check dedup cache    │
│   3. Create outbox events │
└───────────────────────────┘
        │
        ▼
event_outbox (DB)
        │
        ▼
notification-outbox (Queue)
        │
        ▼
(Normal outbox processing flow)
```

---

## Cron Configuration

### Wrangler Setup

```jsonc
// packages/backend/wrangler.jsonc

{
  "triggers": {
    "crons": [
      "0 6 * * *",   // Filing due soon - 06:00 UTC (08:00 CAT)
      "0 7 * * *",   // Filing overdue - 07:00 UTC (09:00 CAT)
      "0 8 * * *",   // Task deadline - 08:00 UTC (10:00 CAT)
      "*/5 * * * *"  // Outbox poll (backup)
    ]
  }
}
```

### Scheduler Entry Point

```typescript
// packages/backend/src/jobs/scheduler.ts

import type { ScheduledEvent, ExecutionContext } from '@cloudflare/workers-types';
import { createDb } from '@repo/database';
import { runFilingDueSoonJob } from './filing-reminders';
import { runFilingOverdueJob } from './filing-reminders';
import { runTaskDeadlineJob } from './task-reminders';
import { runOutboxPoll } from './outbox-poll';
import type { Env } from '../env';

export async function runScheduledJobs(
  event: ScheduledEvent,
  env: Env,
  ctx: ExecutionContext
): Promise<void> {
  const db = createDb(env.DATABASE_URL);
  const cron = event.cron;
  
  console.info('Scheduled job triggered:', { cron, scheduledTime: event.scheduledTime });
  
  try {
    switch (cron) {
      case '0 6 * * *':
        await runFilingDueSoonJob(db, env);
        break;
      
      case '0 7 * * *':
        await runFilingOverdueJob(db, env);
        break;
      
      case '0 8 * * *':
        await runTaskDeadlineJob(db, env);
        break;
      
      case '*/5 * * * *':
        await runOutboxPoll(db, env);
        break;
      
      default:
        console.warn('Unknown cron schedule:', cron);
    }
  } catch (error) {
    console.error('Scheduled job failed:', {
      cron,
      error: error instanceof Error ? error.message : 'Unknown error',
    });
    throw error; // Re-throw for Cloudflare error tracking
  }
}
```

---

## Filing Reminders

### Reminder Schedule

| Days Until Due | Event Type | Severity |
|---------------|------------|----------|
| 7 | `FILING_DUE_SOON` | IMPORTANT |
| 3 | `FILING_DUE_SOON` | IMPORTANT |
| 1 | `FILING_DUE_SOON` | URGENT |

| Days Past Due | Event Type | Severity |
|--------------|------------|----------|
| 1 | `FILING_OVERDUE` | URGENT |
| 3 | `FILING_OVERDUE` | URGENT |
| 7, 14, 21... | `FILING_OVERDUE` | URGENT |

### Implementation

```typescript
// packages/backend/src/jobs/filing-reminders.ts

import { createDb } from '@repo/database';
import { filings, eventOutbox } from '@repo/database';
import { and, eq, gte, lte, notInArray, isNull, sql } from 'drizzle-orm';
import { NotificationEventType, NotificationSeverity } from '@repo/api-services/notifications';
import type { Env } from '../env';

/**
 * Filing Due Soon Job
 * Runs daily at 06:00 UTC
 */
export async function runFilingDueSoonJob(
  db: ReturnType<typeof createDb>,
  env: Env
): Promise<{ created: number }> {
  const today = startOfDay(new Date());
  const reminderDays = [7, 3, 1];
  
  let created = 0;
  
  for (const daysUntilDue of reminderDays) {
    const targetDate = addDays(today, daysUntilDue);
    
    // Find filings due on this specific day
    const filingsToRemind = await db
      .select({
        id: filings.id,
        tenantId: filings.organizationId,
        name: filings.name,
        dueOn: filings.dueOn,
        periodLabel: filings.periodLabel,
        regulatorCode: filings.regulatorCode,
      })
      .from(filings)
      .where(
        and(
          eq(sql`DATE(${filings.dueOn})`, sql`DATE(${targetDate})`),
          notInArray(filings.status, ['accepted', 'waived', 'cancelled']),
          isNull(filings.deletedAt)
        )
      );
    
    for (const filing of filingsToRemind) {
      // Check if we already sent this reminder
      const alreadySent = await hasRecentReminder(
        db,
        filing.tenantId,
        filing.id,
        NotificationEventType.FILING_DUE_SOON,
        daysUntilDue
      );
      
      if (alreadySent) {
        continue;
      }
      
      // Create outbox event
      await db.insert(eventOutbox).values({
        tenantId: filing.tenantId,
        eventType: NotificationEventType.FILING_DUE_SOON,
        payload: {
          tenantId: filing.tenantId,
          entityType: 'filing',
          entityId: filing.id,
          title: `Filing Due in ${daysUntilDue} Days`,
          message: `${filing.name} for ${filing.periodLabel} is due on ${formatDate(filing.dueOn)}`,
          deepLink: `/filings/${filing.id}`,
          severity: daysUntilDue === 1 ? NotificationSeverity.URGENT : NotificationSeverity.IMPORTANT,
          category: 'FILING',
          filingName: filing.name,
          regulatorCode: filing.regulatorCode,
          dueOn: filing.dueOn?.toISOString(),
          daysUntilDue,
          periodLabel: filing.periodLabel,
        },
        status: 'pending',
      });
      
      created++;
    }
  }
  
  // Enqueue outbox processing
  await env.OUTBOX_QUEUE.send({ type: 'poll' });
  
  console.info('Filing due soon job completed:', { created });
  return { created };
}

/**
 * Filing Overdue Job
 * Runs daily at 07:00 UTC
 */
export async function runFilingOverdueJob(
  db: ReturnType<typeof createDb>,
  env: Env
): Promise<{ created: number }> {
  const today = startOfDay(new Date());
  
  // Find all overdue filings
  const overdueFilings = await db
    .select({
      id: filings.id,
      tenantId: filings.organizationId,
      name: filings.name,
      dueOn: filings.dueOn,
      periodLabel: filings.periodLabel,
      regulatorCode: filings.regulatorCode,
    })
    .from(filings)
    .where(
      and(
        lte(filings.dueOn, today),
        notInArray(filings.status, ['accepted', 'waived', 'cancelled']),
        isNull(filings.deletedAt)
      )
    );
  
  let created = 0;
  
  for (const filing of overdueFilings) {
    const daysPastDue = differenceInDays(today, filing.dueOn!);
    
    // Determine if we should send a reminder today
    // Days 1, 3, then every 7 days
    const shouldRemind = 
      daysPastDue === 1 ||
      daysPastDue === 3 ||
      (daysPastDue > 3 && daysPastDue % 7 === 0);
    
    if (!shouldRemind) {
      continue;
    }
    
    // Check if already sent today
    const alreadySent = await hasTodayReminder(
      db,
      filing.tenantId,
      filing.id,
      NotificationEventType.FILING_OVERDUE
    );
    
    if (alreadySent) {
      continue;
    }
    
    // Create outbox event
    await db.insert(eventOutbox).values({
      tenantId: filing.tenantId,
      eventType: NotificationEventType.FILING_OVERDUE,
      payload: {
        tenantId: filing.tenantId,
        entityType: 'filing',
        entityId: filing.id,
        title: `URGENT: Filing Overdue`,
        message: `${filing.name} for ${filing.periodLabel} was due ${daysPastDue} days ago`,
        deepLink: `/filings/${filing.id}`,
        severity: NotificationSeverity.URGENT,
        category: 'FILING',
        filingName: filing.name,
        regulatorCode: filing.regulatorCode,
        dueOn: filing.dueOn?.toISOString(),
        daysPastDue,
        periodLabel: filing.periodLabel,
      },
      status: 'pending',
    });
    
    created++;
  }
  
  // Enqueue outbox processing
  await env.OUTBOX_QUEUE.send({ type: 'poll' });
  
  console.info('Filing overdue job completed:', { created });
  return { created };
}

// Helper functions
function startOfDay(date: Date): Date {
  const d = new Date(date);
  d.setHours(0, 0, 0, 0);
  return d;
}

function addDays(date: Date, days: number): Date {
  const d = new Date(date);
  d.setDate(d.getDate() + days);
  return d;
}

function differenceInDays(later: Date, earlier: Date): number {
  const diff = later.getTime() - earlier.getTime();
  return Math.floor(diff / (1000 * 60 * 60 * 24));
}

function formatDate(date: Date | null): string {
  if (!date) return 'N/A';
  return date.toLocaleDateString('en-GB', {
    day: 'numeric',
    month: 'long',
    year: 'numeric',
  });
}
```

---

## Task Reminders

### Implementation

```typescript
// packages/backend/src/jobs/task-reminders.ts

import { createDb } from '@repo/database';
import { tasks, eventOutbox } from '@repo/database';
import { and, eq, gte, lte, notInArray, isNull } from 'drizzle-orm';
import { NotificationEventType, NotificationSeverity } from '@repo/api-services/notifications';
import type { Env } from '../env';

/**
 * Task Deadline Job
 * Runs daily at 08:00 UTC
 */
export async function runTaskDeadlineJob(
  db: ReturnType<typeof createDb>,
  env: Env
): Promise<{ created: number }> {
  const today = startOfDay(new Date());
  const threeDaysFromNow = addDays(today, 3);
  
  // Find tasks due within 3 days
  const upcomingTasks = await db
    .select({
      id: tasks.id,
      tenantId: tasks.organizationId,
      title: tasks.title,
      dueOn: tasks.dueOn,
      assignedToMemberId: tasks.assignedToMemberId,
      filingId: tasks.filingId,
      serviceRequestId: tasks.serviceRequestId,
    })
    .from(tasks)
    .where(
      and(
        gte(tasks.dueOn, today),
        lte(tasks.dueOn, threeDaysFromNow),
        notInArray(tasks.status, ['done', 'skipped']),
        isNull(tasks.deletedAt)
      )
    );
  
  let created = 0;
  
  for (const task of upcomingTasks) {
    const daysUntilDue = differenceInDays(task.dueOn!, today);
    
    // Only send for tasks due in 3, 1, or 0 days
    if (![3, 1, 0].includes(daysUntilDue)) {
      continue;
    }
    
    // Check if already sent
    const alreadySent = await hasTodayReminder(
      db,
      task.tenantId,
      task.id,
      NotificationEventType.TASK_ACTION_REQUIRED
    );
    
    if (alreadySent) {
      continue;
    }
    
    const severity = daysUntilDue === 0 
      ? NotificationSeverity.URGENT 
      : NotificationSeverity.IMPORTANT;
    
    const title = daysUntilDue === 0
      ? 'Task Due Today'
      : `Task Due in ${daysUntilDue} Days`;
    
    await db.insert(eventOutbox).values({
      tenantId: task.tenantId,
      eventType: NotificationEventType.TASK_ACTION_REQUIRED,
      payload: {
        tenantId: task.tenantId,
        entityType: 'task',
        entityId: task.id,
        title,
        message: `${task.title} is due ${daysUntilDue === 0 ? 'today' : `in ${daysUntilDue} days`}`,
        deepLink: `/tasks/${task.id}`,
        severity,
        category: 'TASK',
        assignedUserId: task.assignedToMemberId,
        taskTitle: task.title,
        requiredAction: 'Complete the task before the deadline',
        dueOn: task.dueOn?.toISOString(),
      },
      status: 'pending',
    });
    
    created++;
  }
  
  // Enqueue outbox processing
  await env.OUTBOX_QUEUE.send({ type: 'poll' });
  
  console.info('Task deadline job completed:', { created });
  return { created };
}
```

---

## Deduplication

### Dedup Strategy

Prevent duplicate reminders using outbox event history:

```typescript
// packages/backend/src/jobs/dedup.ts

import { createDb } from '@repo/database';
import { eventOutbox } from '@repo/database';
import { and, eq, gte, sql } from 'drizzle-orm';

/**
 * Check if a reminder was already sent for this entity today
 */
export async function hasTodayReminder(
  db: ReturnType<typeof createDb>,
  tenantId: string,
  entityId: string,
  eventType: string
): Promise<boolean> {
  const today = new Date();
  today.setHours(0, 0, 0, 0);
  
  const [existing] = await db
    .select({ id: eventOutbox.id })
    .from(eventOutbox)
    .where(
      and(
        eq(eventOutbox.tenantId, tenantId),
        eq(eventOutbox.eventType, eventType),
        eq(sql`${eventOutbox.payload}->>'entityId'`, entityId),
        gte(eventOutbox.createdAt, today)
      )
    )
    .limit(1);
  
  return !!existing;
}

/**
 * Check if a specific reminder bucket was already sent
 * (e.g., "7 days before due" reminder)
 */
export async function hasRecentReminder(
  db: ReturnType<typeof createDb>,
  tenantId: string,
  entityId: string,
  eventType: string,
  daysUntilDue: number
): Promise<boolean> {
  // Look back 24 hours to avoid race conditions
  const yesterday = new Date();
  yesterday.setDate(yesterday.getDate() - 1);
  
  const [existing] = await db
    .select({ id: eventOutbox.id })
    .from(eventOutbox)
    .where(
      and(
        eq(eventOutbox.tenantId, tenantId),
        eq(eventOutbox.eventType, eventType),
        eq(sql`${eventOutbox.payload}->>'entityId'`, entityId),
        eq(sql`(${eventOutbox.payload}->>'daysUntilDue')::int`, daysUntilDue),
        gte(eventOutbox.createdAt, yesterday)
      )
    )
    .limit(1);
  
  return !!existing;
}
```

### Dedup Index

Add an index for efficient dedup queries:

```sql
CREATE INDEX idx_event_outbox_dedup ON event_outbox(
  tenant_id,
  event_type,
  (payload->>'entityId'),
  created_at DESC
);
```

---

## Implementation

### Outbox Poll (Backup)

```typescript
// packages/backend/src/jobs/outbox-poll.ts

import { createDb } from '@repo/database';
import { eventOutbox } from '@repo/database';
import { and, eq, lte, or, isNull } from 'drizzle-orm';
import type { Env } from '../env';

/**
 * Outbox Poll Job
 * Runs every 5 minutes as a backup to queue triggers
 */
export async function runOutboxPoll(
  db: ReturnType<typeof createDb>,
  env: Env
): Promise<{ enqueued: number }> {
  const now = new Date();
  
  // Find pending events that are ready for processing
  const pendingEvents = await db
    .select({
      id: eventOutbox.id,
      tenantId: eventOutbox.tenantId,
      eventType: eventOutbox.eventType,
      attempts: eventOutbox.attempts,
    })
    .from(eventOutbox)
    .where(
      and(
        eq(eventOutbox.status, 'pending'),
        or(
          isNull(eventOutbox.nextRetryAt),
          lte(eventOutbox.nextRetryAt, now)
        )
      )
    )
    .limit(100);
  
  let enqueued = 0;
  
  for (const event of pendingEvents) {
    await env.OUTBOX_QUEUE.send({
      outboxEventId: event.id,
      tenantId: event.tenantId,
      eventType: event.eventType,
      attempt: event.attempts,
    });
    
    enqueued++;
  }
  
  if (enqueued > 0) {
    console.info('Outbox poll enqueued events:', { enqueued });
  }
  
  return { enqueued };
}
```

---

## Monitoring

### Job Metrics

```typescript
// Log structured metrics for each job
console.info('Scheduled job completed:', {
  job: 'filing_due_soon',
  eventsCreated: created,
  durationMs: Date.now() - startTime,
  timestamp: new Date().toISOString(),
});
```

### Alerting Thresholds

| Metric | Warning | Critical |
|--------|---------|----------|
| Job duration | > 30s | > 60s |
| Events created | > 1000 | > 5000 |
| Job failures | 1 per day | 3 per day |

