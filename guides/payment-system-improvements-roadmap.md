---
title: "Payment System Improvements - Implementation Roadmap"
description: "Complete guide to implementing all recommended payment system enhancements."
---

**Last Updated**: 2026-02-10
**Total Estimated Time**: 2-3 sprints
**Difficulty**: Medium to Advanced

---

## Overview

This guide implements all key recommendations from the payment architecture:

1. ✅ **Mock Payment Gateway** (Phase 1 - 1 hour)
2. 🔄 **Payment State Machine Enforcement** (Phase 2 - 3 hours)
3. 🔄 **Regulator Payout Tracking** (Phase 3 - 5 hours)
4. 🔄 **Configurable Handling Fee Rules** (Phase 4 - 4 hours)
5. 🔄 **Payment Analytics Dashboard** (Phase 5 - 6 hours)

---

## Phase 1: Mock Payment Gateway ✅

**Status**: Documented in [implementing-mock-payment-gateway.md](/guides/implementing-mock-payment-gateway)

**Time**: 1 hour

**Deliverables**:
- ✅ Payment gateway interface
- ✅ Mock gateway implementation
- ✅ Mock checkout UI
- ✅ Webhook simulation
- ✅ Testing utilities

---

## Phase 2: Payment State Machine Enforcement

**Goal**: Prevent invalid payment status transitions and enforce business rules.

**Time**: 3 hours

### Step 1: Define State Machine (30 min)

**File**: `packages/api-services/src/domains/compliance/payment-state-machine.ts`

```typescript
import type { PaymentRequestStatus } from "@repo/database/schema";

/**
 * Valid payment status transitions.
 *
 * This state machine prevents invalid transitions like:
 * - paid_platform_verified → required_pending (can't unpay)
 * - cancelled → pending_gateway (can't resurrect)
 */
export const PAYMENT_STATE_TRANSITIONS: Record<
  PaymentRequestStatus,
  PaymentRequestStatus[]
> = {
  required_pending: [
    "pending_gateway",       // User initiates payment
    "paid_platform_verified", // Backoffice manual verification
    "waived",                // Backoffice waives fee
    "cancelled",             // User/system cancels
  ],

  pending_gateway: [
    "paid_gateway_verified",  // Gateway webhook confirms
    "paid_platform_verified", // Backoffice manual override
    "required_pending",       // Payment expired/cancelled at gateway
    "cancelled",              // User cancels
  ],

  paid_gateway_verified: [
    "paid_platform_verified", // Backoffice confirms gateway payment
    "refunded",               // Backoffice initiates refund
  ],

  paid_platform_verified: [
    "refunded",               // Only valid transition
  ],

  waived: [],                 // Terminal state
  refunded: [],               // Terminal state
  cancelled: [],              // Terminal state
};

/**
 * Check if a status transition is valid.
 */
export function canTransitionPaymentStatus(
  from: PaymentRequestStatus,
  to: PaymentRequestStatus
): boolean {
  const validTransitions = PAYMENT_STATE_TRANSITIONS[from];
  return validTransitions.includes(to);
}

/**
 * Get valid next states for a payment.
 */
export function getValidNextStates(
  currentStatus: PaymentRequestStatus
): PaymentRequestStatus[] {
  return PAYMENT_STATE_TRANSITIONS[currentStatus];
}

/**
 * Prerequisites for each status.
 */
export const PAYMENT_STATUS_PREREQUISITES: Record<
  PaymentRequestStatus,
  {
    description: string;
    checks: ((payment: any) => Promise<boolean> | boolean)[];
  }
> = {
  required_pending: {
    description: "Payment is required but not yet initiated",
    checks: [],
  },

  pending_gateway: {
    description: "Payment session created, awaiting user action",
    checks: [
      (payment) => !!payment.externalSessionId,
      (payment) => !!payment.paymentProvider,
    ],
  },

  paid_gateway_verified: {
    description: "Payment confirmed by gateway webhook",
    checks: [
      (payment) => !!payment.externalPaymentId,
      (payment) => !!payment.paidAt,
    ],
  },

  paid_platform_verified: {
    description: "Payment verified by backoffice staff",
    checks: [
      (payment) => !!payment.verifiedAt,
      // Note: verifiedByAgentId can be null for webhook auto-verification
    ],
  },

  waived: {
    description: "Payment fee waived by backoffice",
    checks: [
      (payment) => !!payment.verifiedByAgentId,
    ],
  },

  refunded: {
    description: "Payment refunded to tenant",
    checks: [],
  },

  cancelled: {
    description: "Payment cancelled",
    checks: [],
  },
};
```

### Step 2: Create Transition Service (45 min)

**File**: `packages/api-services/src/domains/compliance/payment-transitions.service.ts`

