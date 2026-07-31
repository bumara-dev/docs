---
title: "Testing"
description: "The mock provider enables testing without external API calls:"
---

## Mock Provider

The mock provider enables testing without external API calls:

```typescript
import { MockGateway } from "@repo/payments";

const mock = new MockGateway();
```

### Enabling Mock Provider

Set environment variable:

```env
PAYMENT_PROVIDER=mock
```

Or create directly in tests:

```typescript
import { createGateway } from "@repo/payments";

const gateway = createGateway("mock");
```

## Mock Provider Features

### In-Memory Storage

The mock provider maintains in-memory stores:

```typescript
class MockGateway implements PaymentGateway {
  private sessions = new Map<string, PaymentSession>();
  private payments = new Map<string, Payment>();
  private subscriptions = new Map<string, Subscription>();
  private customers = new Map<string, Customer>();
}
```

### Deterministic IDs

IDs are generated with predictable prefixes:

| Entity | ID Format |
|--------|-----------|
| Checkout Session | `mock_cs_${uuid}` |
| Payment Session | `mock_ps_${uuid}` |
| Payment | `mock_pi_${uuid}` |
| Subscription | `mock_sub_${uuid}` |
| Customer | `mock_cus_${uuid}` |
| Refund | `mock_re_${uuid}` |

### Simulation Helpers

```typescript
// Simulate successful payment
mock.simulatePaymentSuccess(sessionId: string);

// Simulate subscription creation
mock.simulateSubscriptionCreated(sessionId: string);

// Clear all stored data
mock.clearAll();
```

## Unit Testing

### Testing Gateway Methods

```typescript
import { describe, it, expect, beforeEach } from "vitest";
import { MockGateway } from "@repo/payments";

describe("PaymentGateway", () => {
  let gateway: MockGateway;

  beforeEach(() => {
    gateway = new MockGateway();
  });

  describe("createPaymentSession", () => {
    it("creates a payment session", async () => {
      const session = await gateway.createPaymentSession({
        organizationId: "org_123",
        userId: "user_456",
        amount: 25000,
        currency: "ZMW",
        description: "Test payment",
        successUrl: "https://example.com/success",
        cancelUrl: "https://example.com/cancel",
        metadata: {
          paymentRequestId: "pr_789",
          sourceType: "filing",
          sourceId: "fil_abc",
        },
      });

      expect(session.id).toMatch(/^mock_ps_/);
      expect(session.url).toContain("mock-checkout");
      expect(session.status).toBe("pending");
      expect(session.amount).toBe(25000);
      expect(session.currency).toBe("ZMW");
    });
  });

  describe("createSubscriptionCheckout", () => {
    it("creates a checkout session", async () => {
      const session = await gateway.createSubscriptionCheckout({
        organizationId: "org_123",
        userId: "user_456",
        planId: "plus",
        billingPeriod: "monthly",
        successUrl: "https://example.com/success",
        cancelUrl: "https://example.com/cancel",
      });

      expect(session.id).toMatch(/^mock_cs_/);
      expect(session.url).toContain("mock-checkout");
      expect(session.status).toBe("pending");
    });
  });

  describe("refundPayment", () => {
    it("refunds a payment", async () => {
      // Create payment first
      const session = await gateway.createPaymentSession({
        organizationId: "org_123",
        userId: "user_456",
        amount: 25000,
        currency: "ZMW",
        description: "Test payment",
        successUrl: "https://example.com/success",
        cancelUrl: "https://example.com/cancel",
        metadata: {
          paymentRequestId: "pr_789",
          sourceType: "filing",
          sourceId: "fil_abc",
        },
      });

      // Simulate payment success
      gateway.simulatePaymentSuccess(session.id);

      // Get payment ID
      const payment = await gateway.getPayment(session.paymentId!);

      // Refund
      const refund = await gateway.refundPayment(payment!.id, {
        reason: "requested_by_customer",
      });

      expect(refund.id).toMatch(/^mock_re_/);
      expect(refund.paymentId).toBe(payment!.id);
      expect(refund.amount).toBe(25000);
      expect(refund.status).toBe("succeeded");
    });
  });
});
```

### Testing Service Layer

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";
import { initiatePayment } from "@repo/api-services/domains/compliance";

