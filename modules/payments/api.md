---
title: "Payments API Reference"
description: "The PaymentGateway interface provides all payment operations. Get an instance using the factory:"
---

## Gateway API

The `PaymentGateway` interface provides all payment operations. Get an instance using the factory:

```typescript
import { getPaymentGateway } from "@repo/payments";

const gateway = await getPaymentGateway();
```

---

## Subscription Operations

### createSubscriptionCheckout

Create a checkout session for a new subscription.

```typescript
const session = await gateway.createSubscriptionCheckout({
  organizationId: string;     // Bumara organization ID
  userId: string;             // User initiating checkout
  planId: string;             // Plan: 'start', 'plus', 'pro', 'enterprise'
  billingPeriod: 'monthly' | 'yearly';
  successUrl: string;         // Redirect URL on success
  cancelUrl: string;          // Redirect URL on cancel
  customerEmail?: string;     // Pre-fill email
  trialDays?: number;         // Trial period days
  metadata?: Record<string, string>;
});

// Returns
{
  id: string;           // Session ID
  url: string;          // Checkout URL (redirect user here)
  status: 'pending' | 'complete' | 'expired';
  customerId?: string;
  subscriptionId?: string;
  metadata: Record<string, string>;
}
```

**Example:**
```typescript
const session = await gateway.createSubscriptionCheckout({
  organizationId: 'org_abc123',
  userId: 'user_xyz',
  planId: 'plus',
  billingPeriod: 'monthly',
  successUrl: 'https://app.bumara.com/billing/success?session_id={CHECKOUT_SESSION_ID}',
  cancelUrl: 'https://app.bumara.com/billing',
  customerEmail: 'user@company.com',
  trialDays: 14,
});

// Redirect user
redirect(session.url);
```

### retrieveCheckoutSession

Get details of a checkout session.

```typescript
const session = await gateway.retrieveCheckoutSession(sessionId: string);

// Returns CheckoutSession | null
```

### getSubscription

Retrieve subscription details from the provider.

```typescript
const subscription = await gateway.getSubscription(subscriptionId: string);

// Returns
{
  id: string;
  customerId: string;
  status: 'trialing' | 'active' | 'past_due' | 'canceled' | 'unpaid' | 'paused';
  planId: string;
  priceId?: string;
  currentPeriodStart: Date;
  currentPeriodEnd: Date;
  cancelAtPeriodEnd: boolean;
  canceledAt?: Date;
  trialStart?: Date;
  trialEnd?: Date;
  metadata: Record<string, string>;
}
```

### updateSubscription

Update subscription details (e.g., change plan).

```typescript
const subscription = await gateway.updateSubscription(
  subscriptionId: string,
  {
    priceId?: string;           // New price/plan ID
    cancelAtPeriodEnd?: boolean;
    metadata?: Record<string, string>;
  }
);
```

**Example - Upgrade plan:**
```typescript
await gateway.updateSubscription('sub_abc', {
  priceId: 'price_pro_monthly',
});
```

### cancelSubscription

Cancel a subscription.

```typescript
await gateway.cancelSubscription(
  subscriptionId: string,
  options?: {
    immediate?: boolean;  // true = cancel now, false = cancel at period end
    reason?: string;
  }
);
```

**Example:**
```typescript
// Cancel at period end (default)
await gateway.cancelSubscription('sub_abc');

// Cancel immediately
await gateway.cancelSubscription('sub_abc', { immediate: true });
```

---

## One-Time Payment Operations

### createPaymentSession

Create a checkout session for a one-time payment (compliance fees).

```typescript
const session = await gateway.createPaymentSession({
  organizationId: string;
  userId: string;
  amount: number;           // In minor units (ngwee, cents)
  currency: string;         // 'ZMW', 'USD'
  description: string;      // Payment description
  successUrl: string;
  cancelUrl: string;
  customerEmail?: string;
  metadata: {
    paymentRequestId: string;   // Required - links to payment_requests
    sourceType: 'filing' | 'service_request';
    sourceId: string;
    regulatorId?: string;
  };
});

// Returns
{
  id: string;           // Session ID
  url: string;          // Checkout URL
  status: 'pending' | 'complete' | 'expired';
  paymentId?: string;   // Set when payment created
  amount: number;
  currency: string;
  metadata: Record<string, string>;
}
```