```typescript
import { eq } from "drizzle-orm";
import { paymentRequests } from "@repo/database/schema";
import type { ServiceContext, ServiceDependencies } from "../../core/context";
import { ServiceError } from "../../core/errors";
import { recordAuditLog } from "../audit/audit-log.service";
import {
  canTransitionPaymentStatus,
  PAYMENT_STATUS_PREREQUISITES,
} from "./payment-state-machine";
import type { PaymentRequestStatus } from "@repo/database/schema";

/**
 * Transition payment status with validation.
 *
 * Enforces:
 * 1. Valid state machine transitions
 * 2. Prerequisites for target status
 * 3. Audit logging
 * 4. Side effects (e.g., close tickets, trigger submission)
 */
export async function transitionPaymentStatus(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  input: {
    paymentRequestId: string;
    newStatus: PaymentRequestStatus;
    reason?: string;
    metadata?: Record<string, unknown>;
    skipValidation?: boolean; // For system-level operations
  }
): Promise<void> {
  const { paymentRequestId, newStatus, reason, metadata, skipValidation } = input;

  // 1. Load current payment
  const payment = await deps.db.query.paymentRequests.findFirst({
    where: eq(paymentRequests.id, paymentRequestId),
  });

  if (!payment) {
    throw new ServiceError("NOT_FOUND", "Payment request not found");
  }

  const currentStatus = payment.status;

  // Skip if already in target status
  if (currentStatus === newStatus) {
    return;
  }

  // 2. Validate transition
  if (!skipValidation) {
    if (!canTransitionPaymentStatus(currentStatus, newStatus)) {
      throw new ServiceError(
        "INVALID_TRANSITION",
        `Cannot transition from ${currentStatus} to ${newStatus}`,
        {
          validTransitions: PAYMENT_STATE_TRANSITIONS[currentStatus],
          attempted: newStatus,
        }
      );
    }

    // 3. Check prerequisites
    const prerequisites = PAYMENT_STATUS_PREREQUISITES[newStatus];
    for (const check of prerequisites.checks) {
      const satisfied = await check(payment);
      if (!satisfied) {
        throw new ServiceError(
          "PREREQUISITES_NOT_MET",
          `Prerequisites not met for status ${newStatus}: ${prerequisites.description}`
        );
      }
    }
  }

  // 4. Update status
  const now = deps.now();
  await deps.db
    .update(paymentRequests)
    .set({
      status: newStatus,
      updatedAt: now,
    })
    .where(eq(paymentRequests.id, paymentRequestId));

  // 5. Audit log
  await recordAuditLog(ctx, deps, {
    entityType: "payment_request",
    entityId: paymentRequestId,
    action: "update",
    metadata: {
      fromStatus: currentStatus,
      toStatus: newStatus,
      reason,
      ...metadata,
    },
  });

  // 6. Side effects
  await executeTransitionSideEffects(ctx, deps, {
    payment,
    fromStatus: currentStatus,
    toStatus: newStatus,
  });
}

/**
 * Execute side effects based on status transition.
 */
async function executeTransitionSideEffects(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  input: {
    payment: any;
    fromStatus: PaymentRequestStatus;
    toStatus: PaymentRequestStatus;
  }
): Promise<void> {
  const { payment, fromStatus, toStatus } = input;

  switch (toStatus) {
    case "paid_platform_verified": {
      // Close payment ticket
      await deps.db
        .update(tickets)
        .set({
          status: "resolved",
          resolvedAt: deps.now(),
          updatedAt: deps.now(),
        })
        .where(
          and(
            eq(tickets.paymentRequestId, payment.id),
            eq(tickets.type, "payment_request")
          )
        );

      // Create timeline event
      await deps.db.insert(timelineEvents).values({
        organizationId: payment.organizationId,
        filingId: payment.filingId,
        serviceRequestId: payment.serviceRequestId,
        eventType: "payment",
        title: "Payment verified",
        description: "Payment has been verified by the platform",
        occurredAt: deps.now(),
      });
      break;
    }

    case "waived": {
      // Close payment ticket
      await deps.db
        .update(tickets)
        .set({
          status: "resolved",
          resolvedAt: deps.now(),
          updatedAt: deps.now(),
        })
        .where(
          and(
            eq(tickets.paymentRequestId, payment.id),
            eq(tickets.type, "payment_request")
          )
        );
      break;
    }

    case "refunded": {
      // Create timeline event
      await deps.db.insert(timelineEvents).values({
        organizationId: payment.organizationId,
        filingId: payment.filingId,
        serviceRequestId: payment.serviceRequestId,
        eventType: "payment",
        title: "Payment refunded",
        description: "Payment has been refunded",
        occurredAt: deps.now(),
      });
      break;
    }
  }
}
```

### Step 3: Add Validation Middleware (30 min)

**File**: `packages/backend/src/middleware/payment-validation.ts`

