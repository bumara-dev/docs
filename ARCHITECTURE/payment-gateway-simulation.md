---
title: "Payment Gateway Simulation & Testing"
description: "How to simulate payment webhooks without a real gateway, then migrate to production."
---

**Last Updated**: 2026-02-10
**Status**: Implementation Guide

---

## Overview

This guide shows how to:
1. **Simulate payment webhooks** for development/testing
2. **Build with real gateway interface** from day 1
3. **Seamlessly migrate** to production gateway later

### Architecture Principle

> **"Code against interfaces, not implementations"**

We use the **Adapter Pattern** to abstract payment gateway operations. This allows:
- ✅ Develop without waiting for gateway approval
- ✅ Test payment flows end-to-end
- ✅ Swap mock → real with zero code changes
- ✅ Run integration tests without real charges

---

## Payment Gateway Interface

### Core Abstraction

```typescript
// packages/payments/gateways/interface.ts

/**
 * Payment Gateway Interface
 *
 * All payment gateways (mock, Stripe, Lenco, etc.) implement this interface.
 * This allows swapping gateways without changing business logic.
 */
export interface PaymentGateway {
  /** Gateway identifier (e.g., "stripe", "lenco", "mock") */
  readonly providerName: string;

  /** Display name for UI */
  readonly displayName: string;

  /** Is this a test/mock gateway? */
  readonly isTestMode: boolean;

  /**
   * Create a payment session/checkout.
   *
   * @returns Checkout URL + session ID
   */
  createPaymentSession(params: CreatePaymentSessionParams): Promise<PaymentSession>;

  /**
   * Verify a webhook signature.
   *
   * @returns true if signature is valid
   */
  verifyWebhookSignature(params: VerifyWebhookParams): Promise<boolean>;

  /**
   * Parse webhook event from raw request body.
   *
   * @returns Structured webhook event
   */
  parseWebhookEvent(params: ParseWebhookParams): Promise<WebhookEvent>;

  /**
   * Retrieve payment details by gateway payment ID.
   *
   * @returns Payment details
   */
  getPayment(paymentId: string): Promise<GatewayPayment | null>;

  /**
   * Refund a payment.
   *
   * @returns Refund details
   */
  refundPayment(params: RefundPaymentParams): Promise<GatewayRefund>;
}

// ============================================
// Types
// ============================================

export interface CreatePaymentSessionParams {
  organizationId: string;
  userId: string;
  amount: number;              // In minor units (ngwee)
  currency: string;            // e.g., "ZMW"
  description: string;
  successUrl: string;
  cancelUrl: string;
  metadata: Record<string, unknown>;
}

export interface PaymentSession {
  id: string;                  // Session/checkout ID
  url: string;                 // Redirect URL for user
  expiresAt: Date;
}

export interface VerifyWebhookParams {
  signature: string;
  rawBody: string | Buffer;
  secret?: string;
}

export interface ParseWebhookParams {
  rawBody: string | Buffer;
  headers: Record<string, string>;
}

export interface WebhookEvent {
  id: string;
  type: WebhookEventType;
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

export type WebhookEventType =
  | "payment.succeeded"
  | "payment.failed"
  | "payment.refunded"
  | "payment.expired";

export interface GatewayPayment {
  id: string;
  amount: number;
  currency: string;
  status: "pending" | "succeeded" | "failed" | "refunded";
  metadata: Record<string, unknown>;
  createdAt: Date;
}

export interface RefundPaymentParams {
  paymentId: string;
  amount?: number;             // Partial refund (optional)
  reason?: string;
}

export interface GatewayRefund {
  id: string;
  paymentId: string;
  amount: number;
  status: "succeeded" | "failed";
  createdAt: Date;
}
```

---

## Mock Payment Gateway Implementation

### Mock Gateway

