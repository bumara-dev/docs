---
title: "Webhooks"
description: "The payments module uses webhooks for asynchronous event handling. Events from payment providers are received at provider-specific endpoints in the..."
---

## Overview

The payments module uses webhooks for asynchronous event handling. Events from payment providers are received at provider-specific endpoints in the backend, normalized to a common format, and processed by the webhook event processor.

## Webhook Endpoints

The backend exposes separate endpoints for each payment provider:

```
POST /webhooks/payments/stripe   - Stripe webhook events
POST /webhooks/payments/lenco    - Lenco webhook events
```

Each endpoint:
1. Extracts signature from provider-specific headers
2. Instantiates the appropriate gateway (Stripe or Lenco)
3. Verifies signature using the gateway's `parseWebhookEvent` method
4. Parses event to normalized `WebhookEvent` format
5. Routes to the common event processor
6. Updates database (subscriptions, payment_requests)

## Architecture

```
┌─────────────────┐     ┌─────────────────┐
│     Stripe      │     │     Lenco       │
│    Webhook      │     │    Webhook      │
└────────┬────────┘     └────────┬────────┘
         │                       │
         ▼                       ▼
┌─────────────────┐     ┌─────────────────┐
│ /webhooks/      │     │ /webhooks/      │
│ payments/stripe │     │ payments/lenco  │
└────────┬────────┘     └────────┬────────┘
         │                       │
         │   ┌───────────────┐   │
         └──►│ StripeGateway │◄──┘
             │ LencoGateway  │
             │               │
             │ parseWebhook  │
             │ Event()       │
             └───────┬───────┘
                     │
                     ▼
             ┌───────────────┐
             │ Normalized    │
             │ WebhookEvent  │
             └───────┬───────┘
                     │
                     ▼
             ┌───────────────┐
             │ processPayment│
             │ WebhookEvent()│
             └───────┬───────┘
                     │
         ┌───────────┴───────────┐
         ▼                       ▼
┌─────────────────┐     ┌─────────────────┐
│  subscriptions  │     │payment_requests │
│     table       │     │     table       │
└─────────────────┘     └─────────────────┘
```

## Supported Events

### Subscription Events

| Normalized Event | Description | Actions |
|------------------|-------------|---------|
| `checkout.completed` | User completed subscription checkout | Activate subscription, store external IDs |
| `checkout.expired` | Checkout session expired | Revert payment request to pending |
| `subscription.created` | Subscription created | Link to organization, set status active |
| `subscription.updated` | Subscription changed (plan, status) | Update subscription record |
| `subscription.canceled` | Subscription cancelled | Mark status as cancelled |
| `subscription.renewed` | Subscription renewed | Confirm status active |

### Payment Events (One-time Compliance Payments)

| Normalized Event | Description | Actions |
|------------------|-------------|---------|
| `payment.pending` | Payment processing | Update status |
| `payment.succeeded` | Payment successful | Mark paid_platform_unverified |
| `payment.failed` | Payment failed | Revert to required_pending |
| `payment.refunded` | Payment refunded | Mark as refunded |

### Invoice Events

| Normalized Event | Description | Actions |
|------------------|-------------|---------|
| `invoice.paid` | Invoice paid (subscription renewal) | Confirm subscription active |
| `invoice.payment_failed` | Invoice payment failed | Log warning |

## Event Mapping

### Stripe Events

| Stripe Event | Normalized Event |
|--------------|------------------|
| `checkout.session.completed` | `checkout.completed` |
| `checkout.session.expired` | `checkout.expired` |
| `customer.subscription.created` | `subscription.created` |
| `customer.subscription.updated` | `subscription.updated` |
| `customer.subscription.deleted` | `subscription.canceled` |
| `payment_intent.succeeded` | `payment.succeeded` |
| `payment_intent.payment_failed` | `payment.failed` |
| `payment_intent.processing` | `payment.pending` |
| `charge.refunded` | `payment.refunded` |
| `invoice.paid` | `invoice.paid` |
| `invoice.payment_failed` | `invoice.payment_failed` |

### Lenco Events

| Lenco Event | Normalized Event |
|-------------|------------------|
| `checkout.completed` | `checkout.completed` |
| `subscription.cancelled` | `subscription.canceled` |
| `payment.success` | `payment.succeeded` |
| `payment.failed` | `payment.failed` |

## Backend Implementation

### File Structure

