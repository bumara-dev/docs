---
title: "Payments Data Model"
description: "The payments module uses two primary database tables:"
---

## Overview

The payments module uses two primary database tables:
- `subscriptions` - Platform subscription records
- `payment_requests` - Compliance/one-time payment records

Both tables include provider-agnostic external ID columns for tracking payment gateway references.

## Entity Relationship Diagram

```
┌─────────────────────────┐       ┌─────────────────────────┐
│     organizations       │       │       regulators        │
│─────────────────────────│       │─────────────────────────│
│ id (PK)                 │       │ id (PK)                 │
│ name                    │       │ name                    │
│ ...                     │       │ ...                     │
└───────────┬─────────────┘       └───────────┬─────────────┘
            │                                  │
            │ 1:1                              │
            ▼                                  │
┌─────────────────────────┐                   │
│     subscriptions       │                   │
│─────────────────────────│                   │
│ id (PK)                 │                   │
│ organizationId (FK)     │                   │
│ plan                    │                   │
│ status                  │                   │
│ externalCustomerId      │                   │
│ externalSubscriptionId  │                   │
│ paymentProvider         │                   │
│ ...                     │                   │
└─────────────────────────┘                   │
                                              │
┌─────────────────────────┐       ┌───────────┴─────────────┐
│        filings          │       │    regulator_fees       │
│─────────────────────────│       │─────────────────────────│
│ id (PK)                 │       │ id (PK)                 │
│ organizationId (FK)     │       │ regulatorId (FK)        │
│ ...                     │       │ feeKey                  │
└───────────┬─────────────┘       │ amount                  │
            │                     │ ...                     │
            │                     └─────────────────────────┘
            │ 1:1
            ▼
┌─────────────────────────┐
│   payment_requests      │
│─────────────────────────│
│ id (PK)                 │
│ organizationId (FK)     │
│ filingId (FK)           │◄───── or ─────┐
│ serviceRequestId (FK)   │               │
│ status                  │               │
│ regulatorFee            │       ┌───────┴─────────────┐
│ handlingFee             │       │  service_requests   │
│ totalAmount             │       │─────────────────────│
│ currency                │       │ id (PK)             │
│ externalSessionId       │       │ organizationId (FK) │
│ externalPaymentId       │       │ ...                 │
│ paymentProvider         │       └─────────────────────┘
│ ...                     │
└─────────────────────────┘
```

## Tables

### subscriptions

Platform subscription records for organization billing.

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `organizationId` | TEXT | FK to organizations |
| `plan` | ENUM | Plan tier: start, plus, pro, enterprise |
| `planFeatures` | JSONB | Feature limits and toggles |
| `status` | ENUM | trial, active, past_due, cancelled, etc. |
| `trialStartsAt` | TIMESTAMP | Trial period start |
| `trialEndsAt` | TIMESTAMP | Trial period end |
| `aiEnabled` | BOOLEAN | AI features enabled |
| `addOns` | JSONB | Purchased add-ons |
| `currentPeriodStart` | TIMESTAMP | Billing period start |
| `currentPeriodEnd` | TIMESTAMP | Billing period end |
| `externalCustomerId` | TEXT | Provider's customer ID |
| `externalSubscriptionId` | TEXT | Provider's subscription ID |
| `paymentProvider` | TEXT | Provider name: stripe, lenco |
| `createdAt` | TIMESTAMP | Record creation time |
| `updatedAt` | TIMESTAMP | Last update time |

**Indexes:**
- `idx_subscriptions_by_org` on `organizationId`
- `idx_subscriptions_by_status` on `status`
- `idx_subscriptions_by_plan_status` on `plan, status`
- `idx_subscriptions_external_customer` on `externalCustomerId`
- `idx_subscriptions_external_subscription` on `externalSubscriptionId`