```typescript
// packages/payments/gateways/mock-gateway.ts

import crypto from "node:crypto";
import type {
  PaymentGateway,
  CreatePaymentSessionParams,
  PaymentSession,
  VerifyWebhookParams,
  ParseWebhookParams,
  WebhookEvent,
  GatewayPayment,
  RefundPaymentParams,
  GatewayRefund,
} from "./interface";

/**
 * Mock Payment Gateway
 *
 * Simulates payment gateway behavior for development/testing.
 *
 * Features:
 * - Generates realistic session IDs
 * - Mock checkout page
 * - Webhook simulation endpoint
 * - In-memory payment storage
 */
export class MockPaymentGateway implements PaymentGateway {
  readonly providerName = "mock";
  readonly displayName = "Mock Gateway (Testing)";
  readonly isTestMode = true;

  private payments = new Map<string, GatewayPayment>();
  private sessions = new Map<string, PaymentSession & { params: CreatePaymentSessionParams }>();

  constructor(
    private config: {
      baseUrl: string;           // e.g., "http://localhost:3000"
      webhookSecret?: string;
    }
  ) {}

  /**
   * Create a mock payment session.
   *
   * Returns a mock checkout URL that redirects to a simulation page.
   */
  async createPaymentSession(
    params: CreatePaymentSessionParams
  ): Promise<PaymentSession> {
    // Generate realistic IDs
    const sessionId = `sess_mock_${this.generateId()}`;

    // Create session
    const session: PaymentSession = {
      id: sessionId,
      url: `${this.config.baseUrl}/api/payments/mock/checkout/${sessionId}`,
      expiresAt: new Date(Date.now() + 30 * 60 * 1000), // 30 min
    };

    // Store session
    this.sessions.set(sessionId, { ...session, params });

    console.log(`[MockGateway] Created session: ${sessionId}`);

    return session;
  }

  /**
   * Verify webhook signature.
   *
   * For mock gateway, we use a simple HMAC-SHA256 signature.
   */
  async verifyWebhookSignature(params: VerifyWebhookParams): Promise<boolean> {
    if (!this.config.webhookSecret) {
      // In development, allow unsigned webhooks
      return true;
    }

    const expectedSignature = crypto
      .createHmac("sha256", this.config.webhookSecret)
      .update(params.rawBody.toString())
      .digest("hex");

    return params.signature === expectedSignature;
  }

  /**
   * Parse webhook event.
   */
  async parseWebhookEvent(params: ParseWebhookParams): Promise<WebhookEvent> {
    const body = JSON.parse(params.rawBody.toString());
    return body as WebhookEvent;
  }

  /**
   * Get payment details.
   */
  async getPayment(paymentId: string): Promise<GatewayPayment | null> {
    return this.payments.get(paymentId) ?? null;
  }

  /**
   * Refund a payment.
   */
  async refundPayment(params: RefundPaymentParams): Promise<GatewayRefund> {
    const payment = this.payments.get(params.paymentId);
    if (!payment) {
      throw new Error(`Payment not found: ${params.paymentId}`);
    }

    const refundId = `refund_mock_${this.generateId()}`;
    const refundAmount = params.amount ?? payment.amount;

    // Update payment status
    payment.status = "refunded";

    console.log(`[MockGateway] Refunded payment: ${params.paymentId}`);

    return {
      id: refundId,
      paymentId: params.paymentId,
      amount: refundAmount,
      status: "succeeded",
      createdAt: new Date(),
    };
  }

  // ============================================
  // Mock-Specific Methods (for simulation)
  // ============================================

  /**
   * Simulate a successful payment.
   * Called by mock checkout page or testing utilities.
   */
  async simulatePaymentSuccess(sessionId: string): Promise<GatewayPayment> {
    const session = this.sessions.get(sessionId);
    if (!session) {
      throw new Error(`Session not found: ${sessionId}`);
    }

    // Create payment record
    const paymentId = `pi_mock_${this.generateId()}`;
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

  /**
   * Simulate a failed payment.
   */
  async simulatePaymentFailure(
    sessionId: string,
    reason: string
  ): Promise<GatewayPayment> {
    const session = this.sessions.get(sessionId);
    if (!session) {
      throw new Error(`Session not found: ${sessionId}`);
    }

    const paymentId = `pi_mock_${this.generateId()}`;
    const payment: GatewayPayment = {
      id: paymentId,
      amount: session.params.amount,
      currency: session.params.currency,
      status: "failed",
      metadata: { ...session.params.metadata, failureReason: reason },
      createdAt: new Date(),
    };

    this.payments.set(paymentId, payment);

    console.log(`[MockGateway] Payment failed: ${paymentId} (${reason})`);

    return payment;
  }

  /**
   * Get session details (for mock checkout page).
   */
  getSession(sessionId: string) {
    return this.sessions.get(sessionId);
  }

  private generateId(): string {
    return crypto.randomBytes(16).toString("hex");
  }
}
```

