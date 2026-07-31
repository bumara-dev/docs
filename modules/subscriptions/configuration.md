---
title: "Subscription Configuration"
description: "This is the source of truth for plan features and caps."
---

## Environment Variables

### Backend (packages/backend)

```env
# Stripe Configuration
STRIPE_SECRET_KEY=sk_live_xxxxx          # Stripe secret key
STRIPE_WEBHOOK_SECRET=whsec_xxxxx        # Stripe webhook signing secret
STRIPE_PUBLISHABLE_KEY=pk_live_xxxxx     # Stripe publishable key (for client)

# Optional: Lenco Configuration
LENCO_API_KEY=xxxxx                       # Lenco API key (if using Lenco)
LENCO_WEBHOOK_SECRET=xxxxx                # Lenco webhook signing secret

# Database
DATABASE_URL=postgresql://...             # PostgreSQL connection string
```

### Frontend (apps/app)

```env
# API URL
NEXT_PUBLIC_API_URL=https://api.bumara.com/api/v1

# Stripe (for Stripe.js if needed)
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_live_xxxxx
```

---

## Plan Configuration

### packages/plans/src/lib/features.ts

This is the source of truth for plan features and caps.

```typescript
const BASE: Record<PlanId, PlanFeatures> = {
  start: {
    caps: {
      activeObligations: 3,
      serviceRequestsPerMonth: 2,
      users: 1,
      storageMb: 500,
      regulators: ["zra", "napsa", "pacra"],
    },
    aiCreditsPerMonth: 0,
    payrollEnabled: false,
    invoicingEnabled: false,
    inventoryEnabled: false,
    documentsExpiryAlerts: false,
    prioritySupport: false,
  },
  plus: {
    caps: {
      activeObligations: 4,
      serviceRequestsPerMonth: 3,
      users: 2,
      storageMb: 1024,
      regulators: ["zra", "napsa", "pacra", "nhima", "zppa"],
    },
    aiCreditsPerMonth: 0,
    payrollEnabled: true,
    invoicingEnabled: false,  // Coming soon
    inventoryEnabled: false,
    documentsExpiryAlerts: true,
    prioritySupport: false,
  },
  pro: {
    caps: {
      activeObligations: 7,
      serviceRequestsPerMonth: 5,
      users: 5,
      storageMb: 5120,
      regulators: ["zra", "napsa", "pacra", "nhima", "zppa", "wcf", "local_councils"],
    },
    aiCreditsPerMonth: 100,
    payrollEnabled: true,
    invoicingEnabled: false,  // Coming soon
    inventoryEnabled: false,  // Coming soon
    documentsExpiryAlerts: true,
    prioritySupport: true,
  },
  enterprise: {
    caps: {
      activeObligations: undefined,  // Unlimited
      serviceRequestsPerMonth: undefined,
      users: undefined,
      storageMb: undefined,
      regulators: undefined,  // All regulators
    },
    aiCreditsPerMonth: undefined,  // Custom
    payrollEnabled: true,
    invoicingEnabled: true,
    inventoryEnabled: true,
    documentsExpiryAlerts: true,
    prioritySupport: true,
  },
};
```

---

## Pricing Configuration

### apps/app/config/pricing.ts

Frontend pricing display configuration.

```typescript
export const TIERS: TierConfig[] = [
  {
    id: "start",
    name: "Start",
    tagline: "Essential compliance for small businesses",
    isNew: true,
    accent: "emerald",
    pricing: {
      currency: "ZMW",
      monthly: 1500,
      yearly: 15_000,
      saveLabel: "Save 17%",
    },
    caps: {
      obligations: 3,
      serviceRequests: 2,
      users: 1,
      storage: "500 MB",
      regulators: CORE_REGULATORS,
    },
    features: [
      "3 obligations",
      "2 service requests per month",
      "Core regulators (ZRA, NAPSA, PACRA)",
      "500 MB document storage",
      "Email deadline reminders",
      "1 team member",
      "Standard support",
    ],
  },
  // ... other tiers
];

export const ADDONS: AddOnConfig[] = [
  {
    id: "payroll",
    title: "Payroll",
    description: "Process payroll with PAYE and NAPSA calculations.",
    price: "ZMW 85/employee",
    availableFor: ["plus", "pro"],
    dynamicPricing: true,
  },
  {
    id: "extra_storage",
    title: "Extra Storage",
    description: "Add 10 GB increments for document storage.",
    price: "ZMW 150/mo per 10 GB",
    availableFor: ["start", "plus", "pro"],
  },
  {
    id: "sms_pack",
    title: "SMS Notifications",
    description: "SMS deadline reminders and alerts for your team.",
    price: "ZMW 100/mo",
    availableFor: ["start", "plus", "pro", "enterprise"],
  },
];
```

