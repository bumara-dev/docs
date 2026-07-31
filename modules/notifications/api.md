---
title: "Notifications API"
description: "REST API endpoints for notification management."
---

## Table of Contents

1. [Overview](#overview)
2. [Authentication](#authentication)
3. [Endpoints](#endpoints)
4. [Request/Response Schemas](#requestresponse-schemas)
5. [Implementation](#implementation)

---

## Overview

The Notifications API provides endpoints for:

- Fetching user notifications
- Marking notifications as read
- Managing notification preferences

All endpoints are tenant-scoped and require authentication.

---

## Authentication

All endpoints require a valid Clerk session:

```typescript
// Headers
Authorization: Bearer <clerk_session_token>
```

The `tenantId` and `userId` are extracted from the session claims.

---

## Endpoints

### Notifications

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/notifications` | List user notifications |
| GET | `/notifications/unread-count` | Get unread count |
| POST | `/notifications/:id/read` | Mark single notification read |
| POST | `/notifications/read-all` | Mark all notifications read |
| DELETE | `/notifications/:id` | Archive notification |

### Preferences

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/notification-preferences` | Get user preferences |
| PUT | `/notification-preferences` | Update user preferences |

---

## Request/Response Schemas

### List Notifications

**Request:**

```typescript
GET /notifications?page=1&limit=20&unreadOnly=false&category=FILING
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| page | number | 1 | Page number |
| limit | number | 20 | Items per page (max 100) |
| unreadOnly | boolean | false | Filter unread only |
| category | string | - | Filter by category |

**Response:**

```typescript
{
  data: Notification[],
  pagination: {
    page: number,
    limit: number,
    total: number,
    totalPages: number,
    hasMore: boolean
  }
}

interface Notification {
  id: string;
  category: 'TASK' | 'FILING' | 'PAYMENT' | 'SUBMISSION' | 'SYSTEM';
  severity: 'INFO' | 'IMPORTANT' | 'URGENT';
  title: string;
  body: string;
  deepLink: string | null;
  entityType: string | null;
  entityId: string | null;
  readAt: string | null;
  createdAt: string;
}
```

### Unread Count

**Response:**

```typescript
{
  count: number,
  byCategory: {
    TASK: number,
    FILING: number,
    PAYMENT: number,
    SUBMISSION: number,
    SYSTEM: number
  }
}
```

### Mark Read

**Request:**

```typescript
POST /notifications/:id/read
// No body required
```

**Response:**

```typescript
{
  success: true,
  notification: Notification
}
```

### Mark All Read

**Request:**

```typescript
POST /notifications/read-all
{
  category?: string  // Optional: only mark specific category
}
```

**Response:**

```typescript
{
  success: true,
  count: number  // Number of notifications marked read
}
```

### Get Preferences

**Response:**

```typescript
{
  emailEnabled: boolean,
  whatsAppEnabled: boolean,
  smsEnabled: boolean,
  severityChannels: {
    INFO: string[],
    IMPORTANT: string[],
    URGENT: string[]
  },
  categoryEnabled: Record<string, boolean>,
  quietHoursEnabled: boolean,
  quietHoursStart: string | null,
  quietHoursEnd: string | null,
  timezone: string
}
```

### Update Preferences

**Request:**

```typescript
PUT /notification-preferences
{
  emailEnabled?: boolean,
  whatsAppEnabled?: boolean,
  smsEnabled?: boolean,
  severityChannels?: Record<string, string[]>,
  categoryEnabled?: Record<string, boolean>,
  quietHoursEnabled?: boolean,
  quietHoursStart?: string,
  quietHoursEnd?: string,
  timezone?: string
}
```

**Response:**

```typescript
{
  success: true,
  preferences: NotificationPreferences
}
```

---

## Implementation

### Routes

```typescript
// packages/backend/src/modules/notifications/notifications.routes.ts

import { createRoute } from '@hono/zod-openapi';
import { z } from 'zod';

// Schema definitions
const notificationSchema = z.object({
  id: z.string().uuid(),
  category: z.enum(['TASK', 'FILING', 'PAYMENT', 'SUBMISSION', 'SYSTEM']),
  severity: z.enum(['INFO', 'IMPORTANT', 'URGENT']),
  title: z.string(),
  body: z.string(),
  deepLink: z.string().nullable(),
  entityType: z.string().nullable(),
  entityId: z.string().nullable(),
  readAt: z.string().nullable(),
  createdAt: z.string(),
});

const paginationSchema = z.object({
  page: z.number(),
  limit: z.number(),
  total: z.number(),
  totalPages: z.number(),
  hasMore: z.boolean(),
});

// List notifications
export const listNotificationsRoute = createRoute({
  method: 'get',
  path: '/notifications',
  tags: ['Notifications'],
  request: {
    query: z.object({
      page: z.coerce.number().min(1).default(1),
      limit: z.coerce.number().min(1).max(100).default(20),
      unreadOnly: z.coerce.boolean().default(false),
      category: z.enum(['TASK', 'FILING', 'PAYMENT', 'SUBMISSION', 'SYSTEM']).optional(),
    }),
  },
  responses: {
    200: {
      description: 'List of notifications',
      content: {
        'application/json': {
          schema: z.object({
            data: z.array(notificationSchema),
            pagination: paginationSchema,
          }),
        },
      },
    },
  },
});

// Unread count
export const unreadCountRoute = createRoute({
  method: 'get',
  path: '/notifications/unread-count',
  tags: ['Notifications'],
  responses: {
    200: {
      description: 'Unread notification count',
      content: {
        'application/json': {
          schema: z.object({
            count: z.number(),
            byCategory: z.record(z.number()),
          }),
        },
      },
    },
  },
});

// Mark read
export const markReadRoute = createRoute({
  method: 'post',
  path: '/notifications/{id}/read',
  tags: ['Notifications'],
  request: {
    params: z.object({
      id: z.string().uuid(),
    }),
  },
  responses: {
    200: {
      description: 'Notification marked as read',
      content: {
        'application/json': {
          schema: z.object({
            success: z.boolean(),
            notification: notificationSchema,
          }),
        },
      },
    },
    404: {
      description: 'Notification not found',
    },
  },
});

// Mark all read
export const markAllReadRoute = createRoute({
  method: 'post',
  path: '/notifications/read-all',
  tags: ['Notifications'],
  request: {
    body: {
      content: {
        'application/json': {
          schema: z.object({
            category: z.enum(['TASK', 'FILING', 'PAYMENT', 'SUBMISSION', 'SYSTEM']).optional(),
          }),
        },
      },
    },
  },
  responses: {
    200: {
      description: 'All notifications marked as read',
      content: {
        'application/json': {
          schema: z.object({
            success: z.boolean(),
            count: z.number(),
          }),
        },
      },
    },
  },
});
```

### Handlers

```typescript
// packages/backend/src/modules/notifications/notifications.handlers.ts

import type { RouteHandler } from '@hono/zod-openapi';
import { createDb } from '@repo/database';
import { notifications } from '@repo/database';
import { and, eq, isNull, desc, sql, count } from 'drizzle-orm';
import type {
  listNotificationsRoute,
  unreadCountRoute,
  markReadRoute,
  markAllReadRoute,
} from './notifications.routes';

export const listNotifications: RouteHandler<typeof listNotificationsRoute> = async (c) => {
  const { orgId, userId } = c.get('auth');
  const db = createDb(c.env.DATABASE_URL);
  const { page, limit, unreadOnly, category } = c.req.valid('query');
  
  const offset = (page - 1) * limit;
  
  // Build where conditions
  const conditions = [
    eq(notifications.tenantId, orgId),
    eq(notifications.userId, userId),
    isNull(notifications.archivedAt),
  ];
  
  if (unreadOnly) {
    conditions.push(isNull(notifications.readAt));
  }
  
  if (category) {
    conditions.push(eq(notifications.category, category));
  }
  
  // Get total count
  const [{ total }] = await db
    .select({ total: count() })
    .from(notifications)
    .where(and(...conditions));
  
  // Get paginated results
  const data = await db
    .select()
    .from(notifications)
    .where(and(...conditions))
    .orderBy(desc(notifications.createdAt))
    .limit(limit)
    .offset(offset);
  
  const totalPages = Math.ceil(total / limit);
  
  return c.json({
    data: data.map(mapNotification),
    pagination: {
      page,
      limit,
      total,
      totalPages,
      hasMore: page < totalPages,
    },
  });
};

export const getUnreadCount: RouteHandler<typeof unreadCountRoute> = async (c) => {
  const { orgId, userId } = c.get('auth');
  const db = createDb(c.env.DATABASE_URL);
  
  const results = await db
    .select({
      category: notifications.category,
      count: count(),
    })
    .from(notifications)
    .where(
      and(
        eq(notifications.tenantId, orgId),
        eq(notifications.userId, userId),
        isNull(notifications.readAt),
        isNull(notifications.archivedAt)
      )
    )
    .groupBy(notifications.category);
  
  const byCategory: Record<string, number> = {
    TASK: 0,
    FILING: 0,
    PAYMENT: 0,
    SUBMISSION: 0,
    SYSTEM: 0,
  };
  
  let totalCount = 0;
  
  for (const row of results) {
    byCategory[row.category] = row.count;
    totalCount += row.count;
  }
  
  return c.json({
    count: totalCount,
    byCategory,
  });
};

export const markRead: RouteHandler<typeof markReadRoute> = async (c) => {
  const { orgId, userId } = c.get('auth');
  const db = createDb(c.env.DATABASE_URL);
  const { id } = c.req.valid('param');
  
  const [notification] = await db
    .update(notifications)
    .set({ readAt: new Date(), updatedAt: new Date() })
    .where(
      and(
        eq(notifications.id, id),
        eq(notifications.tenantId, orgId),
        eq(notifications.userId, userId)
      )
    )
    .returning();
  
  if (!notification) {
    return c.json({ error: 'Notification not found' }, 404);
  }
  
  return c.json({
    success: true,
    notification: mapNotification(notification),
  });
};

export const markAllRead: RouteHandler<typeof markAllReadRoute> = async (c) => {
  const { orgId, userId } = c.get('auth');
  const db = createDb(c.env.DATABASE_URL);
  const { category } = c.req.valid('json');
  
  const conditions = [
    eq(notifications.tenantId, orgId),
    eq(notifications.userId, userId),
    isNull(notifications.readAt),
  ];
  
  if (category) {
    conditions.push(eq(notifications.category, category));
  }
  
  const result = await db
    .update(notifications)
    .set({ readAt: new Date(), updatedAt: new Date() })
    .where(and(...conditions));
  
  return c.json({
    success: true,
    count: result.rowCount ?? 0,
  });
};

function mapNotification(row: typeof notifications.$inferSelect) {
  return {
    id: row.id,
    category: row.category,
    severity: row.severity,
    title: row.title,
    body: row.body,
    deepLink: row.deepLink,
    entityType: row.entityType,
    entityId: row.entityId,
    readAt: row.readAt?.toISOString() ?? null,
    createdAt: row.createdAt.toISOString(),
  };
}
```

### Preferences Handlers

```typescript
// packages/backend/src/modules/notifications/preferences.handlers.ts

import type { RouteHandler } from '@hono/zod-openapi';
import { createDb } from '@repo/database';
import { notificationPreferences } from '@repo/database';
import { and, eq } from 'drizzle-orm';

export const getPreferences: RouteHandler<typeof getPreferencesRoute> = async (c) => {
  const { orgId, userId } = c.get('auth');
  const db = createDb(c.env.DATABASE_URL);
  
  let [prefs] = await db
    .select()
    .from(notificationPreferences)
    .where(
      and(
        eq(notificationPreferences.tenantId, orgId),
        eq(notificationPreferences.userId, userId)
      )
    )
    .limit(1);
  
  // Return defaults if no preferences exist
  if (!prefs) {
    prefs = getDefaultPreferences(orgId, userId);
  }
  
  return c.json({
    emailEnabled: prefs.emailEnabled,
    whatsAppEnabled: prefs.whatsAppEnabled,
    smsEnabled: prefs.smsEnabled,
    severityChannels: prefs.severityChannels,
    categoryEnabled: prefs.categoryEnabled,
    quietHoursEnabled: prefs.quietHoursEnabled,
    quietHoursStart: prefs.quietHoursStart,
    quietHoursEnd: prefs.quietHoursEnd,
    timezone: prefs.timezone,
  });
};

export const updatePreferences: RouteHandler<typeof updatePreferencesRoute> = async (c) => {
  const { orgId, userId } = c.get('auth');
  const db = createDb(c.env.DATABASE_URL);
  const updates = c.req.valid('json');
  
  const [prefs] = await db
    .insert(notificationPreferences)
    .values({
      tenantId: orgId,
      userId,
      ...updates,
    })
    .onConflictDoUpdate({
      target: [notificationPreferences.tenantId, notificationPreferences.userId],
      set: {
        ...updates,
        updatedAt: new Date(),
      },
    })
    .returning();
  
  return c.json({
    success: true,
    preferences: prefs,
  });
};

function getDefaultPreferences(tenantId: string, userId: string) {
  return {
    tenantId,
    userId,
    emailEnabled: true,
    whatsAppEnabled: true,
    smsEnabled: false,
    severityChannels: {
      INFO: ['IN_APP'],
      IMPORTANT: ['IN_APP', 'EMAIL', 'WHATSAPP'],
      URGENT: ['IN_APP', 'EMAIL', 'WHATSAPP'],
    },
    categoryEnabled: {},
    quietHoursEnabled: false,
    quietHoursStart: null,
    quietHoursEnd: null,
    timezone: 'Africa/Lusaka',
  };
}
```

### Route Registration

```typescript
// packages/backend/src/modules/notifications/index.ts

import { createRouter } from '@/core/http/create-app';
import { requireAuth } from '@/core/middleware/auth';
import * as routes from './notifications.routes';
import * as handlers from './notifications.handlers';
import * as prefHandlers from './preferences.handlers';

const notificationsRouter = createRouter()
  .use('*', requireAuth)
  .openapi(routes.listNotificationsRoute, handlers.listNotifications)
  .openapi(routes.unreadCountRoute, handlers.getUnreadCount)
  .openapi(routes.markReadRoute, handlers.markRead)
  .openapi(routes.markAllReadRoute, handlers.markAllRead)
  .openapi(routes.getPreferencesRoute, prefHandlers.getPreferences)
  .openapi(routes.updatePreferencesRoute, prefHandlers.updatePreferences);

export default notificationsRouter;
```

Add to module index:

```typescript
// packages/backend/src/modules/index.ts

import notifications from './notifications';

export const router = createRouter()
  // ... existing routes
  .route('/', notifications);
```

