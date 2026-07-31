---
title: "Notifications Workers"
description: "Outbox processor and delivery sender workers for reliable notification processing."
---

## Table of Contents

1. [Overview](#overview)
2. [Cloudflare Queues Configuration](#cloudflare-queues-configuration)
3. [Outbox Processor Worker](#outbox-processor-worker)
4. [Delivery Sender Worker](#delivery-sender-worker)
5. [Retry Strategy](#retry-strategy)
6. [Idempotency](#idempotency)
7. [Error Handling](#error-handling)

---

## Overview

The notification system uses two workers:

| Worker | Queue | Purpose |
|--------|-------|---------|
| Outbox Processor | `notification-outbox` | Processes events from outbox, creates notifications + deliveries |
| Delivery Sender | `notification-delivery` | Sends messages via channel adapters |

### Processing Flow

```
event_outbox (DB)
     │
     │ (trigger: DB insert or cron poll)
     ▼
notification-outbox (Queue)
     │
     ▼
Outbox Processor Worker
     │
     ├── Create notifications rows
     ├── Create notification_deliveries rows
     │
     ▼
notification-delivery (Queue)
     │
     ▼
Delivery Sender Worker
     │
     ├── Call Email Adapter
     ├── Call WhatsApp Adapter
     │
     ▼
Provider APIs (Resend, Meta)
```

---

## Cloudflare Queues Configuration

### Wrangler Configuration

```jsonc
// packages/backend/wrangler.jsonc

{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "bumara-api",
  "main": "src/index.ts",
  "compatibility_date": "2025-09-22",
  "compatibility_flags": ["nodejs_compat"],
  
  // Queue bindings
  "queues": {
    "producers": [
      {
        "queue": "notification-outbox",
        "binding": "OUTBOX_QUEUE"
      },
      {
        "queue": "notification-delivery",
        "binding": "DELIVERY_QUEUE"
      }
    ],
    "consumers": [
      {
        "queue": "notification-outbox",
        "max_batch_size": 10,
        "max_batch_timeout": 30,
        "max_retries": 5,
        "dead_letter_queue": "notification-dlq"
      },
      {
        "queue": "notification-delivery",
        "max_batch_size": 10,
        "max_batch_timeout": 30,
        "max_retries": 5,
        "dead_letter_queue": "notification-dlq"
      }
    ]
  },
  
  // Scheduled triggers for cron jobs
  "triggers": {
    "crons": [
      "0 6 * * *",  // 06:00 UTC - Filing reminders
      "0 7 * * *",  // 07:00 UTC - Overdue check
      "*/5 * * * *" // Every 5 min - Outbox poll (backup)
    ]
  }
}
```

### Environment Types

```typescript
// packages/backend/src/env.ts

export interface Env {
  // Database
  DATABASE_URL: string;
  
  // Queues
  OUTBOX_QUEUE: Queue<OutboxQueueMessage>;
  DELIVERY_QUEUE: Queue<DeliveryQueueMessage>;
  
  // Providers
  RESEND_TOKEN: string;
  RESEND_FROM: string;
  WHATSAPP_PHONE_NUMBER_ID: string;
  WHATSAPP_ACCESS_TOKEN: string;
}

export interface OutboxQueueMessage {
  outboxEventId: string;
  tenantId: string;
  eventType: string;
  attempt: number;
}

export interface DeliveryQueueMessage {
  deliveryId: string;
  notificationId: string;
  tenantId: string;
  channel: string;
  attempt: number;
}
```

---

## Outbox Processor Worker

### Implementation

```typescript
// packages/backend/src/queues/outbox-processor.ts

import type { MessageBatch, ExecutionContext } from '@cloudflare/workers-types';
import { createDb } from '@repo/database';
import { eventOutbox, notifications, notificationDeliveries } from '@repo/database';
import { eq, and } from 'drizzle-orm';
import { resolveRecipients, resolveChannels } from '@repo/api-services/notifications';
import type { Env, OutboxQueueMessage, DeliveryQueueMessage } from '../env';

export async function processOutboxQueue(
  batch: MessageBatch<OutboxQueueMessage>,
  env: Env,
  ctx: ExecutionContext
): Promise<void> {
  const db = createDb(env.DATABASE_URL);
  
  for (const message of batch.messages) {
    try {
      await processOutboxEvent(db, env, message.body);
      message.ack();
    } catch (error) {
      console.error('Outbox processing failed:', {
        outboxEventId: message.body.outboxEventId,
        error: error instanceof Error ? error.message : 'Unknown error',
        attempt: message.body.attempt,
      });
      
      // Retry with backoff
      if (message.body.attempt < 5) {
        message.retry({
          delaySeconds: getBackoffDelay(message.body.attempt),
        });
      } else {
        // Mark as failed in DB
        await markOutboxFailed(db, message.body.outboxEventId, error);
        message.ack(); // Ack to prevent infinite retry
      }
    }
  }
}

async function processOutboxEvent(
  db: ReturnType<typeof createDb>,
  env: Env,
  message: OutboxQueueMessage
): Promise<void> {
  // 1. Fetch outbox event
  const [outboxEvent] = await db
    .select()
    .from(eventOutbox)
    .where(eq(eventOutbox.id, message.outboxEventId))
    .limit(1);
  
  if (!outboxEvent) {
    console.warn('Outbox event not found:', message.outboxEventId);
    return;
  }
  
  // 2. Idempotency check - already processed?
  if (outboxEvent.status === 'processed') {
    console.info('Outbox event already processed:', message.outboxEventId);
    return;
  }
  
  // 3. Mark as processing
  const [updated] = await db
    .update(eventOutbox)
    .set({ status: 'processing', updatedAt: new Date() })
    .where(
      and(
        eq(eventOutbox.id, message.outboxEventId),
        eq(eventOutbox.status, 'pending') // Optimistic lock
      )
    )
    .returning();
  
  if (!updated) {
    console.info('Outbox event already being processed:', message.outboxEventId);
    return;
  }
  
  // 4. Resolve recipients
  const recipients = await resolveRecipients(db, outboxEvent.payload);
  
  if (recipients.length === 0) {
    console.info('No recipients for event:', message.outboxEventId);
    await markOutboxProcessed(db, message.outboxEventId);
    return;
  }
  
  // 5. Create notifications and deliveries
  const deliveryMessages: DeliveryQueueMessage[] = [];
  
  await db.transaction(async (tx) => {
    for (const recipient of recipients) {
      // Create notification (in-app)
      const [notification] = await tx
        .insert(notifications)
        .values({
          tenantId: outboxEvent.tenantId,
          userId: recipient.userId,
          category: outboxEvent.payload.category,
          severity: outboxEvent.payload.severity,
          title: outboxEvent.payload.title,
          body: outboxEvent.payload.message,
          deepLink: outboxEvent.payload.deepLink,
          entityType: outboxEvent.payload.entityType,
          entityId: outboxEvent.payload.entityId,
          outboxEventId: outboxEvent.id,
        })
        .onConflictDoNothing() // Idempotency
        .returning();
      
      if (!notification) {
        // Already exists, skip
        continue;
      }
      
      // Resolve channels for this recipient
      const channels = resolveChannels(outboxEvent.payload, recipient);
      
      // Create delivery records for external channels
      for (const channel of channels) {
        if (channel === 'IN_APP') continue; // Already handled by notification
        
        const to = channel === 'EMAIL' ? recipient.email : recipient.phone;
        if (!to) continue;
        
        const [delivery] = await tx
          .insert(notificationDeliveries)
          .values({
            notificationId: notification.id,
            tenantId: outboxEvent.tenantId,
            channel,
            provider: getProviderForChannel(channel),
            to,
            status: 'queued',
          })
          .onConflictDoNothing() // Idempotency
          .returning();
        
        if (delivery) {
          deliveryMessages.push({
            deliveryId: delivery.id,
            notificationId: notification.id,
            tenantId: outboxEvent.tenantId,
            channel,
            attempt: 0,
          });
        }
      }
    }
    
    // Mark outbox as processed
    await tx
      .update(eventOutbox)
      .set({ status: 'processed', processedAt: new Date(), updatedAt: new Date() })
      .where(eq(eventOutbox.id, message.outboxEventId));
  });
  
  // 6. Enqueue deliveries
  for (const deliveryMessage of deliveryMessages) {
    await env.DELIVERY_QUEUE.send(deliveryMessage);
  }
  
  console.info('Outbox processed:', {
    outboxEventId: message.outboxEventId,
    recipientCount: recipients.length,
    deliveryCount: deliveryMessages.length,
  });
}

function getProviderForChannel(channel: string): string {
  switch (channel) {
    case 'EMAIL':
      return 'RESEND';
    case 'WHATSAPP':
      return 'META';
    case 'SMS':
      return 'AFRICAS_TALKING';
    default:
      return 'UNKNOWN';
  }
}

function getBackoffDelay(attempt: number): number {
  // Exponential backoff: 60s, 300s, 1200s, 3600s, 21600s
  const delays = [60, 300, 1200, 3600, 21600];
  return delays[Math.min(attempt, delays.length - 1)];
}

async function markOutboxProcessed(
  db: ReturnType<typeof createDb>,
  outboxEventId: string
): Promise<void> {
  await db
    .update(eventOutbox)
    .set({ status: 'processed', processedAt: new Date(), updatedAt: new Date() })
    .where(eq(eventOutbox.id, outboxEventId));
}

async function markOutboxFailed(
  db: ReturnType<typeof createDb>,
  outboxEventId: string,
  error: unknown
): Promise<void> {
  await db
    .update(eventOutbox)
    .set({
      status: 'failed',
      lastError: error instanceof Error ? error.message : 'Unknown error',
      updatedAt: new Date(),
    })
    .where(eq(eventOutbox.id, outboxEventId));
}
```

---

## Delivery Sender Worker

### Implementation

```typescript
// packages/backend/src/queues/delivery-sender.ts

import type { MessageBatch, ExecutionContext } from '@cloudflare/workers-types';
import { createDb } from '@repo/database';
import { notifications, notificationDeliveries } from '@repo/database';
import { eq, and } from 'drizzle-orm';
import { sendEmail } from '@repo/notifications/adapters/email';
import { sendWhatsAppTemplate } from '@repo/notifications/adapters/whatsapp';
import type { Env, DeliveryQueueMessage } from '../env';

export async function processDeliveryQueue(
  batch: MessageBatch<DeliveryQueueMessage>,
  env: Env,
  ctx: ExecutionContext
): Promise<void> {
  const db = createDb(env.DATABASE_URL);
  
  for (const message of batch.messages) {
    try {
      await processDelivery(db, env, message.body);
      message.ack();
    } catch (error) {
      console.error('Delivery failed:', {
        deliveryId: message.body.deliveryId,
        channel: message.body.channel,
        error: error instanceof Error ? error.message : 'Unknown error',
        attempt: message.body.attempt,
      });
      
      // Check if retryable
      const isRetryable = isRetryableError(error);
      
      if (isRetryable && message.body.attempt < 5) {
        message.retry({
          delaySeconds: getBackoffDelay(message.body.attempt),
        });
      } else {
        // Mark as failed
        await markDeliveryFailed(db, message.body.deliveryId, error);
        message.ack();
      }
    }
  }
}

async function processDelivery(
  db: ReturnType<typeof createDb>,
  env: Env,
  message: DeliveryQueueMessage
): Promise<void> {
  // 1. Fetch delivery record
  const [delivery] = await db
    .select()
    .from(notificationDeliveries)
    .where(eq(notificationDeliveries.id, message.deliveryId))
    .limit(1);
  
  if (!delivery) {
    console.warn('Delivery not found:', message.deliveryId);
    return;
  }
  
  // 2. Idempotency check
  if (['sent', 'delivered', 'failed', 'bounced'].includes(delivery.status)) {
    console.info('Delivery already processed:', message.deliveryId);
    return;
  }
  
  // 3. Fetch notification for content
  const [notification] = await db
    .select()
    .from(notifications)
    .where(eq(notifications.id, delivery.notificationId))
    .limit(1);
  
  if (!notification) {
    console.warn('Notification not found:', delivery.notificationId);
    await markDeliveryFailed(db, message.deliveryId, new Error('Notification not found'));
    return;
  }
  
  // 4. Mark as sending
  await db
    .update(notificationDeliveries)
    .set({ status: 'sending', attempts: delivery.attempts + 1, updatedAt: new Date() })
    .where(eq(notificationDeliveries.id, message.deliveryId));
  
  // 5. Send via channel adapter
  let providerMessageId: string | undefined;
  
  switch (delivery.channel) {
    case 'EMAIL':
      providerMessageId = await sendEmail(env, {
        to: delivery.to,
        subject: notification.title,
        html: generateEmailHtml(notification),
        text: notification.body,
        metadata: {
          deliveryId: delivery.id,
          notificationId: notification.id,
          tenantId: delivery.tenantId,
        },
      });
      break;
    
    case 'WHATSAPP':
      providerMessageId = await sendWhatsAppTemplate(env, {
        toE164: delivery.to,
        templateName: getWhatsAppTemplate(notification.category, notification.severity),
        language: 'en',
        variables: [notification.title, notification.body],
        metadata: {
          deliveryId: delivery.id,
          notificationId: notification.id,
          tenantId: delivery.tenantId,
        },
      });
      break;
    
    default:
      throw new Error(`Unsupported channel: ${delivery.channel}`);
  }
  
  // 6. Update status to sent
  await db
    .update(notificationDeliveries)
    .set({
      status: 'sent',
      providerMessageId,
      sentAt: new Date(),
      updatedAt: new Date(),
    })
    .where(eq(notificationDeliveries.id, message.deliveryId));
  
  console.info('Delivery sent:', {
    deliveryId: message.deliveryId,
    channel: delivery.channel,
    providerMessageId,
  });
}

function generateEmailHtml(notification: {
  title: string;
  body: string;
  deepLink: string | null;
}): string {
  // Simple email template - should use React Email templates in production
  return `
    <!DOCTYPE html>
    <html>
      <head>
        <meta charset="utf-8">
        <title>${notification.title}</title>
      </head>
      <body style="font-family: sans-serif; padding: 20px;">
        <h2>${notification.title}</h2>
        <p>${notification.body}</p>
        ${notification.deepLink ? `
          <p>
            <a href="${notification.deepLink}" 
               style="background: #2563eb; color: white; padding: 10px 20px; 
                      text-decoration: none; border-radius: 5px;">
              View Details
            </a>
          </p>
        ` : ''}
      </body>
    </html>
  `;
}

function getWhatsAppTemplate(category: string, severity: string): string {
  // Map category + severity to WhatsApp template name
  const templates: Record<string, string> = {
    'TASK_INFO': 'bumara_task_notification',
    'TASK_IMPORTANT': 'bumara_task_urgent',
    'FILING_IMPORTANT': 'bumara_filing_reminder',
    'FILING_URGENT': 'bumara_filing_overdue',
    'PAYMENT_IMPORTANT': 'bumara_payment_required',
    'SUBMISSION_INFO': 'bumara_submission_update',
  };
  
  return templates[`${category}_${severity}`] ?? 'bumara_general_notification';
}

function isRetryableError(error: unknown): boolean {
  if (error instanceof Error) {
    // Rate limits, temporary failures
    const message = error.message.toLowerCase();
    return (
      message.includes('rate limit') ||
      message.includes('timeout') ||
      message.includes('temporarily') ||
      message.includes('503') ||
      message.includes('429')
    );
  }
  return false;
}

function getBackoffDelay(attempt: number): number {
  const delays = [60, 300, 1200, 3600, 21600];
  return delays[Math.min(attempt, delays.length - 1)];
}

async function markDeliveryFailed(
  db: ReturnType<typeof createDb>,
  deliveryId: string,
  error: unknown
): Promise<void> {
  await db
    .update(notificationDeliveries)
    .set({
      status: 'failed',
      failedAt: new Date(),
      lastError: error instanceof Error ? error.message : 'Unknown error',
      updatedAt: new Date(),
    })
    .where(eq(notificationDeliveries.id, deliveryId));
}
```

---

## Retry Strategy

### Backoff Schedule

| Attempt | Delay | Cumulative Time |
|---------|-------|-----------------|
| 1 | Immediate | 0 |
| 2 | 1 minute | 1 min |
| 3 | 5 minutes | 6 min |
| 4 | 20 minutes | 26 min |
| 5 | 1 hour | 1h 26min |
| 6 (max) | 6 hours | Dead letter |

### Dead Letter Queue

Failed messages after max retries go to `notification-dlq` for:
- Manual inspection
- Alerting
- Potential reprocessing

---

## Idempotency

### Guarantees

| Layer | Idempotency Mechanism |
|-------|----------------------|
| Outbox → Notification | Check `outboxEventId` on notification |
| Notification → Delivery | Unique constraint on `(notification_id, channel)` |
| Delivery Status | Check status before processing |
| Queue Message | Queue handles at-least-once delivery |

### Safe Reprocessing

All operations are idempotent - reprocessing the same message will:
1. Skip if already processed (status check)
2. Use `ON CONFLICT DO NOTHING` for inserts
3. Use optimistic locking for status transitions

---

## Error Handling

### Error Categories

| Category | Action | Example |
|----------|--------|---------|
| Transient | Retry with backoff | Rate limit, timeout |
| Permanent | Mark failed, no retry | Invalid email, blocked number |
| Missing Data | Log and skip | Notification not found |
| Configuration | Alert ops, retry later | Invalid API key |

### Logging

```typescript
// Structured logging for observability
console.info('Delivery sent:', {
  deliveryId: 'uuid',
  channel: 'EMAIL',
  providerMessageId: 'resend_msg_123',
  durationMs: 245,
});

console.error('Delivery failed:', {
  deliveryId: 'uuid',
  channel: 'WHATSAPP',
  error: 'Rate limit exceeded',
  attempt: 3,
  willRetry: true,
});
```

---

## Worker Entry Point

```typescript
// packages/backend/src/index.ts

import app from './app';
import { processOutboxQueue } from './queues/outbox-processor';
import { processDeliveryQueue } from './queues/delivery-sender';
import { runScheduledJobs } from './jobs/scheduler';
import type { Env, OutboxQueueMessage, DeliveryQueueMessage } from './env';

export default {
  // HTTP handler
  fetch: app.fetch,
  
  // Queue consumers
  async queue(
    batch: MessageBatch<OutboxQueueMessage | DeliveryQueueMessage>,
    env: Env,
    ctx: ExecutionContext
  ) {
    const queueName = batch.queue;
    
    if (queueName === 'notification-outbox') {
      await processOutboxQueue(batch as MessageBatch<OutboxQueueMessage>, env, ctx);
    } else if (queueName === 'notification-delivery') {
      await processDeliveryQueue(batch as MessageBatch<DeliveryQueueMessage>, env, ctx);
    } else {
      console.warn('Unknown queue:', queueName);
    }
  },
  
  // Scheduled triggers
  async scheduled(
    event: ScheduledEvent,
    env: Env,
    ctx: ExecutionContext
  ) {
    await runScheduledJobs(event, env, ctx);
  },
};
```