// Mock the payments package
vi.mock("@repo/payments", () => ({
  getPaymentGateway: vi.fn().mockResolvedValue({
    providerName: "mock",
    createPaymentSession: vi.fn().mockResolvedValue({
      id: "mock_ps_123",
      url: "https://mock-checkout.com/session",
      status: "pending",
      amount: 25000,
      currency: "ZMW",
      metadata: {},
    }),
  }),
}));

describe("initiatePayment", () => {
  const mockDb = {
    query: {
      paymentRequests: {
        findFirst: vi.fn(),
      },
      organizations: {
        findFirst: vi.fn(),
      },
    },
    update: vi.fn().mockReturnValue({
      set: vi.fn().mockReturnValue({
        where: vi.fn().mockResolvedValue(undefined),
      }),
    }),
  };

  beforeEach(() => {
    vi.clearAllMocks();

    mockDb.query.paymentRequests.findFirst.mockResolvedValue({
      id: "pr_123",
      organizationId: "org_123",
      status: "required_pending",
      totalAmount: 25000,
      currency: "ZMW",
      filingId: "fil_456",
    });

    mockDb.query.organizations.findFirst.mockResolvedValue({
      name: "Test Org",
    });
  });

  it("creates payment session and returns checkout URL", async () => {
    const ctx = { orgId: "org_123", userId: "user_456" };
    const deps = { db: mockDb };

    const result = await initiatePayment(ctx, deps, {
      paymentRequestId: "pr_123",
      successUrl: "https://example.com/success",
      cancelUrl: "https://example.com/cancel",
    });

    expect(result.checkoutUrl).toBe("https://mock-checkout.com/session");
    expect(result.sessionId).toBe("mock_ps_123");
  });

  it("throws if payment not found", async () => {
    mockDb.query.paymentRequests.findFirst.mockResolvedValue(null);

    const ctx = { orgId: "org_123", userId: "user_456" };
    const deps = { db: mockDb };

    await expect(
      initiatePayment(ctx, deps, {
        paymentRequestId: "pr_invalid",
        successUrl: "https://example.com/success",
        cancelUrl: "https://example.com/cancel",
      })
    ).rejects.toThrow("Payment request not found");
  });

  it("throws if payment already processed", async () => {
    mockDb.query.paymentRequests.findFirst.mockResolvedValue({
      id: "pr_123",
      status: "paid_platform_verified", // Already paid
    });

    const ctx = { orgId: "org_123", userId: "user_456" };
    const deps = { db: mockDb };

    await expect(
      initiatePayment(ctx, deps, {
        paymentRequestId: "pr_123",
        successUrl: "https://example.com/success",
        cancelUrl: "https://example.com/cancel",
      })
    ).rejects.toThrow("Cannot initiate payment");
  });
});
```

### Testing Webhooks

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";
import { POST } from "@/app/webhooks/payments/route";

// Mock dependencies
vi.mock("@repo/payments", () => ({
  getPaymentGateway: vi.fn().mockResolvedValue({
    providerName: "mock",
    parseWebhookEvent: vi.fn(),
  }),
  getWebhookSecret: vi.fn().mockReturnValue("test_secret"),
}));

vi.mock("@repo/database", () => ({
  default: {
    update: vi.fn().mockReturnValue({
      set: vi.fn().mockReturnValue({
        where: vi.fn().mockResolvedValue(undefined),
      }),
    }),
  },
}));

describe("Payment Webhook Handler", () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it("handles payment.succeeded event", async () => {
    const { getPaymentGateway } = await import("@repo/payments");

    (getPaymentGateway as any).mockResolvedValue({
      providerName: "mock",
      parseWebhookEvent: vi.fn().mockReturnValue({
        id: "evt_123",
        type: "payment.succeeded",
        data: {
          paymentRequestId: "pr_456",
          paymentId: "pi_789",
          organizationId: "org_123",
          amount: 25000,
          currency: "ZMW",
        },
        provider: "mock",
      }),
    });

    const request = new Request("http://localhost/webhooks/payments", {
      method: "POST",
      body: JSON.stringify({ type: "payment.succeeded" }),
      headers: {
        "stripe-signature": "test_signature",
      },
    });

    const response = await POST(request);
    const data = await response.json();

    expect(response.status).toBe(200);
    expect(data.ok).toBe(true);
  });

  it("rejects invalid signature", async () => {
    const { getPaymentGateway } = await import("@repo/payments");

    (getPaymentGateway as any).mockResolvedValue({
      providerName: "mock",
      parseWebhookEvent: vi.fn().mockImplementation(() => {
        throw new Error("Invalid signature");
      }),
    });

    const request = new Request("http://localhost/webhooks/payments", {
      method: "POST",
      body: JSON.stringify({ type: "payment.succeeded" }),
      headers: {
        "stripe-signature": "invalid_signature",
      },
    });

    const response = await POST(request);

    expect(response.status).toBe(500);
  });
});
```

