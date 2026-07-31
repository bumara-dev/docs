---
title: "Payments Module"
description: "Provider-agnostic payment architecture for Bumara platform."
---

## Overview

The payments module provides a unified interface for handling all payment operations on the Bumara platform. It supports multiple payment providers (Stripe, Lenco, etc.) through a gateway abstraction layer, allowing easy provider switching without modifying business logic.

## Key Features

- **Provider Agnostic**: Switch payment providers by changing a single environment variable
- **Two Payment Types**: Platform subscriptions (recurring) and compliance payments (one-time)
- **Automatic Verification**: Webhook-based payment verification for compliance payments
- **Mock Provider**: Full mock implementation for testing
- **Idempotency**: Built-in support for preventing duplicate payments
- **Multi-tenant**: Full tenant isolation with organization-scoped operations

## Documentation Index

| Document | Description |
|----------|-------------|
| [Architecture](/modules/payments/architecture) | System design, provider pattern, data flow |
| [Fee Calculators](/modules/payments/fee-calculators) | Fee calculation system, calculator types, handling fees |
| [Data Model](/modules/payments/data-model) | Database schema, tables, relationships |
| [API Reference](/modules/payments/api) | Service functions, gateway methods |
| [Webhooks](/modules/payments/webhooks) | Webhook handling, event types |
| [Configuration](/modules/payments/configuration) | Environment variables, provider setup |
| [Testing](/modules/payments/testing) | Mock provider, test strategies |
| [Runbook](/modules/payments/runbook) | Operations, troubleshooting, monitoring |

## Quick Start

### 1. Configure Provider

```env
# .env
PAYMENT_PROVIDER=stripe  # or 'lenco', 'mock'
PAYMENT_API_KEY=sk_test_...
PAYMENT_WEBHOOK_SECRET=whsec_...
```

### 2. Use the Gateway

```typescript
import { getPaymentGateway } from "@repo/payments";

// Get provider instance (singleton)
const gateway = await getPaymentGateway();

// Create subscription checkout
const session = await gateway.createSubscriptionCheckout({
  organizationId: "org_123",
  userId: "user_456",
  planId: "plus",
  billingPeriod: "monthly",
  successUrl: "https://app.bumara.com/billing/success",
  cancelUrl: "https://app.bumara.com/billing/cancel",
});

// Redirect user to session.url
```

### 3. Handle Webhooks

Webhooks are handled by the backend at provider-specific endpoints:

```
POST /webhooks/payments/stripe   - Stripe events
POST /webhooks/payments/lenco    - Lenco events
```

Each handler:
- Verifies signature using the gateway's `parseWebhookEvent`
- Parses events to normalized `WebhookEvent` format
- Routes to common processor (`processPaymentWebhookEvent`)
- Updates database records (subscriptions, payment_requests)

See [Webhooks](/modules/payments/webhooks) for detailed documentation.

## Payment Types

### Platform Subscriptions

Recurring payments for Bumara platform access.

| Plan | Features | Billing |
|------|----------|---------|
| Start | Basic compliance | Monthly/Yearly |
| Plus | Multi-regulator | Monthly/Yearly |
| Pro | Full automation | Monthly/Yearly |
| Enterprise | Custom | Custom |

### Compliance Payments

One-time payments for regulator filing and service fees.

```
Tenant initiates filing/service
    ↓
Calculate fees (regulator + handling)
    ↓
Create payment request
    ↓
gateway.createPaymentSession()
    ↓
Tenant pays via checkout
    ↓
Webhook: payment.succeeded
    ↓
Auto-verify payment
    ↓
Trigger submission
```

## Package Structure

```
packages/payments/
├── index.ts                 # Public exports
├── types.ts                 # Provider-agnostic types
├── errors.ts                # Payment error types
├── config.ts                # Environment configuration
├── legacy.ts                # Backwards compatibility
│
├── calculators/             # Fee calculation system
│   ├── index.ts             # Calculator exports
│   ├── types.ts             # Calculator types
│   ├── config.ts            # Fee configuration, handling rates
│   ├── registry.ts          # Calculator registry
│   ├── errors.ts            # Calculation errors
│   ├── fixed-fee.ts         # Fixed fee calculator
│   ├── contribution-based.ts # NAPSA/NHIMA calculator
│   ├── progressive-tax.ts   # ZRA PAYE calculator
│   └── pre-calculated.ts    # Pre-calculated fees
│
├── gateway/
│   ├── interface.ts         # PaymentGateway interface
│   ├── factory.ts           # Provider factory
│   └── mock.ts              # Mock provider
│
├── providers/
│   ├── stripe/
│   │   ├── index.ts         # Stripe adapter
│   │   └── mappers.ts       # Type mappers
│   └── lenco/
│       └── index.ts         # Lenco adapter (stub)
│
└── utils/
    ├── currency.ts          # Currency helpers
    └── idempotency.ts       # Idempotency keys
```

## Related Documentation

- Subscription Plans
- Compliance Workflow
- Audit Logging
