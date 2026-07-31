---
title: "Payments Architecture"
---

## Design Principles

1. **Provider Abstraction**: Business logic never interacts with provider SDKs directly
2. **Single Source of Truth**: Database records track all payment state
3. **Idempotency**: All operations support retry without side effects
4. **Audit Trail**: Every payment action is logged for compliance
5. **Multi-tenant Isolation**: All queries scoped by `organizationId`

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Application Layer                                  │
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │  Billing Page   │  │ Filing/Service  │  │  Webhook Handler │             │
│  │  (Subscriptions)│  │ (Compliance)    │  │                  │             │
│  └────────┬────────┘  └────────┬────────┘  └────────┬─────────┘             │
│           │                    │                    │                        │
└───────────┼────────────────────┼────────────────────┼────────────────────────┘
            │                    │                    │
            ▼                    ▼                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Service Layer                                       │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    PaymentGateway Interface                          │   │
│  │                                                                      │   │
│  │  Subscriptions:                    One-Time Payments:                │   │
│  │  • createSubscriptionCheckout()    • createPaymentSession()          │   │
│  │  • getSubscription()               • getPayment()                    │   │
│  │  • updateSubscription()            • refundPayment()                 │   │
│  │  • cancelSubscription()                                              │   │
│  │                                                                      │   │
│  │  Customers:                        Webhooks:                         │   │
│  │  • createCustomer()                • verifyWebhookSignature()        │   │
│  │  • getCustomer()                   • parseWebhookEvent()             │   │
│  │  • updateCustomer()                                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
└────────────────────────────────────┼────────────────────────────────────────┘
                                     │
                     ┌───────────────┼───────────────┐
                     ▼               ▼               ▼
              ┌──────────┐   ┌──────────┐   ┌──────────┐
              │  Stripe  │   │  Lenco   │   │  Mock    │
              │ Adapter  │   │ Adapter  │   │ Adapter  │
              └──────────┘   └──────────┘   └──────────┘
                     │               │               │
                     ▼               ▼               ▼
              ┌──────────┐   ┌──────────┐   ┌──────────┐
              │ Stripe   │   │ Lenco    │   │ In-Memory│
              │   API    │   │   API    │   │  Store   │
              └──────────┘   └──────────┘   └──────────┘
```

## Provider Pattern

### Gateway Interface

All providers implement the `PaymentGateway` interface:

```typescript
interface PaymentGateway {
  readonly providerName: string;

  // Subscription checkout
  createSubscriptionCheckout(input: CreateSubscriptionCheckoutInput): Promise<CheckoutSession>;
  retrieveCheckoutSession(sessionId: string): Promise<CheckoutSession | null>;

  // One-time payments
  createPaymentSession(input: CreatePaymentSessionInput): Promise<PaymentSession>;
  getPayment(paymentId: string): Promise<Payment | null>;
  refundPayment(paymentId: string, input: RefundInput): Promise<Refund>;

  // Subscription management
  getSubscription(subscriptionId: string): Promise<Subscription | null>;
  updateSubscription(subscriptionId: string, input: UpdateSubscriptionInput): Promise<Subscription>;
  cancelSubscription(subscriptionId: string, options?: CancelOptions): Promise<void>;

  // Customer management
  createCustomer(input: CreateCustomerInput): Promise<Customer>;
  getCustomer(customerId: string): Promise<Customer | null>;
  updateCustomer(customerId: string, input: UpdateCustomerInput): Promise<Customer>;

  // Webhooks
  verifyWebhookSignature(payload: string, signature: string, secret: string): boolean;
  parseWebhookEvent(payload: string, signature: string, secret: string): WebhookEvent;
}
```

### Provider Factory

The factory returns a singleton gateway instance based on configuration:

```typescript
import { getPaymentGateway } from "@repo/payments";

// Returns cached instance, creates on first call
const gateway = await getPaymentGateway();
console.log(gateway.providerName); // 'stripe', 'lenco', or 'mock'
```

### Provider Selection

```typescript
// Determined by PAYMENT_PROVIDER env var
switch (config().PAYMENT_PROVIDER) {
  case 'stripe':
    return new StripeGateway();
  case 'lenco':
    return new LencoGateway();
  case 'mock':
    return new MockGateway();
}
```

## Data Flow

### Subscription Checkout Flow

```
1. User selects plan on billing page
                ↓
2. Frontend calls API: POST /api/billing/checkout
                ↓
3. API creates checkout session via gateway
   gateway.createSubscriptionCheckout({
     organizationId, planId, billingPeriod, ...
   })
                ↓
4. Gateway returns checkout URL
   { id: 'cs_xxx', url: 'https://checkout.stripe.com/...' }
                ↓
5. User redirected to provider checkout page
                ↓
6. User completes payment
                ↓
7. Provider redirects to successUrl
                ↓
8. Webhook received: checkout.session.completed
                ↓
9. Webhook handler updates subscription:
   - subscriptions.status = 'active'
   - subscriptions.externalSubscriptionId = 'sub_xxx'
   - subscriptions.paymentProvider = 'stripe'
```

### Compliance Payment Flow

```
1. Tenant initiates filing or service request
                ↓
2. System calculates fees:
   calculatePaymentForSource() → FeeBreakdown
   { regulatorFee: 25000, handlingFee: 2500, totalAmount: 27500, currency: 'ZMW' }
                ↓
3. Create payment request record:
   createPaymentRequest() → payment_requests row
   status: 'required_pending'
                ↓
4. Tenant clicks "Pay Now" button
                ↓
5. API calls initiatePayment():
   gateway.createPaymentSession({
     amount: 27500,
     currency: 'ZMW',
     metadata: { paymentRequestId, sourceType, sourceId }
   })
                ↓
