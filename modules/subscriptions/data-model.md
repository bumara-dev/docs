---
title: "Subscriptions Data Model"
description: "Main subscription table linking organizations to their plan."
---

## Database Tables

### subscriptions

Main subscription table linking organizations to their plan.

```sql
CREATE TABLE "subscriptions" (
  "id" uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  "organization_id" text NOT NULL UNIQUE REFERENCES "organizations"("id"),
  "plan_tier" text NOT NULL DEFAULT 'start',  -- 'start', 'plus', 'pro', 'enterprise'
  "status" text NOT NULL DEFAULT 'active',     -- 'active', 'trialing', 'past_due', 'canceled', 'unpaid'
  "billing_period" text NOT NULL DEFAULT 'monthly',  -- 'monthly', 'yearly'
  "current_period_start" timestamp,
  "current_period_end" timestamp,
  "cancel_at_period_end" boolean DEFAULT false,
  "external_subscription_id" text,             -- Stripe subscription ID
  "external_customer_id" text,                 -- Stripe customer ID
  "payment_provider" text,                     -- 'stripe', 'lenco'
  "created_at" timestamp DEFAULT now(),
  "updated_at" timestamp DEFAULT now()
);
```

**Indexes:**
- `idx_subscriptions_org` on `organization_id`
- `idx_subscriptions_external_sub` on `external_subscription_id`

### subscription_usage

Monthly usage tracking per organization.

```sql
CREATE TABLE "subscription_usage" (
  "id" uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  "organization_id" text NOT NULL REFERENCES "organizations"("id") ON DELETE CASCADE,
  "period_key" text NOT NULL,                  -- 'YYYY-MM' format (e.g., '2026-01')
  "period_start" timestamp NOT NULL,
  "period_end" timestamp NOT NULL,
  "service_requests_used" integer NOT NULL DEFAULT 0,
  "storage_used_mb" integer NOT NULL DEFAULT 0,
  "ai_credits_used" integer NOT NULL DEFAULT 0,
  "created_at" timestamp DEFAULT now(),
  "updated_at" timestamp DEFAULT now(),
  CONSTRAINT "uniq_subscription_usage_org_period" UNIQUE("organization_id", "period_key")
);
```

**Indexes:**
- `idx_subscription_usage_org_period` on `(organization_id, period_key)`

### subscription_audit_log

Audit trail for all subscription changes.

```sql
CREATE TABLE "subscription_audit_log" (
  "id" uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  "organization_id" text NOT NULL REFERENCES "organizations"("id") ON DELETE CASCADE,
  "action" text NOT NULL,                      -- 'plan_change_initiated', 'plan_upgraded', etc.
  "before_state" jsonb,                        -- Previous state
  "after_state" jsonb,                         -- New state
  "changed_by_user_id" text,                   -- Who made the change
  "reason" text,                               -- Optional reason
  "created_at" timestamp DEFAULT now() NOT NULL
);
```

**Indexes:**
- `idx_subscription_audit_org` on `organization_id`
- `idx_subscription_audit_org_created` on `(organization_id, created_at)`
- `idx_subscription_audit_action` on `action`

### subscription_payroll_addon

Payroll add-on tracking for per-employee pricing.

```sql
CREATE TABLE "subscription_payroll_addon" (
  "id" uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  "organization_id" text NOT NULL REFERENCES "organizations"("id") ON DELETE CASCADE,
  "employee_count" integer NOT NULL DEFAULT 0,
  "price_per_employee_cents" integer NOT NULL DEFAULT 8500,  -- ZMW 85.00
  "created_at" timestamp DEFAULT now(),
  "updated_at" timestamp DEFAULT now(),
  CONSTRAINT "uniq_subscription_payroll_addon_org" UNIQUE("organization_id")
);
```

**Indexes:**
- `idx_subscription_payroll_addon_org` on `organization_id`

## TypeScript Schema (Drizzle)

### subscriptions.ts

```typescript
// packages/database/src/schema/core/subscriptions.ts

import { pgTable, text, timestamp, uuid, boolean } from "drizzle-orm/pg-core";
import { organizations } from "./organizations";

export const subscriptionStatusEnum = [
  "active",
  "trialing",
  "past_due",
  "canceled",
  "unpaid",
] as const;

export const planTierEnum = ["start", "plus", "pro", "enterprise"] as const;
export const billingPeriodEnum = ["monthly", "yearly"] as const;

export const subscriptions = pgTable("subscriptions", {
  id: uuid("id").primaryKey().defaultRandom(),
  organizationId: text("organization_id")
    .notNull()
    .unique()
    .references(() => organizations.id),
  planTier: text("plan_tier").notNull().default("start"),
  status: text("status").notNull().default("active"),
  billingPeriod: text("billing_period").notNull().default("monthly"),
  currentPeriodStart: timestamp("current_period_start"),
  currentPeriodEnd: timestamp("current_period_end"),
  cancelAtPeriodEnd: boolean("cancel_at_period_end").default(false),
  externalSubscriptionId: text("external_subscription_id"),
  externalCustomerId: text("external_customer_id"),
  paymentProvider: text("payment_provider"),
  createdAt: timestamp("created_at").defaultNow(),
  updatedAt: timestamp("updated_at").defaultNow(),
});
```

### subscription-usage.ts

```typescript
// packages/database/src/schema/billing/subscription-usage.ts

import { pgTable, text, timestamp, uuid, integer, unique } from "drizzle-orm/pg-core";
import { organizations } from "../core/organizations";

export const subscriptionUsage = pgTable(
  "subscription_usage",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    organizationId: text("organization_id")
      .notNull()
      .references(() => organizations.id, { onDelete: "cascade" }),
    periodKey: text("period_key").notNull(),
    periodStart: timestamp("period_start").notNull(),
    periodEnd: timestamp("period_end").notNull(),
    serviceRequestsUsed: integer("service_requests_used").notNull().default(0),
    storageUsedMb: integer("storage_used_mb").notNull().default(0),
    aiCreditsUsed: integer("ai_credits_used").notNull().default(0),
    createdAt: timestamp("created_at").defaultNow(),
    updatedAt: timestamp("updated_at").defaultNow(),
  },
  (table) => ({
    uniqOrgPeriod: unique("uniq_subscription_usage_org_period").on(
      table.organizationId,
      table.periodKey
    ),
  })
);
```

## Relationships

```
organizations (1) ──── (1) subscriptions
     │
     └──── (N) subscription_usage
     │
     └──── (N) subscription_audit_log
     │
     └──── (0..1) subscription_payroll_addon
```

## Usage Tracking Types

| Type | Reset Frequency | Description |
|------|-----------------|-------------|
| `service_requests` | Monthly | Number of service requests created |
| `storage` | Never (cumulative) | Total storage used in MB |
| `ai_credits` | Monthly | AI credits consumed |

**Note:** Obligations are tracked by counting active records, not in the usage table.

## Audit Log Actions

| Action | Description |
|--------|-------------|
| `plan_change_initiated` | User started a plan change |
| `plan_upgraded` | Plan was upgraded (after payment) |
| `plan_downgraded` | Plan was downgraded |
| `subscription_activated` | Subscription activated after checkout |
| `subscription_canceled` | Subscription canceled |
| `subscription_renewed` | Subscription renewed for new period |
| `addon_added` | Add-on was added |
| `addon_removed` | Add-on was removed |
| `payroll_addon_updated` | Payroll employee count changed |

## Migration

Migration file: `packages/database/drizzle/0032_subscription_usage_tracking.sql`

Run migration:
```bash
cd packages/database
pnpm db:migrate
```

Or push directly:
```bash
pnpm db:push
```