**Example:**
```typescript
const session = await gateway.createPaymentSession({
  organizationId: 'org_abc',
  userId: 'user_xyz',
  amount: 27500,  // K275.00 in ngwee
  currency: 'ZMW',
  description: 'PACRA Annual Return Filing',
  successUrl: 'https://app.bumara.com/filings/fil_123/payment/success',
  cancelUrl: 'https://app.bumara.com/filings/fil_123',
  metadata: {
    paymentRequestId: 'pr_456',
    sourceType: 'filing',
    sourceId: 'fil_123',
    regulatorId: 'pacra',
  },
});
```

### getPayment

Retrieve payment details.

```typescript
const payment = await gateway.getPayment(paymentId: string);

// Returns
{
  id: string;
  sessionId?: string;
  amount: number;
  currency: string;
  status: 'pending' | 'processing' | 'succeeded' | 'failed' | 'refunded';
  paidAt?: Date;
  metadata: Record<string, string>;
}
```

### refundPayment

Refund a payment (full or partial).

```typescript
const refund = await gateway.refundPayment(
  paymentId: string,
  {
    amount?: number;    // Partial refund amount (minor units), omit for full
    reason?: 'duplicate' | 'fraudulent' | 'requested_by_customer' | 'other';
    metadata?: Record<string, string>;
  }
);

// Returns
{
  id: string;
  paymentId: string;
  amount: number;
  currency: string;
  status: 'pending' | 'succeeded' | 'failed';
  reason?: string;
  createdAt: Date;
}
```

**Example:**
```typescript
// Full refund
await gateway.refundPayment('pi_abc');

// Partial refund
await gateway.refundPayment('pi_abc', {
  amount: 5000,  // Refund K50.00
  reason: 'requested_by_customer',
});
```

---

## Customer Operations

### createCustomer

Create a customer record with the provider.

```typescript
const customer = await gateway.createCustomer({
  organizationId: string;
  email: string;
  name?: string;
  phone?: string;
  metadata?: Record<string, string>;
});

// Returns
{
  id: string;
  email: string;
  name?: string;
  phone?: string;
  metadata: Record<string, string>;
  createdAt: Date;
}
```

### getCustomer

Retrieve customer details.

```typescript
const customer = await gateway.getCustomer(customerId: string);
// Returns Customer | null
```

### updateCustomer

Update customer details.

```typescript
const customer = await gateway.updateCustomer(
  customerId: string,
  {
    email?: string;
    name?: string;
    phone?: string;
    metadata?: Record<string, string>;
  }
);
```

---

## Webhook Operations

### verifyWebhookSignature

Verify webhook signature is valid.

```typescript
const isValid = gateway.verifyWebhookSignature(
  payload: string,      // Raw request body
  signature: string,    // Signature header value
  secret: string        // Webhook secret
);

// Returns boolean
```

### parseWebhookEvent

Parse and normalize webhook event.

```typescript
const event = gateway.parseWebhookEvent(
  payload: string,
  signature: string,
  secret: string
);

// Returns
{
  id: string;
  type: WebhookEventType;
  data: WebhookEventData;
  provider: string;
  createdAt: Date;
  rawEvent: unknown;
}
```

**WebhookEventType values:**
- `checkout.completed`
- `checkout.expired`
- `subscription.created`
- `subscription.updated`
- `subscription.canceled`
- `payment.pending`
- `payment.succeeded`
- `payment.failed`
- `payment.refunded`
- `invoice.paid`
- `invoice.payment_failed`

---

## Service Layer API

### calculatePaymentForSource

Calculate payment amount for a filing or service request.

```typescript
import { calculatePaymentForSource } from "@repo/api-services/domains/compliance";

const fees = await calculatePaymentForSource(ctx, deps, {
  sourceType: 'filing' | 'service_request';
  sourceId: string;
  regulatorId: string;
  templateId: string;
  entityConditions?: {
    entityType?: string;
    shareCapital?: number;
  };
});

// Returns FeeBreakdown | null
{
  regulatorFee: number;     // Minor units
  handlingFee: number;      // Minor units (10% of regulator fee)
  totalAmount: number;      // Minor units
  currency: string;
  feeSource: 'regulator_fees_table' | 'template_only' | 'company_registration' | 'default';
}
```

### createPaymentRequest

Create a payment request record.

```typescript
import { createPaymentRequest } from "@repo/api-services/domains/compliance";

const result = await createPaymentRequest(ctx, deps, {
  sourceType: 'filing' | 'service_request';
  sourceId: string;
  regulatorId: string;
  templateId: string;
  fees: FeeBreakdown;  // From calculatePaymentForSource
});

// Returns
{
  paymentRequest: PaymentRequest;
  ticket: Ticket;
}
```

