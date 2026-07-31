---
title: "Subscription Enforcement"
description: "Enforcement helpers ensure users cannot exceed their subscription limits. All enforcement happens server-side to prevent bypassing via client manipulation."
---

## Overview

Enforcement helpers ensure users cannot exceed their subscription limits. All enforcement happens server-side to prevent bypassing via client manipulation.

## Enforcement Functions

All enforcement functions are in `packages/api-services/src/domains/subscriptions/enforcement.ts`.

### enforceObligationLimit

Checks if the organization can create more active obligations.

```typescript
import { enforceObligationLimit } from '@repo/api-services';

// In your service function
export async function activateObligation(ctx, deps, input) {
  // Count current active obligations first
  const currentCount = await countActiveObligations(deps.db, ctx.organizationId);

  // Enforce the limit
  await enforceObligationLimit(ctx, deps, currentCount);

  // Proceed with activation...
}
```

**Throws:** `ServiceError` with code `'FORBIDDEN'` if limit exceeded.

---

### enforceServiceRequestLimit

Checks if the organization can create more service requests this month.

```typescript
import { enforceServiceRequestLimit, recordUsage } from '@repo/api-services';

export async function createServiceRequest(ctx, deps, input) {
  // Enforce BEFORE creation
  await enforceServiceRequestLimit(ctx, deps);

  // Create the service request...
  const request = await insertServiceRequest(deps.db, ...);

  // Record usage AFTER success
  await recordUsage(ctx, deps, { usageType: 'service_requests', amount: 1 });

  return request;
}
```

---

### enforceUserLimit

Checks if the organization can add more team members.

```typescript
import { enforceUserLimit } from '@repo/api-services';

export async function inviteTeamMember(ctx, deps, input) {
  await enforceUserLimit(ctx, deps);

  // Proceed with invitation...
}
```

**Note:** Team members are managed via Clerk webhooks, so this is primarily used for validation in the invite flow.

---

### enforceStorageLimit

Checks if the organization has sufficient storage capacity.

```typescript
import { enforceStorageLimit, recordUsage } from '@repo/api-services';

export async function uploadDocument(ctx, deps, input) {
  const fileSizeMb = Math.ceil(input.sizeBytes / 1_048_576);

  // Enforce BEFORE upload
  await enforceStorageLimit(ctx, deps, fileSizeMb);

  // Upload the document...
  const doc = await storeDocument(...);

  // Record usage AFTER success
  await recordUsage(ctx, deps, { usageType: 'storage', amount: fileSizeMb });

  return doc;
}
```

---

### enforceRegulatorAccess

Checks if the organization's plan includes access to a specific regulator.

```typescript
import { enforceRegulatorAccess } from '@repo/api-services';

export async function activateRegulator(ctx, deps, regulatorKey) {
  // Enforce regulator access based on plan
  await enforceRegulatorAccess(ctx, deps, regulatorKey.toLowerCase());

  // Proceed with activation...
}
```

**Regulator Access by Plan:**

| Plan | Regulators |
|------|------------|
| Start | zra, napsa, pacra |
| Plus | + nhima, zppa |
| Pro | + wcf, local councils |
| Enterprise | All |

---

### enforceAICreditsLimit

Checks if the organization has sufficient AI credits for an operation.

```typescript
import { enforceAICreditsLimit, recordUsage } from '@repo/api-services';

export async function processWithAI(ctx, deps, input) {
  const creditsNeeded = calculateCredits(input);

  // Enforce BEFORE processing
  await enforceAICreditsLimit(ctx, deps, creditsNeeded);

  // Process with AI...
  const result = await aiService.process(input);

  // Record usage AFTER success
  await recordUsage(ctx, deps, { usageType: 'ai_credits', amount: creditsNeeded });

  return result;
}
```

---

## Non-Throwing Check

For UI purposes (showing upgrade prompts), use `checkLimit`:

```typescript
import { checkLimit } from '@repo/api-services';

export async function getCapacityInfo(ctx, deps) {
  const result = await checkLimit(ctx, deps, {
    limitType: 'service_requests',
    requestedAmount: 1,
  });

  return {
    canCreate: result.allowed,
    remaining: result.remaining,
    upgradeRequired: result.upgradeRequired,
    suggestedPlan: result.suggestedPlan,
  };
}
```

## Integration Points

The enforcement functions are integrated into these services:

### Service Requests

**File:** `packages/api-services/src/domains/compliance/service-requests.service.ts`

```typescript
export async function createServiceRequest(ctx, deps, input) {
  // Enforce service request limit before creation
  await enforceServiceRequestLimit(ctx, deps);

  // ... create service request ...

  // Record usage after successful creation
  await recordUsage(ctx, deps, { usageType: "service_requests", amount: 1 });
}
```

### Regulator Activation

**File:** `packages/api-services/src/domains/activation/activation.service.ts`

```typescript
export async function activateRegulatorTemplates(ctx, deps, input) {
  // Enforce regulator access based on plan
  await enforceRegulatorAccess(ctx, deps, regulatorKey.toLowerCase());

  // Count current active obligations
  const currentActiveCount = await countActiveObligations(...);

  // Enforce obligation limit if activating new obligations
  if (activatableObligations.length > 0) {
    await enforceObligationLimit(ctx, deps, currentActiveCount);
  }

  // ... proceed with activation ...
}
```

### Document Upload

**File:** `packages/api-services/src/domains/compliance/documents.service.ts`

```typescript
export async function completeDocumentUpload(ctx, deps, input) {
  // Calculate file size in MB
  const fileSizeMb = Math.ceil(input.sizeBytes / 1_048_576);

  // Enforce storage limit
  await enforceStorageLimit(ctx, deps, fileSizeMb);

  // ... complete upload ...

  // Record storage usage
  await recordUsage(ctx, deps, { usageType: "storage", amount: fileSizeMb });
}
```

## Error Handling

When a limit is exceeded, enforcement functions throw a `ServiceError`:

```typescript
throw new ServiceError({
  code: 'FORBIDDEN',
  message: `Service request limit reached. You've used 3/3 service requests this month.`,
  details: {
    limitType: 'service_requests',
    current: 3,
    limit: 3,
    suggestedPlan: 'plus',
  },
});
```

The frontend catches this error and can display an upgrade prompt:

```typescript
try {
  await createServiceRequest(input);
} catch (error) {
  if (error.code === 'FORBIDDEN' && error.details?.suggestedPlan) {
    openUpgradePrompt(error.details.suggestedPlan);
  } else {
    showErrorToast(error.message);
  }
}
```

## Best Practices

1. **Always enforce before the action** - Never record usage before the action succeeds
2. **Record usage after success** - Don't record if the operation fails
3. **Use transactions** - Wrap enforcement + action in a transaction where possible
4. **Include context in errors** - Provide helpful error messages with current/limit values
5. **Log enforcement failures** - Track when users hit limits for analytics
