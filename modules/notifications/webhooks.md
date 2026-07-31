---
title: "Notifications Webhooks"
description: "Delivery receipt webhook handlers for Email and WhatsApp status updates."
---

## Table of Contents

1. [Overview](#overview)
2. [Webhook Endpoints](#webhook-endpoints)
3. [Email Webhooks (Resend)](#email-webhooks-resend)
4. [WhatsApp Webhooks (Meta)](#whatsapp-webhooks-meta)
5. [Security](#security)
6. [Error Handling](#error-handling)

---

## Overview

Webhooks receive delivery status updates from providers, enabling:

- Accurate delivery tracking
- Bounce/complaint handling
- Read receipts (WhatsApp)
- Retry decisions

### Webhook Flow

```
Provider (Resend/Meta)
        │
        │ POST /webhooks/{provider}
        ▼
┌───────────────────────────┐
│   Webhook Handler         │
│   1. Verify signature     │
│   2. Parse payload        │
│   3. Lookup delivery      │
│   4. Update status        │
└───────────────────────────┘
        │
        ▼
notification_deliveries (DB)
   status: SENT → DELIVERED
```

---

## Webhook Endpoints

| Endpoint | Provider | Events |
|----------|----------|--------|
| `POST /webhooks/email` | Resend | delivered, bounced, complained |
| `POST /webhooks/whatsapp` | Meta | delivered, read, failed |

### Backend Routes

```typescript
// packages/backend/src/modules/webhooks/webhooks.routes.ts

import { createRouter } from '@/core/http/create-app';
import { handleEmailWebhook } from './email-webhook.handlers';
import { handleWhatsAppWebhook } from './whatsapp-webhook.handlers';

export const webhookRoutes = createRouter()
  // Resend email webhooks
  .post('/webhooks/email', handleEmailWebhook)
  
  // Meta WhatsApp webhooks
  .get('/webhooks/whatsapp', handleWhatsAppVerification)
  .post('/webhooks/whatsapp', handleWhatsAppWebhook);

export default webhookRoutes;
```

---

## Email Webhooks (Resend)

### Supported Events

| Event | Action | Delivery Status |
|-------|--------|-----------------|
| `email.delivered` | Update status | `delivered` |
| `email.bounced` | Mark failed | `bounced` |
| `email.complained` | Mark complained | `complained` |
| `email.delivery_delayed` | Log warning | (no change) |

### Implementation

```typescript
// packages/backend/src/modules/webhooks/email-webhook.handlers.ts

import type { Context } from 'hono';
import { createDb } from '@repo/database';
import { notificationDeliveries } from '@repo/database';
import { eq, and } from 'drizzle-orm';
import { Webhook } from 'svix';
import type { Env } from '@/env';

interface ResendWebhookPayload {
  type: string;
  created_at: string;
  data: {
    email_id: string;
    from: string;
    to: string[];
    subject: string;
    headers?: Record<string, string>;
    // Bounce-specific
    bounce?: {
      message: string;
    };
  };
}

export async function handleEmailWebhook(c: Context<{ Bindings: Env }>) {
  const db = createDb(c.env.DATABASE_URL);
  
  // 1. Verify webhook signature
  const signature = c.req.header('svix-signature');
  const timestamp = c.req.header('svix-timestamp');
  const webhookId = c.req.header('svix-id');
  
  if (!signature || !timestamp || !webhookId) {
    return c.json({ error: 'Missing webhook headers' }, 400);
  }
  
  const body = await c.req.text();
  
  try {
    // Verify using Resend's Svix-based signatures
    const wh = new Webhook(c.env.RESEND_WEBHOOK_SECRET);
    wh.verify(body, {
      'svix-id': webhookId,
      'svix-timestamp': timestamp,
      'svix-signature': signature,
    });
  } catch (err) {
    console.error('Email webhook signature verification failed:', err);
    return c.json({ error: 'Invalid signature' }, 401);
  }
  
  // 2. Parse payload
  const payload = JSON.parse(body) as ResendWebhookPayload;
  
  // 3. Extract delivery ID from headers
  const deliveryId = payload.data.headers?.['X-Bumara-Delivery-Id'];
  
  if (!deliveryId) {
    // Not our email, ignore
    console.info('Email webhook for non-Bumara email:', payload.data.email_id);
    return c.json({ status: 'ignored' });
  }
  
  // 4. Update delivery status
  const statusMapping: Record<string, string> = {
    'email.delivered': 'delivered',
    'email.bounced': 'bounced',
    'email.complained': 'complained',
  };
  
  const newStatus = statusMapping[payload.type];
  
  if (!newStatus) {
    console.info('Ignoring email webhook event:', payload.type);
    return c.json({ status: 'ignored' });
  }
  
  await db
    .update(notificationDeliveries)
    .set({
      status: newStatus,
      deliveredAt: newStatus === 'delivered' ? new Date() : null,
      failedAt: ['bounced', 'complained'].includes(newStatus) ? new Date() : null,
      lastError: payload.data.bounce?.message,
      webhookPayload: payload,
      updatedAt: new Date(),
    })
    .where(
      and(
        eq(notificationDeliveries.id, deliveryId),
        eq(notificationDeliveries.providerMessageId, payload.data.email_id)
      )
    );
  
  console.info('Email webhook processed:', {
    deliveryId,
    emailId: payload.data.email_id,
    event: payload.type,
    newStatus,
  });
  
  return c.json({ status: 'processed' });
}
```

### Resend Webhook Configuration

1. Go to Resend Dashboard → Webhooks
2. Add endpoint: `https://api.bumara.com/webhooks/email`
3. Select events: `email.delivered`, `email.bounced`, `email.complained`
4. Copy the signing secret to `RESEND_WEBHOOK_SECRET`

---

## WhatsApp Webhooks (Meta)

### Supported Events

| Event | Action | Delivery Status |
|-------|--------|-----------------|
| `message_status: delivered` | Update status | `delivered` |
| `message_status: read` | Update status | `read` |
| `message_status: failed` | Mark failed | `failed` |

### Verification Endpoint

```typescript
// packages/backend/src/modules/webhooks/whatsapp-webhook.handlers.ts

import type { Context } from 'hono';
import { createDb } from '@repo/database';
import { notificationDeliveries } from '@repo/database';
import { eq, and } from 'drizzle-orm';
import type { Env } from '@/env';

/**
 * WhatsApp webhook verification (GET request)
 * Required for initial webhook setup in Meta Developer Console
 */
export async function handleWhatsAppVerification(c: Context<{ Bindings: Env }>) {
  const mode = c.req.query('hub.mode');
  const token = c.req.query('hub.verify_token');
  const challenge = c.req.query('hub.challenge');
  
  if (mode === 'subscribe' && token === c.env.WHATSAPP_VERIFY_TOKEN) {
    console.info('WhatsApp webhook verified');
    return c.text(challenge ?? '');
  }
  
  return c.text('Forbidden', 403);
}
```

### Event Handler

```typescript
/**
 * WhatsApp webhook event handler (POST request)
 */
export async function handleWhatsAppWebhook(c: Context<{ Bindings: Env }>) {
  const db = createDb(c.env.DATABASE_URL);
  
  // 1. Verify signature
  const signature = c.req.header('x-hub-signature-256');
  
  if (!signature) {
    return c.json({ error: 'Missing signature' }, 400);
  }
  
  const body = await c.req.text();
  
  const isValid = await verifyWhatsAppSignature(
    body,
    signature,
    c.env.WHATSAPP_WEBHOOK_SECRET
  );
  
  if (!isValid) {
    console.error('WhatsApp webhook signature verification failed');
    return c.json({ error: 'Invalid signature' }, 401);
  }
  
  // 2. Parse payload
  const payload = JSON.parse(body) as WhatsAppWebhookPayload;
  
  // 3. Process each entry
  for (const entry of payload.entry ?? []) {
    for (const change of entry.changes ?? []) {
      if (change.field !== 'messages') continue;
      
      const value = change.value;
      
      // Process message statuses
      for (const status of value.statuses ?? []) {
        await processMessageStatus(db, status);
      }
    }
  }
  
  return c.json({ status: 'processed' });
}

interface WhatsAppWebhookPayload {
  object: string;
  entry: Array<{
    id: string;
    changes: Array<{
      field: string;
      value: {
        messaging_product: string;
        metadata: { phone_number_id: string };
        statuses?: Array<{
          id: string;
          status: 'sent' | 'delivered' | 'read' | 'failed';
          timestamp: string;
          recipient_id: string;
          errors?: Array<{
            code: number;
            title: string;
            message: string;
          }>;
        }>;
      };
    }>;
  }>;
}

async function processMessageStatus(
  db: ReturnType<typeof createDb>,
  status: WhatsAppWebhookPayload['entry'][0]['changes'][0]['value']['statuses'][0]
): Promise<void> {
  const statusMapping: Record<string, string> = {
    delivered: 'delivered',
    read: 'read',
    failed: 'failed',
  };
  
  const newStatus = statusMapping[status.status];
  
  if (!newStatus) {
    // 'sent' status - we already have this from the API response
    return;
  }
  
  const updateData: Record<string, any> = {
    status: newStatus,
    updatedAt: new Date(),
  };
  
  if (newStatus === 'delivered') {
    updateData.deliveredAt = new Date(parseInt(status.timestamp) * 1000);
  } else if (newStatus === 'failed') {
    updateData.failedAt = new Date(parseInt(status.timestamp) * 1000);
    updateData.lastError = status.errors?.[0]?.message ?? 'Unknown error';
  }
  
  await db
    .update(notificationDeliveries)
    .set(updateData)
    .where(
      and(
        eq(notificationDeliveries.providerMessageId, status.id),
        eq(notificationDeliveries.provider, 'META')
      )
    );
  
  console.info('WhatsApp status update:', {
    messageId: status.id,
    status: newStatus,
    recipient: status.recipient_id,
  });
}

async function verifyWhatsAppSignature(
  body: string,
  signature: string,
  secret: string
): Promise<boolean> {
  const encoder = new TextEncoder();
  const key = await crypto.subtle.importKey(
    'raw',
    encoder.encode(secret),
    { name: 'HMAC', hash: 'SHA-256' },
    false,
    ['sign']
  );
  
  const signatureBuffer = await crypto.subtle.sign(
    'HMAC',
    key,
    encoder.encode(body)
  );
  
  const expectedSignature = 'sha256=' + bufferToHex(signatureBuffer);
  
  return signature === expectedSignature;
}

function bufferToHex(buffer: ArrayBuffer): string {
  return Array.from(new Uint8Array(buffer))
    .map((b) => b.toString(16).padStart(2, '0'))
    .join('');
}
```

### Meta Webhook Configuration

1. Go to Meta Developer Console → Your App → Webhooks
2. Add webhook for WhatsApp Business Account
3. Callback URL: `https://api.bumara.com/webhooks/whatsapp`
4. Verify token: Use value from `WHATSAPP_VERIFY_TOKEN`
5. Subscribe to: `messages` field
6. Get the App Secret for `WHATSAPP_WEBHOOK_SECRET`

---

## Security

### Signature Verification

| Provider | Signature Header | Algorithm |
|----------|-----------------|-----------|
| Resend | `svix-signature` | HMAC-SHA256 (Svix) |
| Meta | `x-hub-signature-256` | HMAC-SHA256 |

### Rate Limiting

```typescript
// Apply rate limiting to webhook endpoints
import { rateLimit } from '@/core/middleware/rate-limit';

const webhookRateLimit = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 1000, // 1000 requests per minute per IP
  message: 'Too many webhook requests',
});

// Apply to routes
webhookRoutes.use('/webhooks/*', webhookRateLimit);
```

### IP Allowlisting (Optional)

For additional security, allowlist provider IPs:

```typescript
// Resend IPs: Check Resend documentation
// Meta IPs: Check Meta Developer documentation

const ALLOWED_WEBHOOK_IPS = [
  // Add provider IPs here
];

function verifySourceIP(c: Context): boolean {
  const clientIP = c.req.header('cf-connecting-ip');
  return ALLOWED_WEBHOOK_IPS.includes(clientIP ?? '');
}
```

---

## Error Handling

### Idempotency

Webhooks may be delivered multiple times. Handle gracefully:

```typescript
// Check current status before updating
const [delivery] = await db
  .select()
  .from(notificationDeliveries)
  .where(eq(notificationDeliveries.providerMessageId, messageId))
  .limit(1);

if (!delivery) {
  console.warn('Delivery not found for webhook:', messageId);
  return; // Ignore
}

// Only update if status progresses (delivered → read is OK, read → delivered is not)
const statusOrder = ['queued', 'sending', 'sent', 'delivered', 'read'];
const currentIndex = statusOrder.indexOf(delivery.status);
const newIndex = statusOrder.indexOf(newStatus);

if (newIndex <= currentIndex && newStatus !== 'failed') {
  console.info('Ignoring older status:', { current: delivery.status, new: newStatus });
  return;
}
```

### Logging

```typescript
// Structured logging for debugging
console.info('Webhook received:', {
  provider: 'META',
  event: 'message_status',
  messageId: status.id,
  status: status.status,
  timestamp: new Date().toISOString(),
});

// Error logging
console.error('Webhook processing failed:', {
  provider: 'RESEND',
  event: payload.type,
  error: error.message,
  payload: JSON.stringify(payload).substring(0, 500), // Truncate
});
```

### Response Codes

| Status | Meaning |
|--------|---------|
| 200 | Successfully processed |
| 400 | Bad request (missing headers) |
| 401 | Invalid signature |
| 500 | Processing error (provider will retry) |

Always return 200 for successfully verified requests to prevent provider retries.

