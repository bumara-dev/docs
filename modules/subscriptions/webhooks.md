---
title: "Subscription Webhooks"
description: "Payment webhooks handle Stripe events for subscription lifecycle management. The webhook processor normalizes events and routes them to appropriate..."
---

## Overview

Payment webhooks handle Stripe events for subscription lifecycle management. The webhook processor normalizes events and routes them to appropriate handlers.

**File:** `packages/backend/src/modules/webhooks/payments/payments-webhook.processor.ts`

## Webhook Endpoints

| Endpoint | Provider | Description |
|----------|----------|-------------|
| `POST /webhooks/stripe` | Stripe | Stripe webhook events |
| `POST /webhooks/lenco` | Lenco | Lenco webhook events (optional) |

## Event Types

### Checkout Events

#### checkout.completed

Triggered when a customer completes a checkout session (initial subscription or plan change).

**Actions:**
1. Update subscription with external IDs
2. Set billing period dates
3. Update plan tier (if changed)
4. Initialize usage tracking record

```typescript
// Handler: handleCheckoutCompleted
await db.update(subscriptions).set({
  externalSubscriptionId: data.subscriptionId,
  externalCustomerId: data.customerId,
  paymentProvider: event.provider,
  status: "active",
  currentPeriodStart,
  currentPeriodEnd,
  planTier: finalPlanTier,  // If provided
});

// Initialize usage tracking
await db.insert(subscriptionUsage).values({
  organizationId,
  periodKey,  // YYYY-MM format
  periodStart: currentPeriodStart,
  periodEnd: currentPeriodEnd,
  serviceRequestsUsed: 0,
  storageUsedMb: 0,
  aiCreditsUsed: 0,
});
```

#### checkout.expired

Triggered when a checkout session expires without completion.

**Actions:**
1. Revert payment request status to pending (for one-time payments)

---

### Subscription Events

#### subscription.created

Triggered when a new subscription is created.

**Actions:**
1. Link external subscription ID to organization
2. Set status to active

```typescript
await db.update(subscriptions).set({
  externalSubscriptionId: data.subscriptionId,
  externalCustomerId: data.customerId,
  paymentProvider: event.provider,
  status: "active",
});
```

#### subscription.updated

Triggered when a subscription is modified (plan change, billing update, etc.).

**Actions:**
1. Update plan tier (if changed)
2. Update billing period dates
3. Update cancel-at-period-end flag

```typescript
const updateSet = { updatedAt: new Date() };

// Update plan tier if changed
if (data.planId && validTiers.includes(data.planId)) {
  updateSet.planTier = data.planId;
}

// Update billing period dates
if (data.currentPeriodStart) {
  updateSet.currentPeriodStart = new Date(data.currentPeriodStart);
}
if (data.currentPeriodEnd) {
  updateSet.currentPeriodEnd = new Date(data.currentPeriodEnd);
}

// Handle cancel at period end
if (typeof data.cancelAtPeriodEnd === "boolean") {
  updateSet.cancelAtPeriodEnd = data.cancelAtPeriodEnd;
}
```

#### subscription.canceled

Triggered when a subscription is canceled.

**Actions:**
1. Update status to 'cancelled'

```typescript
await db.update(subscriptions).set({
  status: "cancelled",
});
```

#### subscription.renewed

Triggered when a subscription renews for a new billing period.

**Actions:**
1. Update billing period dates
2. Reset monthly usage (service requests, AI credits)
3. Keep storage usage (cumulative)

```typescript
// Update subscription
await db.update(subscriptions).set({
  status: "active",
  currentPeriodStart,
  currentPeriodEnd,
});

// Reset monthly usage
await db.insert(subscriptionUsage)
  .values({
    organizationId,
    periodKey,  // New period key
    serviceRequestsUsed: 0,
    storageUsedMb: 0,  // Preserved from previous
    aiCreditsUsed: 0,
  })
  .onConflictDoUpdate({
    target: [subscriptionUsage.organizationId, subscriptionUsage.periodKey],
    set: {
      serviceRequestsUsed: 0,
      aiCreditsUsed: 0,
      // Storage NOT reset - it's cumulative
    },
  });
```

---

### Invoice Events

#### invoice.paid

Triggered when an invoice is successfully paid (subscription renewal).

**Actions:**
1. Confirm subscription is active

#### invoice.payment_failed

Triggered when an invoice payment fails.

**Actions:**
1. Log warning (future: track failure count, suspend after repeated failures)

---

## Webhook Event Data

The webhook processor normalizes provider-specific events into a common `WebhookEvent` format:

```typescript
interface WebhookEvent {
  id: string;                  // Event ID
  type: WebhookEventType;      // Event type
  data: WebhookEventData;      // Normalized event data
  provider: string;            // 'stripe' | 'lenco'
  createdAt: Date;
  rawEvent: unknown;           // Original event for debugging
}

interface WebhookEventData {
  // Common fields
  organizationId?: string;     // From metadata
  customerId?: string;

  // Subscription fields
  subscriptionId?: string;
  planId?: string;             // Maps to plan tier
  currentPeriodStart?: string | number;
  currentPeriodEnd?: string | number;
  cancelAtPeriodEnd?: boolean;

  // Payment fields
  paymentId?: string;
  paymentRequestId?: string;
  amount?: number;
  currency?: string;

  // Additional fields allowed
  [key: string]: unknown;
}
```

## Configuration

### Stripe Webhook Secret

Set in environment:
```env
STRIPE_WEBHOOK_SECRET=whsec_xxxxx
```

### Lenco Webhook Secret

Set in environment:
```env
LENCO_WEBHOOK_SECRET=xxxxx
```

## Testing Webhooks

### Local Development

Use Stripe CLI to forward webhooks:
```bash
stripe listen --forward-to localhost:8787/webhooks/stripe
```

### Test Events

```bash
# Trigger a checkout completed event
stripe trigger checkout.session.completed

# Trigger subscription created
stripe trigger customer.subscription.created

# Trigger subscription renewed
stripe trigger invoice.paid
```

## Error Handling

Webhooks return appropriate HTTP status codes:

| Status | Description |
|--------|-------------|
| 200 | Event processed successfully |
| 400 | Invalid signature or malformed payload |
| 500 | Internal processing error |

Stripe will retry failed webhooks with exponential backoff.

## Monitoring

Log levels for webhook events:

- **INFO** - Successful event processing
- **WARN** - Missing data (e.g., no organizationId)
- **ERROR** - Processing failures

Example log:
```json
{
  "level": "info",
  "eventId": "evt_xxxxx",
  "eventType": "subscription.renewed",
  "subscriptionId": "sub_xxxxx",
  "organizationId": "org_xxxxx",
  "periodKey": "2026-02",
  "message": "Subscription renewed and usage reset"
}
```