```
packages/backend/src/modules/webhooks/
├── webhooks.index.ts          # Main webhooks router (Clerk + Payments)
├── webhooks.routes.ts         # Clerk webhook route
├── webhooks.handlers.ts       # Clerk webhook handler
└── payments/
    ├── index.ts                        # Payments webhook router
    ├── payments-webhook.routes.ts      # Route definitions
    ├── payments-webhook.handlers.ts    # HTTP handlers
    └── payments-webhook.processor.ts   # Event processing logic
```

### Route Definitions

```typescript
// payments-webhook.routes.ts

export const stripeWebhookRoute = createRoute({
  path: "/webhooks/payments/stripe",
  method: "post",
  tags: ["Webhooks"],
  description: "Receive and process Stripe webhook events",
  responses: {
    [HttpStatusCodes.OK]: jsonContent(successResponseSchema, "Processed"),
    [HttpStatusCodes.BAD_REQUEST]: jsonContent(errorResponseSchema, "Invalid"),
    [HttpStatusCodes.INTERNAL_SERVER_ERROR]: jsonContent(errorResponseSchema, "Error"),
  },
});

export const lencoWebhookRoute = createRoute({
  path: "/webhooks/payments/lenco",
  method: "post",
  tags: ["Webhooks"],
  description: "Receive and process Lenco webhook events",
  // ... same response structure
});
```

### Handler Implementation

```typescript
// payments-webhook.handlers.ts

export const stripeWebhookHandler: AppRouteHandler<StripeWebhookRoute> = async (c) => {
  const logger = c.get("logger");

  // Get Stripe signature from headers
  const signature = c.req.header("stripe-signature");
  if (!signature) {
    return c.json({ success: false, error: "BadRequest", message: "Missing signature" }, 400);
  }

  // Get raw body and webhook secret
  const payload = await c.req.text();
  const webhookSecret = getWebhookSecret();

  // Parse and verify using StripeGateway
  const gateway = new StripeGateway();
  const event = gateway.parseWebhookEvent(payload, signature, webhookSecret);

  logger.info({ eventId: event.id, eventType: event.type }, "Processing webhook");

  // Delegate to common processor
  const deps = buildServiceDependencies(c);
  await processPaymentWebhookEvent(event, deps);

  return c.json({ message: `Webhook ${event.type} processed successfully` }, 200);
};
```

### Event Processor

```typescript
// payments-webhook.processor.ts

export async function processPaymentWebhookEvent(
  event: WebhookEvent,
  deps: ServiceDependencies
): Promise<void> {
  const { logger } = deps;

  logger.info({ eventId: event.id, eventType: event.type }, "Processing event");

  switch (event.type) {
    case "checkout.completed":
      await handleCheckoutCompleted(event, deps);
      break;
    case "payment.succeeded":
      await handlePaymentSucceeded(event, deps);
      break;
    case "subscription.canceled":
      await handleSubscriptionCanceled(event, deps);
      break;
    // ... other handlers
    default:
      logger.info({ eventType: event.type }, "Unhandled event type");
  }
}
```

## Database Updates

### Checkout Completed (Subscription)

```typescript
await db.update(subscriptions).set({
  externalSubscriptionId: data.subscriptionId,
  externalCustomerId: data.customerId,
  paymentProvider: event.provider,  // 'stripe' or 'lenco'
  status: "active",
  updatedAt: new Date(),
}).where(eq(subscriptions.organizationId, organizationId));
```

### Payment Succeeded (One-time)

```typescript
await db.update(paymentRequests).set({
  status: "paid_platform_unverified",
  externalPaymentId: data.paymentId,
  paymentProvider: event.provider,
  paidAt: new Date(),
  updatedAt: new Date(),
}).where(eq(paymentRequests.id, data.paymentRequestId));
```

### Subscription Canceled

```typescript
await db.update(subscriptions).set({
  status: "cancelled",
  updatedAt: new Date(),
}).where(eq(subscriptions.externalSubscriptionId, data.subscriptionId));
```

## Metadata Strategy

Critical identifiers are stored in payment metadata to survive the webhook round-trip:

### Required Metadata Fields

| Field | Purpose | Set By |
|-------|---------|--------|
| `organizationId` | Tenant identification | Payment initiation |
| `paymentRequestId` | Link to payment_requests | Payment initiation |
| `sourceType` | `filing` or `service_request` | Payment initiation |
| `sourceId` | Filing or service request ID | Payment initiation |
| `userId` | Who initiated payment | Payment initiation |

### Metadata Flow

