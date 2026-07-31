---
title: "Frontend Integration"
description: "The frontend provides React hooks for subscription management, connected to a pricing UI and billing management page."
---

## Overview

The frontend provides React hooks for subscription management, connected to a pricing UI and billing management page.

## React Query Hooks

**Location:** `apps/app/lib/queries/subscriptions/hooks/use-subscription.ts`

### useSubscription

Fetch the current subscription.

```typescript
import { useSubscription } from '@/lib/queries/subscriptions';

function MyComponent() {
  const { data: subscription, isLoading, error } = useSubscription();

  if (isLoading) return <Spinner />;
  if (error) return <Error message={error.message} />;

  return (
    <div>
      <p>Plan: {subscription.planTier}</p>
      <p>Status: {subscription.status}</p>
    </div>
  );
}
```

**Options:**
- Stale time: 5 minutes
- Automatically refetches on window focus

---

### useUsage

Fetch current period usage.

```typescript
import { useUsage } from '@/lib/queries/subscriptions';

function UsageDisplay() {
  const { data: usage, isLoading } = useUsage();

  return (
    <div>
      <p>Service Requests: {usage.serviceRequestsUsed}</p>
      <p>Storage: {usage.storageUsedMb} MB</p>
    </div>
  );
}
```

**Options:**
- Stale time: 1 minute (usage changes more frequently)

---

### useSubscriptionWithUsage

Fetch both subscription and usage in one call. Ideal for dashboards.

```typescript
import { useSubscriptionWithUsage } from '@/lib/queries/subscriptions';

function Dashboard() {
  const { data, isLoading } = useSubscriptionWithUsage();

  if (!data) return null;

  const { subscription, usage } = data;

  return (
    <div>
      <PlanCard plan={subscription.planTier} />
      <UsageBar
        label="Service Requests"
        used={usage.serviceRequestsUsed}
        limit={subscription.caps.serviceRequestsPerMonth}
      />
    </div>
  );
}
```

---

### useCheckLimit

Check if an action is allowed before attempting it.

```typescript
import { useCheckLimit } from '@/lib/queries/subscriptions';

function CreateServiceRequestButton() {
  const { data: limitCheck } = useCheckLimit({
    limitType: 'service_requests',
    requestedAmount: 1,
  });

  if (!limitCheck?.allowed) {
    return (
      <Button disabled>
        Limit Reached - Upgrade to {limitCheck?.suggestedPlan}
      </Button>
    );
  }

  return <Button onClick={handleCreate}>Create Request</Button>;
}
```

---

### useChangePlan

Mutation to initiate a plan change.

```typescript
import { useChangePlan } from '@/lib/queries/subscriptions';

function UpgradeButton({ targetPlan }) {
  const changePlan = useChangePlan();

  const handleUpgrade = async () => {
    try {
      const result = await changePlan.mutateAsync({
        targetPlan,
        billingPeriod: 'monthly',
      });

      if (result.status === 'checkout_required') {
        // Redirect to Stripe checkout
        window.location.href = result.checkoutUrl;
      } else if (result.status === 'scheduled') {
        toast.success(`Plan change scheduled for ${result.scheduledFor}`);
      }
    } catch (error) {
      toast.error('Failed to change plan');
    }
  };

  return (
    <Button
      onClick={handleUpgrade}
      disabled={changePlan.isPending}
    >
      {changePlan.isPending ? <Spinner /> : 'Upgrade'}
    </Button>
  );
}
```

---

### useUpdatePayrollAddon

Mutation to update payroll add-on employee count.

```typescript
import { useUpdatePayrollAddon } from '@/lib/queries/subscriptions';

function PayrollAddonSettings() {
  const updatePayroll = useUpdatePayrollAddon();
  const [count, setCount] = useState(10);

  const handleSave = async () => {
    const result = await updatePayroll.mutateAsync({ employeeCount: count });
    toast.success(`Updated to ${result.employeeCount} employees`);
  };

  return (
    <div>
      <input
        type="number"
        value={count}
        onChange={(e) => setCount(parseInt(e.target.value))}
      />
      <p>Monthly: ZMW {(count * 85).toLocaleString()}</p>
      <Button onClick={handleSave}>Save</Button>
    </div>
  );
}
```