```typescript
import type { Context, Next } from "hono";
import { canTransitionPaymentStatus } from "@repo/api-services/domains/compliance/payment-state-machine";

/**
 * Middleware to validate payment status transitions.
 *
 * Use on routes that modify payment status.
 */
export async function validatePaymentTransition(c: Context, next: Next) {
  const body = await c.req.json();

  if (body.status) {
    const currentPayment = c.get("payment"); // Set by requirePayment middleware

    if (currentPayment && !canTransitionPaymentStatus(currentPayment.status, body.status)) {
      return c.json(
        {
          error: "INVALID_TRANSITION",
          message: `Cannot transition from ${currentPayment.status} to ${body.status}`,
        },
        400
      );
    }
  }

  await next();
}
```

### Step 4: Update Payment Service (45 min)

Update all payment status changes to use `transitionPaymentStatus`:

**File**: `packages/api-services/src/domains/compliance/payments.service.ts`

```typescript
import { transitionPaymentStatus } from "./payment-transitions.service";

// Update handlePaymentVerification
export async function handlePaymentVerification(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  input: VerifyPaymentInput
): Promise<VerifyPaymentResult> {
  // ... existing validation

  // Use transition service instead of direct update
  await transitionPaymentStatus(ctx, deps, {
    paymentRequestId,
    newStatus: "paid_platform_verified",
    reason: input.verificationNotes,
    metadata: {
      verifiedByAgentId,
      verificationEvidence: input.verificationEvidence,
    },
  });

  // ... rest of function
}

// Add waive payment function
export async function waivePayment(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  input: {
    paymentRequestId: string;
    waivedByAgentId: string;
    reason: string;
  }
): Promise<void> {
  const { paymentRequestId, waivedByAgentId, reason } = input;

  // Transition to waived
  await transitionPaymentStatus(ctx, deps, {
    paymentRequestId,
    newStatus: "waived",
    reason,
    metadata: {
      waivedByAgentId,
    },
  });

  // Update payment record
  await deps.db
    .update(paymentRequests)
    .set({
      verifiedByAgentId: waivedByAgentId,
      verifiedAt: deps.now(),
    })
    .where(eq(paymentRequests.id, paymentRequestId));
}
```

### Step 5: Add Tests (30 min)

**File**: `packages/api-services/src/domains/compliance/__tests__/payment-state-machine.test.ts`

```typescript
import { describe, it, expect } from "vitest";
import {
  canTransitionPaymentStatus,
  getValidNextStates,
} from "../payment-state-machine";

describe("Payment State Machine", () => {
  it("should allow valid transitions", () => {
    expect(canTransitionPaymentStatus("required_pending", "pending_gateway")).toBe(true);
    expect(canTransitionPaymentStatus("pending_gateway", "paid_gateway_verified")).toBe(true);
    expect(canTransitionPaymentStatus("paid_gateway_verified", "paid_platform_verified")).toBe(true);
  });

  it("should reject invalid transitions", () => {
    expect(canTransitionPaymentStatus("paid_platform_verified", "required_pending")).toBe(false);
    expect(canTransitionPaymentStatus("cancelled", "pending_gateway")).toBe(false);
    expect(canTransitionPaymentStatus("refunded", "paid_platform_verified")).toBe(false);
  });

  it("should return valid next states", () => {
    const nextStates = getValidNextStates("required_pending");
    expect(nextStates).toContain("pending_gateway");
    expect(nextStates).toContain("paid_platform_verified");
    expect(nextStates).toContain("waived");
    expect(nextStates).toContain("cancelled");
  });

  it("should not allow transitions from terminal states", () => {
    expect(getValidNextStates("waived")).toEqual([]);
    expect(getValidNextStates("refunded")).toEqual([]);
    expect(getValidNextStates("cancelled")).toEqual([]);
  });
});
```

---

## Phase 3: Regulator Payout Tracking

**Goal**: Track money OUT from Bumara to regulators for reconciliation.

**Time**: 5 hours

### Step 1: Create Payout Schema (30 min)

**File**: `packages/database/src/schema/compliance/regulator-payouts.ts`

