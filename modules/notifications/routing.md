---
title: "Notifications Routing"
description: "Recipient resolution and channel selection logic for the notification system."
---

## Table of Contents

1. [Overview](#overview)
2. [Recipient Resolution](#recipient-resolution)
3. [Channel Resolution](#channel-resolution)
4. [Routing Rules Matrix](#routing-rules-matrix)
5. [Implementation](#implementation)

---

## Overview

The routing module determines:

1. **Who** receives a notification (recipients)
2. **How** they receive it (channels)

All routing decisions respect:
- Tenant isolation (never notify users outside the tenant)
- User preferences (opt-out respected)
- Role-based rules (admins, managers, specific roles)

---

## Recipient Resolution

### Recipient Resolution Flow

```
Event Payload
     │
     ▼
┌─────────────────────────────────────┐
│      resolveRecipients(event)       │
├─────────────────────────────────────┤
│ 1. Extract event type               │
│ 2. Get recipient rules for type     │
│ 3. Query users matching rules       │
│ 4. Filter by tenant membership      │
│ 5. Return recipient list            │
└─────────────────────────────────────┘
     │
     ▼
Recipient[] { userId, email, phone, roles }
```

### Recipient Rules by Event Type

| Event Type | Primary Recipients | Additional Recipients |
|------------|-------------------|----------------------|
| `TASK_ASSIGNED` | Assigned user | - |
| `TASK_ACTION_REQUIRED` | Assigned user | Tenant admins |
| `FILING_DUE_SOON` | - | Admins + Compliance managers |
| `FILING_OVERDUE` | - | Admins + Compliance managers |
| `PAYMENT_REQUIRED` | - | Admins + Finance role |
| `SUBMISSION_STATUS_CHANGED` | Initiator | Admins |

### Recipient Interface

```typescript
interface Recipient {
  userId: string;
  email: string | null;
  phone: string | null; // E.164 format
  roles: string[];
  preferences: NotificationPreferences | null;
}
```

---

## Channel Resolution

### Channel Resolution Flow

```
Recipient + Event
     │
     ▼
┌─────────────────────────────────────┐
│   resolveChannels(event, recipient) │
├─────────────────────────────────────┤
│ 1. Get event severity               │
│ 2. Get default channels for severity│
│ 3. Apply user preferences filter    │
│ 4. Check contact availability       │
│ 5. Return enabled channels          │
└─────────────────────────────────────┘
     │
     ▼
Channel[] ['IN_APP', 'EMAIL', 'WHATSAPP']
```

### Default Channel Rules by Severity

| Severity | Default Channels | Notes |
|----------|-----------------|-------|
| `INFO` | IN_APP only | Low priority, no external channels |
| `IMPORTANT` | IN_APP + EMAIL + WHATSAPP | Medium priority |
| `URGENT` | IN_APP + EMAIL + WHATSAPP + SMS* | All channels, SMS fallback |

*SMS is deferred but architecture supports it.

### Channel Availability Checks

Before including a channel, verify:

| Channel | Availability Check |
|---------|-------------------|
| IN_APP | Always available |
| EMAIL | `recipient.email` is not null |
| WHATSAPP | `recipient.phone` is not null and E.164 valid |
| SMS | `recipient.phone` is not null and E.164 valid |

### Preference Override Logic

```typescript
function applyPreferences(
  defaultChannels: Channel[],
  recipient: Recipient,
  event: EventPayload
): Channel[] {
  const prefs = recipient.preferences;
  
  // No preferences = use defaults
  if (!prefs) return defaultChannels;
  
  return defaultChannels.filter((channel) => {
    // Check channel-level toggle
    if (channel === 'EMAIL' && !prefs.emailEnabled) return false;
    if (channel === 'WHATSAPP' && !prefs.whatsAppEnabled) return false;
    if (channel === 'SMS' && !prefs.smsEnabled) return false;
    
    // Check severity-level channels
    const severityChannels = prefs.severityChannels?.[event.severity] ?? [];
    if (!severityChannels.includes(channel)) return false;
    
    // Check category opt-out
    const categoryEnabled = prefs.categoryEnabled?.[event.category];
    if (categoryEnabled === false) return false;
    
    return true;
  });
}
```

---

## Routing Rules Matrix

### Complete Routing Matrix

| Event Type | Severity | Recipients | Channels |
|------------|----------|------------|----------|
| `TASK_ASSIGNED` | INFO | Assigned user | IN_APP |
| `TASK_ACTION_REQUIRED` | IMPORTANT | Assigned user, Admins | IN_APP, EMAIL, WHATSAPP |
| `FILING_DUE_SOON` | IMPORTANT | Admins, Compliance managers | IN_APP, EMAIL, WHATSAPP |
| `FILING_OVERDUE` | URGENT | Admins, Compliance managers | IN_APP, EMAIL, WHATSAPP |
| `PAYMENT_REQUIRED` | IMPORTANT | Admins, Finance role | IN_APP, EMAIL, WHATSAPP |
| `SUBMISSION_STATUS_CHANGED` | INFO | Initiator, Admins | IN_APP, EMAIL |

### Role Mappings

```typescript
const ROLE_MAPPINGS = {
  admin: ['admin', 'org:admin'],
  complianceManager: ['manager', 'compliance_manager', 'org:manager'],
  finance: ['finance', 'finance_manager'],
  member: ['member', 'org:member'],
} as const;

function hasRole(recipient: Recipient, roleGroup: keyof typeof ROLE_MAPPINGS): boolean {
  const allowedRoles = ROLE_MAPPINGS[roleGroup];
  return recipient.roles.some((role) => allowedRoles.includes(role));
}
```

---

## Implementation

### Routing Module

```typescript
// packages/api-services/src/domains/notifications/routing.ts

import type { DrizzleClient } from '@repo/database';
import { users, organizationMembers, notificationPreferences } from '@repo/database';
import { and, eq, inArray } from 'drizzle-orm';
import {
  NotificationEventType,
  NotificationSeverity,
  type BaseEventPayload,
} from './events';

// ============================================
// Types
// ============================================

export interface Recipient {
  userId: string;
  email: string | null;
  phone: string | null;
  roles: string[];
  preferences: UserPreferences | null;
}

interface UserPreferences {
  emailEnabled: boolean;
  whatsAppEnabled: boolean;
  smsEnabled: boolean;
  severityChannels: Record<string, string[]>;
  categoryEnabled: Record<string, boolean>;
}

export type Channel = 'IN_APP' | 'EMAIL' | 'WHATSAPP' | 'SMS';

// ============================================
// Recipient Resolution
// ============================================

export async function resolveRecipients(
  db: DrizzleClient,
  event: BaseEventPayload
): Promise<Recipient[]> {
  const eventType = getEventTypeFromPayload(event);
  const rules = RECIPIENT_RULES[eventType];
  
  const userIds = new Set<string>();
  
  // Add specific users from payload
  if (rules.includeSpecificUsers) {
    const specificIds = rules.includeSpecificUsers(event);
    specificIds.forEach((id) => userIds.add(id));
  }
  
  // Add users by role
  if (rules.includeRoles && rules.includeRoles.length > 0) {
    const roleUsers = await getUsersByRoles(
      db,
      event.tenantId,
      rules.includeRoles
    );
    roleUsers.forEach((id) => userIds.add(id));
  }
  
  if (userIds.size === 0) {
    return [];
  }
  
  // Fetch full recipient data
  const recipients = await fetchRecipientData(
    db,
    event.tenantId,
    Array.from(userIds)
  );
  
  return recipients;
}

async function getUsersByRoles(
  db: DrizzleClient,
  tenantId: string,
  roles: string[]
): Promise<string[]> {
  const members = await db
    .select({ userId: organizationMembers.userId })
    .from(organizationMembers)
    .where(
      and(
        eq(organizationMembers.organizationId, tenantId),
        inArray(organizationMembers.role, roles)
      )
    );
  
  return members.map((m) => m.userId);
}

async function fetchRecipientData(
  db: DrizzleClient,
  tenantId: string,
  userIds: string[]
): Promise<Recipient[]> {
  const results = await db
    .select({
      userId: users.id,
      email: users.email,
      phone: users.phone,
      role: organizationMembers.role,
      prefs: notificationPreferences,
    })
    .from(users)
    .innerJoin(
      organizationMembers,
      and(
        eq(organizationMembers.userId, users.id),
        eq(organizationMembers.organizationId, tenantId)
      )
    )
    .leftJoin(
      notificationPreferences,
      and(
        eq(notificationPreferences.userId, users.id),
        eq(notificationPreferences.tenantId, tenantId)
      )
    )
    .where(inArray(users.id, userIds));
  
  // Group by user (multiple roles possible)
  const userMap = new Map<string, Recipient>();
  
  for (const row of results) {
    const existing = userMap.get(row.userId);
    if (existing) {
      if (row.role && !existing.roles.includes(row.role)) {
        existing.roles.push(row.role);
      }
    } else {
      userMap.set(row.userId, {
        userId: row.userId,
        email: row.email,
        phone: row.phone,
        roles: row.role ? [row.role] : [],
        preferences: row.prefs
          ? {
              emailEnabled: row.prefs.emailEnabled,
              whatsAppEnabled: row.prefs.whatsAppEnabled,
              smsEnabled: row.prefs.smsEnabled,
              severityChannels: row.prefs.severityChannels ?? {},
              categoryEnabled: row.prefs.categoryEnabled ?? {},
            }
          : null,
      });
    }
  }
  
  return Array.from(userMap.values());
}

// ============================================
// Channel Resolution
// ============================================

export function resolveChannels(
  event: BaseEventPayload,
  recipient: Recipient
): Channel[] {
  // Get default channels for severity
  const defaultChannels = DEFAULT_CHANNELS[event.severity];
  
  // Filter by availability
  const availableChannels = defaultChannels.filter((channel) =>
    isChannelAvailable(channel, recipient)
  );
  
  // Apply user preferences
  const enabledChannels = applyPreferences(
    availableChannels,
    recipient,
    event
  );
  
  // IN_APP is always included
  if (!enabledChannels.includes('IN_APP')) {
    enabledChannels.unshift('IN_APP');
  }
  
  return enabledChannels;
}

function isChannelAvailable(channel: Channel, recipient: Recipient): boolean {
  switch (channel) {
    case 'IN_APP':
      return true;
    case 'EMAIL':
      return !!recipient.email && isValidEmail(recipient.email);
    case 'WHATSAPP':
    case 'SMS':
      return !!recipient.phone && isValidE164(recipient.phone);
    default:
      return false;
  }
}

function applyPreferences(
  channels: Channel[],
  recipient: Recipient,
  event: BaseEventPayload
): Channel[] {
  const prefs = recipient.preferences;
  
  if (!prefs) {
    return channels;
  }
  
  return channels.filter((channel) => {
    // Channel-level toggle
    if (channel === 'EMAIL' && !prefs.emailEnabled) return false;
    if (channel === 'WHATSAPP' && !prefs.whatsAppEnabled) return false;
    if (channel === 'SMS' && !prefs.smsEnabled) return false;
    
    // Severity-level channel filter
    const severityChannels = prefs.severityChannels[event.severity];
    if (severityChannels && !severityChannels.includes(channel)) {
      return false;
    }
    
    // Category opt-out
    if (prefs.categoryEnabled[event.category] === false) {
      return false;
    }
    
    return true;
  });
}

// ============================================
// Validation Helpers
// ============================================

function isValidEmail(email: string): boolean {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}

function isValidE164(phone: string): boolean {
  return /^\+[1-9]\d{6,14}$/.test(phone);
}

// ============================================
// Routing Rules Configuration
// ============================================

interface RecipientRule {
  includeSpecificUsers?: (event: BaseEventPayload) => string[];
  includeRoles?: string[];
}

const RECIPIENT_RULES: Record<NotificationEventType, RecipientRule> = {
  [NotificationEventType.TASK_ASSIGNED]: {
    includeSpecificUsers: (event) => {
      const payload = event as any;
      return payload.assignedUserId ? [payload.assignedUserId] : [];
    },
  },
  [NotificationEventType.TASK_ACTION_REQUIRED]: {
    includeSpecificUsers: (event) => {
      const payload = event as any;
      return payload.assignedUserId ? [payload.assignedUserId] : [];
    },
    includeRoles: ['admin', 'org:admin'],
  },
  [NotificationEventType.FILING_DUE_SOON]: {
    includeRoles: ['admin', 'org:admin', 'manager', 'compliance_manager'],
  },
  [NotificationEventType.FILING_OVERDUE]: {
    includeRoles: ['admin', 'org:admin', 'manager', 'compliance_manager'],
  },
  [NotificationEventType.PAYMENT_REQUIRED]: {
    includeRoles: ['admin', 'org:admin', 'finance', 'finance_manager'],
  },
  [NotificationEventType.SUBMISSION_STATUS_CHANGED]: {
    includeSpecificUsers: (event) => {
      const payload = event as any;
      return payload.initiatorUserId ? [payload.initiatorUserId] : [];
    },
    includeRoles: ['admin', 'org:admin'],
  },
};

const DEFAULT_CHANNELS: Record<NotificationSeverity, Channel[]> = {
  [NotificationSeverity.INFO]: ['IN_APP'],
  [NotificationSeverity.IMPORTANT]: ['IN_APP', 'EMAIL', 'WHATSAPP'],
  [NotificationSeverity.URGENT]: ['IN_APP', 'EMAIL', 'WHATSAPP', 'SMS'],
};

function getEventTypeFromPayload(event: BaseEventPayload): NotificationEventType {
  // Determine event type from payload structure
  // This is set explicitly when emitting the event
  return (event as any).eventType ?? NotificationEventType.TASK_ASSIGNED;
}
```

---

## Testing Routing Logic

### Unit Tests

```typescript
// packages/api-services/src/domains/notifications/__tests__/routing.test.ts

import { describe, it, expect } from 'vitest';
import { resolveChannels } from '../routing';
import { NotificationSeverity, NotificationCategory } from '../events';

describe('resolveChannels', () => {
  const baseEvent = {
    tenantId: 'org_123',
    entityType: 'task' as const,
    entityId: 'task_456',
    title: 'Test',
    message: 'Test message',
    deepLink: '/tasks/task_456',
    severity: NotificationSeverity.INFO,
    category: NotificationCategory.TASK,
  };

  it('returns IN_APP only for INFO severity', () => {
    const recipient = {
      userId: 'user_1',
      email: 'test@example.com',
      phone: '+260971234567',
      roles: ['member'],
      preferences: null,
    };

    const channels = resolveChannels(baseEvent, recipient);
    expect(channels).toEqual(['IN_APP']);
  });

  it('includes EMAIL and WHATSAPP for IMPORTANT severity', () => {
    const event = { ...baseEvent, severity: NotificationSeverity.IMPORTANT };
    const recipient = {
      userId: 'user_1',
      email: 'test@example.com',
      phone: '+260971234567',
      roles: ['admin'],
      preferences: null,
    };

    const channels = resolveChannels(event, recipient);
    expect(channels).toContain('IN_APP');
    expect(channels).toContain('EMAIL');
    expect(channels).toContain('WHATSAPP');
  });

  it('respects user preference to disable email', () => {
    const event = { ...baseEvent, severity: NotificationSeverity.IMPORTANT };
    const recipient = {
      userId: 'user_1',
      email: 'test@example.com',
      phone: '+260971234567',
      roles: ['admin'],
      preferences: {
        emailEnabled: false,
        whatsAppEnabled: true,
        smsEnabled: false,
        severityChannels: {},
        categoryEnabled: {},
      },
    };

    const channels = resolveChannels(event, recipient);
    expect(channels).not.toContain('EMAIL');
    expect(channels).toContain('WHATSAPP');
  });

  it('excludes WHATSAPP if phone is invalid', () => {
    const event = { ...baseEvent, severity: NotificationSeverity.IMPORTANT };
    const recipient = {
      userId: 'user_1',
      email: 'test@example.com',
      phone: 'invalid-phone',
      roles: ['admin'],
      preferences: null,
    };

    const channels = resolveChannels(event, recipient);
    expect(channels).not.toContain('WHATSAPP');
  });
});
```

