---
title: "Implementing Mock Payment Gateway - Quick Start"
description: "Get webhook simulation working in 1 hour."
---

**Estimated Time**: 1 hour
**Difficulty**: Medium
**Prerequisites**: Basic TypeScript, Hono routes

---

## Implementation Checklist

### Phase 1: Core Interface (15 min)

- [ ] Create payment gateway interface
- [ ] Create mock gateway implementation
- [ ] Create gateway factory

### Phase 2: Mock Routes (20 min)

- [ ] Create mock checkout page
- [ ] Create payment simulation endpoint
- [ ] Create manual webhook trigger (dev utility)

### Phase 3: Webhook Handler (15 min)

- [ ] Create unified webhook endpoint
- [ ] Integrate with payment verification service

### Phase 4: Testing (10 min)

- [ ] Test payment flow end-to-end
- [ ] Verify submission auto-trigger

---

## Step-by-Step Implementation

### Step 1: Create Gateway Interface

**File**: `packages/payments/gateways/interface.ts`

```typescript
export interface PaymentGateway {
  readonly providerName: string;
  readonly displayName: string;
  readonly isTestMode: boolean;

  createPaymentSession(params: CreatePaymentSessionParams): Promise<PaymentSession>;
  verifyWebhookSignature(params: VerifyWebhookParams): Promise<boolean>;
  parseWebhookEvent(params: ParseWebhookParams): Promise<WebhookEvent>;
  getPayment(paymentId: string): Promise<GatewayPayment | null>;
  refundPayment(params: RefundPaymentParams): Promise<GatewayRefund>;
}

export interface CreatePaymentSessionParams {
  organizationId: string;
  userId: string;
  amount: number;
  currency: string;
  description: string;
  successUrl: string;
  cancelUrl: string;
  metadata: Record<string, unknown>;
}

export interface PaymentSession {
  id: string;
  url: string;
  expiresAt: Date;
}

export interface WebhookEvent {
  id: string;
  type: "payment.succeeded" | "payment.failed" | "payment.refunded";
  data: {
    paymentId: string;
    sessionId: string;
    amount: number;
    currency: string;
    status: "succeeded" | "failed" | "refunded";
    metadata: Record<string, unknown>;
    failureReason?: string;
  };
  createdAt: Date;
}

// ... other types from the doc
```

### Step 2: Create Mock Gateway

**File**: `packages/payments/gateways/mock-gateway.ts`

```typescript
import crypto from "node:crypto";
import type { PaymentGateway, CreatePaymentSessionParams, PaymentSession } from "./interface";

export class MockPaymentGateway implements PaymentGateway {
  readonly providerName = "mock";
  readonly displayName = "Mock Gateway (Testing)";
  readonly isTestMode = true;

  private sessions = new Map<string, any>();
  private payments = new Map<string, any>();

  constructor(private config: { baseUrl: string; webhookSecret?: string }) {}

  async createPaymentSession(params: CreatePaymentSessionParams): Promise<PaymentSession> {
    const sessionId = `sess_mock_${crypto.randomBytes(16).toString("hex")}`;

    const session = {
      id: sessionId,
      url: `${this.config.baseUrl}/api/payments/mock/checkout/${sessionId}`,
      expiresAt: new Date(Date.now() + 30 * 60 * 1000),
    };

    this.sessions.set(sessionId, { ...session, params });

    return session;
  }

  async verifyWebhookSignature(): Promise<boolean> {
    // In mock mode, always accept
    return true;
  }

  async parseWebhookEvent(params: any): Promise<any> {
    return JSON.parse(params.rawBody.toString());
  }

  async getPayment(paymentId: string): Promise<any> {
    return this.payments.get(paymentId) ?? null;
  }

  async refundPayment(): Promise<any> {
    throw new Error("Not implemented");
  }

  // Mock-specific methods
  async simulatePaymentSuccess(sessionId: string) {
    const session = this.sessions.get(sessionId);
    if (!session) throw new Error("Session not found");

    const paymentId = `pi_mock_${crypto.randomBytes(16).toString("hex")}`;
    const payment = {
      id: paymentId,
      amount: session.params.amount,
      currency: session.params.currency,
      status: "succeeded",
      metadata: session.params.metadata,
      createdAt: new Date(),
    };

    this.payments.set(paymentId, payment);
    return payment;
  }

  getSession(sessionId: string) {
    return this.sessions.get(sessionId);
  }
}
```