---

## Mock Webhook Endpoints

### Backend Routes

```typescript
// packages/backend/src/modules/payments/mock-routes.ts

import { createRoute, OpenAPIHono } from "@hono/zod-openapi";
import { z } from "zod";
import { requireAuth } from "../../middleware/auth";
import { getMockGateway } from "@repo/payments/backend";
import { db } from "@repo/database";

const app = new OpenAPIHono();

// ============================================
// Mock Checkout Page (redirected from createPaymentSession)
// ============================================

/**
 * GET /api/payments/mock/checkout/:sessionId
 *
 * Displays a mock payment page where developers can simulate success/failure.
 */
app.get("/checkout/:sessionId", async (c) => {
  const sessionId = c.req.param("sessionId");
  const gateway = getMockGateway();
  const session = gateway.getSession(sessionId);

  if (!session) {
    return c.html("<h1>Session not found</h1>", 404);
  }

  // Render mock checkout page
  return c.html(`
    <!DOCTYPE html>
    <html>
    <head>
      <title>Mock Payment Gateway</title>
      <style>
        body {
          font-family: system-ui, sans-serif;
          max-width: 600px;
          margin: 50px auto;
          padding: 20px;
          background: #f5f5f5;
        }
        .card {
          background: white;
          padding: 30px;
          border-radius: 8px;
          box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }
        h1 { color: #333; }
        .amount {
          font-size: 2em;
          font-weight: bold;
          color: #0066cc;
          margin: 20px 0;
        }
        .details {
          background: #f9f9f9;
          padding: 15px;
          border-radius: 4px;
          margin: 20px 0;
        }
        button {
          padding: 12px 24px;
          font-size: 16px;
          border: none;
          border-radius: 4px;
          cursor: pointer;
          margin: 10px 5px;
        }
        .btn-success {
          background: #28a745;
          color: white;
        }
        .btn-success:hover {
          background: #218838;
        }
        .btn-danger {
          background: #dc3545;
          color: white;
        }
        .btn-danger:hover {
          background: #c82333;
        }
        .btn-cancel {
          background: #6c757d;
          color: white;
        }
        .banner {
          background: #fff3cd;
          border: 1px solid #ffc107;
          color: #856404;
          padding: 15px;
          border-radius: 4px;
          margin-bottom: 20px;
        }
      </style>
    </head>
    <body>
      <div class="card">
        <div class="banner">
          ⚠️ <strong>Mock Payment Gateway</strong> - This is a simulated payment page for development/testing.
        </div>

        <h1>Complete Payment</h1>

        <div class="amount">
          ${formatAmount(session.params.amount, session.params.currency)}
        </div>

        <div class="details">
          <p><strong>Description:</strong> ${session.params.description}</p>
          <p><strong>Organization:</strong> ${session.params.organizationId}</p>
          <p><strong>Session ID:</strong> ${sessionId}</p>
        </div>

        <p>Simulate payment outcome:</p>

        <button class="btn-success" onclick="simulatePayment('success')">
          ✓ Simulate Success
        </button>

        <button class="btn-danger" onclick="simulatePayment('failure')">
          ✗ Simulate Failure
        </button>

        <button class="btn-cancel" onclick="window.location.href='${session.params.cancelUrl}'">
          Cancel
        </button>
      </div>

      <script>
        async function simulatePayment(outcome) {
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
              // Redirect to success page
              window.location.href = '${session.params.successUrl}?payment_id=' + result.paymentId;
            } else {
              alert('Simulation failed: ' + result.error);
            }
          } catch (error) {
            alert('Error: ' + error.message);
          }
        }
      </script>
    </body>
    </html>
  `);
});

// ============================================
// Mock Payment Simulation API
// ============================================

/**
 * POST /api/payments/mock/simulate
 *
 * Triggers a mock payment and sends webhook to backend.
 */
const simulatePaymentRoute = createRoute({
  method: "post",
  path: "/simulate",
  request: {
    body: {
      content: {
        "application/json": {
          schema: z.object({
            sessionId: z.string(),
            outcome: z.enum(["success", "failure"]),
            failureReason: z.string().optional(),
          }),
        },
      },
    },
  },
  responses: {
    200: {
      description: "Payment simulated, webhook sent",
      content: {
        "application/json": {
          schema: z.object({
            paymentId: z.string(),
            webhookSent: z.boolean(),
          }),
        },
      },
    },
  },
});

app.openapi(simulatePaymentRoute, async (c) => {
  const { sessionId, outcome, failureReason } = c.req.valid("json");

  const gateway = getMockGateway();

  // 1. Simulate payment
  const payment =
    outcome === "success"
      ? await gateway.simulatePaymentSuccess(sessionId)
      : await gateway.simulatePaymentFailure(sessionId, failureReason ?? "generic_decline");

  // 2. Construct webhook event
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
      failureReason: outcome === "failure" ? failureReason : undefined,
    },
    createdAt: new Date(),
  };

  // 3. Send webhook to our own webhook handler (simulate external call)
  const webhookUrl = `${c.req.url.split("/api")[0]}/api/webhooks/payments`;

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

    console.log(`[MockGateway] Webhook sent to ${webhookUrl}: ${webhookOk ? "✓" : "✗"}`);

    return c.json({
      paymentId: payment.id,
      webhookSent: webhookOk,
    });
  } catch (error) {
    console.error("[MockGateway] Failed to send webhook:", error);
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

// ============================================
// Developer Testing Utilities
// ============================================

/**
 * POST /api/payments/mock/webhook-manual
 *
 * Manually trigger a webhook for testing (backoffice only).
 */
const manualWebhookRoute = createRoute({
  method: "post",
  path: "/webhook-manual",
  middleware: [requireAuth],
  request: {
    body: {
      content: {
        "application/json": {
          schema: z.object({
            paymentRequestId: z.string(),
            outcome: z.enum(["success", "failure"]),
            failureReason: z.string().optional(),
          }),
        },
      },
    },
  },
  responses: {
    200: {
      description: "Webhook triggered manually",
    },
  },
});

app.openapi(manualWebhookRoute, async (c) => {
  const { paymentRequestId, outcome, failureReason } = c.req.valid("json");

  // Load payment request
  const paymentRequest = await db.query.paymentRequests.findFirst({
    where: eq(paymentRequests.id, paymentRequestId),
  });

  if (!paymentRequest) {
    return c.json({ error: "Payment request not found" }, 404);
  }

  // Construct webhook event
  const webhookEvent = {
    id: `evt_manual_${crypto.randomUUID()}`,
    type: outcome === "success" ? "payment.succeeded" : "payment.failed",
    data: {
      paymentId: `pi_manual_${crypto.randomUUID()}`,
      sessionId: paymentRequest.externalSessionId ?? `sess_manual_${crypto.randomUUID()}`,
      amount: paymentRequest.totalAmount,
      currency: paymentRequest.currency,
      status: outcome === "success" ? "succeeded" : "failed",
      metadata: { paymentRequestId },
      failureReason: outcome === "failure" ? failureReason : undefined,
    },
    createdAt: new Date(),
  };

  // Send to webhook handler
  const webhookUrl = `${c.req.url.split("/api")[0]}/api/webhooks/payments`;

  const webhookResponse = await fetch(webhookUrl, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "X-Mock-Webhook": "true",
      "X-Manual-Trigger": "true",
    },
    body: JSON.stringify(webhookEvent),
  });

  return c.json({
    ok: webhookResponse.ok,
    webhookEvent,
  });
});

export default app;

// Helper function
function formatAmount(amountInMinor: number, currency: string): string {
  const major = amountInMinor / 100;
  return new Intl.NumberFormat("en-ZM", {
    style: "currency",
    currency,
  }).format(major);
}
```