---

### useRemainingCapacity

Computed hook for displaying remaining capacity.

```typescript
import { useRemainingCapacity } from '@/lib/queries/subscriptions';

function CapacityOverview() {
  const { remaining, isLoading } = useRemainingCapacity();

  if (isLoading || !remaining) return <Spinner />;

  return (
    <div>
      <CapacityBar label="Obligations" remaining={remaining.obligations} />
      <CapacityBar label="Service Requests" remaining={remaining.serviceRequests} />
      <CapacityBar label="Storage" remaining={remaining.storageMb} suffix="MB" />
    </div>
  );
}
```

---

## Components

### PricingSection

Full pricing page with plan cards and comparison table.

**Location:** `apps/app/components/shared/pricing/index.tsx`

```typescript
import { PricingSection } from '@/components/shared/pricing';

function PricingPage() {
  return (
    <div className="container">
      <PricingSection />
    </div>
  );
}
```

**Features:**
- Monthly/yearly billing toggle
- Current plan highlight
- Plan change integration
- Feature comparison table

---

### UpgradeSheet

Modal for upgrade prompts when limits are hit.

**Location:** `apps/app/components/shared/upgrade/upgrade-guard.tsx`

```typescript
import { UpgradeSheet, openUpgradeForCap } from '@/components/shared/upgrade';

// In your app layout
function Layout({ children }) {
  return (
    <>
      {children}
      <UpgradeSheet />
    </>
  );
}

// Trigger upgrade prompt when limit hit
function handleLimitExceeded() {
  openUpgradeForCap({
    plan: 'start',
    capName: 'serviceRequestsPerMonth',
    current: 2,
    cap: 2,
  });
}
```

---

### Billing Page

Subscription management page.

**Location:** `apps/app/app/(authenticated)/(general)/billing/page.tsx`

**Features:**
- Current plan display
- Billing period info
- Usage progress bars
- Regulator access list
- Change plan button

---

## Query Keys

For cache invalidation:

```typescript
import { subscriptionQueryKeys } from '@/lib/queries/subscriptions';

// Invalidate all subscription data
queryClient.invalidateQueries({ queryKey: subscriptionQueryKeys.all });

// Invalidate specific queries
queryClient.invalidateQueries({ queryKey: subscriptionQueryKeys.current() });
queryClient.invalidateQueries({ queryKey: subscriptionQueryKeys.usage() });
```

---

## Configuration

### Pricing Config

**Location:** `apps/app/config/pricing.ts`

```typescript
export const TIERS: TierConfig[] = [
  {
    id: "start",
    name: "Start",
    tagline: "Essential compliance for small businesses",
    pricing: {
      currency: "ZMW",
      monthly: 1500,
      yearly: 15_000,
    },
    caps: {
      obligations: 3,
      serviceRequests: 2,
      users: 1,
      storage: "500 MB",
      regulators: ["ZRA", "NAPSA", "PACRA"],
    },
    // ...
  },
  // ... other tiers
];

export const ADDONS: AddOnConfig[] = [
  {
    id: "payroll",
    title: "Payroll",
    price: "ZMW 85/employee",
    availableFor: ["plus", "pro"],
  },
  // ... other addons
];
```

---

## Error Handling

Handle enforcement errors gracefully:

```typescript
import { toast } from '@repo/design-system/utils/sonner';
import { openUpgradeForCap } from '@/components/shared/upgrade';

async function handleServiceRequestCreate(input) {
  try {
    await createServiceRequest(input);
    toast.success('Service request created');
  } catch (error) {
    // Check if it's a limit error
    if (error.message?.includes('limit')) {
      openUpgradeForCap({
        plan: 'start',  // Current plan
        capName: 'serviceRequestsPerMonth',
        current: 2,
        cap: 2,
      });
    } else {
      toast.error(error.message);
    }
  }
}
```

---

## Best Practices

1. **Prefetch subscription data** - Load on app mount for instant access
2. **Show loading states** - Use skeletons while fetching
3. **Handle errors gracefully** - Show upgrade prompts for limit errors
4. **Cache appropriately** - Usage data needs shorter stale times than subscription data
5. **Invalidate on mutations** - Clear cache after plan changes