```typescript
import {
  index,
  integer,
  jsonb,
  pgTable,
  text,
  timestamp,
  uuid,
} from "drizzle-orm/pg-core";
import { timestamps } from "../../helpers/timestamps";
import { regulators } from "../core/regulators";
import { backOfficeAgents } from "../core/back-office-agents";
import { documents } from "./documents";

/**
 * Payout Status Enum
 */
export const payoutStatusEnum = pgEnum("payout_status", [
  "pending",           // Created, awaiting approval
  "approved",          // Approved, awaiting transfer
  "processing",        // Transfer in progress
  "completed",         // Transfer completed
  "failed",            // Transfer failed
  "cancelled",         // Cancelled before processing
]);

/**
 * Regulator Payouts
 *
 * Tracks money OUT from Bumara to regulators.
 *
 * Flow:
 * 1. Create payout batch for a period (monthly/weekly)
 * 2. Approve payout
 * 3. Transfer funds to regulator
 * 4. Upload proof of payment
 * 5. Mark as completed
 */
export const regulatorPayouts = pgTable(
  "regulator_payouts",
  {
    id: uuid("id").primaryKey().defaultRandom(),

    // Regulator receiving payout
    regulatorId: uuid("regulator_id")
      .notNull()
      .references(() => regulators.id, { onDelete: "restrict" }),

    // Payout amount in minor units (ngwee)
    amount: integer("amount").notNull(),
    currency: text("currency").default("ZMW").notNull(),

    // Status
    status: payoutStatusEnum("status").default("pending").notNull(),

    // Period covered by this payout
    periodStart: timestamp("period_start", { mode: "date" }).notNull(),
    periodEnd: timestamp("period_end", { mode: "date" }).notNull(),

    // Payment requests included in this payout
    paymentRequestIds: jsonb("payment_request_ids")
      .$type<string[]>()
      .notNull()
      .default([]),

    // Summary stats
    paymentCount: integer("payment_count").notNull().default(0),
    totalRegulatorFees: integer("total_regulator_fees").notNull().default(0),
    totalHandlingFees: integer("total_handling_fees").notNull().default(0),

    // Processing
    approvedByAgentId: uuid("approved_by_agent_id").references(
      () => backOfficeAgents.id,
      { onDelete: "set null" }
    ),
    approvedAt: timestamp("approved_at", { mode: "date" }),

    processedByAgentId: uuid("processed_by_agent_id").references(
      () => backOfficeAgents.id,
      { onDelete: "set null" }
    ),
    processedAt: timestamp("processed_at", { mode: "date" }),

    // Proof of payment
    proofDocumentId: uuid("proof_document_id").references(() => documents.id, {
      onDelete: "set null",
    }),

    // Transfer details
    transferReference: text("transfer_reference"),
    transferMethod: text("transfer_method"), // "bank_transfer", "mobile_money", etc.

    // Metadata
    metadata: jsonb("metadata").$type<Record<string, unknown>>(),
    notes: text("notes"),

    ...timestamps,
  },
  (table) => [
    index("idx_regulator_payouts_regulator").on(table.regulatorId, table.status),
    index("idx_regulator_payouts_period").on(table.periodStart, table.periodEnd),
    index("idx_regulator_payouts_status").on(table.status),
  ]
);

export type RegulatorPayout = typeof regulatorPayouts.$inferSelect;
export type NewRegulatorPayout = typeof regulatorPayouts.$inferInsert;
```

**Migration**: `packages/database/migrations/add-regulator-payouts.sql`

```sql
-- Create payout status enum
CREATE TYPE payout_status AS ENUM (
  'pending',
  'approved',
  'processing',
  'completed',
  'failed',
  'cancelled'
);

-- Create regulator_payouts table
CREATE TABLE regulator_payouts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  regulator_id UUID NOT NULL REFERENCES regulators(id) ON DELETE RESTRICT,
  amount INTEGER NOT NULL,
  currency TEXT NOT NULL DEFAULT 'ZMW',
  status payout_status NOT NULL DEFAULT 'pending',
  period_start TIMESTAMP NOT NULL,
  period_end TIMESTAMP NOT NULL,
  payment_request_ids JSONB NOT NULL DEFAULT '[]',
  payment_count INTEGER NOT NULL DEFAULT 0,
  total_regulator_fees INTEGER NOT NULL DEFAULT 0,
  total_handling_fees INTEGER NOT NULL DEFAULT 0,
  approved_by_agent_id UUID REFERENCES back_office_agents(id) ON DELETE SET NULL,
  approved_at TIMESTAMP,
  processed_by_agent_id UUID REFERENCES back_office_agents(id) ON DELETE SET NULL,
  processed_at TIMESTAMP,
  proof_document_id UUID REFERENCES documents(id) ON DELETE SET NULL,
  transfer_reference TEXT,
  transfer_method TEXT,
  metadata JSONB,
  notes TEXT,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_regulator_payouts_regulator ON regulator_payouts(regulator_id, status);
CREATE INDEX idx_regulator_payouts_period ON regulator_payouts(period_start, period_end);
CREATE INDEX idx_regulator_payouts_status ON regulator_payouts(status);
```

### Step 2: Create Payout Service (2 hours)

**File**: `packages/api-services/src/domains/compliance/regulator-payouts.service.ts`