### Step 3: Create Gateway Factory

**File**: `packages/payments/backend.ts`

```typescript
import type { PaymentGateway } from "./gateways/interface";
import { MockPaymentGateway } from "./gateways/mock-gateway";

let cachedGateway: PaymentGateway | null = null;

export async function getPaymentGateway(): Promise<PaymentGateway> {
  if (cachedGateway) return cachedGateway;

  const provider = process.env.PAYMENT_GATEWAY ?? "mock";

  switch (provider) {
    case "mock":
      cachedGateway = new MockPaymentGateway({
        baseUrl: process.env.BASE_URL ?? "http://localhost:3000",
        webhookSecret: process.env.MOCK_WEBHOOK_SECRET,
      });
      break;

    case "stripe":
      throw new Error("Stripe gateway not yet implemented");

    default:
      throw new Error(`Unknown payment gateway: ${provider}`);
  }

  return cachedGateway;
}

export function getMockGateway(): MockPaymentGateway {
  if (!(cachedGateway instanceof MockPaymentGateway)) {
    throw new Error("Mock gateway not active");
  }
  return cachedGateway;
}
```

### Step 4: Update Payment Service

**File**: `packages/api-services/src/domains/compliance/payments.service.ts`

Update the `initiatePayment` function to use the gateway:

```typescript
export async function initiatePayment(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  input: { paymentRequestId: string; successUrl: string; cancelUrl: string }
): Promise<{ checkoutUrl: string; sessionId: string }> {
  const payment = await loadPayment(deps, input.paymentRequestId);

  // Get gateway
  const { getPaymentGateway } = await import("@repo/payments/backend");
  const gateway = await getPaymentGateway();

  // Create session
  const session = await gateway.createPaymentSession({
    organizationId: payment.organizationId,
    userId: ctx.userId!,
    amount: payment.totalAmount,
    currency: payment.currency,
    description: `Payment for ${payment.organizationId}`,
    successUrl: input.successUrl,
    cancelUrl: input.cancelUrl,
    metadata: { paymentRequestId: payment.id },
  });

  // Update payment request
  await deps.db.update(paymentRequests)
    .set({
      status: "pending_gateway",
      externalSessionId: session.id,
      paymentProvider: gateway.providerName,
      updatedAt: new Date(),
    })
    .where(eq(paymentRequests.id, payment.id));

  return {
    checkoutUrl: session.url,
    sessionId: session.id,
  };
}
```

### Step 5: Create Mock Routes

**File**: `packages/backend/src/modules/payments/mock.ts`

