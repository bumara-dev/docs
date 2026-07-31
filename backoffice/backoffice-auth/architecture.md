---
title: "Security Architecture"
description: "Defense-in-depth security model for the Bumara backoffice."
---

The backoffice implements multiple security layers to ensure that only authorized Bumara staff can access internal operations.

---

## Security Layers Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Browser Request                                │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ LAYER 1: Next.js Middleware (proxy.ts)                                   │
│ ├── Clerk Authentication (JWT validation)                               │
│ ├── Email Domain Validation (@bumara.com, @bumara.co.zm)                │
│ ├── Organization Membership Check (CLERK_INTERNAL_ORG_ID)               │
│ └── Backoffice Role Validation                                          │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ LAYER 2: Server Component Guards                                         │
│ ├── AuthGuard → Ensures user is authenticated                           │
│ ├── ServerOrganizationGuard → Ensures active org context                │
│ ├── BackofficeOrgGuard → Verifies org === CLERK_INTERNAL_ORG_ID         │
│ └── MfaGuard → Requires twoFactorEnabled === true                       │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ LAYER 3: Hono API Middleware Chain                                       │
│ ├── clerkAuth → Validates JWT token                                     │
│ ├── attachAuthToContext → Extracts user/org from claims                 │
│ ├── requireAuth → Verifies user exists in database                      │
│ ├── requireBackofficeOrg → Validates org === internal org               │
│ ├── requireActiveStaff → Checks back_office_agents.is_active            │
│ └── requireBackoffice → Validates backoffice role                       │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ LAYER 4: Business Logic                                                  │
│ ├── Permission checks (requirePermission, requireRole)                  │
│ └── Audit logging (actorType: 'STAFF')                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Layer 1: Next.js Middleware

**File:** `apps/backoffice/proxy.ts`

The middleware runs on every request before reaching the application. It performs:

### 1.1 Route Classification

```typescript
// Public routes (no auth required)
const isPublicRoute = createRouteMatcher([
  "/sign-in(.*)",
  "/access-denied(.*)",
]);

// Routes exempt from org context
const isOrgSelectRoute = createRouteMatcher([
  "/org-select(.*)",
  "/sign-in(.*)",
  "/access-denied(.*)",
  "/setup-mfa(.*)",
]);

// Routes that bypass backoffice checks
const isBackofficeCheckExempt = createRouteMatcher([
  "/api(.*)",
  "/_next(.*)",
  "/sign-in(.*)",
  "/access-denied(.*)",
  "/org-select(.*)",
  "/setup-mfa(.*)",
]);
```

### 1.2 Access Check Flow

The `checkBackofficeAccess()` function validates:

1. **Email Domain** - Primary email must end with allowed domain
2. **Organization Membership** - User must be member of internal org
3. **Role Assignment** - User must have a valid backoffice role

```typescript
const accessCheck = await checkBackofficeAccess(userId, {
  internalOrgId: env.CLERK_INTERNAL_ORG_ID,
  allowedDomains: ["@bumara.com", "@bumara.co.zm"],
});

if (!accessCheck.hasAccess) {
  // Redirect to /access-denied with reason
}
```

### 1.3 Access Denial Reasons

| Reason | Description | Resolution |
|--------|-------------|------------|
| `invalid_domain` | Email not from company domain | Use company email |
| `not_in_organization` | Not member of backoffice org | Request invitation |
| `wrong_organization` | Active org is not backoffice | Switch organization |
| `no_role_assigned` | No role in backoffice org | Admin assigns role |

---

## Layer 2: Server Component Guards

**File:** `apps/backoffice/app/(authenticated)/layout.tsx`

Server components wrap the authenticated layout with security guards:

```tsx
const AppLayout = ({ children }: AppLayoutProperties) => (
  <AuthGuard>
    <ServerOrganizationGuard>
      <BackofficeOrgGuard>
        <MfaGuard>
          <AuthenticatedLayout>{children}</AuthenticatedLayout>
        </MfaGuard>
      </BackofficeOrgGuard>
    </ServerOrganizationGuard>
  </AuthGuard>
);
```

### Guard Components

| Guard | Purpose | Redirect |
|-------|---------|----------|
| `AuthGuard` | Ensures user is signed in | `/sign-in` |
| `ServerOrganizationGuard` | Ensures active organization | `/org-select` |
| `BackofficeOrgGuard` | Verifies backoffice org membership | `/access-denied` |
| `MfaGuard` | Requires MFA enabled | `/setup-mfa` |

### MFA Guard Details

The `MfaGuard` enforces mandatory two-factor authentication:

```tsx
export async function MfaGuard({ children, redirectTo = "/setup-mfa" }) {
  // Skip if MFA not required by environment
  if (!env.REQUIRE_MFA) {
    return <>{children}</>;
  }

  // Exempt certain paths to prevent redirect loops
  const MFA_EXEMPT_PATHS = [
    "/setup-mfa",
    "/sign-in",
    "/sign-out",
    "/access-denied",
  ];

  const user = await currentUser();
  
  if (!user?.twoFactorEnabled) {
    redirect(redirectTo);
  }

  return <>{children}</>;
}
```