```typescript
import { and, between, eq, inArray, isNull, sql } from "drizzle-orm";
import {
  paymentRequests,
  regulatorPayouts,
  regulators,
  filings,
  serviceRequests,
  timelineEvents,
} from "@repo/database/schema";
import type { ServiceContext, ServiceDependencies } from "../../core/context";
import { ServiceError } from "../../core/errors";
import { recordAuditLog } from "../audit/audit-log.service";

/**
 * Create a regulator payout batch for a period.
 *
 * Aggregates all verified payments for a regulator and creates
 * a payout record for reconciliation.
 */
export async function createRegulatorPayout(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  input: {
    regulatorId: string;
    periodStart: Date;
    periodEnd: Date;
    notes?: string;
  }
): Promise<{
  payoutId: string;
  amount: number;
  paymentCount: number;
}> {
  const { regulatorId, periodStart, periodEnd, notes } = input;

  // 1. Validate regulator exists
  const regulator = await deps.db.query.regulators.findFirst({
    where: eq(regulators.id, regulatorId),
  });

  if (!regulator) {
    throw new ServiceError("NOT_FOUND", "Regulator not found");
  }

  // 2. Find all verified payments for this regulator in the period
  // that haven't been included in a previous payout
  const payments = await deps.db
    .select({
      paymentRequest: paymentRequests,
      regulatorId: sql<string>`COALESCE(
        ${filings.regulatorId},
        ${serviceRequests.regulatorId}
      )`.as("regulator_id"),
    })
    .from(paymentRequests)
    .leftJoin(filings, eq(paymentRequests.filingId, filings.id))
    .leftJoin(serviceRequests, eq(paymentRequests.serviceRequestId, serviceRequests.id))
    .where(
      and(
        eq(paymentRequests.status, "paid_platform_verified"),
        between(paymentRequests.verifiedAt, periodStart, periodEnd),
        sql`COALESCE(${filings.regulatorId}, ${serviceRequests.regulatorId}) = ${regulatorId}`,
        // Exclude payments already in a payout
        sql`NOT EXISTS (
          SELECT 1 FROM regulator_payouts rp
          WHERE ${paymentRequests.id} = ANY(rp.payment_request_ids)
        )`
      )
    );

  if (payments.length === 0) {
    throw new ServiceError(
      "NO_PAYMENTS_FOUND",
      `No payments found for regulator ${regulator.name} in period ${periodStart.toISOString()} - ${periodEnd.toISOString()}`
    );
  }

  // 3. Calculate totals
  const totalRegulatorFees = payments.reduce(
    (sum, p) => sum + p.paymentRequest.regulatorFee,
    0
  );
  const totalHandlingFees = payments.reduce(
    (sum, p) => sum + p.paymentRequest.handlingFee,
    0
  );
  const payoutAmount = totalRegulatorFees; // Only regulator fees, not handling fees

  // 4. Create payout record
  const [payout] = await deps.db
    .insert(regulatorPayouts)
    .values({
      regulatorId,
      amount: payoutAmount,
      currency: "ZMW",
      status: "pending",
      periodStart,
      periodEnd,
      paymentRequestIds: payments.map((p) => p.paymentRequest.id),
      paymentCount: payments.length,
      totalRegulatorFees,
      totalHandlingFees,
      notes,
      createdAt: deps.now(),
      updatedAt: deps.now(),
    })
    .returning();

  // 5. Audit log
  await recordAuditLog(ctx, deps, {
    entityType: "regulator_payout",
    entityId: payout.id,
    action: "create",
    metadata: {
      regulatorId,
      regulatorName: regulator.name,
      periodStart: periodStart.toISOString(),
      periodEnd: periodEnd.toISOString(),
      amount: payoutAmount,
      paymentCount: payments.length,
    },
  });

  return {
    payoutId: payout.id,
    amount: payoutAmount,
    paymentCount: payments.length,
  };
}

/**
 * Approve a regulator payout.
 *
 * Requires backoffice admin permission.
 */
export async function approveRegulatorPayout(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  input: {
    payoutId: string;
    approvedByAgentId: string;
    notes?: string;
  }
): Promise<void> {
  const { payoutId, approvedByAgentId, notes } = input;

  // Load payout
  const payout = await deps.db.query.regulatorPayouts.findFirst({
    where: eq(regulatorPayouts.id, payoutId),
  });

  if (!payout) {
    throw new ServiceError("NOT_FOUND", "Payout not found");
  }

  if (payout.status !== "pending") {
    throw new ServiceError(
      "INVALID_STATUS",
      `Payout is ${payout.status}, expected pending`
    );
  }

  // Update status
  await deps.db
    .update(regulatorPayouts)
    .set({
      status: "approved",
      approvedByAgentId,
      approvedAt: deps.now(),
      notes: notes ?? payout.notes,
      updatedAt: deps.now(),
    })
    .where(eq(regulatorPayouts.id, payoutId));

  // Audit log
  await recordAuditLog(ctx, deps, {
    entityType: "regulator_payout",
    entityId: payoutId,
    action: "approve",
    metadata: {
      approvedByAgentId,
      amount: payout.amount,
      paymentCount: payout.paymentCount,
    },
  });
}

/**
 * Complete a regulator payout.
 *
 * Called after funds are transferred to regulator.
 */
export async function completeRegulatorPayout(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  input: {
    payoutId: string;
    processedByAgentId: string;
    transferReference: string;
    transferMethod: string;
    proofDocumentId?: string;
    notes?: string;
  }
): Promise<void> {
  const {
    payoutId,
    processedByAgentId,
    transferReference,
    transferMethod,
    proofDocumentId,
    notes,
  } = input;

  // Load payout
  const payout = await deps.db.query.regulatorPayouts.findFirst({
    where: eq(regulatorPayouts.id, payoutId),
    with: {
      regulator: true,
    },
  });

  if (!payout) {
    throw new ServiceError("NOT_FOUND", "Payout not found");
  }

  if (payout.status !== "approved") {
    throw new ServiceError(
      "INVALID_STATUS",
      `Payout is ${payout.status}, expected approved`
    );
  }

  // Update status
  await deps.db
    .update(regulatorPayouts)
    .set({
      status: "completed",
      processedByAgentId,
      processedAt: deps.now(),
      transferReference,
      transferMethod,
      proofDocumentId,
      notes: notes ?? payout.notes,
      updatedAt: deps.now(),
    })
    .where(eq(regulatorPayouts.id, payoutId));

  // Audit log
  await recordAuditLog(ctx, deps, {
    entityType: "regulator_payout",
    entityId: payoutId,
    action: "complete",
    metadata: {
      processedByAgentId,
      transferReference,
      transferMethod,
      amount: payout.amount,
      regulatorName: payout.regulator.name,
    },
  });
}

/**
 * Get payout reconciliation report.
 */
export async function getPayoutReconciliationReport(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  input: {
    regulatorId?: string;
    startDate: Date;
    endDate: Date;
  }
): Promise<{
  totalPayments: number;
  totalRegulatorFees: number;
  totalHandlingFees: number;
  totalPayoutAmount: number;
  payouts: Array<{
    id: string;
    regulatorName: string;
    amount: number;
    paymentCount: number;
    status: string;
    periodStart: Date;
    periodEnd: Date;
  }>;
}> {
  const { regulatorId, startDate, endDate } = input;

  const filters = [
    between(regulatorPayouts.periodStart, startDate, endDate),
  ];

  if (regulatorId) {
    filters.push(eq(regulatorPayouts.regulatorId, regulatorId));
  }

  const payoutsData = await deps.db
    .select({
      payout: regulatorPayouts,
      regulatorName: regulators.name,
    })
    .from(regulatorPayouts)
    .innerJoin(regulators, eq(regulatorPayouts.regulatorId, regulators.id))
    .where(and(...filters));

  const totalPayments = payoutsData.reduce((sum, p) => sum + p.payout.paymentCount, 0);
  const totalRegulatorFees = payoutsData.reduce(
    (sum, p) => sum + p.payout.totalRegulatorFees,
    0
  );
  const totalHandlingFees = payoutsData.reduce(
    (sum, p) => sum + p.payout.totalHandlingFees,
    0
  );
  const totalPayoutAmount = payoutsData.reduce((sum, p) => sum + p.payout.amount, 0);

  return {
    totalPayments,
    totalRegulatorFees,
    totalHandlingFees,
    totalPayoutAmount,
    payouts: payoutsData.map((p) => ({
      id: p.payout.id,
      regulatorName: p.regulatorName,
      amount: p.payout.amount,
      paymentCount: p.payout.paymentCount,
      status: p.payout.status,
      periodStart: p.payout.periodStart,
      periodEnd: p.payout.periodEnd,
    })),
  };
}
```