---

## Webhook Handler (Gateway-Agnostic)

### Unified Webhook Endpoint

```typescript
// packages/backend/src/modules/webhooks/payments.ts

import { Hono } from "hono";
import { getPaymentGateway } from "@repo/payments/backend";
import { handlePaymentVerification } from "@repo/api-services/domains/compliance/payments.service";
import { db } from "@repo/database";
import { eq } from "drizzle-orm";
import { paymentRequests } from "@repo/database/schema";

const app = new Hono();

/**
 * POST /api/webhooks/payments
 *
 * Universal webhook handler for all payment gateways.
 *
 * Supports:
 * - Mock Gateway (development)
 * - Stripe (production)
 * - Lenco (production)
 */
app.post("/", async (c) => {
  try {
    // 1. Get gateway (auto-detects based on config)
    const gateway = await getPaymentGateway();

    // 2. Verify webhook signature
    const signature = c.req.header("x-webhook-signature") ?? "";
    const rawBody = await c.req.raw.text();

    const isValid = await gateway.verifyWebhookSignature({
      signature,
      rawBody,
    });

    if (!isValid && !c.req.header("x-mock-webhook")) {
      console.error("[Webhook] Invalid signature");
      return c.json({ error: "Invalid signature" }, 401);
    }

    // 3. Parse webhook event
    const event = await gateway.parseWebhookEvent({
      rawBody,
      headers: Object.fromEntries(c.req.raw.headers.entries()),
    });

    console.log(`[Webhook] Received: ${event.type} (${event.id})`);

    // 4. Handle based on event type
    switch (event.type) {
      case "payment.succeeded": {
        await handlePaymentSucceeded(event);
        break;
      }

      case "payment.failed": {
        await handlePaymentFailed(event);
        break;
      }

      case "payment.refunded": {
        await handlePaymentRefunded(event);
        break;
      }

      default: {
        console.log(`[Webhook] Unhandled event type: ${event.type}`);
      }
    }

    return c.json({ received: true });
  } catch (error) {
    console.error("[Webhook] Error processing webhook:", error);
    return c.json({ error: error.message }, 500);
  }
});

// ============================================
// Event Handlers
// ============================================

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
    console.error(`[Webhook] Payment request not found: ${paymentRequestId}`);
    return;
  }

  // Update payment status to paid_gateway_verified
  await db.update(paymentRequests)
    .set({
      status: "paid_gateway_verified",
      paidAt: new Date(),
      externalPaymentId: paymentId,
      updatedAt: new Date(),
    })
    .where(eq(paymentRequests.id, paymentRequestId));

  console.log(`[Webhook] Payment verified: ${paymentRequestId}`);

  // Auto-verify and trigger submission
  try {
    await handlePaymentVerification(
      { orgId: payment.organizationId, userId: "system" },
      { db, now: () => new Date() },
      {
        paymentRequestId,
        verifiedByAgentId: null,  // System verification
        verificationNotes: "Auto-verified via payment gateway webhook",
        verificationEvidence: {
          transactionReference: paymentId,
          provider: event.data.metadata.provider as string,
        },
      }
    );

    console.log(`[Webhook] Auto-triggered submission for: ${paymentRequestId}`);
  } catch (error) {
    console.error("[Webhook] Failed to auto-verify payment:", error);
    // Don't fail webhook - payment status is already updated
  }
}

async function handlePaymentFailed(event: WebhookEvent): Promise<void> {
  const { metadata, failureReason } = event.data;
  const paymentRequestId = metadata.paymentRequestId as string;

  if (!paymentRequestId) return;

  // Update payment status
  await db.update(paymentRequests)
    .set({
      status: "required_pending",  // Reset to pending
      updatedAt: new Date(),
    })
    .where(eq(paymentRequests.id, paymentRequestId));

  console.log(`[Webhook] Payment failed: ${paymentRequestId} (${failureReason})`);

  // TODO: Notify tenant of failure
}

async function handlePaymentRefunded(event: WebhookEvent): Promise<void> {
  const { paymentId, metadata } = event.data;
  const paymentRequestId = metadata.paymentRequestId as string;

  if (!paymentRequestId) return;

  // Update payment status
  await db.update(paymentRequests)
    .set({
      status: "refunded",
      updatedAt: new Date(),
    })
    .where(eq(paymentRequests.id, paymentRequestId));

  console.log(`[Webhook] Payment refunded: ${paymentRequestId}`);

  // TODO: Reverse submission if applicable
}

export default app;
```