---

## Layer 3: API Middleware Chain

**File:** `packages/backend/src/core/middleware/auth.ts`

The Hono backend applies a middleware chain to all backoffice API routes:

### Standard Backoffice Chain

```typescript
middleware: [
  clerkAuth,              // Validate JWT
  attachAuthToContext,    // Extract user/org claims
  requireAuth,            // Verify user in database
  requireBackofficeOrg,   // Verify internal org
  requireActiveStaff,     // Check is_active flag
  requireBackoffice,      // Validate backoffice role
]
```

### Middleware Reference

| Middleware | Purpose | Error Code |
|------------|---------|------------|
| `clerkAuth` | Validates Clerk JWT token | 401 |
| `attachAuthToContext` | Extracts userId, orgId, orgRole from claims | - |
| `requireAuth` | Verifies user exists in `users` table | 401/404 |
| `requireOrg` | Verifies org exists in `organizations` table | 403/404 |
| `requireBackofficeOrg` | Validates `orgId === CLERK_INTERNAL_ORG_ID` | 403 |
| `requireActiveStaff` | Checks `back_office_agents.is_active = true` | 403 |
| `requireBackoffice` | Validates backoffice role | 403 |
| `requireBackofficeApprover` | Requires admin or manager role | 403 |
| `requirePermission(perm)` | Checks specific permission | 403 |

### requireBackofficeOrg Implementation

```typescript
export const requireBackofficeOrg = createMiddleware<AppBindings>(
  async (c, next) => {
    const orgId = c.get("orgId");
    const backofficeOrgId = process.env.CLERK_INTERNAL_ORG_ID;

    if (orgId !== backofficeOrgId) {
      return c.json({
        success: false,
        error: "Forbidden",
        message: "Access denied. Backoffice organization membership required.",
      }, 403);
    }

    await next();
  }
);
```

### requireActiveStaff Implementation

This provides instant revocation capability:

```typescript
export const requireActiveStaff = createMiddleware<AppBindings>(
  async (c, next) => {
    const userId = c.get("userId");

    const [staffRecord] = await db
      .select({ id: backOfficeAgents.id, isActive: backOfficeAgents.isActive })
      .from(backOfficeAgents)
      .where(eq(backOfficeAgents.clerkUserId, userId))
      .limit(1);

    if (!staffRecord) {
      return c.json({ message: "Staff registration required." }, 403);
    }

    if (!staffRecord.isActive) {
      return c.json({ message: "Your staff account has been deactivated." }, 403);
    }

    await next();
  }
);
```

---

## Layer 4: Audit Logging

All sensitive backoffice actions are logged with actor type:

```typescript
await recordAuditLog(ctx, deps, {
  action: 'approve',
  entityType: 'payout',
  entityId: payoutId,
  actorType: 'STAFF',  // Distinguishes from TENANT actions
  changes: {
    before: { status: 'pending' },
    after: { status: 'approved' },
  },
});
```

### Audit Log Schema

| Field | Description |
|-------|-------------|
| `org_id` | Organization context |
| `actor_id` | User who performed action |
| `actor_type` | 'STAFF', 'TENANT', or 'SYSTEM' |
| `resource_type` | Entity type (filing, payout, etc.) |
| `resource_id` | Entity ID |
| `from_status` | Previous status |
| `to_status` | New status |
| `reason` | Optional reason for change |
| `metadata` | Additional context (JSON) |
| `created_at` | Timestamp |

---

## Security Features Summary

| Feature | Implementation | Purpose |
|---------|---------------|---------|
| No public sign-up | Sign-up route removed | Invite-only access |
| Domain validation | `checkBackofficeAccess()` | Only company emails |
| Org verification | `requireBackofficeOrg` | Prevent cross-realm access |
| Staff gate | `requireActiveStaff` | Instant revocation capability |
| Mandatory MFA | `MfaGuard` + `/setup-mfa` | Second factor required |
| CORS hardening | Origin allowlist | Prevent XSS/CSRF |
| Audit logging | `actorType: 'STAFF'` | Forensic traceability |

---

## Route Layout Structure

```
apps/backoffice/app/
├── (unauthenticated)/           # No auth required
│   ├── sign-in/                 # Custom sign-in form
│   ├── org-select/              # Organization selector
│   └── access-denied/           # Access denied page
│
├── (mfa-setup)/                 # Auth required, MFA exempt
│   └── setup-mfa/               # MFA setup wizard
│
└── (authenticated)/             # Full security chain
    └── (home)/                  # Dashboard and features
        ├── layout.tsx           # Applies all guards
        └── ...
```

---

## Next Steps

- [Authentication Flow](/backoffice/backoffice-auth/authentication-flow) - Detailed sign-in and MFA process
- [Authorization](/backoffice/backoffice-auth/authorization) - Role and permission details
- [Development Guide](/backoffice/backoffice-auth/development-guide) - How to protect new routes