```typescript
import { Hono } from "hono";
import { getMockGateway } from "@repo/payments/backend";

const app = new Hono();

// Mock checkout page
app.get("/checkout/:sessionId", async (c) => {
  const sessionId = c.req.param("sessionId");
  const gateway = getMockGateway();
  const session = gateway.getSession(sessionId);

  if (!session) {
    return c.html("<h1>Session not found</h1>", 404);
  }

  return c.html(`
    <!DOCTYPE html>
    <html>
    <head>
      <title>Mock Payment</title>
      <style>
        body { font-family: sans-serif; max-width: 600px; margin: 50px auto; padding: 20px; }
        .card { background: white; padding: 30px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
        button { padding: 12px 24px; margin: 10px 5px; font-size: 16px; border: none; border-radius: 4px; cursor: pointer; }
        .success { background: #28a745; color: white; }
        .danger { background: #dc3545; color: white; }
      </style>
    </head>
    <body>
      <div class="card">
        <h1>Mock Payment Gateway</h1>
        <p><strong>Amount:</strong> ${session.params.amount / 100} ${session.params.currency}</p>
        <p><strong>Description:</strong> ${session.params.description}</p>

        <button class="success" onclick="simulate('success')">✓ Simulate Success</button>
        <button class="danger" onclick="simulate('failure')">✗ Simulate Failure</button>
      </div>

      <script>
        async function simulate(outcome) {
          const response = await fetch('/api/payments/mock/simulate', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ sessionId: '${sessionId}', outcome })
          });

          if (response.ok) {
            const result = await response.json();
            window.location.href = '${session.params.successUrl}?payment_id=' + result.paymentId;
          }
        }
      </script>
    </body>
    </html>
  `);
});

// Payment simulation endpoint
app.post("/simulate", async (c) => {
  const { sessionId, outcome } = await c.req.json();
  const gateway = getMockGateway();

  // Simulate payment
  const payment = outcome === "success"
    ? await gateway.simulatePaymentSuccess(sessionId)
    : await gateway.simulatePaymentFailure(sessionId, "insufficient_funds");

  // Send webhook
  const webhookEvent = {
    id: `evt_mock_${crypto.randomUUID()}`,
    type: outcome === "success" ? "payment.succeeded" : "payment.failed",
    data: {
      paymentId: payment.id,
      sessionId,
      amount: payment.amount,
      currency: payment.currency,
      status: payment.status,
      metadata: payment.metadata,
    },
    createdAt: new Date(),
  };

  const webhookUrl = `${new URL(c.req.url).origin}/api/webhooks/payments`;

  await fetch(webhookUrl, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(webhookEvent),
  });

  return c.json({ paymentId: payment.id, webhookSent: true });
});

export default app;
```

**Register routes**: `packages/backend/src/modules/payments/index.ts`

```typescript
import { Hono } from "hono";
import mockRoutes from "./mock";

const app = new Hono();

app.route("/mock", mockRoutes);

export default app;
```

### Step 6: Create Webhook Handler

**File**: `packages/backend/src/modules/webhooks/payments.ts`

```typescript
import { Hono } from "hono";
import { getPaymentGateway } from "@repo/payments/backend";
import { handlePaymentVerification } from "@repo/api-services/domains/compliance/payments.service";
import { db } from "@repo/database";
import { eq } from "drizzle-orm";
import { paymentRequests } from "@repo/database/schema";

const app = new Hono();

app.post("/", async (c) => {
  const gateway = await getPaymentGateway();

  // Verify signature
  const signature = c.req.header("x-webhook-signature") ?? "";
  const rawBody = await c.req.raw.text();

  const isValid = await gateway.verifyWebhookSignature({ signature, rawBody });
  if (!isValid) {
    return c.json({ error: "Invalid signature" }, 401);
  }

  // Parse event
  const event = await gateway.parseWebhookEvent({ rawBody, headers: {} });

  // Handle event
  if (event.type === "payment.succeeded") {
    const paymentRequestId = event.data.metadata.paymentRequestId as string;

    const payment = await db.query.paymentRequests.findFirst({
      where: eq(paymentRequests.id, paymentRequestId),
    });

    if (payment) {
      // Update to paid_gateway_verified
      await db.update(paymentRequests)
        .set({
          status: "paid_gateway_verified",
          paidAt: new Date(),
          externalPaymentId: event.data.paymentId,
        })
        .where(eq(paymentRequests.id, paymentRequestId));

      // Auto-verify and trigger submission
      await handlePaymentVerification(
        { orgId: payment.organizationId, userId: "system" },
        { db, now: () => new Date() },
        {
          paymentRequestId,
          verifiedByAgentId: null,
          verificationNotes: "Auto-verified via webhook",
          verificationEvidence: {
            transactionReference: event.data.paymentId,
          },
        }
      );
    }
  }

  return c.json({ received: true });
});

export default app;
```

**Register webhook**: `packages/backend/src/index.ts`

```typescript
import { Hono } from "hono";
import paymentsWebhook from "./modules/webhooks/payments";

const app = new Hono();

app.route("/api/webhooks/payments", paymentsWebhook);

export default app;
```

### Step 7: Update Environment

**File**: `.env.local`

```bash
PAYMENT_GATEWAY=mock
BASE_URL=http://localhost:3000
MOCK_WEBHOOK_SECRET=dev_secret_123
```

---

## Testing the Flow

### Test 1: End-to-End Payment