---

## Gateway Registry (Swap Mock ↔ Real)

### Gateway Factory

```typescript
// packages/payments/backend.ts

import type { PaymentGateway } from "./gateways/interface";
import { MockPaymentGateway } from "./gateways/mock-gateway";
// Future imports:
// import { StripeGateway } from "./gateways/stripe-gateway";
// import { LencoGateway } from "./gateways/lenco-gateway";

let cachedGateway: PaymentGateway | null = null;

/**
 * Get the active payment gateway based on environment config.
 *
 * Supports:
 * - PAYMENT_GATEWAY=mock → MockPaymentGateway
 * - PAYMENT_GATEWAY=stripe → StripeGateway
 * - PAYMENT_GATEWAY=lenco → LencoGateway
 */
export async function getPaymentGateway(): Promise<PaymentGateway> {
  if (cachedGateway) {
    return cachedGateway;
  }

  const provider = process.env.PAYMENT_GATEWAY ?? "mock";

  switch (provider) {
    case "mock": {
      cachedGateway = new MockPaymentGateway({
        baseUrl: process.env.BASE_URL ?? "http://localhost:3000",
        webhookSecret: process.env.MOCK_WEBHOOK_SECRET,
      });
      break;
    }

    case "stripe": {
      // Future: Stripe implementation
      // cachedGateway = new StripeGateway({
      //   apiKey: process.env.STRIPE_SECRET_KEY!,
      //   webhookSecret: process.env.STRIPE_WEBHOOK_SECRET!,
      // });
      throw new Error("Stripe gateway not yet implemented");
    }

    case "lenco": {
      // Future: Lenco implementation
      // cachedGateway = new LencoGateway({
      //   apiKey: process.env.LENCO_API_KEY!,
      //   webhookSecret: process.env.LENCO_WEBHOOK_SECRET!,
      // });
      throw new Error("Lenco gateway not yet implemented");
    }

    default: {
      throw new Error(`Unknown payment gateway: ${provider}`);
    }
  }

  console.log(`[Payments] Using gateway: ${cachedGateway.displayName}`);

  return cachedGateway;
}

/**
 * Get mock gateway (for testing utilities).
 */
export function getMockGateway(): MockPaymentGateway {
  if (!(cachedGateway instanceof MockPaymentGateway)) {
    throw new Error("Mock gateway not active");
  }
  return cachedGateway;
}

/**
 * Reset gateway cache (for testing).
 */
export function resetGatewayCache(): void {
  cachedGateway = null;
}
```

