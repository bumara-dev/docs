---
title: "Notifications Security & Permissions"
description: "Security considerations, tenant isolation, and access control for the notification system."
---

## Table of Contents

1. [Overview](#overview)
2. [Tenant Isolation](#tenant-isolation)
3. [Data Security](#data-security)
4. [API Security](#api-security)
5. [Webhook Security](#webhook-security)
6. [Provider Security](#provider-security)
7. [Audit Trail](#audit-trail)

---

## Overview

Security is critical for the notification system due to:

- Multi-tenant architecture (data must never leak between organizations)
- Sensitive compliance information in notification content
- External provider integrations (API keys, webhooks)
- Personal contact information (emails, phone numbers)

### Security Principles

| Principle | Implementation |
|-----------|----------------|
| Tenant Isolation | Every query includes `tenant_id` filter |
| Least Privilege | Users see only their own notifications |
| Minimal Payloads | No PII in notification content |
| Secure Secrets | Provider keys in environment variables only |
| Signature Verification | All webhooks verify signatures |
| Audit Logging | State changes are logged |

---

## Tenant Isolation

### Non-Negotiable Rules

1. **Every database query MUST filter by `tenant_id`**
2. **`tenant_id` is derived from Clerk session, never from client input**
3. **Never join or return data across tenants**

### Implementation Pattern

```typescript
// CORRECT: tenant_id from auth context
export async function listNotifications(c: Context) {
  const { orgId, userId } = c.get('auth'); // From Clerk middleware
  
  const notifications = await db
    .select()
    .from(notifications)
    .where(
      and(
        eq(notifications.tenantId, orgId), // Always first
        eq(notifications.userId, userId)
      )
    );
  
  return notifications;
}

// WRONG: tenant_id from request
export async function listNotifications(c: Context) {
  const { tenantId } = c.req.query(); // NEVER DO THIS
  // ...
}
```

### Outbox Event Isolation

Events in the outbox are tenant-scoped:

```typescript
// Creating outbox event - tenant_id is mandatory
await db.insert(eventOutbox).values({
  tenantId: ctx.orgId, // From auth context
  eventType: 'TASK_ASSIGNED',
  payload: {
    tenantId: ctx.orgId, // Also in payload for verification
    // ...
  },
});
```

### Recipient Resolution Isolation

Recipients are resolved only within the tenant:

```typescript
// Only query users within the tenant's org membership
const recipients = await db
  .select()
  .from(users)
  .innerJoin(
    organizationMembers,
    and(
      eq(organizationMembers.userId, users.id),
      eq(organizationMembers.organizationId, tenantId) // Tenant filter
    )
  )
  .where(inArray(users.id, userIds));
```

---

## Data Security

### PII Minimization

Notification payloads should NOT contain:

- Full names (use IDs or first names only)
- TPIN/TIN numbers
- NAPSA numbers
- Bank account details
- Salary information
- Full addresses

**Do include:**
- Entity IDs (task ID, filing ID)
- Deep links (URLs to view details)
- Generic descriptions
- Due dates
- Status changes

```typescript
// CORRECT: Minimal payload
const payload = {
  title: 'Task Assigned',
  message: 'You have been assigned a new task',
  deepLink: `/tasks/${taskId}`, // User navigates to see details
};

// WRONG: PII in payload
const payload = {
  title: 'PAYE Filing Due',
  message: `Submit PAYE for Acme Corp (TPIN: 1234567890)`, // NO!
};
```

### Email/Phone Storage

Contact information is stored in the `users` table (existing):

```typescript
// Phone numbers stored in E.164 format
// Validation before storage
function validateAndStorePhone(phone: string): string {
  const normalized = normalizeToE164(phone);
  if (!isValidE164(normalized)) {
    throw new Error('Invalid phone number');
  }
  return normalized;
}
```

### Delivery Records

Delivery records store the destination address for retry purposes:

```typescript
// notification_deliveries.to contains email/phone
// This is necessary for delivery retries
// Access is restricted to:
// - The notification system (workers)
// - Audit queries (with proper authorization)
```

---

## API Security

### Authentication

All notification API endpoints require authentication:

```typescript
// packages/backend/src/modules/notifications/index.ts

import { requireAuth } from '@/core/middleware/auth';

const notificationsRouter = createRouter()
  .use('*', requireAuth) // All routes require auth
  .openapi(routes.listNotificationsRoute, handlers.listNotifications);
```

### Authorization

Users can only access their own notifications:

```typescript
// Handler enforces user-level access
export const listNotifications = async (c: Context) => {
  const { orgId, userId } = c.get('auth');
  
  // Query filters by BOTH tenant and user
  const notifications = await db
    .select()
    .from(notifications)
    .where(
      and(
        eq(notifications.tenantId, orgId),
        eq(notifications.userId, userId) // User can only see their own
      )
    );
};
```

### Preferences Access

Users can only view/modify their own preferences:

```typescript
// Same user-level restriction
export const updatePreferences = async (c: Context) => {
  const { orgId, userId } = c.get('auth');
  
  await db
    .update(notificationPreferences)
    .set(updates)
    .where(
      and(
        eq(notificationPreferences.tenantId, orgId),
        eq(notificationPreferences.userId, userId)
      )
    );
};
```

### Admin Access (Backoffice)

Backoffice users may need broader access for support:

```typescript
// Backoffice endpoint with role check
export const listTenantNotifications = async (c: Context) => {
  const { userId, role } = c.get('auth');
  
  // Verify backoffice role
  if (!['backoffice_admin', 'backoffice_manager'].includes(role)) {
    return c.json({ error: 'Forbidden' }, 403);
  }
  
  const { tenantId } = c.req.valid('param');
  
  // Log access for audit
  await logAuditEvent({
    actorId: userId,
    actorType: 'STAFF',
    action: 'notifications.list',
    resourceType: 'organization',
    resourceId: tenantId,
  });
  
  // Backoffice can query any tenant's notifications
  const notifications = await db
    .select()
    .from(notifications)
    .where(eq(notifications.tenantId, tenantId));
  
  return notifications;
};
```

---

## Webhook Security

### Signature Verification

All incoming webhooks MUST verify signatures:

```typescript
// Resend webhooks (Svix-based)
import { Webhook } from 'svix';

async function verifyResendWebhook(body: string, headers: Headers): Promise<boolean> {
  const wh = new Webhook(env.RESEND_WEBHOOK_SECRET);
  
  try {
    wh.verify(body, {
      'svix-id': headers.get('svix-id')!,
      'svix-timestamp': headers.get('svix-timestamp')!,
      'svix-signature': headers.get('svix-signature')!,
    });
    return true;
  } catch {
    return false;
  }
}

// Meta webhooks (HMAC-SHA256)
async function verifyMetaWebhook(body: string, signature: string): Promise<boolean> {
  const encoder = new TextEncoder();
  const key = await crypto.subtle.importKey(
    'raw',
    encoder.encode(env.WHATSAPP_WEBHOOK_SECRET),
    { name: 'HMAC', hash: 'SHA-256' },
    false,
    ['sign']
  );
  
  const computed = await crypto.subtle.sign('HMAC', key, encoder.encode(body));
  const expected = 'sha256=' + bufferToHex(computed);
  
  return signature === expected;
}
```

### Rate Limiting

Webhook endpoints are rate-limited:

```typescript
// Apply strict rate limiting to webhooks
const webhookRateLimit = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 1000, // 1000 requests per minute
});

webhookRoutes.use('/webhooks/*', webhookRateLimit);
```

### Response Handling

Never leak internal errors to webhook callers:

```typescript
export async function handleWebhook(c: Context) {
  try {
    await processWebhook(c);
    return c.json({ status: 'ok' });
  } catch (error) {
    // Log internally but don't expose details
    console.error('Webhook error:', error);
    return c.json({ status: 'error' }, 500); // Generic response
  }
}
```

---

## Provider Security

### API Key Management

Provider API keys are stored in environment variables only:

```typescript
// Environment variables (never in code)
RESEND_TOKEN=re_xxxxx
WHATSAPP_ACCESS_TOKEN=EAAxxxxx
WHATSAPP_WEBHOOK_SECRET=xxxxx
```

### Key Rotation

Document key rotation procedures:

1. Generate new key in provider dashboard
2. Update environment variable in Cloudflare dashboard
3. Deploy worker (picks up new env vars)
4. Verify webhook signatures work with new secret
5. Revoke old key in provider dashboard

### Access Token Scoping

Use minimum required permissions:

| Provider | Required Scopes |
|----------|-----------------|
| Resend | Send emails, Webhooks |
| Meta WhatsApp | `whatsapp_business_messaging`, `whatsapp_business_management` |

### Secrets in Logs

Never log provider credentials:

```typescript
// WRONG
console.log('Sending email with token:', env.RESEND_TOKEN);

// CORRECT
console.log('Sending email to:', { to: redactEmail(email) });

function redactEmail(email: string): string {
  const [local, domain] = email.split('@');
  return `${local.substring(0, 2)}***@${domain}`;
}
```

---

## Audit Trail

### What to Audit

| Event | Logged Fields |
|-------|---------------|
| Notification created | tenant_id, user_id, category, entity_type, entity_id |
| Notification read | notification_id, user_id, timestamp |
| Preference changed | user_id, field, old_value, new_value |
| Delivery sent | delivery_id, channel, provider |
| Delivery failed | delivery_id, channel, error_code (not message) |
| Webhook received | provider, event_type, message_id |

### Audit Implementation

```typescript
// Use existing audit log service
import { recordAuditLog } from '@repo/api-services/audit';

// When notification is marked read
await recordAuditLog(ctx, deps, {
  action: 'notification.read',
  entityType: 'notification',
  entityId: notificationId,
  metadata: {
    markedAt: new Date().toISOString(),
  },
});

// When preferences updated
await recordAuditLog(ctx, deps, {
  action: 'notification_preferences.update',
  entityType: 'notification_preferences',
  entityId: preferencesId,
  changes: {
    before: { emailEnabled: true },
    after: { emailEnabled: false },
  },
});
```

### Log Retention

Notification system logs should follow the same retention policy as audit logs:

- Production: 90 days minimum
- Compliance events: 7 years (regulatory requirement)

---

## Security Checklist

### Before Deployment

- [ ] All API endpoints require authentication
- [ ] All queries include tenant_id filter
- [ ] Webhook signature verification implemented
- [ ] API keys in environment variables only
- [ ] No PII in notification payloads
- [ ] Rate limiting on all endpoints
- [ ] Audit logging for state changes

### Regular Review

- [ ] Monthly: Review webhook failure rates
- [ ] Quarterly: Rotate API keys
- [ ] Quarterly: Review access logs for anomalies
- [ ] Annually: Security audit of notification flows