### Step 3: Create Payout Routes (1 hour)

**File**: `packages/backend/src/modules/compliance/regulator-payouts/routes.ts`

```typescript
import { createRoute, OpenAPIHono, z } from "@hono/zod-openapi";
import { requireAuth, requireBackofficeAdmin } from "../../../middleware/auth";
import {
  createRegulatorPayout,
  approveRegulatorPayout,
  completeRegulatorPayout,
  getPayoutReconciliationReport,
} from "@repo/api-services/domains/compliance/regulator-payouts.service";

const app = new OpenAPIHono();

// Create payout
const createPayoutRoute = createRoute({
  method: "post",
  path: "/",
  middleware: [requireAuth, requireBackofficeAdmin],
  request: {
    body: {
      content: {
        "application/json": {
          schema: z.object({
            regulatorId: z.string(),
            periodStart: z.string().datetime(),
            periodEnd: z.string().datetime(),
            notes: z.string().optional(),
          }),
        },
      },
    },
  },
  responses: {
    200: {
      description: "Payout created",
      content: {
        "application/json": {
          schema: z.object({
            payoutId: z.string(),
            amount: z.number(),
            paymentCount: z.number(),
          }),
        },
      },
    },
  },
});

app.openapi(createPayoutRoute, async (c) => {
  const { regulatorId, periodStart, periodEnd, notes } = c.req.valid("json");

  const result = await createRegulatorPayout(
    c.get("ctx"),
    c.get("deps"),
    {
      regulatorId,
      periodStart: new Date(periodStart),
      periodEnd: new Date(periodEnd),
      notes,
    }
  );

  return c.json(result);
});

// Approve payout
const approvePayoutRoute = createRoute({
  method: "post",
  path: "/:id/approve",
  middleware: [requireAuth, requireBackofficeAdmin],
  request: {
    body: {
      content: {
        "application/json": {
          schema: z.object({
            notes: z.string().optional(),
          }),
        },
      },
    },
  },
  responses: {
    200: { description: "Payout approved" },
  },
});

app.openapi(approvePayoutRoute, async (c) => {
  const payoutId = c.req.param("id");
  const { notes } = c.req.valid("json");
  const agentId = c.get("ctx").userId!;

  await approveRegulatorPayout(c.get("ctx"), c.get("deps"), {
    payoutId,
    approvedByAgentId: agentId,
    notes,
  });

  return c.json({ ok: true });
});

// Complete payout
const completePayoutRoute = createRoute({
  method: "post",
  path: "/:id/complete",
  middleware: [requireAuth, requireBackofficeAdmin],
  request: {
    body: {
      content: {
        "application/json": {
          schema: z.object({
            transferReference: z.string(),
            transferMethod: z.string(),
            proofDocumentId: z.string().optional(),
            notes: z.string().optional(),
          }),
        },
      },
    },
  },
  responses: {
    200: { description: "Payout completed" },
  },
});

app.openapi(completePayoutRoute, async (c) => {
  const payoutId = c.req.param("id");
  const { transferReference, transferMethod, proofDocumentId, notes } =
    c.req.valid("json");
  const agentId = c.get("ctx").userId!;

  await completeRegulatorPayout(c.get("ctx"), c.get("deps"), {
    payoutId,
    processedByAgentId: agentId,
    transferReference,
    transferMethod,
    proofDocumentId,
    notes,
  });

  return c.json({ ok: true });
});

// Get reconciliation report
const reconciliationReportRoute = createRoute({
  method: "get",
  path: "/reconciliation",
  middleware: [requireAuth, requireBackofficeAdmin],
  request: {
    query: z.object({
      regulatorId: z.string().optional(),
      startDate: z.string().datetime(),
      endDate: z.string().datetime(),
    }),
  },
  responses: {
    200: {
      description: "Reconciliation report",
    },
  },
});

app.openapi(reconciliationReportRoute, async (c) => {
  const { regulatorId, startDate, endDate } = c.req.valid("query");

  const report = await getPayoutReconciliationReport(c.get("ctx"), c.get("deps"), {
    regulatorId,
    startDate: new Date(startDate),
    endDate: new Date(endDate),
  });

  return c.json(report);
});

export default app;
```