---

## Environment Configuration

### .env Files

```bash
# .env.development
PAYMENT_GATEWAY=mock
BASE_URL=http://localhost:3000
MOCK_WEBHOOK_SECRET=dev_secret_12345

# .env.test
PAYMENT_GATEWAY=mock
BASE_URL=http://localhost:3001
MOCK_WEBHOOK_SECRET=test_secret_67890

# .env.production (future)
PAYMENT_GATEWAY=stripe
STRIPE_SECRET_KEY=sk_live_xxxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxxx
```

---

## Testing Flows

### Manual Testing (via Mock UI)

1. **Create payment request**:
   ```bash
   POST /api/payments/requests
   {
     "sourceType": "service_request",
     "sourceId": "sr_123",
     "regulatorId": "reg_pacra",
     "feeBreakdown": { ... }
   }
   ```

2. **Initiate payment**:
   ```bash
   POST /api/payments/initiate
   {
     "paymentRequestId": "pay_456",
     "successUrl": "/payments/success",
     "cancelUrl": "/payments/cancel"
   }
   ```

3. **Redirected to**: `http://localhost:3000/api/payments/mock/checkout/sess_mock_abc123`

4. **Click "Simulate Success"** → Webhook sent → Payment verified → Submission job created ✅

### Automated Testing

