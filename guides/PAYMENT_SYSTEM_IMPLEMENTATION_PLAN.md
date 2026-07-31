---
title: "Payment System - Detailed Implementation Plan"
description: "Comprehensive sprint plan for implementing the complete payment system with webhooks, state machine, and regulator payouts."
---

**Created**: 2026-02-10
**Owner**: Engineering Team
**Timeline**: 4 weeks (4 sprints)
**Estimated Effort**: 80 hours

---

## Executive Summary

### Goals
1. ✅ Enable payment testing without real gateway (mock system)
2. ✅ Enforce payment status transitions (state machine)
3. ✅ Track regulator payouts for reconciliation
4. ✅ Add configurable handling fee rules
5. ✅ Build payment analytics dashboard

### Success Criteria
- [ ] End-to-end payment flow works with mock gateway
- [ ] Invalid payment transitions are blocked
- [ ] Regulator payouts can be created and tracked
- [ ] Handling fees are configurable per regulator
- [ ] Analytics dashboard shows revenue metrics

### Risk Mitigation
- **Low Risk**: Building on existing solid architecture
- **Mock Gateway First**: No dependency on external approval
- **Incremental Rollout**: Each sprint delivers value independently
- **Backward Compatible**: No breaking changes to existing flows

---

## Sprint 0: Preparation (Pre-work)

**Duration**: 2 days (before Sprint 1)
**Owner**: Tech Lead + 1 Developer

### Tasks

#### Day 1: Documentation Review & Architecture Validation

**Task 0.1**: Review Architecture Documentation (2 hours)
- [ ] Read [regulator-payment-system.md](/ARCHITECTURE/regulator-payment-system)
- [ ] Review current payment flow in codebase
- [ ] Identify integration points
- [ ] Document current state vs. desired state

**Deliverable**: Architecture review notes document

**Acceptance Criteria**:
- Team understands current payment flow
- Integration points identified
- No blockers to starting implementation

---

**Task 0.2**: Set Up Project Structure (1 hour)
- [ ] Create `packages/payments/` workspace
- [ ] Set up TypeScript config
- [ ] Add necessary dependencies
- [ ] Create folder structure

```bash
packages/payments/
├── gateways/
│   ├── interface.ts
│   ├── mock-gateway.ts
│   └── stripe-gateway.ts (placeholder)
├── calculators/
│   ├── index.ts
│   ├── types.ts
│   ├── config.ts
│   └── registry.ts
├── backend.ts
├── package.json
└── tsconfig.json
```

**Commands**:
```bash
cd packages
mkdir -p payments/gateways payments/calculators
cd payments
pnpm init
pnpm add -D typescript @types/node
pnpm add drizzle-orm zod
```

**Deliverable**: Empty project structure ready for code

**Acceptance Criteria**:
- Packages install successfully
- TypeScript compiles
- No import errors

---

**Task 0.3**: Database Schema Review (1 hour)
- [ ] Review `payment_requests` table schema
- [ ] Review `regulator_fees` table schema
- [ ] Verify all needed columns exist
- [ ] Identify any missing indexes

**Verification Query**:
```sql
-- Check payment_requests schema
\d payment_requests

-- Check indexes
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'payment_requests';
```

**Deliverable**: Schema validation checklist

**Acceptance Criteria**:
- All required columns exist
- Indexes are in place
- No schema migrations needed for Sprint 1

---

**Task 0.4**: Set Up Testing Infrastructure (2 hours)
- [ ] Configure Vitest for `packages/payments`
- [ ] Set up test database
- [ ] Create test fixtures
- [ ] Write first smoke test

**File**: `packages/payments/__tests__/setup.ts`
```typescript
import { beforeAll, afterAll, afterEach } from "vitest";
import { db } from "@repo/database";

beforeAll(async () => {
  // Set up test database
  await db.execute(sql`CREATE SCHEMA IF NOT EXISTS test`);
});

afterEach(async () => {
  // Clean up test data
  await db.execute(sql`TRUNCATE TABLE payment_requests CASCADE`);
});

afterAll(async () => {
  // Tear down
  await db.execute(sql`DROP SCHEMA test CASCADE`);
});
```

**Deliverable**: Working test suite

**Acceptance Criteria**:
- `pnpm test` runs successfully
- Test database is isolated
- Fixtures load correctly

---

**Task 0.5**: Create Implementation Tracking Board (1 hour)
- [ ] Set up project board (GitHub Projects / Linear / Jira)
- [ ] Create epics for each sprint
- [ ] Break down tasks into tickets
- [ ] Assign story points

**Deliverable**: Project board with all tasks

**Acceptance Criteria**:
- All sprints have epics
- All tasks have tickets
- Dependencies mapped
- Team has access

---

## Sprint 1: Mock Payment Gateway

**Duration**: 1 week (5 days)
**Owner**: 2 Developers
**Effort**: 20 hours
**Priority**: P0 (Critical - Unblocks development)

### Sprint Goal
Enable payment testing without a real payment gateway using webhook simulation.

### Sprint Outcomes
✅ Mock gateway implemented
✅ Checkout UI working
✅ Webhook simulation functional
✅ Auto-verification triggering submission jobs

---

### Day 1: Gateway Interface & Mock Implementation

**Task 1.1**: Create Payment Gateway Interface (2 hours)
**Owner**: Dev 1
**Complexity**: Medium

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
  type: "payment.succeeded" | "payment.failed" | "payment.refunded" | "payment.expired";
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

// ... other interfaces
```

**Subtasks**:
- [ ] Define all interface types
- [ ] Add JSDoc documentation
- [ ] Export all types
- [ ] Verify TypeScript compilation

**Acceptance Criteria**:
- ✅ All interfaces defined
- ✅ TypeScript compiles with no errors
- ✅ JSDoc comments added
- ✅ Exports are correct

**Testing**: N/A (interfaces only)

---

**Task 1.2**: Implement Mock Gateway (3 hours)
**Owner**: Dev 1
**Complexity**: High

**File**: `packages/payments/gateways/mock-gateway.ts`

**Implementation Checklist**:
- [ ] Implement `PaymentGateway` interface
- [ ] Add in-memory session storage
- [ ] Add in-memory payment storage
- [ ] Implement `createPaymentSession`
- [ ] Implement `simulatePaymentSuccess` (mock-specific)
- [ ] Implement `simulatePaymentFailure` (mock-specific)
- [ ] Implement `verifyWebhookSignature` (always true for mock)
- [ ] Implement `parseWebhookEvent`
- [ ] Implement `getPayment`
- [ ] Add logging for debugging

**Key Methods**:
```typescript
export class MockPaymentGateway implements PaymentGateway {
  private sessions = new Map<string, SessionData>();
  private payments = new Map<string, GatewayPayment>();