**Status Values:**
```typescript
type SubscriptionStatus =
  | 'trial'           // In trial period
  | 'active'          // Paid and active
  | 'past_due'        // Payment failed, grace period
  | 'cancelled'       // User cancelled
  | 'incomplete'      // Initial payment pending
  | 'incomplete_expired'; // Initial payment failed
```

### payment_requests

Compliance payment records for filing and service request fees.

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `organizationId` | TEXT | FK to organizations |
| `filingId` | UUID | FK to filings (nullable) |
| `serviceRequestId` | UUID | FK to service_requests (nullable) |
| `status` | ENUM | Payment request status |
| `currency` | TEXT | ISO 4217 currency code |
| `regulatorFee` | INTEGER | Regulator fee in minor units |
| `handlingFee` | INTEGER | Handling fee in minor units |
| `totalAmount` | INTEGER | Total amount in minor units |
| `requestedAt` | TIMESTAMP | When payment was requested |
| `paidAt` | TIMESTAMP | When payment was made |
| `verifiedAt` | TIMESTAMP | When payment was verified |
| `verifiedByAgentId` | UUID | FK to back_office_agents (manual verification) |
| `externalSessionId` | TEXT | Gateway checkout session ID |
| `externalPaymentId` | TEXT | Gateway payment/transaction ID |
| `paymentProvider` | TEXT | Provider name: stripe, lenco |
| `createdAt` | TIMESTAMP | Record creation time |
| `updatedAt` | TIMESTAMP | Last update time |

**Indexes:**
- `idx_payment_requests_org_status` on `organizationId, status`
- `idx_payment_requests_external_session` on `externalSessionId`
- `idx_payment_requests_external_payment` on `externalPaymentId`

**Constraints:**
- `chk_payment_amounts_positive`: All amounts >= 0
- `chk_payment_total_equals_sum`: totalAmount = regulatorFee + handlingFee

**Status Values:**
```typescript
type PaymentRequestStatus =
  | 'not_required'           // No payment needed
  | 'required_pending'       // Payment needed, not started
  | 'pending_gateway'        // Checkout session created
  | 'paid_pending_verify'    // Paid, awaiting verification (manual)
  | 'paid_platform_verified' // Verified via webhook (automatic)
  | 'paid_regulator_pending' // Paid to Bumara, pending regulator payment
  | 'completed'              // Fully completed
  | 'refunded'               // Payment refunded
  | 'waived';                // Payment requirement waived
```

## Amount Storage

All monetary amounts are stored in **minor units** (smallest currency denomination):

| Currency | Major Unit | Minor Unit | Example |
|----------|------------|------------|---------|
| ZMW | Kwacha | Ngwee | K250.00 = 25000 |
| USD | Dollar | Cents | $25.00 = 2500 |

**Conversion Utilities:**

```typescript
import { toMinorUnits, toMajorUnits, formatCurrency } from "@repo/payments";

// Convert for storage
const minorUnits = toMinorUnits(250.00, 'ZMW'); // 25000

// Convert for display
const majorUnits = toMajorUnits(25000, 'ZMW'); // 250.00

// Format for UI
const formatted = formatCurrency(25000, 'ZMW'); // 'K250.00'
```

## External ID Strategy

External IDs enable bi-directional lookup between Bumara and payment providers:

### Subscription External IDs

| Column | Source | Example |
|--------|--------|---------|
| `externalCustomerId` | Provider customer ID | `cus_ABC123` (Stripe) |
| `externalSubscriptionId` | Provider subscription ID | `sub_XYZ789` (Stripe) |
| `paymentProvider` | Provider name | `stripe` |

**Usage:**
```typescript
// Lookup by external ID (webhook processing)
const subscription = await db.query.subscriptions.findFirst({
  where: eq(subscriptions.externalSubscriptionId, 'sub_XYZ789')
});

// Lookup by org (normal operations)
const subscription = await db.query.subscriptions.findFirst({
  where: eq(subscriptions.organizationId, orgId)
});
```

### Payment Request External IDs