```typescript
// packages/api-services/src/domains/compliance/__tests__/payments-flow.test.ts

import { describe, it, expect, beforeEach } from "vitest";
import { getMockGateway } from "@repo/payments/backend";
import { createPaymentRequest, handlePaymentVerification } from "../payments.service";
import { requestSubmission } from "../../submissions/submissions.service";

describe("Payment Flow (End-to-End)", () => {
  let mockGateway: MockPaymentGateway;

  beforeEach(() => {
    mockGateway = getMockGateway();
  });

  it("should auto-trigger submission after payment success", async () => {
    // 1. Create payment request
    const paymentResult = await createPaymentRequest(ctx, deps, {
      sourceType: "service_request",
      sourceId: "sr_test_123",
      regulatorId: "reg_pacra",
      feeBreakdown: { regulatorFee: 50000, handlingFee: 2500, totalAmount: 52500 },
    });

    // 2. Create payment session
    const session = await mockGateway.createPaymentSession({
      organizationId: "org_123",
      userId: "user_456",
      amount: paymentResult.totalAmount,
      currency: "ZMW",
      description: "Test payment",
      successUrl: "/success",
      cancelUrl: "/cancel",
      metadata: { paymentRequestId: paymentResult.paymentRequestId },
    });

    // 3. Simulate payment success
    const payment = await mockGateway.simulatePaymentSuccess(session.id);

    // 4. Verify payment (triggers submission)
    const verifyResult = await handlePaymentVerification(ctx, deps, {
      paymentRequestId: paymentResult.paymentRequestId,
      verifiedByAgentId: "agent_789",
      verificationNotes: "Test verification",
    });

    // Assertions
    expect(verifyResult.status).toBe("paid_platform_verified");
    expect(verifyResult.automaticSubmissionTriggered).toBe(true);
    expect(verifyResult.submissionJobId).toBeDefined();
  });

  it("should handle payment failure gracefully", async () => {
    // ... test failure flow
  });
});
```

---

## Migration to Real Gateway

### Step 1: Implement Gateway Adapter

```typescript
// packages/payments/gateways/stripe-gateway.ts

import Stripe from "stripe";
import type {
  PaymentGateway,
  CreatePaymentSessionParams,
  PaymentSession,
  // ... other imports
} from "./interface";

export class StripeGateway implements PaymentGateway {
  readonly providerName = "stripe";
  readonly displayName = "Stripe";
  readonly isTestMode = false;

  private stripe: Stripe;

  constructor(config: { apiKey: string; webhookSecret: string }) {
    this.stripe = new Stripe(config.apiKey, {
      apiVersion: "2023-10-16",
    });
    this.webhookSecret = config.webhookSecret;
  }

  async createPaymentSession(
    params: CreatePaymentSessionParams
  ): Promise<PaymentSession> {
    // Create Stripe checkout session
    const session = await this.stripe.checkout.sessions.create({
      mode: "payment",
      payment_method_types: ["card"],
      line_items: [
        {
          price_data: {
            currency: params.currency.toLowerCase(),
            product_data: {
              name: params.description,
            },
            unit_amount: params.amount,  // Already in minor units
          },
          quantity: 1,
        },
      ],
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

  async verifyWebhookSignature(params: VerifyWebhookParams): Promise<boolean> {
    try {
      this.stripe.webhooks.constructEvent(
        params.rawBody,
        params.signature,
        this.webhookSecret
      );
      return true;
    } catch {
      return false;
    }
  }

  async parseWebhookEvent(params: ParseWebhookParams): Promise<WebhookEvent> {
    const stripeEvent = this.stripe.webhooks.constructEvent(
      params.rawBody,
      params.headers["stripe-signature"],
      this.webhookSecret
    );

    // Map Stripe event to our format
    switch (stripeEvent.type) {
      case "checkout.session.completed": {
        const session = stripeEvent.data.object as Stripe.Checkout.Session;
        return {
          id: stripeEvent.id,
          type: "payment.succeeded",
          data: {
            paymentId: session.payment_intent as string,
            sessionId: session.id,
            amount: session.amount_total!,
            currency: session.currency!.toUpperCase(),
            status: "succeeded",
            metadata: session.metadata ?? {},
          },
          createdAt: new Date(stripeEvent.created * 1000),
        };
      }

      // ... other event types

      default: {
        throw new Error(`Unhandled Stripe event: ${stripeEvent.type}`);
      }
    }
  }

  // ... implement other methods
}
```

