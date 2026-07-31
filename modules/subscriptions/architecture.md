---
title: "Subscriptions Architecture"
---

## System Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              Frontend (app)                              │
├─────────────────────────────────────────────────────────────────────────┤
│  Pricing UI  │  Billing Page  │  Upgrade Guard  │  Usage Display        │
│     ↓              ↓                ↓                  ↓                │
│  useChangePlan  useSubscription  useCheckLimit  useRemainingCapacity    │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           Backend API (Hono)                             │
├─────────────────────────────────────────────────────────────────────────┤
│  /subscriptions/current      GET   - Get current subscription           │
│  /subscriptions/usage        GET   - Get current period usage           │
│  /subscriptions/summary      GET   - Get subscription + usage           │
│  /subscriptions/check-limit  POST  - Check if action is allowed         │
│  /subscriptions/change-plan  POST  - Initiate plan upgrade/downgrade    │
│  /subscriptions/payroll-addon POST - Update payroll add-on              │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        Service Layer (api-services)                      │
├─────────────────────────────────────────────────────────────────────────┤
│  subscriptions.service.ts    - Business logic                           │
│  enforcement.ts              - Limit enforcement helpers                 │
│  subscriptions.schema.ts     - Zod validation schemas                   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Repository Layer (database)                      │
├─────────────────────────────────────────────────────────────────────────┤
│  subscriptions.ts            - Database queries                          │
│  Schema: subscriptions, subscription_usage, subscription_audit_log      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            External Services                             │
├─────────────────────────────────────────────────────────────────────────┤
│  Stripe                      - Payment processing & webhooks            │
│  Lenco (optional)            - Alternative payment provider             │
└─────────────────────────────────────────────────────────────────────────┘
```

## Package Structure

### packages/plans

Defines plan features and caps shared across frontend and backend.

```
packages/plans/
├── src/
│   ├── types/
│   │   └── plans.ts           # PlanId, PlanFeatures types
│   └── lib/
│       └── features.ts        # BASE config, featuresFor(), computeFlags()
└── package.json
```

**Key exports:**
- `PlanId` - Union type: `'start' | 'plus' | 'pro' | 'enterprise'`
- `PlanFeatures` - Interface with caps and feature flags
- `featuresFor(planId, addons)` - Compute features for a plan
- `computeFlags(features)` - Derive boolean flags from features

### packages/database

Database schema and repository functions.

```
packages/database/
├── src/
│   ├── schema/
│   │   ├── core/
│   │   │   └── subscriptions.ts         # subscriptions table
│   │   └── billing/
│   │       ├── subscription-usage.ts    # subscription_usage table
│   │       └── subscription-audit-log.ts # audit log table
│   └── repositories/
│       └── subscriptions.ts             # Query functions
└── drizzle/
    └── 0032_subscription_usage_tracking.sql  # Migration
```

### packages/api-services

Business logic and enforcement.

```
packages/api-services/
└── src/
    └── domains/
        └── subscriptions/
            ├── subscriptions.service.ts  # Core service functions
            ├── subscriptions.schema.ts   # Zod schemas
            └── enforcement.ts            # Limit enforcement
```

### packages/backend

HTTP routes and handlers.

```
packages/backend/
└── src/
    └── modules/
        ├── subscriptions/
        │   ├── index.ts                  # Router setup
        │   ├── subscriptions.routes.ts   # OpenAPI route definitions
        │   └── subscriptions.handlers.ts # Request handlers
        └── webhooks/
            └── payments/
                └── payments-webhook.processor.ts  # Stripe webhook handling
```

### apps/app (Frontend)

React components and hooks.

```
apps/app/
├── config/
│   └── pricing.ts                        # TIERS, ADDONS, COMPARISON_DATA
├── lib/
│   └── queries/
│       └── subscriptions/
│           ├── types.ts                  # TypeScript types
│           ├── fetchers/
│           │   └── subscriptions.ts      # API fetch functions
│           ├── hooks/
│           │   └── use-subscription.ts   # React Query hooks
│           └── index.ts
├── components/
│   └── shared/
│       ├── pricing/
│       │   └── index.tsx                 # PricingSection component
│       └── upgrade/
│           └── upgrade-guard.tsx         # UpgradeSheet component
└── app/
    └── (authenticated)/
        └── (general)/
            ├── pricing/
            │   └── page.tsx              # Pricing page
            └── billing/
                └── page.tsx              # Billing management page
```

## Data Flow

### 1. Checking Subscription Limits

```
User Action → useCheckLimit() → GET /subscriptions/check-limit
                                         ↓
                              checkSubscriptionLimit()
                                         ↓
                              findSubscriptionByOrganizationId()
                                         ↓
                              getOrCreateUsage()
                                         ↓
                              Compare current vs caps
                                         ↓
                              Return { allowed, remaining, ... }
```

### 2. Enforcing Limits (Server-side)

```
Service Request Creation → enforceServiceRequestLimit()
                                    ↓
                          Check usage < cap
                                    ↓
                          If exceeded: throw ServiceError('FORBIDDEN')
                                    ↓
                          Create service request
                                    ↓
                          recordUsage('service_requests', 1)
```

### 3. Plan Change Flow

```
User clicks "Upgrade" → useChangePlan() → POST /subscriptions/change-plan
                                                    ↓
                                          changePlan()
                                                    ↓
                                          Create Stripe checkout session
                                                    ↓
                                          Return { checkoutUrl }
                                                    ↓
                                          Redirect to Stripe
                                                    ↓
                                          User completes payment
                                                    ↓
                                          Stripe webhook → handleCheckoutCompleted()
                                                    ↓
                                          Update subscription status
```

## Multi-Tenant Isolation

All queries are scoped to `organizationId`:

```typescript
// Repository layer
export async function findSubscriptionByOrganizationId(
  db: DatabaseClient,
  organizationId: string
) {
  return db
    .select()
    .from(subscriptions)
    .where(eq(subscriptions.organizationId, organizationId))  // Always scoped
    .limit(1);
}
```

## Key Design Decisions

### 1. Server-side Enforcement

All limit checks happen on the server, never trusting client-side validation.

```typescript
// In service-requests.service.ts
export async function createServiceRequest(ctx, deps, input) {
  // Enforce BEFORE creation
  await enforceServiceRequestLimit(ctx, deps);

  // ... create request ...

  // Record usage AFTER success
  await recordUsage(ctx, deps, { usageType: 'service_requests', amount: 1 });
}
```

### 2. Non-throwing Limit Checks for UI

The `checkLimit()` function returns a result object instead of throwing, allowing the UI to show upgrade prompts gracefully.

```typescript
const result = await checkLimit(ctx, deps, { limitType: 'obligations' });
// result: { allowed: false, current: 3, limit: 3, upgradeRequired: true, suggestedPlan: 'plus' }
```

### 3. Monthly Usage Reset

Service requests and AI credits reset monthly. Storage is cumulative. The reset happens via webhook when Stripe renews the subscription.

### 4. Audit Trail

All plan changes are logged to `subscription_audit_log` for compliance:

```typescript
await createSubscriptionAuditLog(db, {
  organizationId,
  action: 'plan_change_initiated',
  beforeState: { plan: currentPlan },
  afterState: { targetPlan, billingPeriod },
  changedByUserId: userId,
});
```