| Column | Source | Example |
|--------|--------|---------|
| `externalSessionId` | Checkout session ID | `cs_ABC123` (Stripe) |
| `externalPaymentId` | Payment/transaction ID | `pi_XYZ789` (Stripe) |
| `paymentProvider` | Provider name | `stripe` |

**Usage:**
```typescript
// On payment initiation
await db.update(paymentRequests).set({
  status: 'pending_gateway',
  externalSessionId: session.id,
  paymentProvider: gateway.providerName,
});

// On webhook (payment succeeded)
await db.update(paymentRequests).set({
  status: 'paid_platform_verified',
  externalPaymentId: event.data.paymentId,
  paidAt: now,
  verifiedAt: now,
});
```

## Migration SQL

```sql
-- Add columns to subscriptions table
ALTER TABLE subscriptions
ADD COLUMN external_customer_id TEXT,
ADD COLUMN external_subscription_id TEXT,
ADD COLUMN payment_provider TEXT;

-- Add columns to payment_requests table
ALTER TABLE payment_requests
ADD COLUMN external_session_id TEXT,
ADD COLUMN external_payment_id TEXT,
ADD COLUMN payment_provider TEXT;

-- Create indexes for webhook lookups
CREATE INDEX idx_subscriptions_external_customer
  ON subscriptions(external_customer_id);
CREATE INDEX idx_subscriptions_external_subscription
  ON subscriptions(external_subscription_id);
CREATE INDEX idx_payment_requests_external_session
  ON payment_requests(external_session_id);
CREATE INDEX idx_payment_requests_external_payment
  ON payment_requests(external_payment_id);
```

## TypeScript Types

### Subscription Type

```typescript
type Subscription = {
  id: string;
  organizationId: string;
  plan: 'start' | 'plus' | 'pro' | 'enterprise';
  planFeatures: {
    maxUsers?: number;
    maxTasks?: number;
    maxRegulators?: number;
    maxDocumentsStorage?: number;
    supportLevel?: 'basic' | 'priority' | 'dedicated';
  };
  status: SubscriptionStatus;
  trialStartsAt: Date | null;
  trialEndsAt: Date | null;
  aiEnabled: boolean;
  addOns: AddOn[];
  currentPeriodStart: Date | null;
  currentPeriodEnd: Date | null;
  externalCustomerId: string | null;
  externalSubscriptionId: string | null;
  paymentProvider: string | null;
  createdAt: Date;
  updatedAt: Date;
};
```

### Payment Request Type

```typescript
type PaymentRequest = {
  id: string;
  organizationId: string;
  filingId: string | null;
  serviceRequestId: string | null;
  status: PaymentRequestStatus;
  currency: string;
  regulatorFee: number;      // Minor units
  handlingFee: number;       // Minor units
  totalAmount: number;       // Minor units
  requestedAt: Date | null;
  paidAt: Date | null;
  verifiedAt: Date | null;
  verifiedByAgentId: string | null;
  externalSessionId: string | null;
  externalPaymentId: string | null;
  paymentProvider: string | null;
  createdAt: Date;
  updatedAt: Date;
};
```

## Query Examples

### Get organization's active subscription

```typescript
const subscription = await db.query.subscriptions.findFirst({
  where: and(
    eq(subscriptions.organizationId, orgId),
    eq(subscriptions.status, 'active')
  ),
});
```

### Get pending payment requests for a filing

```typescript
const payments = await db.query.paymentRequests.findMany({
  where: and(
    eq(paymentRequests.filingId, filingId),
    eq(paymentRequests.status, 'required_pending')
  ),
});
```

### Update payment on webhook

```typescript
await db
  .update(paymentRequests)
  .set({
    status: 'paid_platform_verified',
    paidAt: new Date(),
    verifiedAt: new Date(),
    externalPaymentId: event.data.paymentId,
    paymentProvider: event.provider,
    updatedAt: new Date(),
  })
  .where(eq(paymentRequests.id, paymentRequestId));
```