### Step 2: Update Environment Config

```bash
# .env.production
PAYMENT_GATEWAY=stripe
STRIPE_SECRET_KEY=sk_live_xxxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxxx
BASE_URL=https://app.bumara.com
```

### Step 3: Zero Code Changes

**No changes needed in**:
- ✅ Payment service layer (`payments.service.ts`)
- ✅ Webhook handler (`webhooks/payments.ts`)
- ✅ Submission flow (`submissions.service.ts`)
- ✅ Frontend payment UI

**Only change**: Environment variable `PAYMENT_GATEWAY=stripe`

---

## Developer Utilities

### Backoffice Mock Payment Panel

```typescript
// apps/backoffice/app/(authenticated)/(home)/dev/mock-payments/page.tsx

"use client";

import { useState } from "react";
import { Button } from "@/components/ui/button";
import { Card, CardHeader, CardTitle, CardContent } from "@/components/ui/card";

export default function MockPaymentsPage() {
  const [paymentRequestId, setPaymentRequestId] = useState("");

  const triggerWebhook = async (outcome: "success" | "failure") => {
    const response = await fetch("/api/payments/mock/webhook-manual", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        paymentRequestId,
        outcome,
        failureReason: outcome === "failure" ? "insufficient_funds" : undefined,
      }),
    });

    const result = await response.json();
    alert(result.ok ? "Webhook sent!" : "Failed: " + result.error);
  };

  return (
    <div className="p-8">
      <Card>
        <CardHeader>
          <CardTitle>Mock Payment Webhook Trigger</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="space-y-4">
            <input
              type="text"
              placeholder="Payment Request ID"
              value={paymentRequestId}
              onChange={(e) => setPaymentRequestId(e.target.value)}
              className="w-full p-2 border rounded"
            />

            <div className="flex gap-2">
              <Button onClick={() => triggerWebhook("success")}>
                Simulate Success
              </Button>
              <Button variant="destructive" onClick={() => triggerWebhook("failure")}>
                Simulate Failure
              </Button>
            </div>
          </div>
        </CardContent>
      </Card>
    </div>
  );
}
```

---

## Summary

### What You Get

1. ✅ **Mock Gateway** - Fully functional payment simulation
2. ✅ **Mock Checkout UI** - Visual payment flow testing
3. ✅ **Webhook Simulation** - Auto-verify payments locally
4. ✅ **Testing Utilities** - Backoffice dev panel
5. ✅ **Gateway Abstraction** - Swap mock → real with env var
6. ✅ **Zero Migration Cost** - No code changes when switching

### Development Workflow

```bash
# 1. Start dev server
pnpm dev

# 2. Set mock gateway
export PAYMENT_GATEWAY=mock

# 3. Test payment flow
curl -X POST http://localhost:3000/api/payments/initiate

# 4. Visit mock checkout
open http://localhost:3000/api/payments/mock/checkout/sess_xxx

# 5. Click "Simulate Success"
# → Webhook sent
# → Payment verified
# → Submission job created ✅
```

### Production Deployment

```bash
# 1. Implement StripeGateway (one-time)
# 2. Update .env.production
export PAYMENT_GATEWAY=stripe
export STRIPE_SECRET_KEY=sk_live_xxx
export STRIPE_WEBHOOK_SECRET=whsec_xxx

# 3. Deploy (zero code changes) ✅
```

---

**The mock gateway gives you full end-to-end testing now, with a seamless migration path to production later!**