  async createPaymentSession(params: CreatePaymentSessionParams): Promise<PaymentSession> {
    const sessionId = `sess_mock_${crypto.randomBytes(16).toString("hex")}`;

    const session = {
      id: sessionId,
      url: `${this.config.baseUrl}/api/payments/mock/checkout/${sessionId}`,
      expiresAt: new Date(Date.now() + 30 * 60 * 1000),
    };

    this.sessions.set(sessionId, { ...session, params });

    console.log(`[MockGateway] Session created: ${sessionId}`);

    return session;
  }

  async simulatePaymentSuccess(sessionId: string): Promise<GatewayPayment> {
    const session = this.sessions.get(sessionId);
    if (!session) throw new Error("Session not found");

    const paymentId = `pi_mock_${crypto.randomBytes(16).toString("hex")}`;
    const payment: GatewayPayment = {
      id: paymentId,
      amount: session.params.amount,
      currency: session.params.currency,
      status: "succeeded",
      metadata: session.params.metadata,
      createdAt: new Date(),
    };

    this.payments.set(paymentId, payment);

    console.log(`[MockGateway] Payment succeeded: ${paymentId}`);

    return payment;
  }
}
```

**Acceptance Criteria**:
- ✅ All interface methods implemented
- ✅ Session IDs are unique and realistic
- ✅ Payment IDs are unique and realistic
- ✅ In-memory storage works correctly
- ✅ Logging is clear and helpful

**Testing**:
```typescript
// packages/payments/__tests__/mock-gateway.test.ts
import { describe, it, expect } from "vitest";
import { MockPaymentGateway } from "../gateways/mock-gateway";

describe("MockPaymentGateway", () => {
  const gateway = new MockPaymentGateway({
    baseUrl: "http://localhost:3000",
  });

  it("should create payment session", async () => {
    const session = await gateway.createPaymentSession({
      organizationId: "org_123",
      userId: "user_456",
      amount: 52500,
      currency: "ZMW",
      description: "Test payment",
      successUrl: "/success",
      cancelUrl: "/cancel",
      metadata: { test: true },
    });

    expect(session.id).toMatch(/^sess_mock_/);
    expect(session.url).toContain("/api/payments/mock/checkout/");
    expect(session.expiresAt).toBeInstanceOf(Date);
  });

  it("should simulate payment success", async () => {
    const session = await gateway.createPaymentSession({...});
    const payment = await gateway.simulatePaymentSuccess(session.id);

    expect(payment.id).toMatch(/^pi_mock_/);
    expect(payment.status).toBe("succeeded");
    expect(payment.amount).toBe(52500);
  });

  it("should retrieve payment by ID", async () => {
    const session = await gateway.createPaymentSession({...});
    const payment = await gateway.simulatePaymentSuccess(session.id);

    const retrieved = await gateway.getPayment(payment.id);

    expect(retrieved).toEqual(payment);
  });
});
```

**Time Estimate**: 3 hours
**Dependencies**: Task 1.1 complete

---

**Task 1.3**: Create Gateway Factory (1 hour)
**Owner**: Dev 1
**Complexity**: Low

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

  console.log(`[Payments] Using gateway: ${cachedGateway.displayName}`);

  return cachedGateway;
}

export function getMockGateway(): MockPaymentGateway {
  if (!(cachedGateway instanceof MockPaymentGateway)) {
    throw new Error("Mock gateway not active");
  }
  return cachedGateway;
}

export function resetGatewayCache(): void {
  cachedGateway = null;
}
```

**Acceptance Criteria**:
- ✅ Gateway factory returns mock gateway
- ✅ Singleton pattern works
- ✅ Environment variable controls gateway selection
- ✅ Helper functions work correctly

**Testing**:
```typescript
describe("Gateway Factory", () => {
  beforeEach(() => {
    resetGatewayCache();
  });

  it("should return mock gateway by default", async () => {
    const gateway = await getPaymentGateway();
    expect(gateway.providerName).toBe("mock");
  });

  it("should cache gateway instance", async () => {
    const gateway1 = await getPaymentGateway();
    const gateway2 = await getPaymentGateway();
    expect(gateway1).toBe(gateway2);
  });

  it("should return mock gateway helper", async () => {
    await getPaymentGateway();
    const mockGateway = getMockGateway();
    expect(mockGateway).toBeInstanceOf(MockPaymentGateway);
  });
});
```

**Time Estimate**: 1 hour
**Dependencies**: Task 1.2 complete

---

### Day 2: Mock Checkout UI

**Task 1.4**: Create Mock Checkout Route (2 hours)
**Owner**: Dev 2
**Complexity**: Medium

**File**: `packages/backend/src/modules/payments/mock.ts`

**Implementation Checklist**:
- [ ] Create Hono app
- [ ] Add GET `/checkout/:sessionId` route
- [ ] Generate HTML checkout page
- [ ] Add CSS styling
- [ ] Add JavaScript for simulation
- [ ] Handle session not found