### initiatePayment

Initiate payment via the gateway.

```typescript
import { initiatePayment } from "@repo/api-services/domains/compliance";

const result = await initiatePayment(ctx, deps, {
  paymentRequestId: string;
  successUrl: string;
  cancelUrl: string;
});

// Returns
{
  checkoutUrl: string;
  sessionId: string;
}
```

**Full flow example:**
```typescript
// 1. Calculate fees
const fees = await calculatePaymentForSource(ctx, deps, {
  sourceType: 'filing',
  sourceId: filing.id,
  regulatorId: filing.regulatorId,
  templateId: filing.templateId,
});

if (!fees) {
  // No payment required
  return;
}

// 2. Create payment request
const { paymentRequest } = await createPaymentRequest(ctx, deps, {
  sourceType: 'filing',
  sourceId: filing.id,
  regulatorId: filing.regulatorId,
  templateId: filing.templateId,
  fees,
});

// 3. Initiate payment (when user clicks Pay)
const { checkoutUrl } = await initiatePayment(ctx, deps, {
  paymentRequestId: paymentRequest.id,
  successUrl: `${baseUrl}/filings/${filing.id}/payment/success`,
  cancelUrl: `${baseUrl}/filings/${filing.id}`,
});

// 4. Redirect user to checkout
redirect(checkoutUrl);
```

### verifyPayment

Manually verify a payment (backoffice).

```typescript
import { verifyPayment } from "@repo/api-services/domains/compliance";

const result = await verifyPayment(ctx, deps, {
  paymentRequestId: string;
  verificationNotes?: string;
});

// Returns
{
  paymentRequest: PaymentRequest;
  shouldSubmit: boolean;
}
```

---

## Utility Functions

### Currency Utilities

```typescript
import {
  toMinorUnits,
  toMajorUnits,
  formatCurrency,
  getCurrencyConfig,
} from "@repo/payments";

// Convert to minor units for storage
toMinorUnits(250.00, 'ZMW');  // 25000

// Convert to major units for display
toMajorUnits(25000, 'ZMW');   // 250.00

// Format for UI
formatCurrency(25000, 'ZMW'); // 'K250.00'
formatCurrency(25000, 'ZMW', { showCode: true }); // 'ZMW 250.00'

// Get currency configuration
getCurrencyConfig('ZMW');
// { code: 'ZMW', symbol: 'K', decimals: 2, name: 'Zambian Kwacha' }
```

### Idempotency Utilities

```typescript
import {
  generateIdempotencyKey,
  paymentSessionKey,
  subscriptionCheckoutKey,
  refundKey,
  customerKey,
  generateTimeWindowKey,
} from "@repo/payments";

// Generic key
generateIdempotencyKey('operation', 'id1', 'id2');

// Payment session key
paymentSessionKey(organizationId, paymentRequestId);

// Subscription checkout key
subscriptionCheckoutKey(organizationId, planId, billingPeriod);

// Refund key
refundKey(paymentId, amount);

// Customer key
customerKey(organizationId, email);

// Time-windowed key (allows retry after window)
generateTimeWindowKey('payment_retry', 60, paymentRequestId);
```

---

## Error Handling

```typescript
import { PaymentError } from "@repo/payments";

try {
  await gateway.createPaymentSession({ ... });
} catch (error) {
  if (error instanceof PaymentError) {
    switch (error.code) {
      case 'CONFIGURATION_ERROR':
        // Missing API key or configuration
        break;
      case 'INVALID_AMOUNT':
        // Amount validation failed
        break;
      case 'PAYMENT_FAILED':
        // Payment processing failed
        break;
      case 'ALREADY_REFUNDED':
        // Payment already refunded
        break;
    }
  }
}
```

**Error codes:**
| Code | Description |
|------|-------------|
| `PROVIDER_NOT_IMPLEMENTED` | Provider method not yet implemented |
| `CONFIGURATION_ERROR` | Missing or invalid configuration |
| `WEBHOOK_VERIFICATION_FAILED` | Invalid webhook signature |
| `WEBHOOK_PARSE_ERROR` | Failed to parse webhook payload |
| `PAYMENT_FAILED` | Payment processing failed |
| `ALREADY_REFUNDED` | Payment was already refunded |
| `INVALID_AMOUNT` | Invalid payment amount |
| `CUSTOMER_NOT_FOUND` | Customer not found |
| `SUBSCRIPTION_NOT_FOUND` | Subscription not found |