### Step 4: Create Backoffice UI (1.5 hours)

**File**: `apps/backoffice/app/(authenticated)/(home)/finance/payouts/page.tsx`

```typescript
"use client";

import { useState } from "react";
import { useQuery, useMutation } from "@tanstack/react-query";
import { Button } from "@/components/ui/button";
import { Card, CardHeader, CardTitle, CardContent } from "@/components/ui/card";
import { Badge } from "@/components/ui/badge";
import { formatCurrency } from "@/lib/utils";

export default function PayoutsPage() {
  const [selectedRegulator, setSelectedRegulator] = useState<string | null>(null);
  const [periodStart, setPeriodStart] = useState(
    new Date(new Date().getFullYear(), new Date().getMonth(), 1).toISOString().split("T")[0]
  );
  const [periodEnd, setPeriodEnd] = useState(
    new Date().toISOString().split("T")[0]
  );

  // Fetch payouts
  const { data: payouts, refetch } = useQuery({
    queryKey: ["regulator-payouts", selectedRegulator, periodStart, periodEnd],
    queryFn: async () => {
      const params = new URLSearchParams({
        startDate: new Date(periodStart).toISOString(),
        endDate: new Date(periodEnd).toISOString(),
      });
      if (selectedRegulator) params.set("regulatorId", selectedRegulator);

      const response = await fetch(`/api/regulator-payouts/reconciliation?${params}`);
      return response.json();
    },
  });

  // Create payout mutation
  const createPayout = useMutation({
    mutationFn: async (regulatorId: string) => {
      const response = await fetch("/api/regulator-payouts", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          regulatorId,
          periodStart: new Date(periodStart).toISOString(),
          periodEnd: new Date(periodEnd).toISOString(),
        }),
      });
      return response.json();
    },
    onSuccess: () => refetch(),
  });

  return (
    <div className="p-8 space-y-6">
      <div className="flex justify-between items-center">
        <h1 className="text-3xl font-bold">Regulator Payouts</h1>
        <Button onClick={() => createPayout.mutate(selectedRegulator!)}>
          Create Payout
        </Button>
      </div>

      {/* Period Selector */}
      <Card>
        <CardHeader>
          <CardTitle>Period Selection</CardTitle>
        </CardHeader>
        <CardContent className="flex gap-4">
          <input
            type="date"
            value={periodStart}
            onChange={(e) => setPeriodStart(e.target.value)}
            className="border rounded px-3 py-2"
          />
          <span className="flex items-center">to</span>
          <input
            type="date"
            value={periodEnd}
            onChange={(e) => setPeriodEnd(e.target.value)}
            className="border rounded px-3 py-2"
          />
        </CardContent>
      </Card>

      {/* Summary */}
      {payouts && (
        <div className="grid grid-cols-4 gap-4">
          <Card>
            <CardContent className="pt-6">
              <div className="text-sm text-muted-foreground">Total Payments</div>
              <div className="text-2xl font-bold">{payouts.totalPayments}</div>
            </CardContent>
          </Card>
          <Card>
            <CardContent className="pt-6">
              <div className="text-sm text-muted-foreground">Regulator Fees</div>
              <div className="text-2xl font-bold">
                {formatCurrency(payouts.totalRegulatorFees)}
              </div>
            </CardContent>
          </Card>
          <Card>
            <CardContent className="pt-6">
              <div className="text-sm text-muted-foreground">Handling Fees</div>
              <div className="text-2xl font-bold">
                {formatCurrency(payouts.totalHandlingFees)}
              </div>
            </CardContent>
          </Card>
          <Card>
            <CardContent className="pt-6">
              <div className="text-sm text-muted-foreground">Total Payout</div>
              <div className="text-2xl font-bold">
                {formatCurrency(payouts.totalPayoutAmount)}
              </div>
            </CardContent>
          </Card>
        </div>
      )}

      {/* Payouts Table */}
      <Card>
        <CardHeader>
          <CardTitle>Payouts</CardTitle>
        </CardHeader>
        <CardContent>
          <table className="w-full">
            <thead>
              <tr className="border-b">
                <th className="text-left p-2">Regulator</th>
                <th className="text-left p-2">Period</th>
                <th className="text-right p-2">Amount</th>
                <th className="text-right p-2">Payments</th>
                <th className="text-left p-2">Status</th>
                <th className="text-left p-2">Actions</th>
              </tr>
            </thead>
            <tbody>
              {payouts?.payouts.map((payout) => (
                <tr key={payout.id} className="border-b">
                  <td className="p-2">{payout.regulatorName}</td>
                  <td className="p-2">
                    {new Date(payout.periodStart).toLocaleDateString()} -{" "}
                    {new Date(payout.periodEnd).toLocaleDateString()}
                  </td>
                  <td className="p-2 text-right">
                    {formatCurrency(payout.amount)}
                  </td>
                  <td className="p-2 text-right">{payout.paymentCount}</td>
                  <td className="p-2">
                    <PayoutStatusBadge status={payout.status} />
                  </td>
                  <td className="p-2">
                    <PayoutActions payout={payout} onSuccess={refetch} />
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        </CardContent>
      </Card>
    </div>
  );
}

function PayoutStatusBadge({ status }: { status: string }) {
  const variants = {
    pending: "secondary",
    approved: "default",
    processing: "default",
    completed: "success",
    failed: "destructive",
    cancelled: "outline",
  };

  return <Badge variant={variants[status]}>{status}</Badge>;
}

function PayoutActions({ payout, onSuccess }) {
  // Approve/Complete actions based on status
  // ...
}
```