## Integration Testing

### Stripe Test Mode

Use Stripe test mode for integration tests:

```env
PAYMENT_PROVIDER=stripe
PAYMENT_API_KEY=sk_test_xxxxxxxxxxxx
```

#### Test Card Numbers

| Card | Result |
|------|--------|
| `4242424242424242` | Success |
| `4000000000000002` | Decline |
| `4000000000009995` | Insufficient funds |
| `4000000000000069` | Expired card |

### Webhook Testing

#### Stripe CLI

Forward webhooks to local development:

```bash
# Install Stripe CLI
brew install stripe/stripe-cli/stripe

# Login
stripe login

# Forward webhooks
stripe listen --forward-to localhost:3000/api/webhooks/payments

# In another terminal, trigger events
stripe trigger payment_intent.succeeded
stripe trigger checkout.session.completed
```

#### Manual Trigger

```bash
# Trigger specific event
stripe trigger payment_intent.succeeded \
  --override payment_intent:metadata.organizationId=org_123 \
  --override payment_intent:metadata.paymentRequestId=pr_456
```

### End-to-End Testing

```typescript
import { test, expect } from "@playwright/test";

test.describe("Payment Flow", () => {
  test("completes payment checkout", async ({ page }) => {
    // Navigate to filing
    await page.goto("/filings/fil_123");

    // Click pay button
    await page.click('button:has-text("Pay Now")');

    // Should redirect to checkout
    await expect(page).toHaveURL(/checkout/);

    // Fill test card (Stripe test mode)
    await page.fill('[data-testid="card-number"]', "4242424242424242");
    await page.fill('[data-testid="card-expiry"]', "12/25");
    await page.fill('[data-testid="card-cvc"]', "123");

    // Submit payment
    await page.click('button:has-text("Pay")');

    // Should redirect to success page
    await expect(page).toHaveURL(/payment\/success/);
  });
});
```

## Test Fixtures

### Payment Request Fixture

```typescript
export const paymentRequestFixture = {
  id: "pr_test_123",
  organizationId: "org_test_123",
  filingId: "fil_test_456",
  serviceRequestId: null,
  status: "required_pending",
  currency: "ZMW",
  regulatorFee: 25000,
  handlingFee: 2500,
  totalAmount: 27500,
  requestedAt: new Date("2024-01-15"),
  paidAt: null,
  verifiedAt: null,
  externalSessionId: null,
  externalPaymentId: null,
  paymentProvider: null,
  createdAt: new Date("2024-01-15"),
  updatedAt: new Date("2024-01-15"),
};
```

### Subscription Fixture

```typescript
export const subscriptionFixture = {
  id: "sub_test_123",
  organizationId: "org_test_123",
  plan: "plus",
  planFeatures: {
    maxUsers: 5,
    maxRegulators: 3,
  },
  status: "active",
  trialStartsAt: null,
  trialEndsAt: null,
  currentPeriodStart: new Date("2024-01-01"),
  currentPeriodEnd: new Date("2024-02-01"),
  externalCustomerId: "cus_test_123",
  externalSubscriptionId: "sub_stripe_123",
  paymentProvider: "stripe",
  createdAt: new Date("2024-01-01"),
  updatedAt: new Date("2024-01-01"),
};
```

## Test Coverage

### Critical Paths to Test

1. **Subscription checkout flow**
   - Session creation
   - Webhook processing
   - Subscription activation

2. **Compliance payment flow**
   - Fee calculation
   - Payment request creation
   - Session creation
   - Webhook processing
   - Auto-verification

3. **Error handling**
   - Invalid signatures
   - Missing metadata
   - Already processed payments
   - Refund edge cases

4. **Provider switching**
   - Factory returns correct provider
   - Configuration validation

### Coverage Goals

| Area | Target |
|------|--------|
| Gateway interface | 90% |
| Webhook handlers | 95% |
| Service layer | 85% |
| Error handling | 100% |
