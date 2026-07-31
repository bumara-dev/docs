---
title: "Notifications Data Model"
description: "Database schema and migrations for the notification system."
---

## Table of Contents

1. [Overview](#overview)
2. [Tables](#tables)
3. [Enums](#enums)
4. [Indexes](#indexes)
5. [Migration Guide](#migration-guide)

---

## Overview

The notification system uses four core tables:

| Table | Purpose |
|-------|---------|
| `event_outbox` | Transactional outbox for reliable event processing |
| `notifications` | In-app notification feed (source of truth) |
| `notification_deliveries` | External channel delivery tracking |
| `notification_preferences` | User-level channel preferences |

---

## Tables

### 2.1 event_outbox

The outbox table ensures reliable event processing using the transactional outbox pattern.

```typescript
// packages/database/src/schema/notifications/event-outbox.ts

import {
  index,
  integer,
  jsonb,
  pgTable,
  text,
  timestamp,
  uuid,
} from 'drizzle-orm/pg-core';
import { timestamps } from '../../helpers/timestamps';
import { organizations } from '../core';
import { outboxStatusEnum } from './enums';

export const eventOutbox = pgTable(
  'event_outbox',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    
    // Tenant isolation aka organization_id
    tenantId: text('tenant_id')
      .notNull()
      .references(() => organizations.id, { onDelete: 'cascade' }),
    
    // Event identification
    eventType: text('event_type').notNull(),
    
    // Event payload (JSON)
    payload: jsonb('payload')
      .$type<EventPayload>()
      .notNull(),
    
    // Processing state
    status: outboxStatusEnum('status').notNull().default('pending'),
    
    // Retry handling
    attempts: integer('attempts').notNull().default(0),
    maxAttempts: integer('max_attempts').notNull().default(5),
    nextRetryAt: timestamp('next_retry_at', { mode: 'date' }),
    lastError: text('last_error'),
    
    // Timestamps
    processedAt: timestamp('processed_at', { mode: 'date' }),
    ...timestamps,
  },
  (table) => [
    // Primary query: pending events ready for processing
    index('idx_event_outbox_status_retry').on(
      table.status,
      table.nextRetryAt
    ),
    // Tenant + time for audit/debugging
    index('idx_event_outbox_tenant_created').on(
      table.tenantId,
      table.createdAt
    ),
    // Event type filtering
    index('idx_event_outbox_event_type').on(table.eventType),
  ]
);

// Type for the payload column
interface EventPayload {
  entityType: string;
  entityId: string;
  title: string;
  message: string;
  deepLink: string;
  severity: 'INFO' | 'IMPORTANT' | 'URGENT';
  metadata?: Record<string, unknown>;
}
```

### 2.2 notifications

The in-app notification feed table.

```typescript
// packages/database/src/schema/notifications/notifications.ts

import {
  index,
  jsonb,
  pgTable,
  text,
  timestamp,
  uuid,
} from 'drizzle-orm/pg-core';
import { timestamps } from '../../helpers/timestamps';
import { organizations } from '../core';
import { users } from '../core/user';
import { notificationSeverityEnum, notificationCategoryEnum } from './enums';

export const notifications = pgTable(
  'notifications',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    
    // Tenant isolation aka organization_id
    tenantId: text('tenant_id')
      .notNull()
      .references(() => organizations.id, { onDelete: 'cascade' }),
    
    // Recipient
    userId: text('user_id')
      .notNull()
      .references(() => users.id, { onDelete: 'cascade' }),
    
    // Classification
    category: notificationCategoryEnum('category').notNull(),
    severity: notificationSeverityEnum('severity').notNull().default('INFO'),
    
    // Content
    title: text('title').notNull(),
    body: text('body').notNull(),
    
    // Navigation
    deepLink: text('deep_link'),
    
    // Entity reference (for deduplication and grouping)
    entityType: text('entity_type'),
    entityId: text('entity_id'),
    
    // Source event (for tracing)
    outboxEventId: uuid('outbox_event_id'),
    
    // Read state
    readAt: timestamp('read_at', { mode: 'date' }),
    
    // Archive state (soft delete from feed)
    archivedAt: timestamp('archived_at', { mode: 'date' }),
    
    // Additional data for UI
    metadata: jsonb('metadata').$type<Record<string, unknown>>(),
    
    ...timestamps,
  },
  (table) => [
    // Primary query: user's notification feed
    index('idx_notifications_tenant_user_created').on(
      table.tenantId,
      table.userId,
      table.createdAt
    ),
    // Unread notifications
    index('idx_notifications_tenant_user_read').on(
      table.tenantId,
      table.userId,
      table.readAt
    ),
    // Entity grouping (for "view all" by entity)
    index('idx_notifications_entity').on(
      table.tenantId,
      table.entityType,
      table.entityId
    ),
  ]
);
```

### 2.3 notification_deliveries

Tracks external channel delivery attempts and status.

```typescript
// packages/database/src/schema/notifications/notification-deliveries.ts

import {
  index,
  integer,
  jsonb,
  pgTable,
  text,
  timestamp,
  unique,
  uuid,
} from 'drizzle-orm/pg-core';
import { timestamps } from '../../helpers/timestamps';
import { organizations } from '../core';
import { notifications } from './notifications';
import { 
  deliveryChannelEnum, 
  deliveryStatusEnum, 
  deliveryProviderEnum 
} from './enums';

export const notificationDeliveries = pgTable(
  'notification_deliveries',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    
    // Link to notification
    notificationId: uuid('notification_id')
      .notNull()
      .references(() => notifications.id, { onDelete: 'cascade' }),
    
    // Tenant isolation
    tenantId: text('tenant_id')
      .notNull()
      .references(() => organizations.id, { onDelete: 'cascade' }),
    
    // Channel configuration
    channel: deliveryChannelEnum('channel').notNull(),
    provider: deliveryProviderEnum('provider').notNull(),
    
    // Destination
    to: text('to').notNull(), // email address or E.164 phone
    
    // Provider tracking
    providerMessageId: text('provider_message_id'),
    
    // Status tracking
    status: deliveryStatusEnum('status').notNull().default('queued'),
    
    // Retry handling
    attempts: integer('attempts').notNull().default(0),
    maxAttempts: integer('max_attempts').notNull().default(5),
    nextRetryAt: timestamp('next_retry_at', { mode: 'date' }),
    lastError: text('last_error'),
    
    // Webhook data (for debugging)
    webhookPayload: jsonb('webhook_payload').$type<Record<string, unknown>>(),
    
    // Timestamps
    sentAt: timestamp('sent_at', { mode: 'date' }),
    deliveredAt: timestamp('delivered_at', { mode: 'date' }),
    failedAt: timestamp('failed_at', { mode: 'date' }),
    ...timestamps,
  },
  (table) => [
    // Idempotency: one delivery per notification per channel
    unique('uq_notification_deliveries_notification_channel').on(
      table.notificationId,
      table.channel
    ),
    
    // Primary query: pending deliveries
    index('idx_notification_deliveries_status_retry').on(
      table.status,
      table.nextRetryAt
    ),
    
    // Provider message lookup (for webhooks)
    index('idx_notification_deliveries_provider_msg').on(
      table.provider,
      table.providerMessageId
    ),
    
    // Tenant + channel for analytics
    index('idx_notification_deliveries_tenant_channel').on(
      table.tenantId,
      table.channel,
      table.status
    ),
  ]
);
```

### 2.4 notification_preferences

User-level notification preferences.

```typescript
// packages/database/src/schema/notifications/notification-preferences.ts

import {
  boolean,
  integer,
  jsonb,
  pgTable,
  text,
  time,
  unique,
  uuid,
} from 'drizzle-orm/pg-core';
import { timestamps } from '../../helpers/timestamps';
import { organizations } from '../core';
import { users } from '../core/user';

export const notificationPreferences = pgTable(
  'notification_preferences',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    
    // Tenant + User
    tenantId: text('tenant_id')
      .notNull()
      .references(() => organizations.id, { onDelete: 'cascade' }),
    userId: text('user_id')
      .notNull()
      .references(() => users.id, { onDelete: 'cascade' }),
    
    // Channel toggles
    emailEnabled: boolean('email_enabled').notNull().default(true),
    whatsAppEnabled: boolean('whatsapp_enabled').notNull().default(true),
    smsEnabled: boolean('sms_enabled').notNull().default(false),
    
    // Severity filters (JSON object for flexibility)
    // { "INFO": ["IN_APP"], "IMPORTANT": ["IN_APP", "EMAIL"], "URGENT": ["IN_APP", "EMAIL", "WHATSAPP"] }
    severityChannels: jsonb('severity_channels')
      .$type<Record<string, string[]>>()
      .default({
        INFO: ['IN_APP'],
        IMPORTANT: ['IN_APP', 'EMAIL', 'WHATSAPP'],
        URGENT: ['IN_APP', 'EMAIL', 'WHATSAPP'],
      }),
    
    // Category toggles (opt-out specific categories)
    // { "TASK_ASSIGNED": false, "FILING_DUE_SOON": true }
    categoryEnabled: jsonb('category_enabled')
      .$type<Record<string, boolean>>()
      .default({}),
    
    // Quiet hours (optional)
    quietHoursEnabled: boolean('quiet_hours_enabled').notNull().default(false),
    quietHoursStart: time('quiet_hours_start'), // e.g., "22:00"
    quietHoursEnd: time('quiet_hours_end'),     // e.g., "07:00"
    timezone: text('timezone').default('Africa/Lusaka'),
    
    // Digest preferences (future)
    digestEnabled: boolean('digest_enabled').notNull().default(false),
    digestFrequency: text('digest_frequency').default('daily'), // daily | weekly
    
    ...timestamps,
  },
  (table) => [
    // One preference record per user per tenant
    unique('uq_notification_preferences_tenant_user').on(
      table.tenantId,
      table.userId
    ),
  ]
);
```

---

## Enums

```typescript
// packages/database/src/schema/notifications/enums.ts

import { pgEnum } from 'drizzle-orm/pg-core';

// Outbox processing status
export const outboxStatusEnum = pgEnum('outbox_status', [
  'pending',      // Waiting to be processed
  'processing',   // Currently being processed
  'processed',    // Successfully processed
  'failed',       // Failed after max retries (dead letter)
]);

// Notification severity
export const notificationSeverityEnum = pgEnum('notification_severity', [
  'INFO',         // Low priority, in-app only
  'IMPORTANT',    // Medium priority, in-app + email + whatsapp
  'URGENT',       // High priority, all channels
]);

// Notification category (for filtering/preferences)
export const notificationCategoryEnum = pgEnum('notification_category', [
  'TASK',         // Task-related notifications
  'FILING',       // Filing/compliance notifications
  'PAYMENT',      // Payment-related notifications
  'SUBMISSION',   // Submission status notifications
  'SYSTEM',       // System notifications
]);

// Delivery channel
export const deliveryChannelEnum = pgEnum('delivery_channel', [
  'IN_APP',       // In-app notification (always created)
  'EMAIL',        // Email delivery
  'WHATSAPP',     // WhatsApp template message
  'SMS',          // SMS (future)
]);

// Delivery status
export const deliveryStatusEnum = pgEnum('delivery_status', [
  'queued',       // Created, waiting to send
  'sending',      // Currently being sent
  'sent',         // Sent to provider
  'delivered',    // Confirmed delivered (webhook)
  'read',         // Confirmed read (WhatsApp only)
  'failed',       // Permanent failure
  'bounced',      // Email bounced
  'complained',   // Email marked as spam
]);

// Provider (for webhook routing)
export const deliveryProviderEnum = pgEnum('delivery_provider', [
  'RESEND',       // Resend for email
  'META',         // Meta WhatsApp Cloud API
  'TWILIO',       // Twilio (future SMS)
  'AFRICAS_TALKING', // Africa's Talking (future SMS)
]);
```

---

## Indexes

### Performance-Critical Queries

| Query | Index |
|-------|-------|
| Fetch pending outbox events | `idx_event_outbox_status_retry` |
| Fetch user's notification feed | `idx_notifications_tenant_user_created` |
| Count unread notifications | `idx_notifications_tenant_user_read` |
| Fetch pending deliveries | `idx_notification_deliveries_status_retry` |
| Webhook message lookup | `idx_notification_deliveries_provider_msg` |

---

## Migration Guide

### Step 1: Add Enums

```sql
-- Add new enums for notifications
CREATE TYPE outbox_status AS ENUM ('pending', 'processing', 'processed', 'failed');
CREATE TYPE notification_severity AS ENUM ('INFO', 'IMPORTANT', 'URGENT');
CREATE TYPE notification_category AS ENUM ('TASK', 'FILING', 'PAYMENT', 'SUBMISSION', 'SYSTEM');
CREATE TYPE delivery_channel AS ENUM ('IN_APP', 'EMAIL', 'WHATSAPP', 'SMS');
CREATE TYPE delivery_status AS ENUM ('queued', 'sending', 'sent', 'delivered', 'read', 'failed', 'bounced', 'complained');
CREATE TYPE delivery_provider AS ENUM ('RESEND', 'META', 'TWILIO', 'AFRICAS_TALKING');
```

### Step 2: Create Tables

Run Drizzle migration:

```bash
cd packages/database
pnpm drizzle-kit generate
pnpm drizzle-kit migrate
```

### Step 3: Migrate Existing Notifications Table

The existing `notifications` table needs to be enhanced:

```sql
-- Add new columns to existing notifications table
ALTER TABLE notifications
  ADD COLUMN category notification_category,
  ADD COLUMN severity notification_severity DEFAULT 'INFO',
  ADD COLUMN entity_type TEXT,
  ADD COLUMN entity_id TEXT,
  ADD COLUMN outbox_event_id UUID,
  ADD COLUMN archived_at TIMESTAMP,
  ADD COLUMN metadata JSONB;

-- Rename columns for consistency
ALTER TABLE notifications
  RENAME COLUMN organization_id TO tenant_id;
ALTER TABLE notifications
  RENAME COLUMN action_url TO deep_link;
ALTER TABLE notifications
  RENAME COLUMN message TO body;

-- Drop deprecated columns
ALTER TABLE notifications
  DROP COLUMN is_read,
  DROP COLUMN type,
  DROP COLUMN priority;

-- Backfill category (based on old type if available)
UPDATE notifications SET category = 'SYSTEM' WHERE category IS NULL;
ALTER TABLE notifications ALTER COLUMN category SET NOT NULL;

-- Add new indexes
CREATE INDEX idx_notifications_tenant_user_created ON notifications(tenant_id, user_id, created_at DESC);
CREATE INDEX idx_notifications_tenant_user_read ON notifications(tenant_id, user_id, read_at);
CREATE INDEX idx_notifications_entity ON notifications(tenant_id, entity_type, entity_id);
```

---

## Schema Export

```typescript
// packages/database/src/schema/notifications/index.ts

export * from './enums';
export * from './event-outbox';
export * from './notifications';
export * from './notification-deliveries';
export * from './notification-preferences';
```

Add to main schema index:

```typescript
// packages/database/src/schema/index.ts

// ... existing exports
export * from './notifications';
```