**HTML Template**:
```typescript
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
      <title>Mock Payment Gateway</title>
      <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
          font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
          background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
          min-height: 100vh;
          display: flex;
          align-items: center;
          justify-content: center;
          padding: 20px;
        }
        .card {
          background: white;
          border-radius: 16px;
          box-shadow: 0 20px 60px rgba(0,0,0,0.3);
          max-width: 500px;
          width: 100%;
          padding: 40px;
        }
        .banner {
          background: #fff3cd;
          border: 2px solid #ffc107;
          color: #856404;
          padding: 16px;
          border-radius: 8px;
          margin-bottom: 24px;
          display: flex;
          align-items: center;
          gap: 12px;
        }
        h1 {
          font-size: 28px;
          margin-bottom: 8px;
          color: #1a202c;
        }
        .amount {
          font-size: 48px;
          font-weight: 700;
          color: #667eea;
          margin: 24px 0;
        }
        .details {
          background: #f7fafc;
          padding: 20px;
          border-radius: 8px;
          margin: 24px 0;
        }
        .details p {
          margin: 8px 0;
          color: #4a5568;
        }
        .details strong {
          color: #2d3748;
        }
        button {
          width: 100%;
          padding: 16px;
          font-size: 16px;
          font-weight: 600;
          border: none;
          border-radius: 8px;
          cursor: pointer;
          transition: all 0.2s;
          margin: 8px 0;
        }
        .btn-success {
          background: #48bb78;
          color: white;
        }
        .btn-success:hover {
          background: #38a169;
          transform: translateY(-2px);
          box-shadow: 0 4px 12px rgba(72, 187, 120, 0.4);
        }
        .btn-danger {
          background: #f56565;
          color: white;
        }
        .btn-danger:hover {
          background: #e53e3e;
          transform: translateY(-2px);
          box-shadow: 0 4px 12px rgba(245, 101, 101, 0.4);
        }
        .btn-cancel {
          background: #e2e8f0;
          color: #4a5568;
        }
        .btn-cancel:hover {
          background: #cbd5e0;
        }
        .loading {
          display: none;
          text-align: center;
          margin: 20px 0;
        }
        .spinner {
          border: 3px solid #f3f3f3;
          border-top: 3px solid #667eea;
          border-radius: 50%;
          width: 40px;
          height: 40px;
          animation: spin 1s linear infinite;
          margin: 0 auto;
        }
        @keyframes spin {
          0% { transform: rotate(0deg); }
          100% { transform: rotate(360deg); }
        }
      </style>
    </head>
    <body>
      <div class="card">
        <div class="banner">
          <span style="font-size: 24px;">⚠️</span>
          <div>
            <strong>Mock Payment Gateway</strong><br>
            <small>Development/Testing Mode - No real charges</small>
          </div>
        </div>

        <h1>Complete Payment</h1>

        <div class="amount">
          ${formatAmount(session.params.amount, session.params.currency)}
        </div>

        <div class="details">
          <p><strong>Description:</strong> ${session.params.description}</p>
          <p><strong>Organization:</strong> ${session.params.organizationId}</p>
          <p><strong>Session ID:</strong> <code>${sessionId}</code></p>
        </div>

        <p style="margin: 20px 0; color: #4a5568;">
          Simulate payment outcome:
        </p>

        <button class="btn-success" onclick="simulatePayment('success')">
          ✓ Simulate Success
        </button>

        <button class="btn-danger" onclick="simulatePayment('failure')">
          ✗ Simulate Failure
        </button>

        <button class="btn-cancel" onclick="window.location.href='${session.params.cancelUrl}'">
          Cancel Payment
        </button>

        <div class="loading" id="loading">
          <div class="spinner"></div>
          <p style="margin-top: 12px; color: #4a5568;">Processing...</p>
        </div>
      </div>

      <script>
        async function simulatePayment(outcome) {
          const loadingEl = document.getElementById('loading');
          loadingEl.style.display = 'block';

          try {
            const response = await fetch('/api/payments/mock/simulate', {
              method: 'POST',
              headers: { 'Content-Type': 'application/json' },
              body: JSON.stringify({
                sessionId: '${sessionId}',
                outcome: outcome,
                failureReason: outcome === 'failure' ? 'insufficient_funds' : undefined
              })
            });

            const result = await response.json();

            if (response.ok) {
              window.location.href = '${session.params.successUrl}?payment_id=' + result.paymentId;
            } else {
              alert('Simulation failed: ' + result.error);
              loadingEl.style.display = 'none';
            }
          } catch (error) {
            alert('Error: ' + error.message);
            loadingEl.style.display = 'none';
          }
        }
      </script>
    </body>
    </html>
  `);
});

function formatAmount(amountInMinor: number, currency: string): string {
  const major = amountInMinor / 100;
  return new Intl.NumberFormat("en-ZM", {
    style: "currency",
    currency,
  }).format(major);
}
```

**Acceptance Criteria**:
- ✅ Checkout page renders correctly
- ✅ Session details displayed
- ✅ Styling looks professional
- ✅ Responsive design works
- ✅ Buttons are clickable

**Manual Testing**:
1. Start dev server
2. Visit `/api/payments/mock/checkout/test_session`
3. Verify page renders
4. Check styling
5. Test responsiveness

**Time Estimate**: 2 hours
**Dependencies**: Task 1.2 complete

---

**Task 1.5**: Create Payment Simulation Endpoint (2 hours)
**Owner**: Dev 2
**Complexity**: Medium

**File**: `packages/backend/src/modules/payments/mock.ts` (continued)

```typescript
app.post("/simulate", async (c) => {
  const { sessionId, outcome, failureReason } = await c.req.json();

  const gateway = getMockGateway();

  // 1. Simulate payment
  const payment =
    outcome === "success"
      ? await gateway.simulatePaymentSuccess(sessionId)
      : await gateway.simulatePaymentFailure(sessionId, failureReason ?? "generic_decline");

  // 2. Construct webhook event
  const webhookEvent: WebhookEvent = {
    id: `evt_mock_${crypto.randomUUID()}`,
    type: outcome === "success" ? "payment.succeeded" : "payment.failed",
    data: {
      paymentId: payment.id,
      sessionId,
      amount: payment.amount,
      currency: payment.currency,
      status: payment.status,
      metadata: payment.metadata,
      failureReason: outcome === "failure" ? failureReason : undefined,
    },
    createdAt: new Date(),
  };

  // 3. Send webhook to our own webhook handler
  const webhookUrl = `${new URL(c.req.url).origin}/api/webhooks/payments`;

  try {
    const webhookResponse = await fetch(webhookUrl, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "X-Mock-Webhook": "true",
      },
      body: JSON.stringify(webhookEvent),
    });

    const webhookOk = webhookResponse.ok;

    console.log(`[MockGateway] Webhook sent: ${webhookOk ? "✓" : "✗"}`);

    return c.json({
      paymentId: payment.id,
      webhookSent: webhookOk,
    });
  } catch (error) {
    console.error("[MockGateway] Webhook failed:", error);
    return c.json(
      {
        paymentId: payment.id,
        webhookSent: false,
        error: error.message,
      },
      500
    );
  }
});
```

**Acceptance Criteria**:
- ✅ Endpoint validates input
- ✅ Payment simulation works for success/failure
- ✅ Webhook event is constructed correctly
- ✅ Webhook is sent to correct URL
- ✅ Errors are handled gracefully

**Testing**:
```typescript
describe("Mock Simulation Endpoint", () => {
  it("should simulate successful payment and send webhook", async () => {
    const gateway = getMockGateway();
    const session = await gateway.createPaymentSession({...});

    const response = await app.request("/api/payments/mock/simulate", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        sessionId: session.id,
        outcome: "success",
      }),
    });

    expect(response.status).toBe(200);
    const result = await response.json();
    expect(result.paymentId).toMatch(/^pi_mock_/);
    expect(result.webhookSent).toBe(true);
  });
});
```

**Time Estimate**: 2 hours
**Dependencies**: Task 1.4 complete

---

### Day 3: Webhook Handler & Integration

**Task 1.6**: Create Webhook Handler (3 hours)
**Owner**: Dev 1
**Complexity**: High

**File**: `packages/backend/src/modules/webhooks/payments.ts`

**Implementation Checklist**:
- [ ] Create Hono app
- [ ] Add POST `/` route
- [ ] Implement signature verification
- [ ] Implement event parsing
- [ ] Handle `payment.succeeded` event
- [ ] Handle `payment.failed` event
- [ ] Handle `payment.refunded` event
- [ ] Add error handling
- [ ] Add logging

```typescript
import { Hono } from "hono";
import { getPaymentGateway } from "@repo/payments/backend";
import { db } from "@repo/database";
import { eq, and } from "drizzle-orm";
import { paymentRequests, tickets, timelineEvents } from "@repo/database/schema";

