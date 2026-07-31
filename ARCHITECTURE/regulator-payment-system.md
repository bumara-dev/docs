---
title: "Regulator Payment System Architecture"
description: "Comprehensive guide to handling tenant payments to Bumara for regulator services."
---

**Last Updated**: 2026-02-10
**Status**: Architectural Design
**Author**: Senior Engineering Team

---

## Table of Contents

1. [Overview](#overview)
2. [Current System Analysis](#current-system-analysis)
3. [Payment Flow](#payment-flow)
4. [Fee Calculation Architecture](#fee-calculation-architecture)
5. [Integration with Submission Jobs](#integration-with-submission-jobs)
6. [Adding New Regulators](#adding-new-regulators)
7. [Best Practices & Improvements](#best-practices--improvements)
8. [Security & Compliance](#security--compliance)
9. [Migration & Rollout](#migration--rollout)

---

## Overview

### Purpose

The regulator payment system handles:
- **Tenant-to-Bumara payments** for compliance services
- **Regulator-specific fee calculations** (PACRA, ZRA, NAPSA, NHIMA, etc.)
- **Payment verification** and automatic submission triggering
- **Bumara-to-regulator remittances** (future: regulator payouts)

### Key Principles

1. **Multi-Regulator Support**: Pluggable fee calculators for different regulators
2. **Separation of Concerns**: Tenant payments ≠ Regulator remittances
3. **Extensibility**: Easy to add new regulators without code changes
4. **Audit Trail**: Full payment lifecycle tracking
5. **Idempotency**: Safe retries and duplicate prevention
6. **Multi-Tenant Isolation**: Strict org-level data boundaries

---

## Current System Analysis

### Database Schema

```typescript
// Tenant Payments (money IN to Bumara)
payment_requests {
  id: uuid                          // Payment request ID
  organizationId: string            // Tenant who pays
  filingId?: uuid                   // Source: filing
  serviceRequestId?: uuid           // Source: service request

  // Fee breakdown (in ngwee - minor units)
  regulatorFee: integer             // What regulator charges
  handlingFee: integer              // Bumara service fee
  totalAmount: integer              // regulatorFee + handlingFee
  currency: string                  // Default: ZMW

  // Lifecycle
  status: PaymentRequestStatus      // State machine
  requestedAt: timestamp
  paidAt?: timestamp
  verifiedAt?: timestamp
  verifiedByAgentId?: uuid          // Backoffice staff

  // Gateway integration
  externalSessionId?: string        // Stripe/Lenco session
  externalPaymentId?: string        // Transaction ID
  paymentProvider?: string          // 'stripe', 'lenco', etc.
}

// Regulator Fee Catalog
regulator_fees {
  id: uuid
  regulatorId: uuid                 // Which regulator
  feeKey: string                    // e.g., "PACRA_ANNUAL_RETURN_COMPANY"
  name: string
  amount: integer                   // In minor units (ngwee)
  currency: string

  // Conditional fees (tiered pricing)
  conditions?: {
    entityType?: string[]
    companyType?: string[]
    shareCapitalMin?: number
    shareCapitalMax?: number
  }

  // Fee history
  effectiveFrom: timestamp
  effectiveUntil?: timestamp

  // Audit
  sourceReference: string           // "PACRA Gazette Notice Jan 2025"
  updatedByAgentId: uuid
  notes: text
}

// Regulator Remittances (money OUT to regulators) - FUTURE
regulator_payouts {
  id: uuid
  regulatorId: uuid
  amount: integer                   // Total to remit
  currency: string
  status: PayoutStatus              // pending, processing, completed
  periodStart: date
  periodEnd: date
  paymentRequestIds: uuid[]         // Which payments included
  proofDocumentId?: uuid            // Receipt/proof of payment
  processedAt?: timestamp
}
```

### Payment Request State Machine

```typescript
type PaymentRequestStatus =
  | "required_pending"              // Payment needed, not yet paid
  | "pending_gateway"               // Checkout session created, awaiting payment
  | "paid_gateway_verified"         // Gateway confirmed payment (webhook)
  | "paid_platform_verified"        // Backoffice manually verified
  | "waived"                        // Fee waived by backoffice
  | "refunded"                      // Payment refunded
  | "cancelled";                    // Payment cancelled

// Valid transitions
const VALID_TRANSITIONS = {
  required_pending: ["pending_gateway", "paid_platform_verified", "waived", "cancelled"],
  pending_gateway: ["paid_gateway_verified", "paid_platform_verified", "cancelled"],
  paid_gateway_verified: ["paid_platform_verified", "refunded"],
  paid_platform_verified: ["refunded"],
  waived: [],
  refunded: [],
  cancelled: [],
};
```

---

## Payment Flow

### End-to-End Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    TENANT APP (User Journey)                     │
└─────────────────────────────────────────────────────────────────┘

1. User completes filing/service request data
   └─> All tasks done
   └─> All documents uploaded
   └─> Status: ready_for_submission

2. System checks payment requirements
   └─> GET /api/payments/calculate
       {
         sourceType: "filing" | "service_request",
         sourceId: "fil_123",
         regulatorId: "reg_pacra",
         templateId: "tmpl_456",
         entityConditions: { companyType: "private" }
       }

3. Fee Calculator resolves fee
   ┌──────────────────────────────────┐
   │  Fee Calculator Registry         │
   │  ┌────────────────────────────┐ │
   │  │ 1. Map feeKey → calculator │ │
   │  │ 2. Load regulator_fees     │ │
   │  │ 3. Apply conditions        │ │
   │  │ 4. Calculate handling fee  │ │
   │  │ 5. Return breakdown        │ │
   │  └────────────────────────────┘ │
   └──────────────────────────────────┘

   Response:
   {
     regulatorFee: 50000,      // K500 in ngwee
     handlingFee: 2500,        // K25 (5% of K500, service request)
     totalAmount: 52500,       // K525
     currency: "ZMW",
     feeKey: "PACRA_ANNUAL_RETURN_COMPANY",
     lineItems: [...]
   }

4. Create payment request
   └─> POST /api/payments/requests
   └─> Idempotent: returns existing if already created
   └─> Creates payment_requests record
   └─> Creates payment ticket (type: payment_request, status: waiting_on_tenant)

5. User initiates payment
   └─> POST /api/payments/initiate
       {
         paymentRequestId: "pay_123",
         successUrl: "/payments/success",
         cancelUrl: "/payments/cancel"
       }

6. Payment gateway checkout
   ┌──────────────────────────────────┐
   │  Payment Gateway (Stripe/Lenco)  │
   │  ┌────────────────────────────┐ │
   │  │ 1. Create checkout session │ │
   │  │ 2. Redirect to gateway     │ │
   │  │ 3. User pays               │ │
   │  │ 4. Gateway webhook         │ │
   │  └────────────────────────────┘ │
   └──────────────────────────────────┘

   └─> Webhook: POST /api/webhooks/payments
       {
         event: "payment.succeeded",
         paymentId: "pi_abc123",
         metadata: { paymentRequestId: "pay_123" }
       }

7. Auto-verify and trigger submission
   ┌──────────────────────────────────┐
   │  Automatic Submission Trigger    │
   │  ┌────────────────────────────┐ │
   │  │ 1. Update payment → paid   │ │
   │  │ 2. Close payment ticket    │ │
   │  │ 3. Check submission ready  │ │
   │  │ 4. Create submission job   │ │
   │  │ 5. Notify tenant & staff   │ │
   │  └────────────────────────────┘ │
   └──────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   BACKOFFICE (Staff Workflow)                    │
└─────────────────────────────────────────────────────────────────┘

8. Staff claims submission job
   └─> Submission job status: queued → assigned → in_progress

9. Staff completes submission to regulator
   └─> Submission job status: submitted → accepted

10. Regulator payout (periodic batch)
    └─> Monthly/weekly: Aggregate all paid fees
    └─> Create regulator_payouts record
    └─> Transfer funds to regulator
    └─> Mark payout as completed
```

### Handling Fee Rules

```typescript
// Current Configuration
const HANDLING_FEE_CONFIG = {
  // Source-based rates
  rates: {
    filing: 0,              // 0% for filings (statutory filings are included)
    service_request: 0.05,  // 5% for service requests (ad-hoc services)
  },

  // Floor and ceiling
  minimum: 5000,            // K50 minimum (in ngwee)
  maximum: 500000,          // K5,000 maximum (in ngwee)

  // Currency
  defaultCurrency: "ZMW",
};

// Calculation Logic
function calculateHandlingFee(
  regulatorFee: number,
  sourceType: "filing" | "service_request"
): number {
  const rate = HANDLING_FEE_CONFIG.rates[sourceType];

  // Calculate base handling fee
  const baseFee = Math.round(regulatorFee * rate);

  // Apply floor and ceiling
  if (baseFee === 0) return 0;  // No fee for filings
  if (baseFee < HANDLING_FEE_CONFIG.minimum) {
    return HANDLING_FEE_CONFIG.minimum;
  }
  if (baseFee > HANDLING_FEE_CONFIG.maximum) {
    return HANDLING_FEE_CONFIG.maximum;
  }

  return baseFee;
}
```

---

## Fee Calculation Architecture

### Calculator Registry Pattern

```typescript
/**
 * Fee Calculator Interface (Strategy Pattern)
 *
 * Each calculator handles a specific fee type and knows:
 * 1. How to validate input
 * 2. Where to get fee data
 * 3. How to compute the fee
 * 4. How to structure the breakdown
 */
interface FeeCalculator<TContext extends BaseFeeContext> {
  calculatorId: string;
  calculationType: FeeCalculationType;
  displayName: string;

  canHandle(context: BaseFeeContext): boolean;
  validate(context: TContext): string[];
  calculate(context: TContext): Promise<DetailedFeeBreakdown>;
}

/**
 * Fee Calculator Types
 */
type FeeCalculationType =
  | "fixed"               // Table lookup (PACRA annual returns)
  | "percentage_capital"  // % of share capital (PACRA company reg)
  | "contribution_based"  // Pre-calculated payroll % (NAPSA, NHIMA)
  | "progressive_tax"     // Tax bands (ZRA PAYE)
  | "pre_calculated"      // Stored on source (Turnover Tax)
  | "custom";             // Regulator-specific logic

/**
 * Registry: Auto-routing based on fee key
 */
class FeeCalculatorRegistry {
  private calculators = new Map<FeeCalculationType, FeeCalculator>();

  register(calculator: FeeCalculator) {
    this.calculators.set(calculator.calculationType, calculator);
  }

  async calculate(context: BaseFeeContext): Promise<DetailedFeeBreakdown> {
    // 1. Map fee key to calculator type
    const calculatorType = getCalculatorTypeForFeeKey(context.feeKey);

    // 2. Get calculator
    const calculator = this.calculators.get(calculatorType);
    if (!calculator) {
      throw new FeeCalculationError("CALCULATOR_NOT_FOUND", ...);
    }

    // 3. Validate input
    const errors = calculator.validate(context);
    if (errors.length > 0) {
      throw new FeeCalculationError("VALIDATION_ERROR", ...);
    }

    // 4. Calculate
    return calculator.calculate(context);
  }
}
```

### Fee Key → Calculator Mapping

```typescript
// packages/payments/calculators/config.ts

export const FEE_KEY_CALCULATOR_MAP: Record<string, FeeCalculationType> = {
  // PACRA Fixed Fees
  PACRA_ANNUAL_RETURN_COMPANY: "fixed",
  PACRA_ANNUAL_RETURN_BUSINESS_NAME: "fixed",
  PACRA_NAME_CLEARANCE: "fixed",
  PACRA_NAME_RESERVATION: "fixed",
  PACRA_CHANGE_DIRECTORS: "fixed",
  PACRA_CHANGE_REGISTERED_OFFICE: "fixed",
  PACRA_CERTIFICATE_GOOD_STANDING: "fixed",
  PACRA_CERTIFICATE_INCORPORATION: "fixed",

  // PACRA Percentage of Capital
  PACRA_COMPANY_REGISTRATION: "percentage_capital",

  // NAPSA/NHIMA Contribution-based
  NAPSA_MONTHLY_CONTRIBUTION: "contribution_based",
  NHIMA_MONTHLY_CONTRIBUTION: "contribution_based",

  // ZRA Progressive Tax
  ZRA_PAYE_MONTHLY: "progressive_tax",

  // ZRA Pre-calculated
  ZRA_TURNOVER_TAX_MONTHLY: "pre_calculated",
  ZRA_PROVISIONAL_TAX_QUARTERLY: "pre_calculated",
  ZRA_WITHHOLDING_TAX: "pre_calculated",

  // Future: Custom calculators
  ECZ_LICENSE_APPLICATION: "custom",
  ZEMA_ENVIRONMENTAL_PERMIT: "custom",
};

export function getCalculatorTypeForFeeKey(feeKey: string): FeeCalculationType {
  return FEE_KEY_CALCULATOR_MAP[feeKey] ?? "fixed";  // Default to fixed
}
```

### Example: Fixed Fee Calculator

```typescript
// packages/payments/calculators/fixed-fee.ts

export class FixedFeeCalculator implements FeeCalculator<FixedFeeContext> {
  readonly calculatorId = "fixed_fee";
  readonly calculationType = "fixed" as const;
  readonly displayName = "Fixed Fee Calculator";

  constructor(private deps: CalculatorDependencies) {}

  canHandle(context: BaseFeeContext): boolean {
    const type = getCalculatorTypeForFeeKey(context.feeKey);
    return type === "fixed";
  }

  validate(context: FixedFeeContext): string[] {
    const errors: string[] = [];

    if (!context.feeKey) {
      errors.push("feeKey is required");
    }
    if (!context.regulatorId) {
      errors.push("regulatorId is required");
    }

    return errors;
  }

  async calculate(context: FixedFeeContext): Promise<DetailedFeeBreakdown> {
    const { regulatorId, feeKey, sourceType, entityConditions } = context;

    // 1. Lookup fee from regulator_fees table
    const fee = await this.deps.db.query.regulatorFees.findFirst({
      where: and(
        eq(regulatorFees.regulatorId, regulatorId),
        eq(regulatorFees.feeKey, feeKey),
        isNull(regulatorFees.effectiveUntil)  // Active fees only
      ),
    });

    if (!fee) {
      throw new FeeCalculationError(
        "FEE_NOT_FOUND",
        `Fee not found: ${feeKey} for regulator ${regulatorId}`
      );
    }

    // 2. Check conditions (if variable fee)
    const regulatorFee = this.applyConditions(fee, entityConditions);

    // 3. Calculate handling fee based on source type
    const handlingFee = calculateHandlingFee(regulatorFee, sourceType);

    // 4. Build breakdown
    const lineItems: FeeLineItem[] = [
      {
        code: "REGULATOR_FEE",
        label: fee.name,
        amount: regulatorFee,
        metadata: { feeId: fee.id, feeKey },
      },
    ];

    if (handlingFee > 0) {
      lineItems.push({
        code: "HANDLING_FEE",
        label: "Bumara Service Fee",
        amount: handlingFee,
        metadata: { rate: HANDLING_FEE_CONFIG.rates[sourceType] },
      });
    }

    return {
      regulatorFee,
      handlingFee,
      totalAmount: regulatorFee + handlingFee,
      currency: fee.currency,
      feeKey,
      feeSource: "regulator_fees_table",
      calculationType: "fixed",
      lineItems,
      handlingFeeInfo: {
        rate: HANDLING_FEE_CONFIG.rates[sourceType],
        calculatedFrom: "regulator_fee",
        minimumApplied: handlingFee === HANDLING_FEE_CONFIG.minimum,
        maximumApplied: handlingFee === HANDLING_FEE_CONFIG.maximum,
      },
    };
  }

  private applyConditions(
    fee: RegulatorFee,
    entityConditions?: { entityType?: string; companyType?: string }
  ): number {
    // Simple implementation - no conditions check
    // For advanced: check fee.conditions against entityConditions
    return fee.amount;
  }
}
```

### Example: Custom Calculator (Regulator-Specific)

```typescript
// packages/payments/calculators/custom/zema-environmental-permit.ts

/**
 * ZEMA Environmental Permit Calculator
 *
 * Fee structure:
 * - Project cost < K100,000: K500 flat
 * - Project cost K100,000 - K1M: K1,000 + 0.5% of project cost
 * - Project cost > K1M: K5,000 + 0.3% of project cost
 *
 * This is regulator-specific logic that doesn't fit other calculators.
 */
export class ZemaEnvironmentalPermitCalculator implements FeeCalculator {
  readonly calculatorId = "zema_environmental_permit";
  readonly calculationType = "custom" as const;
  readonly displayName = "ZEMA Environmental Permit Calculator";

  canHandle(context: BaseFeeContext): boolean {
    return context.feeKey === "ZEMA_ENVIRONMENTAL_PERMIT";
  }

  validate(context: BaseFeeContext): string[] {
    const errors: string[] = [];

    // Expect projectCost in metadata
    if (!context.metadata?.projectCost) {
      errors.push("projectCost is required in metadata");
    }

    return errors;
  }

  async calculate(context: BaseFeeContext): Promise<DetailedFeeBreakdown> {
    const projectCost = context.metadata.projectCost as number; // In ZMW major units

    // Tiered calculation
    let regulatorFee: number;
    let tier: string;

    if (projectCost < 100000) {
      regulatorFee = 50000;  // K500 flat
      tier = "small";
    } else if (projectCost <= 1000000) {
      regulatorFee = 100000 + Math.round(projectCost * 0.005 * 100);  // K1,000 + 0.5%
      tier = "medium";
    } else {
      regulatorFee = 500000 + Math.round(projectCost * 0.003 * 100);  // K5,000 + 0.3%
      tier = "large";
    }

    const handlingFee = calculateHandlingFee(regulatorFee, context.sourceType);

    return {
      regulatorFee,
      handlingFee,
      totalAmount: regulatorFee + handlingFee,
      currency: "ZMW",
      feeKey: "ZEMA_ENVIRONMENTAL_PERMIT",
      feeSource: "calculated",
      calculationType: "custom",
      lineItems: [
        {
          code: "BASE_FEE",
          label: `ZEMA Permit Fee (${tier} project)`,
          amount: regulatorFee,
          metadata: { projectCost, tier },
        },
        {
          code: "HANDLING_FEE",
          label: "Bumara Service Fee",
          amount: handlingFee,
        },
      ],
      handlingFeeInfo: {
        rate: HANDLING_FEE_CONFIG.rates[context.sourceType],
        calculatedFrom: "regulator_fee",
      },
      notes: `Project cost: K${projectCost.toLocaleString()} (${tier} tier)`,
    };
  }
}

// Register in registry initialization
registerHandler(new ZemaEnvironmentalPermitCalculator(deps));
```

---

## Integration with Submission Jobs

### Submission Readiness Gates

```typescript
/**
 * Compute submission readiness for a filing or service request.
 *
 * Gates:
 * 1. All required tasks done
 * 2. All required documents uploaded
 * 3. Payment verified (if required)
 */
async function computeSubmissionReadiness(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  input: { sourceType: SourceType; sourceId: string }
): Promise<ReadinessResult> {
  const orgId = requireOrganizationContext(ctx);

  // 1. Check tasks
  const tasks = await loadTasks(deps, orgId, input.sourceType, input.sourceId);
  const tasksGate = {
    total: tasks.length,
    done: tasks.filter(t => t.status === "done").length,
    required: tasks.filter(t => t.required).length,
    requiredDone: tasks.filter(t => t.required && t.status === "done").length,
    missingTaskIds: tasks
      .filter(t => t.required && t.status !== "done")
      .map(t => t.id),
  };

  // 2. Check documents
  const docRequirements = await loadDocRequirements(deps, input.sourceType, templateId);
  const uploadedDocs = await loadUploadedDocs(deps, orgId, input.sourceType, input.sourceId);
  const docsGate = {
    required: docRequirements.filter(r => r.required).length,
    satisfied: docRequirements.filter(r =>
      r.required && uploadedDocs.has(r.key)
    ).length,
    missingRequirementKeys: docRequirements
      .filter(r => r.required && !uploadedDocs.has(r.key))
      .map(r => r.key),
  };

  // 3. Check payment
  const payment = await loadPaymentForSource(deps, orgId, input.sourceType, input.sourceId);
  const paymentGate = {
    required: !!payment,
    status: payment?.status ?? null,
    paymentRequestId: payment?.id ?? null,
  };

  // Determine readiness
  const isReady =
    tasksGate.requiredDone === tasksGate.required &&
    docsGate.satisfied === docsGate.required &&
    (!paymentGate.required || paymentGate.status === "paid_platform_verified");

  return {
    isReady,
    blockers: {
      tasks: tasksGate.missingTaskIds,
      documents: docsGate.missingRequirementKeys,
      payment: !paymentGate.required || paymentGate.status === "paid_platform_verified"
        ? null
        : { status: paymentGate.status, paymentRequestId: paymentGate.paymentRequestId },
    },
    snapshot: {
      tasks: tasksGate,
      documents: docsGate,
      payment: paymentGate,
    },
  };
}
```

### Automatic Submission Trigger (After Payment Verification)

```typescript
/**
 * Verify payment and automatically trigger submission.
 *
 * This is called by:
 * 1. Backoffice staff (manual verification)
 * 2. Payment gateway webhook (automatic verification)
 *
 * Flow:
 * 1. Update payment → paid_platform_verified
 * 2. Close payment ticket
 * 3. Check if source is ready for submission
 * 4. If ready → Create submission job automatically
 * 5. Return result with submissionJobId
 */
async function handlePaymentVerification(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  input: VerifyPaymentInput
): Promise<VerifyPaymentResult> {
  const { paymentRequestId, verifiedByAgentId } = input;

  // 1. Load and validate payment
  const payment = await loadPayment(deps, paymentRequestId);
  if (payment.status === "paid_platform_verified") {
    throw new ServiceError("CONFLICT", "Payment already verified");
  }

  // 2. Update payment status
  await deps.db.update(paymentRequests)
    .set({
      status: "paid_platform_verified",
      verifiedAt: new Date(),
      verifiedByAgentId,
    })
    .where(eq(paymentRequests.id, paymentRequestId));

  // 3. Close payment ticket
  await closePaymentTicket(deps, payment.organizationId, paymentRequestId);

  // 4. Audit log
  await recordAuditLog(ctx, deps, {
    entityType: "payment_request",
    entityId: paymentRequestId,
    action: "approve",
    metadata: { verifiedByAgentId, amount: payment.totalAmount },
  });

  // 5. AUTOMATIC SUBMISSION TRIGGER
  const sourceType = payment.filingId ? "filing" : "service_request";
  const sourceId = (payment.filingId ?? payment.serviceRequestId) as string;

  let submissionJobId: string | null = null;
  let automaticSubmissionTriggered = false;

  try {
    // Check readiness
    const readiness = await computeSubmissionReadiness(
      { orgId: payment.organizationId, userId: verifiedByAgentId },
      deps,
      { sourceType, sourceId }
    );

    if (readiness.isReady) {
      // Create submission job
      const submissionResult = await requestSubmission(
        { orgId: payment.organizationId, userId: verifiedByAgentId },
        deps,
        { sourceType, sourceId }
      );

      if (submissionResult.ok) {
        submissionJobId = submissionResult.submissionJobId;
        automaticSubmissionTriggered = true;

        // Transition source status
        if (sourceType === "filing") {
          await transitionFilingStatus(deps, sourceId, "submission_in_progress");
        } else {
          await transitionServiceRequestStatus(deps, sourceId, "submission_in_progress");
        }
      }
    }
  } catch (error) {
    // Don't fail verification if submission creation fails
    deps.logger?.error("Failed to auto-trigger submission after payment", {
      paymentRequestId,
      sourceType,
      sourceId,
      error,
    });
  }

  return {
    paymentRequestId,
    status: "paid_platform_verified",
    verifiedAt: new Date(),
    verifiedByAgentId,
    automaticSubmissionTriggered,
    submissionJobId,
  };
}
```

### Submission Job Creation

```typescript
/**
 * Request submission for a filing or service request.
 *
 * Prerequisites:
 * 1. All required tasks done
 * 2. All required documents uploaded
 * 3. Payment verified (if required)
 *
 * Creates a submission job in the backoffice queue.
 */
async function requestSubmission(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  input: { sourceType: SourceType; sourceId: string }
): Promise<{ ok: boolean; submissionJobId?: string; errors?: string[] }> {
  const orgId = requireOrganizationContext(ctx);

  // 1. Check readiness
  const readiness = await computeSubmissionReadiness(ctx, deps, input);

  if (!readiness.isReady) {
    return {
      ok: false,
      errors: [
        ...readiness.blockers.tasks.map(id => `Task ${id} not complete`),
        ...readiness.blockers.documents.map(key => `Document ${key} missing`),
        readiness.blockers.payment ? `Payment ${readiness.blockers.payment.status}` : null,
      ].filter(Boolean),
    };
  }

  // 2. Load source entity
  const source = await loadSourceEntity(deps, orgId, input.sourceType, input.sourceId);

  // 3. Get regulator key
  const regulatorKey = await getRegulatorKey(deps, source.regulatorId);

  // 4. Create submission job
  const [submissionJob] = await deps.db.insert(submissionJobs).values({
    organizationId: orgId,
    filingId: input.sourceType === "filing" ? input.sourceId : null,
    serviceRequestId: input.sourceType === "service_request" ? input.sourceId : null,
    regulatorKey,
    templateId: source.templateId,
    status: "queued",
    priority: "normal",
    requestedByUserId: ctx.userId,
    requestedAt: new Date(),
  }).returning({ id: submissionJobs.id });

  // 5. Create gate snapshot (for audit/reproducibility)
  await deps.db.insert(submissionGateSnapshots).values({
    submissionJobId: submissionJob.id,
    organizationId: orgId,
    gateSnapshot: readiness.snapshot,
    createdAt: new Date(),
  });

  // 6. Audit log
  await recordAuditLog(ctx, deps, {
    entityType: "submission_job",
    entityId: submissionJob.id,
    action: "create",
    metadata: {
      sourceType: input.sourceType,
      sourceId: input.sourceId,
      regulatorKey,
      status: "queued",
    },
  });

  return {
    ok: true,
    submissionJobId: submissionJob.id,
  };
}
```

---

## Adding New Regulators

### Step-by-Step Guide

#### Step 1: Seed Regulator

```typescript
// packages/database/src/seeds/regulators/new-regulator.ts

export const NEW_REGULATOR = {
  code: "new_regulator",
  name: "New Regulatory Authority",
  shortName: "NRA",
  category: "licensing",
  industry: "Healthcare",
  minimumPlanRequired: "start",
  isPublicAccess: true,
  isActive: true,
  isNational: true,
  config: {
    supportEmail: "support@nra.gov.zm",
    supportPhone: "+260 123 456 789",
  },
};

// Seed function
async function seedNewRegulator(db) {
  const [regulator] = await db.insert(regulators)
    .values(NEW_REGULATOR)
    .returning();

  console.log("✓ Seeded regulator:", regulator.code);
  return regulator;
}
```

#### Step 2: Seed Regulator Fees

```typescript
// packages/database/src/seeds/fees/new-regulator-fees.ts

export const NEW_REGULATOR_FEES = [
  {
    feeKey: "NRA_LICENSE_APPLICATION",
    name: "License Application Fee",
    amount: 150000,  // K1,500 in ngwee
    currency: "ZMW",
    effectiveFrom: new Date("2025-01-01"),
    sourceReference: "NRA Fees Gazette Notice 2025",
  },
  {
    feeKey: "NRA_ANNUAL_RENEWAL",
    name: "Annual License Renewal",
    amount: 100000,  // K1,000 in ngwee
    currency: "ZMW",
    effectiveFrom: new Date("2025-01-01"),
    sourceReference: "NRA Fees Gazette Notice 2025",
  },
  {
    feeKey: "NRA_INSPECTION_FEE",
    name: "Facility Inspection Fee",
    amount: 50000,   // K500 in ngwee
    currency: "ZMW",
    effectiveFrom: new Date("2025-01-01"),
    sourceReference: "NRA Fees Gazette Notice 2025",
  },
];

async function seedNewRegulatorFees(db, regulatorId: string) {
  const fees = NEW_REGULATOR_FEES.map(fee => ({
    ...fee,
    regulatorId,
    id: crypto.randomUUID(),
  }));

  await db.insert(regulatorFees).values(fees);
  console.log(`✓ Seeded ${fees.length} fees for new regulator`);
}
```

#### Step 3: Configure Fee Calculators

```typescript
// packages/payments/calculators/config.ts

// Add fee key mappings
export const FEE_KEY_CALCULATOR_MAP = {
  // ... existing mappings

  // New Regulator - Fixed Fees
  NRA_LICENSE_APPLICATION: "fixed",
  NRA_ANNUAL_RENEWAL: "fixed",
  NRA_INSPECTION_FEE: "fixed",
};
```

#### Step 4: Create Service Templates

```typescript
// packages/database/src/seeds/templates/new-regulator-templates.ts

export const NRA_LICENSE_APPLICATION_TEMPLATE = {
  templateKey: "NRA_LICENSE_APPLICATION_V1",
  templateVersion: 1,
  name: "License Application",
  description: "Apply for a new operating license",
  regulator: "new_regulator",

  intakeFieldsSchema: [
    {
      key: "facilityName",
      label: "Facility Name",
      type: "text",
      required: true,
    },
    {
      key: "facilityAddress",
      label: "Facility Address",
      type: "textarea",
      required: true,
    },
    {
      key: "licenseType",
      label: "License Type",
      type: "select",
      required: true,
      options: [
        { value: "basic", label: "Basic License" },
        { value: "advanced", label: "Advanced License" },
      ],
    },
  ],

  activationRules: {
    requiresConnection: false,  // No pre-connection needed
    minimumPlanTier: "start",
  },

  taskTemplateConfigs: [
    {
      key: "upload_facility_plan",
      title: "Upload Facility Floor Plan",
      taskType: "upload_document",
      required: true,
      sequence: 1,
      actionKind: "doc_upload",
      actionRefTemplate: {
        docRequirementGroup: "facility_plan",
      },
    },
    {
      key: "fill_application_form",
      title: "Complete Application Form",
      taskType: "fill_form",
      required: true,
      sequence: 2,
      actionKind: "form_section",
      actionRefTemplate: {
        formKey: "nra_license_application",
        section: "facility_details",
      },
    },
  ],

  docRequirementConfigs: [
    {
      key: "facility_plan",
      name: "Facility Floor Plan",
      kind: "source",
      required: true,
    },
    {
      key: "business_permit",
      name: "Business Permit",
      kind: "source",
      required: true,
    },
  ],

  paymentRuleConfig: {
    paymentRequired: true,
    feeKey: "NRA_LICENSE_APPLICATION",
  },

  billingTag: "service_fee",
  status: "active",
};

async function seedNewRegulatorTemplates(db, regulatorId: string) {
  const template = {
    ...NRA_LICENSE_APPLICATION_TEMPLATE,
    regulatorId,
    id: crypto.randomUUID(),
  };

  await db.insert(serviceTemplates).values(template);
  console.log("✓ Seeded service template:", template.templateKey);
}
```

#### Step 5: (Optional) Custom Fee Calculator

```typescript
// packages/payments/calculators/custom/nra-license-calculator.ts

/**
 * NRA License Calculator
 *
 * Custom logic: Fee varies by facility size
 * - Small (<50 sqm): K1,000
 * - Medium (50-200 sqm): K2,000
 * - Large (>200 sqm): K3,500
 */
export class NRALicenseCalculator implements FeeCalculator {
  readonly calculatorId = "nra_license_application";
  readonly calculationType = "custom" as const;
  readonly displayName = "NRA License Application Calculator";

  canHandle(context: BaseFeeContext): boolean {
    return context.feeKey === "NRA_LICENSE_APPLICATION";
  }

  validate(context: BaseFeeContext): string[] {
    const errors: string[] = [];

    if (!context.metadata?.facilitySizeSquareMeters) {
      errors.push("facilitySizeSquareMeters is required in metadata");
    }

    return errors;
  }

  async calculate(context: BaseFeeContext): Promise<DetailedFeeBreakdown> {
    const facilitySize = context.metadata.facilitySizeSquareMeters as number;

    let regulatorFee: number;
    let tier: string;

    if (facilitySize < 50) {
      regulatorFee = 100000;  // K1,000
      tier = "small";
    } else if (facilitySize <= 200) {
      regulatorFee = 200000;  // K2,000
      tier = "medium";
    } else {
      regulatorFee = 350000;  // K3,500
      tier = "large";
    }

    const handlingFee = calculateHandlingFee(regulatorFee, context.sourceType);

    return {
      regulatorFee,
      handlingFee,
      totalAmount: regulatorFee + handlingFee,
      currency: "ZMW",
      feeKey: "NRA_LICENSE_APPLICATION",
      feeSource: "calculated",
      calculationType: "custom",
      lineItems: [
        {
          code: "LICENSE_FEE",
          label: `License Fee (${tier} facility)`,
          amount: regulatorFee,
          metadata: { facilitySize, tier },
        },
        {
          code: "HANDLING_FEE",
          label: "Bumara Service Fee",
          amount: handlingFee,
        },
      ],
      handlingFeeInfo: {
        rate: HANDLING_FEE_CONFIG.rates[context.sourceType],
        calculatedFrom: "regulator_fee",
      },
      notes: `Facility size: ${facilitySize} sqm (${tier} tier)`,
    };
  }
}

// Register in registry
registerHandler(new NRALicenseCalculator(deps));
```

#### Step 6: Run Seeds

```bash
# Seed regulator + fees + templates
pnpm --filter @repo/database seed:new-regulator

# Verify
pnpm --filter @repo/database db:studio
```

---

## Best Practices & Improvements

### Current Strengths ✅

1. **Calculator Registry Pattern**: Clean separation of fee logic
2. **Idempotent Payment Creation**: Safe retries
3. **Automatic Submission Trigger**: No manual step after payment
4. **Multi-Tenant Isolation**: Strict org scoping
5. **Audit Trail**: Full payment lifecycle logged
6. **Template-Driven**: No code changes for most services

### Recommended Improvements 🚀

#### 1. Enhanced Payment Gateway Integration

```typescript
// packages/payments/gateways/webhook-handler.ts

/**
 * Webhook Handler for Payment Gateways
 *
 * Handles:
 * - payment.succeeded
 * - payment.failed
 * - payment.refunded
 */
export async function handlePaymentWebhook(
  req: Request,
  provider: "stripe" | "lenco"
): Promise<Response> {
  // 1. Verify webhook signature
  const signature = req.headers.get("x-webhook-signature");
  const isValid = await verifyWebhookSignature(provider, signature, req.body);

  if (!isValid) {
    return new Response("Invalid signature", { status: 401 });
  }

  // 2. Parse event
  const event = await req.json();

  // 3. Handle based on event type
  switch (event.type) {
    case "payment.succeeded": {
      const { paymentId, metadata } = event.data;
      const { paymentRequestId } = metadata;

      // Find payment request
      const payment = await db.query.paymentRequests.findFirst({
        where: eq(paymentRequests.id, paymentRequestId),
      });

      if (!payment) {
        logger.error("Payment request not found for webhook", { paymentRequestId });
        return new Response("Payment not found", { status: 404 });
      }

      // Auto-verify payment
      await handlePaymentVerification(
        { orgId: payment.organizationId, userId: "system" },
        deps,
        {
          paymentRequestId,
          verifiedByAgentId: null,  // System verification
          verificationNotes: `Auto-verified via ${provider} webhook`,
          verificationEvidence: {
            transactionReference: paymentId,
            provider,
          },
        }
      );

      return new Response("OK", { status: 200 });
    }

    case "payment.failed": {
      // Update payment status to failed
      // Notify tenant
      break;
    }

    case "payment.refunded": {
      // Update payment status to refunded
      // Reverse submission if not yet completed
      break;
    }
  }

  return new Response("Event not handled", { status: 200 });
}
```

#### 2. Regulator Payout System

```typescript
// packages/api-services/src/domains/compliance/regulator-payouts.service.ts

/**
 * Create regulator payout batch for a period.
 *
 * Aggregates all verified payments for a regulator in a period
 * and creates a payout record for reconciliation.
 */
export async function createRegulatorPayout(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  input: {
    regulatorId: string;
    periodStart: Date;
    periodEnd: Date;
  }
): Promise<{ payoutId: string; totalAmount: number; paymentCount: number }> {
  const { regulatorId, periodStart, periodEnd } = input;

  // 1. Find all verified payments for regulator in period
  const payments = await deps.db.query.paymentRequests.findMany({
    where: and(
      eq(paymentRequests.status, "paid_platform_verified"),
      gte(paymentRequests.verifiedAt, periodStart),
      lte(paymentRequests.verifiedAt, periodEnd),
      // TODO: Add regulator filter via filing/serviceRequest join
    ),
  });

  // 2. Calculate total regulator fees (exclude Bumara handling fees)
  const totalRegulatorFees = payments.reduce(
    (sum, p) => sum + p.regulatorFee,
    0
  );

  // 3. Create payout record
  const [payout] = await deps.db.insert(regulatorPayouts).values({
    regulatorId,
    amount: totalRegulatorFees,
    currency: "ZMW",
    status: "pending",
    periodStart,
    periodEnd,
    paymentRequestIds: payments.map(p => p.id),
    createdAt: new Date(),
  }).returning();

  // 4. Audit log
  await recordAuditLog(ctx, deps, {
    entityType: "regulator_payout",
    entityId: payout.id,
    action: "create",
    metadata: {
      regulatorId,
      periodStart,
      periodEnd,
      totalAmount: totalRegulatorFees,
      paymentCount: payments.length,
    },
  });

  return {
    payoutId: payout.id,
    totalAmount: totalRegulatorFees,
    paymentCount: payments.length,
  };
}

/**
 * Mark regulator payout as completed.
 *
 * Called after funds are transferred to regulator.
 */
export async function completeRegulatorPayout(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  input: {
    payoutId: string;
    proofDocumentId: string;
    transferReference: string;
  }
): Promise<void> {
  const { payoutId, proofDocumentId, transferReference } = input;

  // Update payout
  await deps.db.update(regulatorPayouts)
    .set({
      status: "completed",
      proofDocumentId,
      processedAt: new Date(),
      metadata: { transferReference },
    })
    .where(eq(regulatorPayouts.id, payoutId));

  // Audit log
  await recordAuditLog(ctx, deps, {
    entityType: "regulator_payout",
    entityId: payoutId,
    action: "complete",
    metadata: { proofDocumentId, transferReference },
  });
}
```

#### 3. Configurable Handling Fee Rules

```typescript
// packages/database/src/schema/core/handling-fee-rules.ts

/**
 * Handling Fee Rules Table
 *
 * Allows per-regulator or per-service handling fee configuration.
 */
export const handlingFeeRules = pgTable("handling_fee_rules", {
  id: uuid("id").primaryKey().defaultRandom(),

  // Scope
  regulatorId: uuid("regulator_id").references(() => regulators.id),
  serviceTemplateId: uuid("service_template_id").references(() => serviceTemplates.id),
  sourceType: text("source_type"),  // "filing" | "service_request" | null (all)

  // Fee configuration
  feeType: text("fee_type").notNull(),  // "percentage" | "fixed" | "none"
  percentageRate: numeric("percentage_rate"),  // e.g., 0.05 for 5%
  fixedAmount: integer("fixed_amount"),        // In minor units
  minimumFee: integer("minimum_fee"),
  maximumFee: integer("maximum_fee"),

  // Priority for conflict resolution (higher = more specific)
  priority: integer("priority").default(0),

  // Active dates
  effectiveFrom: timestamp("effective_from").defaultNow(),
  effectiveUntil: timestamp("effective_until"),

  ...timestamps,
});

// Usage in calculator
async function getHandlingFeeRule(
  deps: ServiceDependencies,
  context: {
    regulatorId: string;
    serviceTemplateId?: string;
    sourceType: SourceType;
  }
): Promise<HandlingFeeRule | null> {
  // Find most specific rule (highest priority)
  const rule = await deps.db.query.handlingFeeRules.findFirst({
    where: and(
      or(
        eq(handlingFeeRules.regulatorId, context.regulatorId),
        eq(handlingFeeRules.serviceTemplateId, context.serviceTemplateId),
        isNull(handlingFeeRules.regulatorId)  // Global rules
      ),
      or(
        eq(handlingFeeRules.sourceType, context.sourceType),
        isNull(handlingFeeRules.sourceType)  // All source types
      ),
      isNull(handlingFeeRules.effectiveUntil)  // Active only
    ),
    orderBy: desc(handlingFeeRules.priority),
  });

  return rule;
}
```

#### 4. Payment Analytics & Reporting

```typescript
// packages/api-services/src/domains/compliance/payment-analytics.service.ts

/**
 * Generate payment analytics for a period.
 */
export async function getPaymentAnalytics(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  input: {
    startDate: Date;
    endDate: Date;
    regulatorId?: string;
  }
): Promise<PaymentAnalyticsReport> {
  const { startDate, endDate, regulatorId } = input;

  // Aggregate queries
  const analytics = await deps.db
    .select({
      regulator: regulators.name,
      totalPayments: count(paymentRequests.id),
      totalRegulatorFees: sum(paymentRequests.regulatorFee),
      totalHandlingFees: sum(paymentRequests.handlingFee),
      totalRevenue: sum(paymentRequests.totalAmount),
      avgPaymentAmount: avg(paymentRequests.totalAmount),
    })
    .from(paymentRequests)
    .leftJoin(filings, eq(paymentRequests.filingId, filings.id))
    .leftJoin(regulators, eq(filings.regulatorId, regulators.id))
    .where(
      and(
        gte(paymentRequests.verifiedAt, startDate),
        lte(paymentRequests.verifiedAt, endDate),
        eq(paymentRequests.status, "paid_platform_verified"),
        regulatorId ? eq(regulators.id, regulatorId) : undefined
      )
    )
    .groupBy(regulators.name);

  return {
    period: { start: startDate, end: endDate },
    summary: {
      totalPayments: analytics.reduce((sum, a) => sum + a.totalPayments, 0),
      totalRegulatorFees: analytics.reduce((sum, a) => sum + a.totalRegulatorFees, 0),
      totalHandlingFees: analytics.reduce((sum, a) => sum + a.totalHandlingFees, 0),
      totalRevenue: analytics.reduce((sum, a) => sum + a.totalRevenue, 0),
    },
    byRegulator: analytics,
  };
}
```

#### 5. Payment State Machine Enforcement

```typescript
// packages/api-services/src/domains/compliance/payment-state-machine.ts

/**
 * Valid payment status transitions.
 */
const VALID_PAYMENT_TRANSITIONS: Record<PaymentRequestStatus, PaymentRequestStatus[]> = {
  required_pending: ["pending_gateway", "paid_platform_verified", "waived", "cancelled"],
  pending_gateway: ["paid_gateway_verified", "paid_platform_verified", "cancelled"],
  paid_gateway_verified: ["paid_platform_verified", "refunded"],
  paid_platform_verified: ["refunded"],
  waived: [],
  refunded: [],
  cancelled: [],
};

/**
 * Transition payment status with validation.
 */
export async function transitionPaymentStatus(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  input: {
    paymentRequestId: string;
    newStatus: PaymentRequestStatus;
    reason?: string;
    metadata?: Record<string, unknown>;
  }
): Promise<void> {
  const { paymentRequestId, newStatus, reason, metadata } = input;

  // 1. Load current payment
  const payment = await deps.db.query.paymentRequests.findFirst({
    where: eq(paymentRequests.id, paymentRequestId),
  });

  if (!payment) {
    throw new ServiceError("NOT_FOUND", "Payment request not found");
  }

  // 2. Validate transition
  const validTransitions = VALID_PAYMENT_TRANSITIONS[payment.status];
  if (!validTransitions.includes(newStatus)) {
    throw new ServiceError(
      "INVALID_TRANSITION",
      `Cannot transition from ${payment.status} to ${newStatus}`
    );
  }

  // 3. Update status
  await deps.db.update(paymentRequests)
    .set({
      status: newStatus,
      updatedAt: new Date(),
    })
    .where(eq(paymentRequests.id, paymentRequestId));

  // 4. Audit log
  await recordAuditLog(ctx, deps, {
    entityType: "payment_request",
    entityId: paymentRequestId,
    action: "update",
    metadata: {
      fromStatus: payment.status,
      toStatus: newStatus,
      reason,
      ...metadata,
    },
  });
}
```

---

## Security & Compliance

### Multi-Tenant Isolation

```typescript
// CRITICAL: Every payment query MUST filter by organizationId

// ✅ CORRECT
const payment = await db.query.paymentRequests.findFirst({
  where: and(
    eq(paymentRequests.id, paymentId),
    eq(paymentRequests.organizationId, orgId)  // ← REQUIRED
  ),
});

// ❌ WRONG - Tenant data leak!
const payment = await db.query.paymentRequests.findFirst({
  where: eq(paymentRequests.id, paymentId),
});
```

### Authorization Checks

```typescript
// Backoffice-only operations
const BACKOFFICE_PAYMENT_ACTIONS = [
  "verify_payment",
  "waive_payment",
  "refund_payment",
  "view_all_payments",
  "create_regulator_payout",
];

async function requireBackofficePermission(
  ctx: ServiceContext,
  action: string
): Promise<void> {
  // Check if user is backoffice staff with appropriate role
  const agent = await loadBackofficeAgent(ctx.userId);

  if (!agent || !agent.permissions.includes(action)) {
    throw new ServiceError("FORBIDDEN", `Insufficient permissions: ${action}`);
  }
}

// Usage
await requireBackofficePermission(ctx, "verify_payment");
await handlePaymentVerification(ctx, deps, input);
```

### PCI Compliance

```typescript
/**
 * PCI DSS Compliance Notes:
 *
 * 1. NEVER store card details in our database
 * 2. Use payment gateway tokens only
 * 3. Log transaction IDs, not card info
 * 4. Encrypt sensitive fields at rest
 * 5. Use HTTPS for all payment endpoints
 */

// ✅ CORRECT - Store gateway reference
await db.insert(paymentRequests).values({
  externalPaymentId: "pi_abc123",  // Gateway transaction ID
  paymentProvider: "stripe",
});

// ❌ WRONG - NEVER store card info
await db.insert(paymentRequests).values({
  cardNumber: "4242424242424242",  // PCI VIOLATION!
  cvv: "123",
});
```

### Audit Logging

```typescript
/**
 * Required audit events for payments:
 */
const PAYMENT_AUDIT_EVENTS = [
  "payment_request.created",
  "payment_request.updated",
  "payment_request.verified",
  "payment_request.waived",
  "payment_request.refunded",
  "payment_request.cancelled",
  "regulator_payout.created",
  "regulator_payout.completed",
];

// All payment operations MUST create audit logs
await recordAuditLog(ctx, deps, {
  entityType: "payment_request",
  entityId: paymentRequestId,
  action: "verify",
  metadata: {
    verifiedByAgentId,
    amount,
    currency,
    fromStatus: "pending_gateway",
    toStatus: "paid_platform_verified",
  },
});
```

---

## Migration & Rollout

### Phase 1: Existing System Stabilization

**Goals:**
- Document current payment flows
- Add missing audit logs
- Strengthen multi-tenant isolation
- Add payment state machine validation

**Tasks:**
- [ ] Audit all payment-related queries for org isolation
- [ ] Add state machine validation to payment updates
- [ ] Add missing audit logs for payment lifecycle events
- [ ] Create payment reconciliation report

### Phase 2: Enhanced Calculator System

**Goals:**
- Migrate all regulators to calculator registry
- Add custom calculators for complex fees
- Centralize handling fee configuration

**Tasks:**
- [ ] Audit all regulator fees in database
- [ ] Create missing fee calculators
- [ ] Add handling fee rules table
- [ ] Migrate hardcoded fee logic to calculators

### Phase 3: Webhook Integration

**Goals:**
- Automatic payment verification
- Eliminate manual verification for gateway payments

**Tasks:**
- [ ] Implement webhook handler for Stripe/Lenco
- [ ] Add webhook signature verification
- [ ] Add retry logic for failed webhooks
- [ ] Monitor webhook success rate

### Phase 4: Regulator Payouts

**Goals:**
- Track Bumara → Regulator fund transfers
- Reconciliation reports
- Proof of payment tracking

**Tasks:**
- [ ] Create regulator_payouts table
- [ ] Build payout batch creation logic
- [ ] Add proof of payment upload
- [ ] Generate reconciliation reports

### Rollout Strategy

```typescript
/**
 * Feature Flag Rollout
 */
const PAYMENT_FEATURE_FLAGS = {
  // Phase 2
  USE_CALCULATOR_REGISTRY: true,
  USE_CUSTOM_CALCULATORS: true,
  USE_HANDLING_FEE_RULES: false,  // Beta testing

  // Phase 3
  AUTO_VERIFY_GATEWAY_PAYMENTS: true,
  REQUIRE_MANUAL_VERIFICATION: false,  // Legacy mode

  // Phase 4
  ENABLE_REGULATOR_PAYOUTS: false,  // Not yet ready
};

// Usage in payment service
if (PAYMENT_FEATURE_FLAGS.AUTO_VERIFY_GATEWAY_PAYMENTS) {
  await handlePaymentVerification(ctx, deps, input);
} else {
  // Legacy: require manual verification
  await createPaymentTicket(ctx, deps, ticketInput);
}
```

---

## Conclusion

### Current System Strengths ✅

1. ✅ **Calculator Registry**: Clean, extensible fee calculation
2. ✅ **Template-Driven**: Add regulators via config, not code
3. ✅ **Automatic Submission**: Payment verification triggers submission
4. ✅ **Multi-Tenant Safe**: Org isolation enforced
5. ✅ **Audit Trail**: Full payment lifecycle logged

### Recommended Next Steps 🚀

**Immediate (Sprint 1-2):**
1. Add payment state machine validation
2. Strengthen multi-tenant isolation checks
3. Add webhook handler for automatic verification

**Short-Term (Sprint 3-4):**
1. Implement regulator payout tracking
2. Add handling fee configuration table
3. Build payment analytics dashboard

**Medium-Term (Quarter 2):**
1. Migrate all regulators to calculator registry
2. Add custom calculators for complex fees
3. Build reconciliation reporting

### Key Takeaways

- **Separation of Concerns**: Tenant payments (IN) ≠ Regulator payouts (OUT)
- **Extensibility**: Add new regulators without code changes
- **Automation**: Payment verification auto-triggers submission
- **Safety**: Multi-tenant isolation + state machine validation
- **Compliance**: Audit logs + PCI compliance + proper authorization

---

**Questions? Issues?**
Contact: @engineering-team or file an issue in `/docs/architecture/issues/`