---

## Stripe Product Configuration

### Stripe Dashboard Setup

1. **Create Products** for each plan tier:
   - Product name: "Bumara Start", "Bumara Plus", "Bumara Pro"
   - Metadata: `{ "plan_id": "start" }`, etc.

2. **Create Prices** for each product:
   - Monthly recurring price
   - Yearly recurring price
   - Currency: ZMW

3. **Configure Webhooks**:
   - Endpoint: `https://api.bumara.com/webhooks/stripe`
   - Events to listen:
     - `checkout.session.completed`
     - `checkout.session.expired`
     - `customer.subscription.created`
     - `customer.subscription.updated`
     - `customer.subscription.deleted`
     - `invoice.paid`
     - `invoice.payment_failed`

---

## Database Schema Configuration

### Migration

Ensure the subscription tables exist:

```bash
cd packages/database
pnpm db:migrate
```

Or push directly:
```bash
pnpm db:push
```

### Required Tables

1. `subscriptions` - Core subscription data
2. `subscription_usage` - Monthly usage tracking
3. `subscription_audit_log` - Plan change audit trail
4. `subscription_payroll_addon` - Payroll add-on tracking

---

## Feature Flags

### Add-on Availability

| Add-on | Start | Plus | Pro | Enterprise |
|--------|-------|------|-----|------------|
| Payroll | - | Yes | Yes | Yes |
| Smart Invoicing | - | Soon | Soon | Yes |
| Inventory | - | - | Soon | Yes |
| AI Assistance | - | - | Soon | Yes |
| Extra Storage | Yes | Yes | Yes | Custom |
| SMS Pack | Yes | Yes | Yes | Yes |

### Feature Flags by Plan

| Feature | Start | Plus | Pro | Enterprise |
|---------|-------|------|-----|------------|
| Document Expiry Alerts | - | Yes | Yes | Yes |
| Automated Schedules | - | Yes | Yes | Yes |
| Approval Workflows | - | - | Yes | Yes |
| Multi-entity Support | - | - | Yes | Yes |
| Priority Support | - | - | Yes | Yes |
| Dedicated Account Manager | - | - | - | Yes |
| White Label | - | - | - | Yes |
| SSO & Domain Capture | - | - | - | Yes |

---

## Regulator Access

### By Plan Tier

```typescript
export const CORE_REGULATORS = ["ZRA", "NAPSA", "PACRA"];
export const PLUS_REGULATORS = [...CORE_REGULATORS, "NHIMA", "ZPPA"];
export const PRO_REGULATORS = [...PLUS_REGULATORS, "WCF", "LOCAL COUNCILS"];
export const ALL_REGULATORS = PRO_REGULATORS;
```

### Validation

Regulator slugs are lowercase in the database:
- `zra`, `napsa`, `pacra`, `nhima`, `zppa`, `wcf`, `local_councils`

---

## Changing Pricing

To update pricing:

1. **Update Stripe**:
   - Create new prices in Stripe Dashboard
   - Archive old prices (don't delete)

2. **Update Frontend Config**:
   - Edit `apps/app/config/pricing.ts`
   - Update `pricing.monthly` and `pricing.yearly` values

3. **Update Plans Package**:
   - Edit `packages/plans/src/lib/features.ts` if caps change

4. **No backend changes needed** - pricing is handled by Stripe