```
1. Service creates payment session:
   gateway.createPaymentSession({
     metadata: {
       organizationId: 'org_123',
       paymentRequestId: 'pr_456',
       sourceType: 'filing',
       sourceId: 'fil_789',
     }
   })
        ↓
2. Provider stores metadata with payment
        ↓
3. Payment completes
        ↓
4. Provider sends webhook with metadata
        ↓
5. Handler extracts from event.data:
   event.data.paymentRequestId === 'pr_456'
```

## Signature Verification

### Stripe

```
Header: stripe-signature
Format: t=timestamp,v1=signature
Algorithm: HMAC-SHA256
```

### Lenco

```
Header: x-lenco-signature (or x-webhook-signature)
Format: sha256=signature
Algorithm: HMAC-SHA256
```

### Verification Process

The gateway handles provider-specific verification internally:

```typescript
// Gateway.parseWebhookEvent() does both verification and parsing
const event = gateway.parseWebhookEvent(payload, signature, webhookSecret);

// Throws PaymentError if verification fails:
// - WEBHOOK_VERIFICATION_FAILED: Invalid signature
// - WEBHOOK_PARSE_ERROR: Malformed payload
```

## Provider Configuration

### Stripe Dashboard

1. Go to Developers → Webhooks
2. Add endpoint: `https://api.bumara.com/webhooks/payments/stripe`
3. Select events:
   - `checkout.session.completed`
   - `checkout.session.expired`
   - `customer.subscription.created`
   - `customer.subscription.updated`
   - `customer.subscription.deleted`
   - `payment_intent.succeeded`
   - `payment_intent.payment_failed`
   - `charge.refunded`
   - `invoice.paid`
   - `invoice.payment_failed`
4. Copy signing secret to `STRIPE_WEBHOOK_SECRET`

### Lenco Dashboard

1. Go to Settings → Webhooks
2. Add endpoint: `https://api.bumara.com/webhooks/payments/lenco`
3. Enable events:
   - Checkout events
   - Payment events
   - Subscription events
4. Copy webhook secret to `LENCO_WEBHOOK_SECRET`

## Environment Variables

```bash
# Stripe webhook secret (starts with whsec_)
STRIPE_WEBHOOK_SECRET=whsec_...

# Lenco webhook secret
LENCO_WEBHOOK_SECRET=...

# Or use generic key (provider auto-detected)
PAYMENT_WEBHOOK_SECRET=...
```

## Testing Webhooks

### Local Development with Stripe CLI

```bash
# Forward Stripe webhooks to local backend
stripe listen --forward-to localhost:3000/webhooks/payments/stripe

# The CLI provides a webhook signing secret for local testing
# Add this to your .env as STRIPE_WEBHOOK_SECRET
```

### Mock Provider

The mock provider is useful for unit tests:

```typescript
import { MockGateway } from "@repo/payments";

const mock = new MockGateway();

// Parse a mock event (no signature verification)
const event = mock.parseWebhookEvent(
  JSON.stringify({ type: "payment.succeeded", ... }),
  "any-signature",
  "mock-secret"
);
```

## Error Handling

### Retry Policy

Payment providers automatically retry failed webhook deliveries:

| Provider | Initial Retry | Max Retries | Backoff |
|----------|---------------|-------------|---------|
| Stripe | 1 hour | 72 hours | Exponential |
| Lenco | 5 minutes | 24 hours | Exponential |

### Idempotency

The processor handles duplicate webhooks gracefully by checking current state:

```typescript
// Subscription already linked - no-op
if (existingSub.externalSubscriptionId === data.subscriptionId) {
  logger.info("Subscription already linked");
  return;
}
```

### Error Responses

| Status | Meaning | Provider Action |
|--------|---------|-----------------|
| 200 | Success | Mark delivered |
| 400 | Bad request (invalid signature) | Don't retry |
| 500 | Server error | Retry |

## Monitoring

### Key Metrics

- Webhook delivery success rate
- Processing latency per event type
- Event type distribution
- Verification failures (possible attacks)

### Logging

All events are logged with structured context:

```typescript
logger.info(
  { eventId: event.id, eventType: event.type, provider: event.provider },
  "Processing payment webhook event"
);

logger.warn(
  { code: error.code },
  "Webhook verification failed"
);
```

### Alerts

Set up alerts for:
- High verification failure rate (possible signature misconfiguration)
- Unusual event volume (spikes or drops)
- Processing errors (database issues)
- Missing expected events (e.g., no payment.succeeded after checkout)