const app = new Hono();

app.post("/", async (c) => {
  try {
    // 1. Get gateway
    const gateway = await getPaymentGateway();

    // 2. Verify signature
    const signature = c.req.header("x-webhook-signature") ?? "";
    const rawBody = await c.req.raw.text();

    const isValid = await gateway.verifyWebhookSignature({
      signature,
      rawBody,
    });

    // Allow mock webhooks without signature in development
    if (!isValid && !c.req.header("x-mock-webhook")) {
      console.error("[Webhook] Invalid signature");
      return c.json({ error: "Invalid signature" }, 401);
    }

    // 3. Parse event
    const event = await gateway.parseWebhookEvent({
      rawBody,
      headers: Object.fromEntries(c.req.raw.headers.entries()),
    });

    console.log(`[Webhook] Received: ${event.type} (${event.id})`);

    // 4. Handle based on event type
    switch (event.type) {
      case "payment.succeeded":
        await handlePaymentSucceeded(event);
        break;

      case "payment.failed":
        await handlePaymentFailed(event);
        break;

      case "payment.refunded":
        await handlePaymentRefunded(event);
        break;

      default:
        console.log(`[Webhook] Unhandled event: ${event.type}`);
    }

    return c.json({ received: true });
  } catch (error) {
    console.error("[Webhook] Error:", error);
    return c.json({ error: error.message }, 500);
  }
});

async function handlePaymentSucceeded(event: WebhookEvent): Promise<void> {
  const { paymentId, metadata } = event.data;
  const paymentRequestId = metadata.paymentRequestId as string;

  if (!paymentRequestId) {
    console.error("[Webhook] No paymentRequestId in metadata");
    return;
  }

  // Load payment request
  const payment = await db.query.paymentRequests.findFirst({
    where: eq(paymentRequests.id, paymentRequestId),
  });

  if (!payment) {
    console.error(`[Webhook] Payment not found: ${paymentRequestId}`);
    return;
  }

  // Update to paid_gateway_verified
  await db
    .update(paymentRequests)
    .set({
      status: "paid_gateway_verified",
      paidAt: new Date(),
      externalPaymentId: paymentId,
      updatedAt: new Date(),
    })
    .where(eq(paymentRequests.id, paymentRequestId));

  console.log(`[Webhook] Payment verified: ${paymentRequestId}`);

  // Close payment ticket
  await db
    .update(tickets)
    .set({
      status: "resolved",
      resolvedAt: new Date(),
      updatedAt: new Date(),
    })
    .where(
      and(
        eq(tickets.paymentRequestId, paymentRequestId),
        eq(tickets.type, "payment_request")
      )
    );

  // Create timeline event
  await db.insert(timelineEvents).values({
    organizationId: payment.organizationId,
    filingId: payment.filingId,
    serviceRequestId: payment.serviceRequestId,
    eventType: "payment",
    title: "Payment received",
    description: "Payment has been verified via payment gateway",
    occurredAt: new Date(),
  });

  // Auto-verify and trigger submission (imported from payments.service.ts)
  try {
    const { handlePaymentVerification } = await import(
      "@repo/api-services/domains/compliance/payments.service"
    );

    await handlePaymentVerification(
      { orgId: payment.organizationId, userId: "system" },
      { db, now: () => new Date(), logger: console },
      {
        paymentRequestId,
        verifiedByAgentId: null,
        verificationNotes: "Auto-verified via webhook",
        verificationEvidence: {
          transactionReference: paymentId,
          provider: event.data.metadata.provider as string,
        },
      }
    );

    console.log(`[Webhook] Auto-triggered submission: ${paymentRequestId}`);
  } catch (error) {
    console.error("[Webhook] Failed to trigger submission:", error);
    // Don't fail webhook - payment is already verified
  }
}

async function handlePaymentFailed(event: WebhookEvent): Promise<void> {
  const { metadata, failureReason } = event.data;
  const paymentRequestId = metadata.paymentRequestId as string;

  if (!paymentRequestId) return;

  await db
    .update(paymentRequests)
    .set({
      status: "required_pending",
      updatedAt: new Date(),
    })
    .where(eq(paymentRequests.id, paymentRequestId));

  console.log(`[Webhook] Payment failed: ${paymentRequestId} (${failureReason})`);

  // TODO: Notify tenant
}

async function handlePaymentRefunded(event: WebhookEvent): Promise<void> {
  const { metadata } = event.data;
  const paymentRequestId = metadata.paymentRequestId as string;

  if (!paymentRequestId) return;

  await db
    .update(paymentRequests)
    .set({
      status: "refunded",
      updatedAt: new Date(),
    })
    .where(eq(paymentRequests.id, paymentRequestId));

  console.log(`[Webhook] Payment refunded: ${paymentRequestId}`);

  // TODO: Handle refund side effects
}