---

## Phase 4: Configurable Handling Fee Rules

**Goal**: Allow per-regulator or per-service handling fee configuration.

**Time**: 4 hours

### Implementation Summary

Create `handling_fee_rules` table with:
- Per-regulator rates
- Per-service-template rates
- Priority-based rule resolution
- Effective date ranges

See full code in the architecture doc section "Recommended Improvements > Configurable Handling Fee Rules".

---

## Phase 5: Payment Analytics Dashboard

**Goal**: Business intelligence and reporting for payments.

**Time**: 6 hours

### Implementation Summary

Create analytics service with:
- Revenue by regulator
- Payment success rates
- Handling fee collection
- Time-series charts
- Export to CSV/Excel

---

## Summary & Timeline

### Sprint 1 (Week 1)
- [x] Mock Payment Gateway (1 hour)
- [ ] Payment State Machine (3 hours)
- [ ] Tests & Documentation (2 hours)

### Sprint 2 (Week 2)
- [ ] Regulator Payout Schema (30 min)
- [ ] Payout Service (2 hours)
- [ ] Payout Routes (1 hour)
- [ ] Backoffice UI (1.5 hours)

### Sprint 3 (Week 3)
- [ ] Handling Fee Rules Schema (1 hour)
- [ ] Fee Rules Service (2 hours)
- [ ] Update Fee Calculators (1 hour)

### Sprint 4 (Week 4)
- [ ] Payment Analytics Service (3 hours)
- [ ] Analytics Dashboard UI (3 hours)

### Total: ~20 hours over 4 weeks

---

## Next Steps

1. **Start with Phase 2** (Payment State Machine) - Highest ROI
2. **Then Phase 3** (Regulator Payouts) - Critical for accounting
3. **Phase 4 & 5** can be done in parallel if needed

All phases build on each other and follow the same patterns established in Phase 1 (Mock Gateway).

---

**Questions or need help with implementation?** Refer to the architecture docs or create an issue!