6. User redirected to checkout
                ↓
7. Payment completed
                ↓
8. Webhook: payment.succeeded
                ↓
9. handlePaymentSucceeded():
   - payment_requests.status = 'paid_platform_verified'
   - payment_requests.externalPaymentId = 'pi_xxx'
   - Close payment ticket
   - Log for auto-submission
                ↓
10. Submission job picks up and submits to regulator
```

## Webhook Architecture

### Event Normalization

Provider-specific events are mapped to normalized types:

| Stripe Event | Lenco Event | Normalized Event |
|--------------|-------------|------------------|
| `checkout.session.completed` | `checkout.completed` | `checkout.completed` |
| `customer.subscription.deleted` | `subscription.cancelled` | `subscription.canceled` |
| `payment_intent.succeeded` | `payment.success` | `payment.succeeded` |
| `payment_intent.payment_failed` | `payment.failed` | `payment.failed` |

### Event Data Extraction

```typescript
interface WebhookEvent {
  id: string;                    // Event ID
  type: WebhookEventType;        // Normalized type
  data: WebhookEventData;        // Extracted data
  provider: string;              // 'stripe', 'lenco'
  createdAt: Date;
  rawEvent: unknown;             // Original provider event
}



interface WebhookEventData {
  // Common
  organizationId?: string;       // From metadata
  customerId?: string;

  // Subscription events
  subscriptionId?: string;
  planId?: string;

  // Payment events
  paymentId?: string;
  paymentRequestId?: string;     // From metadata
  sourceType?: string;           // From metadata
  sourceId?: string;             // From metadata
  amount?: number;
  currency?: string;
}
```

### Metadata Strategy

Critical identifiers are stored in provider metadata to survive the webhook round-trip:

```typescript
// When creating payment session
await gateway.createPaymentSession({
  metadata: {
    organizationId: 'org_123',      // For tenant lookup
    paymentRequestId: 'pr_456',     // For payment request lookup
    sourceType: 'filing',           // filing or service_request
    sourceId: 'fil_789',            // Filing or service request ID
    userId: 'user_abc',             // Who initiated
  }
});

// Metadata returned in webhook
// event.data.organizationId === 'org_123'
// event.data.paymentRequestId === 'pr_456'
```

## Error Handling

### Error Types

```typescript
class PaymentError extends Error {
  code: PaymentErrorCode;
  details?: Record<string, unknown>;
}

type PaymentErrorCode =
  | 'PROVIDER_NOT_IMPLEMENTED'
  | 'CONFIGURATION_ERROR'
  | 'WEBHOOK_VERIFICATION_FAILED'
  | 'WEBHOOK_PARSE_ERROR'
  | 'PAYMENT_FAILED'
  | 'ALREADY_REFUNDED'
  | 'INVALID_AMOUNT'
  | 'CUSTOMER_NOT_FOUND'
  | 'SUBSCRIPTION_NOT_FOUND';
```

### Error Wrapping

Provider errors are wrapped with context:

```typescript
try {
  return await this.client.paymentIntents.retrieve(id);
} catch (error) {
  throw wrapProviderError(error, this.providerName);
  // PaymentError with original error in details
}
```

## Security Considerations

### Webhook Verification

All webhooks must pass signature verification:

```typescript
// Stripe uses HMAC-SHA256
const isValid = gateway.verifyWebhookSignature(
  rawBody,
  signature,  // From stripe-signature header
  webhookSecret
);

if (!isValid) {
  return Response.json({ error: 'Invalid signature' }, { status: 401 });
}
```

### Tenant Isolation

All operations enforce `organizationId` scope:

```typescript
// Payment request lookup
const payment = await db.query.paymentRequests.findFirst({
  where: and(
    eq(paymentRequests.id, input.paymentRequestId),
    eq(paymentRequests.organizationId, ctx.orgId)  // Always scoped
  ),
});
```

### Sensitive Data

- API keys stored in environment variables only
- No card details stored in database
- External IDs stored for reference only
- PII not logged

## Idempotency

### Key Generation

```typescript
import { generateIdempotencyKey, paymentSessionKey } from "@repo/payments";

// For payment sessions
const key = paymentSessionKey(organizationId, paymentRequestId);
// Output: sha256('payment_session:org_123:pr_456').slice(0, 32)

// For subscriptions
const key = subscriptionCheckoutKey(organizationId, planId, billingPeriod);
```

### Time-Windowed Keys

For operations that should be allowed after a time window:

```typescript
import { generateTimeWindowKey } from "@repo/payments";

// Allow retry after 60 minutes
const key = generateTimeWindowKey('payment_retry', 60, paymentRequestId);
```

## Provider Implementation Guide

### Adding a New Provider

1. Create adapter directory: `packages/payments/providers/newprovider/`

2. Implement gateway interface:
```typescript
export class NewProviderGateway implements PaymentGateway {
  readonly providerName = 'newprovider';

  async createSubscriptionCheckout(input) {
    // Provider-specific implementation
  }

  // ... implement all interface methods
}
```

3. Add type mappers:
```typescript
export function mapNewProviderSession(session: ProviderSession): CheckoutSession {
  return {
    id: session.id,
    url: session.checkout_url,
    status: mapStatus(session.state),
    // ...
  };
}
```

4. Register in factory:
```typescript
case 'newprovider': {
  const { NewProviderGateway } = await import('../providers/newprovider');
  return new NewProviderGateway();
}
```

5. Update config:
```typescript
PAYMENT_PROVIDER: z.enum(['stripe', 'lenco', 'newprovider', 'mock'])
```