export default app;
```

**Acceptance Criteria**:
- ✅ Webhook signature verified
- ✅ Events parsed correctly
- ✅ Payment status updated
- ✅ Tickets closed
- ✅ Timeline events created
- ✅ Submission auto-triggered
- ✅ Errors handled gracefully

**Testing**:
```typescript
describe("Webhook Handler", () => {
  it("should handle payment.succeeded", async () => {
    // Create payment request
    const [payment] = await db.insert(paymentRequests).values({...}).returning();

    // Send webhook
    const webhookEvent = {
      id: "evt_test_123",
      type: "payment.succeeded",
      data: {
        paymentId: "pi_test_456",
        sessionId: "sess_test_789",
        amount: 52500,
        currency: "ZMW",
        status: "succeeded",
        metadata: { paymentRequestId: payment.id },
      },
      createdAt: new Date(),
    };

    const response = await app.request("/api/webhooks/payments", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "X-Mock-Webhook": "true",
      },
      body: JSON.stringify(webhookEvent),
    });

    expect(response.status).toBe(200);

    // Verify payment updated
    const updated = await db.query.paymentRequests.findFirst({
      where: eq(paymentRequests.id, payment.id),
    });

    expect(updated.status).toBe("paid_gateway_verified");
    expect(updated.externalPaymentId).toBe("pi_test_456");
  });
});
```

**Time Estimate**: 3 hours
**Dependencies**: Task 1.3 complete

---

**Task 1.7**: Update Payment Service (2 hours)
**Owner**: Dev 1
**Complexity**: Medium

**File**: `packages/api-services/src/domains/compliance/payments.service.ts`

**Changes**:
- [ ] Import `getPaymentGateway` from `@repo/payments/backend`
- [ ] Update `initiatePayment` to use gateway
- [ ] Remove hardcoded payment URL
- [ ] Add error handling

```typescript
export async function initiatePayment(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  input: {
    paymentRequestId: string;
    successUrl: string;
    cancelUrl: string;
  }
): Promise<{ checkoutUrl: string; sessionId: string }> {
  const { orgId, userId } = ctx;
  const { paymentRequestId, successUrl, cancelUrl } = input;

  // Load payment request
  const payment = await deps.db.query.paymentRequests.findFirst({
    where: and(
      eq(paymentRequests.id, paymentRequestId),
      eq(paymentRequests.organizationId, orgId)
    ),
  });

  if (!payment) {
    throw new ServiceError("NOT_FOUND", "Payment request not found");
  }

  if (payment.status !== "required_pending") {
    throw new ServiceError(
      "INVALID_STATUS",
      `Cannot initiate payment: status is '${payment.status}'`
    );
  }

  // Get organization for description
  const org = await deps.db.query.organizations.findFirst({
    where: eq(organizations.id, orgId),
    columns: { name: true },
  });

  // Get gateway
  const { getPaymentGateway } = await import("@repo/payments/backend");
  const gateway = await getPaymentGateway();

  // Create payment session
  const session = await gateway.createPaymentSession({
    organizationId: orgId,
    userId: userId!,
    amount: payment.totalAmount,
    currency: payment.currency,
    description: `Payment for ${org?.name ?? "organization"}`,
    successUrl,
    cancelUrl,
    metadata: { paymentRequestId: payment.id },
  });

  // Update payment request
  await deps.db
    .update(paymentRequests)
    .set({
      status: "pending_gateway",
      externalSessionId: session.id,
      paymentProvider: gateway.providerName,
      updatedAt: deps.now(),
    })
    .where(eq(paymentRequests.id, payment.id));

  // Audit log
  await recordAuditLog(ctx, deps, {
    entityType: "payment_request",
    entityId: payment.id,
    action: "update",
    metadata: {
      sessionId: session.id,
      provider: gateway.providerName,
      fromStatus: "required_pending",
      toStatus: "pending_gateway",
    },
  });

  return {
    checkoutUrl: session.url,
    sessionId: session.id,
  };
}
```

**Acceptance Criteria**:
- ✅ Gateway integration works
- ✅ Payment session created
- ✅ Payment status updated
- ✅ Audit log created
- ✅ Error handling works

**Testing**:
```typescript
describe("Payment Service Integration", () => {
  it("should create payment session via gateway", async () => {
    const payment = await createPaymentRequest(...);

    const result = await initiatePayment(ctx, deps, {
      paymentRequestId: payment.id,
      successUrl: "/success",
      cancelUrl: "/cancel",
    });

    expect(result.checkoutUrl).toContain("/api/payments/mock/checkout/");
    expect(result.sessionId).toMatch(/^sess_mock_/);

    // Verify payment updated
    const updated = await db.query.paymentRequests.findFirst({
      where: eq(paymentRequests.id, payment.id),
    });

    expect(updated.status).toBe("pending_gateway");
    expect(updated.externalSessionId).toBe(result.sessionId);
  });
});
```

**Time Estimate**: 2 hours
**Dependencies**: Task 1.6 complete

---

### Day 4: End-to-End Testing

**Task 1.8**: Manual E2E Testing (3 hours)
**Owner**: Dev 1 + Dev 2
**Complexity**: Medium

**Test Scenarios**:

**Scenario 1: Successful Payment Flow**
- [ ] Create filing or service request
- [ ] Complete all tasks
- [ ] Upload all documents
- [ ] Click "Request Payment"
- [ ] Verify payment request created
- [ ] Click "Pay Now"
- [ ] Redirected to mock checkout
- [ ] Verify session details displayed
- [ ] Click "Simulate Success"
- [ ] Verify redirected to success page
- [ ] Verify payment status = `paid_platform_verified`
- [ ] Verify submission job created
- [ ] Verify filing/service request status = `submission_in_progress`

**Scenario 2: Failed Payment Flow**
- [ ] Create payment request
- [ ] Click "Pay Now"
- [ ] Click "Simulate Failure"
- [ ] Verify payment status = `required_pending`
- [ ] Verify no submission job created
- [ ] Verify can retry payment

**Scenario 3: Cancelled Payment**
- [ ] Create payment request
- [ ] Click "Pay Now"
- [ ] Click "Cancel"
- [ ] Verify redirected to cancel URL
- [ ] Verify payment status unchanged

**Test Matrix**:
| Scenario | Payment Status Before | Action | Payment Status After | Submission Job | Pass/Fail |
|----------|----------------------|--------|---------------------|----------------|-----------|
| Success  | required_pending | Simulate Success | paid_platform_verified | Created | ✅ |
| Failure  | required_pending | Simulate Failure | required_pending | Not Created | ✅ |
| Cancel   | required_pending | Click Cancel | required_pending | Not Created | ✅ |

**Acceptance Criteria**:
- ✅ All scenarios pass
- ✅ No console errors
- ✅ UI behaves correctly
- ✅ Data persists correctly

**Time Estimate**: 3 hours

---

**Task 1.9**: Automated Integration Tests (2 hours)
**Owner**: Dev 2
**Complexity**: Medium

**File**: `packages/api-services/src/domains/compliance/__tests__/payment-flow.e2e.test.ts`

```typescript
import { describe, it, expect, beforeEach } from "vitest";
import { getMockGateway } from "@repo/payments/backend";
import {
  createPaymentRequest,
  initiatePayment,
  handlePaymentVerification,
} from "../payments.service";
import { requestSubmission } from "../../submissions/submissions.service";
import { db } from "@repo/database";