```bash
# 1. Start dev server
pnpm dev

# 2. Create a filing/service request (via UI or API)

# 3. Click "Pay Now" button

# 4. Redirected to: http://localhost:3000/api/payments/mock/checkout/sess_xxx

# 5. Click "Simulate Success"

# 6. Verify:
#    - Payment status → paid_platform_verified
#    - Submission job created
#    - Tenant redirected to success page
```

### Test 2: Manual Webhook Trigger

```typescript
// Via backoffice dev panel or API
POST /api/payments/mock/webhook-manual
{
  "paymentRequestId": "pay_123",
  "outcome": "success"
}
```

---

## Troubleshooting

### Issue: Webhook not received

**Solution**: Check that webhook URL is correct:
```typescript
const webhookUrl = `${new URL(c.req.url).origin}/api/webhooks/payments`;
console.log("Webhook URL:", webhookUrl);
```

### Issue: Payment not verified

**Solution**: Check payment status transitions:
```sql
SELECT id, status, created_at, updated_at
FROM payment_requests
WHERE id = 'pay_123'
ORDER BY updated_at DESC;
```

### Issue: Submission not auto-triggered

**Solution**: Check submission readiness:
```typescript
const readiness = await computeSubmissionReadiness(ctx, deps, {
  sourceType: "service_request",
  sourceId: "sr_123"
});
console.log("Readiness:", readiness);
```

---

## Migration to Stripe (Future)

### Step 1: Install Stripe SDK

```bash
pnpm add stripe
```

### Step 2: Create Stripe Gateway

```typescript
// packages/payments/gateways/stripe-gateway.ts

import Stripe from "stripe";
import type { PaymentGateway } from "./interface";

export class StripeGateway implements PaymentGateway {
  readonly providerName = "stripe";
  readonly displayName = "Stripe";
  readonly isTestMode = false;

  private stripe: Stripe;

  constructor(config: { apiKey: string; webhookSecret: string }) {
    this.stripe = new Stripe(config.apiKey, { apiVersion: "2023-10-16" });
  }

  async createPaymentSession(params) {
    const session = await this.stripe.checkout.sessions.create({
      mode: "payment",
      payment_method_types: ["card"],
      line_items: [{
        price_data: {
          currency: params.currency.toLowerCase(),
          product_data: { name: params.description },
          unit_amount: params.amount,
        },
        quantity: 1,
      }],
      metadata: params.metadata,
      success_url: params.successUrl,
      cancel_url: params.cancelUrl,
    });

    return {
      id: session.id,
      url: session.url!,
      expiresAt: new Date(session.expires_at * 1000),
    };
  }

  // ... implement other methods
}
```

### Step 3: Update Gateway Factory

```typescript
// packages/payments/backend.ts

case "stripe": {
  const { StripeGateway } = await import("./gateways/stripe-gateway");
  cachedGateway = new StripeGateway({
    apiKey: process.env.STRIPE_SECRET_KEY!,
    webhookSecret: process.env.STRIPE_WEBHOOK_SECRET!,
  });
  break;
}
```

### Step 4: Update Environment

```bash
# Production
PAYMENT_GATEWAY=stripe
STRIPE_SECRET_KEY=sk_live_xxxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxxx
```

**Zero code changes needed!** 🎉

---

## Summary

### What You Built

1. ✅ Payment gateway abstraction layer
2. ✅ Mock gateway with checkout UI
3. ✅ Webhook simulation
4. ✅ Auto-verification on payment success
5. ✅ Auto-submission trigger
6. ✅ Easy migration path to real gateway

### Time to Production

- **Mock implementation**: 1 hour ✅
- **Testing & refinement**: 2 hours
- **Stripe migration**: 2 hours
- **Total**: ~5 hours

### Next Steps

1. [ ] Implement mock gateway (this guide)
2. [ ] Test payment flow end-to-end
3. [ ] Add Stripe gateway when ready
4. [ ] Switch env var to `PAYMENT_GATEWAY=stripe`
5. [ ] Deploy to production ✅

---

**Questions?** Check the full documentation: [payment-gateway-simulation.md](/ARCHITECTURE/payment-gateway-simulation)
