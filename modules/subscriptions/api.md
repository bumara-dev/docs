---
title: "Subscriptions API Reference"
description: "All endpoints require authentication and organization context."
---

All endpoints require authentication and organization context.

## Endpoints

### GET /subscriptions/current

Get the current subscription for the organization.

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "organizationId": "org_xxx",
    "planTier": "plus",
    "status": "active",
    "billingPeriod": "monthly",
    "currentPeriodStart": "2026-01-01T00:00:00Z",
    "currentPeriodEnd": "2026-02-01T00:00:00Z",
    "cancelAtPeriodEnd": false,
    "caps": {
      "activeObligations": 4,
      "serviceRequestsPerMonth": 3,
      "users": 2,
      "storageMb": 1024,
      "regulators": ["zra", "napsa", "pacra", "nhima", "zppa"],
      "aiCreditsPerMonth": null
    },
    "addons": [
      {
        "id": "payroll",
        "type": "payroll",
        "quantity": 10,
        "pricePerUnit": 8500
      }
    ]
  }
}
```

**Error Responses:**
- `401 Unauthorized` - Missing or invalid authentication
- `403 Forbidden` - Organization context required
- `500 Internal Server Error` - Failed to fetch subscription

---

### GET /subscriptions/usage

Get current period usage for the organization.

**Response:**
```json
{
  "success": true,
  "data": {
    "periodKey": "2026-01",
    "periodStart": "2026-01-01T00:00:00Z",
    "periodEnd": "2026-02-01T00:00:00Z",
    "serviceRequestsUsed": 2,
    "storageUsedMb": 150,
    "aiCreditsUsed": 0,
    "activeObligations": 3,
    "usersCount": 1
  }
}
```

---

### GET /subscriptions/summary

Get subscription and usage in a single call. Useful for dashboard display.

**Response:**
```json
{
  "success": true,
  "data": {
    "subscription": { /* same as /subscriptions/current */ },
    "usage": { /* same as /subscriptions/usage */ }
  }
}
```

---

### POST /subscriptions/check-limit

Check if a specific action is allowed within subscription limits.

**Request Body:**
```json
{
  "limitType": "service_requests",
  "requestedAmount": 1,
  "regulatorSlug": null
}
```

**Limit Types:**
- `obligations` - Active obligations count
- `service_requests` - Monthly service request count
- `users` - Team member count
- `storage` - Storage in MB
- `ai_credits` - Monthly AI credits
- `regulators` - Access to specific regulator (requires `regulatorSlug`)

**Response:**
```json
{
  "success": true,
  "data": {
    "allowed": false,
    "current": 3,
    "limit": 3,
    "remaining": 0,
    "upgradeRequired": true,
    "suggestedPlan": "plus"
  }
}
```

**Usage Example:**
```typescript
const result = await checkLimit(getToken, {
  limitType: 'service_requests',
  requestedAmount: 1
});

if (!result.allowed) {
  openUpgradePrompt(result.suggestedPlan);
}
```

---

### POST /subscriptions/change-plan

Initiate a plan upgrade or downgrade.

**Request Body:**
```json
{
  "targetPlan": "pro",
  "billingPeriod": "monthly"
}
```

**Target Plans:** `start`, `plus`, `pro`, `enterprise`

**Response (Upgrade - Checkout Required):**
```json
{
  "success": true,
  "data": {
    "status": "checkout_required",
    "checkoutUrl": "https://checkout.stripe.com/...",
    "message": "Please complete checkout to activate your new plan"
  }
}
```

**Response (Downgrade - Scheduled):**
```json
{
  "success": true,
  "data": {
    "status": "scheduled",
    "scheduledFor": "2026-02-01T00:00:00Z",
    "message": "Plan change scheduled for end of billing period"
  }
}
```

**Response (Same Plan):**
```json
{
  "success": true,
  "data": {
    "status": "completed",
    "message": "Billing period updated"
  }
}
```

**Error Responses:**
- `400 Bad Request` - Invalid target plan
- `403 Forbidden` - Plan change not allowed

---

### POST /subscriptions/payroll-addon

Update the payroll add-on employee count.

**Request Body:**
```json
{
  "employeeCount": 25
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "employeeCount": 25,
    "pricePerEmployee": 8500,
    "totalMonthlyPrice": 212500
  }
}
```

**Error Responses:**
- `403 Forbidden` - Payroll add-on not available on current plan (requires Plus or Pro)

---

## Zod Schemas

### Input Schemas

```typescript
// Check limit input
const checkLimitInputSchema = z.object({
  limitType: z.enum([
    'obligations',
    'service_requests',
    'users',
    'storage',
    'ai_credits',
    'regulators',
  ]),
  requestedAmount: z.number().int().positive().optional(),
  regulatorSlug: z.string().optional(),
});

// Change plan input
const changePlanInputSchema = z.object({
  targetPlan: z.enum(['start', 'plus', 'pro', 'enterprise']),
  billingPeriod: z.enum(['monthly', 'yearly']),
});

// Update payroll addon input
const updatePayrollAddonInputSchema = z.object({
  employeeCount: z.number().int().min(0),
});
```

### Response Schemas

```typescript
// Subscription caps
const capsResponseSchema = z.object({
  activeObligations: z.number().nullable(),
  serviceRequestsPerMonth: z.number().nullable(),
  users: z.number().nullable(),
  storageMb: z.number().nullable(),
  regulators: z.array(z.string()),
  aiCreditsPerMonth: z.number().nullable(),
});

// Subscription response
const subscriptionResponseSchema = z.object({
  id: z.string(),
  organizationId: z.string(),
  planTier: z.enum(['start', 'plus', 'pro', 'enterprise']),
  status: z.enum(['active', 'trialing', 'past_due', 'canceled', 'unpaid']),
  billingPeriod: z.enum(['monthly', 'yearly']),
  currentPeriodStart: z.string().nullable(),
  currentPeriodEnd: z.string().nullable(),
  cancelAtPeriodEnd: z.boolean(),
  caps: capsResponseSchema,
  addons: z.array(z.object({
    id: z.string(),
    type: z.string(),
    quantity: z.number(),
    pricePerUnit: z.number(),
  })),
});

// Usage response
const usageResponseSchema = z.object({
  periodKey: z.string(),
  periodStart: z.string(),
  periodEnd: z.string(),
  serviceRequestsUsed: z.number(),
  storageUsedMb: z.number(),
  aiCreditsUsed: z.number(),
  activeObligations: z.number(),
  usersCount: z.number(),
});

// Check limit response
const checkLimitResponseSchema = z.object({
  allowed: z.boolean(),
  current: z.number(),
  limit: z.number().nullable(),
  remaining: z.number().nullable(),
  upgradeRequired: z.boolean(),
  suggestedPlan: z.enum(['start', 'plus', 'pro', 'enterprise']).nullable(),
});

// Change plan response
const changePlanResponseSchema = z.object({
  status: z.enum(['checkout_required', 'scheduled', 'completed']),
  checkoutUrl: z.string().optional(),
  scheduledFor: z.string().optional(),
  message: z.string(),
});
```

## Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `UNAUTHORIZED` | 401 | Missing or invalid authentication |
| `FORBIDDEN` | 403 | Organization context required or limit exceeded |
| `BAD_REQUEST` | 400 | Invalid request body |
| `INTERNAL_ERROR` | 500 | Server error |

## Rate Limiting

All subscription endpoints are protected by rate limiting:
- Standard rate limit applies (see API documentation)
- Webhook endpoints have separate limits