describe("Payment Flow E2E", () => {
  let mockGateway: MockPaymentGateway;
  let ctx: ServiceContext;
  let deps: ServiceDependencies;

  beforeEach(() => {
    mockGateway = getMockGateway();
    ctx = { orgId: "org_test", userId: "user_test" };
    deps = { db, now: () => new Date(), logger: console };
  });

  it("should complete full payment flow with auto-submission", async () => {
    // 1. Create service request (assuming helper exists)
    const serviceRequest = await createTestServiceRequest(ctx, deps);

    // 2. Complete prerequisites
    await completeAllTasks(serviceRequest.id);
    await uploadAllDocuments(serviceRequest.id);

    // 3. Calculate and create payment
    const feeBreakdown = await calculatePaymentForSource(ctx, deps, {
      sourceType: "service_request",
      sourceId: serviceRequest.id,
      regulatorId: serviceRequest.regulatorId,
      templateId: serviceRequest.templateId,
    });

    const paymentResult = await createPaymentRequest(ctx, deps, {
      sourceType: "service_request",
      sourceId: serviceRequest.id,
      organizationId: ctx.orgId,
      regulatorId: serviceRequest.regulatorId,
      feeBreakdown,
    });

    // 4. Initiate payment
    const initiateResult = await initiatePayment(ctx, deps, {
      paymentRequestId: paymentResult.paymentRequestId,
      successUrl: "/success",
      cancelUrl: "/cancel",
    });

    expect(initiateResult.checkoutUrl).toContain("/mock/checkout/");

    // 5. Simulate successful payment
    const payment = await mockGateway.simulatePaymentSuccess(
      initiateResult.sessionId
    );

    expect(payment.status).toBe("succeeded");

    // 6. Trigger webhook manually (simulating gateway callback)
    const webhookEvent: WebhookEvent = {
      id: `evt_test_${crypto.randomUUID()}`,
      type: "payment.succeeded",
      data: {
        paymentId: payment.id,
        sessionId: initiateResult.sessionId,
        amount: payment.amount,
        currency: payment.currency,
        status: "succeeded",
        metadata: { paymentRequestId: paymentResult.paymentRequestId },
      },
      createdAt: new Date(),
    };

    // Simulate webhook handler
    await handlePaymentVerification(ctx, deps, {
      paymentRequestId: paymentResult.paymentRequestId,
      verifiedByAgentId: null,
      verificationNotes: "Auto-verified via webhook",
      verificationEvidence: {
        transactionReference: payment.id,
      },
    });

    // 7. Verify payment verified
    const verifiedPayment = await db.query.paymentRequests.findFirst({
      where: eq(paymentRequests.id, paymentResult.paymentRequestId),
    });

    expect(verifiedPayment.status).toBe("paid_platform_verified");
    expect(verifiedPayment.paidAt).toBeDefined();

    // 8. Verify submission job created
    const submissionJob = await db.query.submissionJobs.findFirst({
      where: eq(submissionJobs.serviceRequestId, serviceRequest.id),
    });

    expect(submissionJob).toBeDefined();
    expect(submissionJob.status).toBe("queued");

    // 9. Verify service request status updated
    const updatedRequest = await db.query.serviceRequests.findFirst({
      where: eq(serviceRequests.id, serviceRequest.id),
    });

    expect(updatedRequest.status).toBe("submission_in_progress");
  });

  it("should handle payment failure correctly", async () => {
    const serviceRequest = await createTestServiceRequest(ctx, deps);
    const paymentResult = await createPaymentRequest(ctx, deps, {...});
    const initiateResult = await initiatePayment(ctx, deps, {...});

    // Simulate failure
    const payment = await mockGateway.simulatePaymentFailure(
      initiateResult.sessionId,
      "insufficient_funds"
    );

    expect(payment.status).toBe("failed");

    // Payment should remain in required_pending
    const unchangedPayment = await db.query.paymentRequests.findFirst({
      where: eq(paymentRequests.id, paymentResult.paymentRequestId),
    });

    expect(unchangedPayment.status).toBe("pending_gateway");

    // No submission job should be created
    const submissionJob = await db.query.submissionJobs.findFirst({
      where: eq(submissionJobs.serviceRequestId, serviceRequest.id),
    });

    expect(submissionJob).toBeUndefined();
  });
});

// Helper functions
async function createTestServiceRequest(ctx, deps) {
  // Implementation
}

async function completeAllTasks(serviceRequestId: string) {
  // Implementation
}

async function uploadAllDocuments(serviceRequestId: string) {
  // Implementation
}
```

**Acceptance Criteria**:
- ✅ All tests pass
- ✅ Success flow tested
- ✅ Failure flow tested
- ✅ Edge cases covered

**Time Estimate**: 2 hours
**Dependencies**: Task 1.8 complete

---

### Day 5: Documentation & Sprint Review

**Task 1.10**: Create User Documentation (2 hours)
**Owner**: Dev 2
**Complexity**: Low

**Files to Create**:
- [ ] `docs/guides/testing-with-mock-gateway.md`
- [ ] `docs/guides/migrating-to-stripe.md`
- [ ] Update `README.md` with payment instructions

**Content**:
```markdown
# Testing with Mock Gateway

## Quick Start

1. Set environment variable:
   \`\`\`bash
   PAYMENT_GATEWAY=mock
   BASE_URL=http://localhost:3000
   \`\`\`

2. Create a payment request in the app

3. Click "Pay Now" - you'll be redirected to the mock checkout

4. Click "Simulate Success" to complete payment

5. Verify submission job created automatically

## For Developers

### Triggering webhooks manually:

\`\`\`bash
POST /api/payments/mock/webhook-manual
{
  "paymentRequestId": "pay_123",
  "outcome": "success"
}
\`\`\`

### Checking payment status:

\`\`\`sql
SELECT id, status, external_payment_id
FROM payment_requests
WHERE id = 'pay_123';
\`\`\`
```

**Acceptance Criteria**:
- ✅ Documentation is clear
- ✅ Examples work
- ✅ Screenshots included

**Time Estimate**: 2 hours

---

**Task 1.11**: Sprint 1 Review & Demo (1 hour)
**Owner**: Tech Lead + Team
**Complexity**: N/A

**Agenda**:
1. Demo mock gateway checkout UI (10 min)
2. Demo successful payment flow (10 min)
3. Demo failed payment flow (5 min)
4. Show auto-triggered submission job (5 min)
5. Review code quality (10 min)
6. Discuss learnings (10 min)
7. Plan Sprint 2 (10 min)

**Deliverables**:
- ✅ Working demo
- ✅ Sprint retrospective notes
- ✅ Sprint 2 ready to start

**Time Estimate**: 1 hour

---

### Sprint 1 Summary

**Total Effort**: 20 hours
**Duration**: 5 days
**Team**: 2 developers

**Deliverables**:
✅ Payment gateway interface
✅ Mock gateway implementation
✅ Mock checkout UI
✅ Webhook handler
✅ Payment service integration
✅ E2E tests
✅ Documentation

**Metrics**:
- Lines of code: ~1,500
- Test coverage: >80%
- Files created: 8
- Tests written: 15+

---

## Sprint 2: Payment State Machine

**Duration**: 1 week (5 days)
**Owner**: 2 Developers
**Effort**: 15 hours
**Priority**: P0 (Critical - Data integrity)

### Sprint Goal
Enforce valid payment status transitions and prevent invalid state changes.

### Tasks Overview

**Day 1-2**: State Machine Implementation (6 hours)
- Task 2.1: Define state machine (1 hour)
- Task 2.2: Create transition service (2 hours)
- Task 2.3: Add validation middleware (1 hour)
- Task 2.4: Write unit tests (2 hours)

**Day 3**: Service Integration (4 hours)
- Task 2.5: Update payment service (2 hours)
- Task 2.6: Update webhook handler (1 hour)
- Task 2.7: Add integration tests (1 hour)

**Day 4**: Side Effects & Cleanup (3 hours)
- Task 2.8: Implement side effects (2 hours)
- Task 2.9: Update audit logging (1 hour)

**Day 5**: Testing & Review (2 hours)
- Task 2.10: Manual testing (1 hour)
- Task 2.11: Sprint review (1 hour)

[Detailed tasks available in Phase 2 of the roadmap document]

---

## Sprint 3: Regulator Payout Tracking

**Duration**: 1 week (5 days)
**Owner**: 2 Developers
**Effort**: 20 hours
**Priority**: P1 (High - Accounting)

### Sprint Goal
Enable tracking of money OUT from Bumara to regulators for reconciliation.

### Tasks Overview

**Day 1**: Database Schema (4 hours)
- Task 3.1: Create payout schema (2 hours)
- Task 3.2: Write migration (1 hour)
- Task 3.3: Run migration & verify (1 hour)

**Day 2-3**: Payout Service (8 hours)
- Task 3.4: Create payout service (4 hours)
- Task 3.5: Create payout routes (2 hours)
- Task 3.6: Write tests (2 hours)

**Day 4**: Backoffice UI (6 hours)
- Task 3.7: Create payout list page (3 hours)
- Task 3.8: Create payout detail page (2 hours)
- Task 3.9: Add reconciliation report (1 hour)

**Day 5**: Testing & Review (2 hours)
- Task 3.10: Manual testing (1 hour)
- Task 3.11: Sprint review (1 hour)

[Detailed tasks available in Phase 3 of the roadmap document]

---

## Sprint 4: Analytics & Polish

**Duration**: 1 week (5 days)
**Owner**: 2 Developers
**Effort**: 20 hours
**Priority**: P2 (Medium - Nice to have)

### Sprint Goal
Build payment analytics dashboard and configurable handling fee rules.

### Tasks Overview

**Day 1-2**: Handling Fee Rules (8 hours)
- Task 4.1: Create fee rules schema (2 hours)
- Task 4.2: Implement fee rules service (3 hours)
- Task 4.3: Update calculators (2 hours)
- Task 4.4: Write tests (1 hour)

**Day 3-4**: Analytics (10 hours)
- Task 4.5: Create analytics service (4 hours)
- Task 4.6: Build dashboard UI (4 hours)
- Task 4.7: Add charts & exports (2 hours)

**Day 5**: Final Review (2 hours)
- Task 4.8: End-to-end testing (1 hour)
- Task 4.9: Final demo & retrospective (1 hour)

---

## Post-Sprint: Production Migration

**Duration**: 1 week
**Owner**: Tech Lead + 1 Developer
**Effort**: 10 hours
**Priority**: P0 (When ready for production)

### Tasks

**Task 5.1**: Implement Stripe Gateway (4 hours)
- [ ] Install Stripe SDK
- [ ] Create StripeGateway class
- [ ] Implement all interface methods
- [ ] Add webhook signature verification
- [ ] Test in Stripe sandbox

**Task 5.2**: Staging Deployment (2 hours)
- [ ] Update staging environment variables
- [ ] Deploy code
- [ ] Test with real Stripe sandbox
- [ ] Verify webhooks received

**Task 5.3**: Production Deployment (2 hours)
- [ ] Set up Stripe production account
- [ ] Configure webhook endpoints
- [ ] Update production environment variables
- [ ] Deploy to production
- [ ] Monitor first payments

**Task 5.4**: Documentation & Handoff (2 hours)
- [ ] Update runbooks
- [ ] Create troubleshooting guide
- [ ] Train support team
- [ ] Set up monitoring alerts

---

## Timeline Gantt Chart

```
Week 1 (Sprint 1: Mock Gateway)
Mon  Tue  Wed  Thu  Fri
[====Gateway====][==UI==][=Webhook=][=Tests=][Review]
Dev1 ████████████████████████████████░░░░░░░░
Dev2 ░░░░░░░░░░░░████████████████████████░░░░

Week 2 (Sprint 2: State Machine)
Mon  Tue  Wed  Thu  Fri
[=State Machine=][=Integration=][=Effects=][Review]
Dev1 ████████████████████░░░░░░░░░░
Dev2 ████████████████████░░░░░░░░░░

Week 3 (Sprint 3: Payouts)
Mon  Tue  Wed  Thu  Fri
[=Schema=][==Service==][===UI===][Review]
Dev1 ████████████████████████████░░
Dev2 ░░░░░░░░░░░░████████████████░░

Week 4 (Sprint 4: Analytics)
Mon  Tue  Wed  Thu  Fri
[=Fee Rules=][=Analytics=][Review]
Dev1 ████████████████████░░░░░░
Dev2 ████████████████████░░░░░░
```

---

## Resource Allocation

### Team Members

**Dev 1** (Backend Focus)
- Sprint 1: Gateway interface, mock gateway, webhook handler
- Sprint 2: State machine, transition service
- Sprint 3: Payout service, routes
- Sprint 4: Fee rules service, analytics service

**Dev 2** (Frontend/Full-Stack)
- Sprint 1: Mock checkout UI, simulation endpoint, tests
- Sprint 2: Integration, middleware, tests
- Sprint 3: Backoffice UI, reconciliation
- Sprint 4: Analytics dashboard, charts

**Tech Lead** (Code Review & Guidance)
- Review all PRs
- Unblock technical issues
- Conduct sprint reviews
- Plan production migration

---

## Definition of Done (DoD)

For each task to be considered complete:

### Code Quality
- [ ] Code follows project style guide
- [ ] No TypeScript errors
- [ ] No ESLint warnings
- [ ] All imports resolved

### Testing
- [ ] Unit tests written (>80% coverage)
- [ ] Integration tests pass
- [ ] Manual testing completed
- [ ] No regressions introduced

### Documentation
- [ ] JSDoc comments added
- [ ] README updated if needed
- [ ] User docs created
- [ ] Migration guide written (if applicable)

### Review
- [ ] PR created and linked to ticket
- [ ] Code reviewed by peer
- [ ] Feedback addressed
- [ ] PR approved and merged

### Deployment
- [ ] Deployed to staging
- [ ] Smoke tests pass
- [ ] Ready for production

---

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Existing payment flows break | Low | High | Thorough testing, backward compatibility |
| Webhook delivery fails | Medium | Medium | Retry logic, manual fallback |
| Database migration issues | Low | High | Test migration on staging first |
| Team velocity lower than expected | Medium | Medium | Buffer time in estimates, prioritize P0 tasks |
| Stripe approval delayed | Low | Low | Mock gateway allows development to continue |

---

## Success Metrics

### Sprint 1
- ✅ 100% of manual test scenarios pass
- ✅ >80% automated test coverage
- ✅ Zero critical bugs

### Sprint 2
- ✅ All invalid transitions blocked
- ✅ No production incidents due to state issues
- ✅ Audit logs capture all changes

### Sprint 3
- ✅ Payout batches created successfully
- ✅ Reconciliation reports accurate
- ✅ Finance team can use UI

### Sprint 4
- ✅ Analytics dashboard loads &lt;2 seconds
- ✅ Fee rules configurable without code deploy
- ✅ Reports exportable to Excel

### Production
- ✅ First payment processed successfully
- ✅ Webhooks delivered &lt;30 seconds
- ✅ Zero payment data loss

---

## Communication Plan

### Daily Standups (15 min)
- What did you do yesterday?
- What will you do today?
- Any blockers?

### Sprint Planning (2 hours)
- Review sprint goals
- Break down tasks
- Assign ownership
- Estimate effort

### Sprint Review (1 hour)
- Demo completed work
- Gather feedback
- Update backlog

### Sprint Retrospective (1 hour)
- What went well?
- What could improve?
- Action items for next sprint

### Ad-hoc Sync (as needed)
- Technical pairing
- Unblocking
- Quick decisions

---

## Rollout Strategy

### Phase 1: Development (Weeks 1-2)
- Mock gateway fully functional
- State machine enforced
- All tests passing

### Phase 2: Internal Testing (Week 3)
- Regulator payouts tested with finance team
- Bug fixes and refinements
- Documentation complete

### Phase 3: Staging (Week 4)
- Deploy all features to staging
- Full E2E testing
- Performance testing

### Phase 4: Production (Week 5+)
- Implement Stripe gateway
- Deploy to production
- Monitor closely
- Gradual rollout (10% → 50% → 100%)

---

## Monitoring & Alerts

### Metrics to Track
- Payment success rate
- Webhook delivery time
- Average checkout time
- Failed payment reasons
- Payout reconciliation accuracy

### Alerts to Set Up
- Payment failure rate >5%
- Webhook delivery >60 seconds
- Database errors
- Gateway downtime

### Dashboards
- Real-time payment status
- Daily revenue charts
- Regulator payout tracking
- System health metrics

---

## Appendix: Tools & Technologies

### Development
- **Language**: TypeScript
- **Backend**: Hono (web framework)
- **Database**: PostgreSQL + Drizzle ORM
- **Testing**: Vitest
- **Linting**: ESLint + Biome

### Infrastructure
- **Payment Gateway**: Stripe (production)
- **Mock Gateway**: Custom implementation
- **Webhooks**: Hono routes
- **Deployment**: TBD (Vercel/Railway/AWS)

### Monitoring
- **Logging**: Console (development), Sentry (production)
- **Analytics**: TBD
- **Error Tracking**: Sentry

---

## Next Steps

1. ✅ **Review this plan** with the team
2. ✅ **Create project board** with all tasks
3. ✅ **Assign ownership** for Sprint 1
4. ✅ **Schedule sprint planning** meeting
5. ✅ **Begin Sprint 1** on Monday!

---

**Questions or concerns?** Discuss in team meeting or Slack.

**Ready to start?** Let's build an amazing payment system! 🚀
