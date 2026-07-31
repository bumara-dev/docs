---
title: "Bumara Backoffice Authentication System"
description: "Comprehensive documentation of the backoffice authentication, authorization, and security architecture."
---

## Table of Contents

1. [Overview](#1-overview)
2. [Authentication Provider (Clerk)](#2-authentication-provider-clerk)
3. [Authentication Flow](#3-authentication-flow)
4. [Authorization & RBAC](#4-authorization--rbac)
5. [Permission System](#5-permission-system)
6. [Middleware Stack](#6-middleware-stack)
7. [Frontend Route Protection](#7-frontend-route-protection)
8. [Multi-Tenancy & Isolation](#8-multi-tenancy--isolation)
9. [Session Management](#9-session-management)
10. [Multi-Factor Authentication (MFA)](#10-multi-factor-authentication-mfa)
11. [Security Features](#11-security-features)
12. [Audit Logging](#12-audit-logging)
13. [Staff Management](#13-staff-management)
14. [Environment Configuration](#14-environment-configuration)
15. [Suggestions for Improvements](#15-suggestions-for-improvements)
16. [Suggestions for Additions](#16-suggestions-for-additions)

---

## 1. Overview

The Bumara backoffice uses a **multi-layered defense-in-depth authentication system** built on **Clerk** as the identity provider. The system enforces:

- **Separation of Concerns**: Backoffice staff operate in a dedicated internal organization, completely isolated from tenant organizations
- **RBAC**: Fine-grained roles and permissions at both role and feature levels
- **Mandatory MFA**: All staff must use two-factor authentication
- **Comprehensive Auditing**: Every sensitive action is logged with actor context
- **Domain Validation**: Only company email addresses can access backoffice

### Key Files

| Component | Location |
|-----------|----------|
| Backend Auth Middleware | `packages/backend/src/core/middleware/auth.ts` |
| Rate Limiting | `packages/backend/src/core/middleware/rate-limit.ts` |
| Auth Helpers | `packages/auth/helpers.ts` |
| Frontend Layout Guards | `apps/backoffice/app/(authenticated)/layout.tsx` |
| Sign-In Form | `apps/backoffice/components/auth/sign-in-form.tsx` |
| MFA Guard | `apps/backoffice/components/shared/auth/mfa-guard.tsx` |
| Backoffice Org Guard | `apps/backoffice/components/shared/organization/backoffice-org-guard.tsx` |
| Audit Logs Schema | `packages/database/src/schema/system/audit-logs.ts` |
| Staff Schema | `packages/database/src/schema/core/back-office-agents.ts` |

---

## 2. Authentication Provider (Clerk)

The system uses **Clerk** as the third-party authentication provider, handling:

- User identity management
- Password policies & hashing
- Session management
- JWT token issuance
- Organization management
- MFA/TOTP management

### Environment Variables

```env
# Server-side
CLERK_SECRET_KEY=sk_live_xxx
CLERK_INTERNAL_ORG_ID=org_xxx           # Dedicated backoffice organization
CLERK_ALLOWED_BACKOFFICE_DOMAINS="@bumara.com,@bumara.co.zm"

# Client-side
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_live_xxx
NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-up

# MFA Enforcement
REQUIRE_MFA=true
```

### Clerk SDK Integration

```typescript
// Server-side (packages/auth/server.ts)
import { auth, currentUser } from "@clerk/nextjs/server";

// Client-side (packages/auth/client.ts)
import { useSignIn, useUser, useOrganization } from "@clerk/nextjs";

// Backend (Hono)
import { clerkMiddleware, getAuth } from "@hono/clerk-auth";
```

---

## 3. Authentication Flow

### 3.1 Sign-In Process (3-Step Flow)

Located in: `apps/backoffice/components/auth/sign-in-form.tsx`

```
┌─────────────────────────────────────────────────────────────────┐
│                     SIGN-IN FLOW                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STEP 1: IDENTIFIER                                            │
│  ┌─────────────────────────────────────────────────────┐       │
│  │ User enters email address                            │       │
│  │ → signIn.create({ identifier })                      │       │
│  │ → If valid: status = "needs_first_factor"            │       │
│  └─────────────────────────────────────────────────────┘       │
│                          ↓                                      │
│  STEP 2: PASSWORD                                               │
│  ┌─────────────────────────────────────────────────────┐       │
│  │ User enters password                                 │       │
│  │ → signIn.attemptFirstFactor({ strategy: "password" })│       │
│  │ → If MFA disabled: status = "complete" → Dashboard   │       │
│  │ → If MFA enabled: status = "needs_second_factor"     │       │
│  └─────────────────────────────────────────────────────┘       │
│                          ↓                                      │
│  STEP 3: TWO-FACTOR AUTHENTICATION                              │
│  ┌─────────────────────────────────────────────────────┐       │
│  │ User enters 6-digit TOTP code (or backup code)       │       │
│  │ → signIn.attemptSecondFactor({ strategy: "totp" })   │       │
│  │ → Auto-submits when 6 digits entered                 │       │
│  │ → On success: setActive({ session }) → Dashboard     │       │
│  └─────────────────────────────────────────────────────┘       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Organization Selection

After authentication, users select their organization:

```typescript
// apps/backoffice/components/auth/org-selector.tsx
const { setActive } = useOrganizationList();

// Switch to selected organization
await setActive({ organization: organizationId });
```

---

## 4. Authorization & RBAC

### 4.1 Role Definitions

Located in: `packages/backend/src/core/middleware/auth.ts`

```typescript
export const ClerkOrgRoles = {
  // Standard organization roles
  ADMIN: "org:admin",           // Full access (owner/creator)
  MANAGER: "org:manager",       // Manages submissions/approvals
  MEMBER: "org:member",         // Basic access

  // Backoffice-specific roles
  BACKOFFICE_ADMIN: "org:backoffice_admin",     // Full backoffice access
  BACKOFFICE_MANAGER: "org:backoffice_manager", // Manage/approve tasks
  BACKOFFICE_MEMBER: "org:backoffice_member",   // Review/submit only
} as const;
```

### 4.2 Role Hierarchy

```
Level 3 (Highest): org:admin, org:backoffice_admin
Level 2 (Middle):  org:manager, org:backoffice_manager
Level 1 (Lowest):  org:member, org:backoffice_member
```

### 4.3 Role Groups

```typescript
export const RoleGroups = {
  ADMINS: ["org:admin", "org:backoffice_admin"],
  MANAGERS: ["org:admin", "org:manager", "org:backoffice_admin", "org:backoffice_manager"],
  BACKOFFICE: [...all_backoffice_roles, ...standard_roles],
  ORGANIZATION: ["org:admin", "org:manager", "org:member"],
} as const;
```

### 4.4 Role Hierarchy Check

```typescript
// packages/auth/helpers.ts
export function hasRoleOrHigher(
  userRole: BackofficeRole | null | undefined,
  requiredRole: BackofficeRole
): boolean {
  const roleHierarchy: Record<BackofficeRole, number> = {
    "org:admin": 3,
    "org:backoffice_admin": 3,
    "org:manager": 2,
    "org:backoffice_manager": 2,
    "org:member": 1,
    "org:backoffice_member": 1,
  };
  return roleHierarchy[userRole] >= roleHierarchy[requiredRole];
}
```

---

## 5. Permission System

### 5.1 Feature-Based Permissions

Format: `org:feature:action`

```typescript
export const Permissions = {
  // Compliance
  COMPLIANCE_READ: "org:compliance:read",
  COMPLIANCE_CREATE: "org:compliance:create",
  COMPLIANCE_SUBMIT: "org:compliance:submit",
  COMPLIANCE_APPROVE: "org:compliance:approve",
  COMPLIANCE_MANAGE: "org:compliance:manage",

  // Payroll
  PAYROLL_READ: "org:payroll:read",
  PAYROLL_MANAGE: "org:payroll:manage",
  PAYROLL_RUN: "org:payroll:run",
  PAYROLL_APPROVE: "org:payroll:approve",

  // Invoice
  INVOICE_READ: "org:invoice:read",
  INVOICE_CREATE: "org:invoice:create",
  INVOICE_SEND: "org:invoice:send",
  INVOICE_MANAGE: "org:invoice:manage",

  // Documents
  DOCUMENTS_READ: "org:documents:read",
  DOCUMENTS_UPLOAD: "org:documents:upload",
  DOCUMENTS_DELETE: "org:documents:delete",
  DOCUMENTS_MANAGE: "org:documents:manage",

  // AI
  AI_USE: "org:ai:use",
  AI_INSIGHTS: "org:ai:insights",

  // Payments
  PAYMENTS_READ: "org:payments:read",
  PAYMENTS_CREATE: "org:payments:create",
  PAYMENTS_PROCESS: "org:payments:process",
  PAYMENTS_APPROVE: "org:payments:approve",
  PAYMENTS_REFUND: "org:payments:refund",
} as const;
```

### 5.2 Permission Groups

```typescript
export const PermissionGroups = {
  VIEWER: [COMPLIANCE_READ, PAYROLL_READ, INVOICE_READ, DOCUMENTS_READ, PAYMENTS_READ],
  APPROVER: [COMPLIANCE_APPROVE, PAYROLL_APPROVE, PAYMENTS_APPROVE],
  COMPLIANCE_FULL: [COMPLIANCE_READ, COMPLIANCE_CREATE, COMPLIANCE_SUBMIT, COMPLIANCE_APPROVE],
  PAYROLL_FULL: [PAYROLL_READ, PAYROLL_MANAGE, PAYROLL_RUN, PAYROLL_APPROVE],
  // ... etc
} as const;
```

---

## 6. Middleware Stack

### 6.1 Backend Middleware Chain

Located in: `packages/backend/src/core/middleware/auth.ts`

```
┌─────────────────────────────────────────────────────────────────┐
│                    MIDDLEWARE CHAIN                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. clerkAuth                                                   │
│     └─ Validates JWT token from Clerk                           │
│                          ↓                                      │
│  2. attachAuthToContext                                         │
│     └─ Extracts: userId, sessionId, orgId, orgRole, permissions│
│                          ↓                                      │
│  3. requireAuth                                                 │
│     └─ Verifies user exists in database                         │
│                          ↓                                      │
│  4. requireBackofficeOrg                                        │
│     └─ Verifies org === CLERK_INTERNAL_ORG_ID                   │
│                          ↓                                      │
│  5. requireActiveStaff                                          │
│     └─ Verifies user has active back_office_agents record       │
│                          ↓                                      │
│  6. requireBackoffice / requireRole([...])                      │
│     └─ Verifies user has appropriate role                       │
│                          ↓                                      │
│  7. requirePermission(...) (optional)                           │
│     └─ Verifies user has specific permission                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 Available Middleware

```typescript
// Authentication
export const clerkAuth = clerkMiddleware();
export const attachAuthToContext = createMiddleware(...);
export const requireAuth = createMiddleware(...);

// Organization
export const requireOrg = createMiddleware(...);
export const requireBackofficeOrg = createMiddleware(...);

// Staff Status
export const requireActiveStaff = createMiddleware(...);

// Role-based
export const requireRole = (allowedRoles: string[]) => createMiddleware(...);
export const requireAdmin = requireRole([ClerkOrgRoles.ADMIN]);
export const requireAnyAdmin = requireRole([...RoleGroups.ADMINS]);
export const requireManager = requireRole([...RoleGroups.MANAGERS]);
export const requireBackoffice = requireRole([...RoleGroups.BACKOFFICE]);
export const requireBackofficeApprover = requireRole([...]);

// Permission-based
export const requirePermission = (permission: string) => createMiddleware(...);
export const requireAnyPermission = (permissions: string[]) => createMiddleware(...);
export const requireAllPermissions = (permissions: string[]) => createMiddleware(...);
```

### 6.3 Example Route Configuration

```typescript
// packages/backend/src/modules/backoffice-users/routes.ts
.openapi(route, handler, {
  middleware: [
    requireAuth,
    requireBackofficeOrg,
    requireActiveStaff,
    requireBackofficeApprover,  // Only admins/managers can access
  ],
});
```

---

## 7. Frontend Route Protection

### 7.1 Guard Components (Defense in Depth)

Located in: `apps/backoffice/app/(authenticated)/layout.tsx`

```tsx
const AppLayout = ({ children }: AppLayoutProperties) => (
  <AuthGuard>                       {/* 1. Check authentication */}
    <ServerOrganizationGuard>       {/* 2. Check org context exists */}
      <BackofficeOrgGuard>          {/* 3. Check org is internal org */}
        <MfaGuard>                  {/* 4. Check MFA is enabled */}
          <AuthenticatedLayout>
            {children}
          </AuthenticatedLayout>
        </MfaGuard>
      </BackofficeOrgGuard>
    </ServerOrganizationGuard>
  </AuthGuard>
);
```

### 7.2 Guard Implementations

**AuthGuard** - Checks userId exists:
```typescript
// Redirects to /sign-in if not authenticated
```

**ServerOrganizationGuard** - Checks orgId exists:
```typescript
// Redirects to /org-select if no organization
```

**BackofficeOrgGuard** - Checks org is internal:
```typescript
export const BackofficeOrgGuard = async ({ children }) => {
  const { orgId } = await auth();
  const backofficeOrgId = env.CLERK_INTERNAL_ORG_ID;

  if (!isBackofficeScope(orgId, backofficeOrgId)) {
    redirect("/access-denied?reason=wrong_organization");
  }

  return <>{children}</>;
};
```

**MfaGuard** - Checks MFA is enabled:
```typescript
export async function MfaGuard({ children, redirectTo = "/setup-mfa" }) {
  if (!env.REQUIRE_MFA) {
    return <>{children}</>;
  }

  const user = await currentUser();
  if (!user?.twoFactorEnabled) {
    redirect(redirectTo);
  }

  return <>{children}</>;
}
```

### 7.3 Route Structure

```
/(unauthenticated)/
├── /sign-in              # Clerk sign-in form
├── /sign-up              # Clerk sign-up form
├── /org-select           # Organization switcher
├── /access-denied        # Access denied page
├── /pending-approval     # Awaiting role assignment
└── /setup-mfa            # MFA setup page

/(authenticated)/         # Protected by all guards
├── / (dashboard)
├── /admin/*
├── /cases/*
├── /catalog/*
├── /inbox/*
├── /tickets/*
└── /payments/*
```

---

## 8. Multi-Tenancy & Isolation

### 8.1 Realm Separation

```typescript
export type Realm = "backoffice" | "tenant";

export function getRealm(
  currentOrgId: string | null | undefined,
  backofficeOrgId: string | null | undefined
): Realm | null {
  if (!currentOrgId) return null;
  return isBackofficeScope(currentOrgId, backofficeOrgId) ? "backoffice" : "tenant";
}
```

### 8.2 Organization Scoping

**Critical Pattern**: Every database query MUST be scoped by `organizationId`:

```typescript
// In services
const orgId = requireOrganizationContext(ctx);
const result = await db
  .select(...)
  .from(table)
  .where(and(
    eq(table.organizationId, orgId),  // ALWAYS include org filter
    ...otherFilters
  ));
```

### 8.3 Backoffice Cross-Org Access

Backoffice staff can view data across organizations for support purposes:

Cross-org reach is a property of the actor, not a flag on the context: the staff
guard sets `staffAgentId`, `buildServiceContext` derives a `staff` actor from it,
and `canReachAcrossOrgs(ctx)` is true for `staff` and `system` actors only.

```typescript
// Derived by buildServiceContext when requireActiveStaff has run
{
  ...baseCtx,
  actor: { kind: "staff", userId, staffAgentId },
}
```

### 8.4 Domain Validation

```typescript
// packages/auth/helpers.ts
export function isCompanyEmail(email: string, allowedDomains: string[]): boolean {
  const normalizedEmail = email.toLowerCase().trim();
  return allowedDomains.some((domain) =>
    normalizedEmail.endsWith(domain.toLowerCase().trim())
  );
}

// Usage
if (!isCompanyEmail(email, ["@bumara.com", "@bumara.co.zm"])) {
  return { hasAccess: false, reason: "invalid_domain" };
}
```

---

## 9. Session Management

### 9.1 Session Creation

Handled by Clerk when user signs in:
1. JWT token issued with claims
2. Token stored in HTTP-only cookies (managed by Clerk)
3. Token validated on each request

### 9.2 JWT Claims Structure

```typescript
type ClerkSessionClaims = {
  sub?: string;              // User ID
  sid?: string;              // Session ID
  org_id?: string;           // Organization ID
  org_role?: string;         // e.g., "org:admin"
  org_slug?: string;         // Organization slug
  org_permissions?: string[]; // Permission array
  azp?: string;              // Authorized party (frontend URL)
};
```

### 9.3 Session Extraction

```typescript
export const attachAuthToContext = createMiddleware(async (c, next) => {
  const auth = getAuth(c);
  const claims = auth?.sessionClaims;

  // Extract with fallbacks
  const userId = auth?.userId || claims?.sub || claims?.user_id;
  const orgId = auth?.orgId || claims?.org_id || claims?.organization_id;
  const orgRole = auth?.orgRole || claims?.org_role;
  const orgPermissions = auth?.orgPermissions || claims?.org_permissions || [];

  // Attach to Hono context
  c.set("userId", userId);
  c.set("orgId", orgId);
  c.set("orgRole", orgRole);
  c.set("orgPermissions", orgPermissions);

  return next();
});
```

---

## 10. Multi-Factor Authentication (MFA)

### 10.1 MFA Requirement

- Controlled by `REQUIRE_MFA` environment variable
- All backoffice staff **must** have 2FA enabled in production
- Users without MFA are redirected to `/setup-mfa`

### 10.2 Supported Strategies

1. **TOTP** (Time-based One-Time Password)
   - 6-digit codes from authenticator apps (Google Authenticator, Authy, etc.)
   - Auto-submits when 6 digits entered

2. **Backup Codes**
   - Fallback for account recovery
   - One-time use codes

### 10.3 MFA Exempt Paths

```typescript
const MFA_EXEMPT_PATHS = [
  "/setup-mfa",       // Users setting up MFA
  "/sign-in",         // During login
  "/sign-out",        // Logout
  "/access-denied",   // Access denied page
];
```

### 10.4 Development Mode

In development (`NODE_ENV=development`):
- "Skip 2FA" button available on sign-in form
- Allows testing without configuring MFA

---

## 11. Security Features

### 11.1 Rate Limiting

Located in: `packages/backend/src/core/middleware/rate-limit.ts`

```typescript
// Preset limiters
export const organizationRateLimit = rateLimit({
  windowMs: 60 * 1000,  // 1 minute
  max: 5,               // 5 requests per minute
});

export const strictRateLimit = rateLimit({
  windowMs: 60 * 1000,
  max: 60,
});

export const generalRateLimit = rateLimit({
  windowMs: 60 * 1000,
  max: 60,
});
```

**Key Generator**: `IP:userId` combination prevents brute force

**Response Headers**:
- `X-RateLimit-Limit`: Maximum requests allowed
- `X-RateLimit-Remaining`: Requests remaining
- `X-RateLimit-Reset`: Unix timestamp when limit resets
- `Retry-After`: Seconds to wait (when rate limited)

### 11.2 CSRF Protection

- Implicitly handled by SameSite cookie policy (Clerk manages)
- Next.js server actions use automatic CSRF tokens

### 11.3 Password Policies

Managed by Clerk Dashboard:
- Minimum length requirements
- Complexity requirements
- Breach detection

### 11.4 Secure Logging

```typescript
// Never log:
// - Passwords
// - Tokens
// - API keys
// - PII in certain contexts

logger?.warn({
  userId: c.get("userId"),  // OK to log
  path: c.req.path,         // OK to log
  // password: xxx           // NEVER log
}, "Auth failed");
```

### 11.5 Staff Deactivation

`requireActiveStaff` middleware allows instant access revocation:

```typescript
// Look up staff record
const [staffRecord] = await db
  .select({ id: backOfficeAgents.id, isActive: backOfficeAgents.isActive })
  .from(backOfficeAgents)
  .where(eq(backOfficeAgents.clerkUserId, userId))
  .limit(1);

// Deny if deactivated
if (!staffRecord?.isActive) {
  return c.json({ error: "Access denied. Your staff account has been deactivated." }, 403);
}
```

---

## 12. Audit Logging

### 12.1 Audit Log Schema

Located in: `packages/database/src/schema/system/audit-logs.ts`

```typescript
export const auditLogs = pgTable("audit_logs", {
  id: uuid("id").primaryKey().defaultRandom(),
  organizationId: text("organization_id"),
  userId: text("user_id"),

  // Critical: Distinguishes actor type
  actorType: text("actor_type").$type<"STAFF" | "TENANT" | "SYSTEM">(),

  // Request context
  ipAddress: text("ip_address"),
  userAgent: text("user_agent"),

  // Action details
  action: text("action").notNull(),       // e.g., "UPDATE_ROLE"
  entityType: text("entity_type").notNull(), // e.g., "backoffice_user"
  entityId: text("entity_id"),

  // State changes
  changes: jsonb("changes").$type<{
    before?: Record<string, unknown>;
    after?: Record<string, unknown>;
  }>(),

  metadata: jsonb("metadata"),
  timestamp: timestamp("timestamp").defaultNow().notNull(),
});
```

### 12.2 Actor Types

| Actor Type | Description |
|------------|-------------|
| `STAFF` | Internal Bumara backoffice staff |
| `TENANT` | Customer organization user |
| `SYSTEM` | Automated system action (webhooks, crons) |

### 12.3 Indexed Columns

```typescript
// Optimized indexes for common queries
index("idx_audit_logs_org_timestamp").on(table.organizationId, table.timestamp);
index("idx_audit_logs_entity").on(table.entityType, table.entityId);
index("idx_audit_logs_user").on(table.userId, table.timestamp);
index("idx_audit_logs_actor_type").on(table.actorType, table.timestamp);
```

---

## 13. Staff Management

### 13.1 Staff Schema

Located in: `packages/database/src/schema/core/back-office-agents.ts`

```typescript
export const backOfficeAgents = pgTable("back_office_agents", {
  id: uuid("id").primaryKey().defaultRandom(),

  // Link to Clerk user
  clerkUserId: text("clerk_user_id").notNull().references(() => users.id),
  role: agentRoleEnum("role").notNull().default("member"),
  department: varchar("department", { length: 100 }),

  // Specializations (regulators/tasks they handle)
  specializations: jsonb("specializations").$type<string[]>().default([]),

  // Capacity management
  maxConcurrentTasks: integer("max_concurrent_tasks").default(10),
  currentTaskLoad: integer("current_task_load").default(0),

  // Availability
  isAvailable: boolean("is_available").default(true),
  isActive: boolean("is_active").default(true),  // For instant deactivation

  // Performance metrics
  tasksCompleted: integer("tasks_completed").default(0),
  averageCompletionTime: decimal("average_completion_time"),
  customerSatisfactionScore: decimal("customer_satisfaction_score"),

  // Employment
  hireDate: date("hire_date"),
  notes: text("notes"),

  ...timestamps,
});
```

### 13.2 Access Request Flow

1. User authenticates with company email
2. If not in internal org → Shows "pending approval" page
3. Admin receives access request
4. Admin assigns role via backoffice user management
5. User can now access backoffice

---

## 14. Environment Configuration

### 14.1 Required Variables

```env
# Clerk Authentication
CLERK_SECRET_KEY=sk_live_xxx
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_live_xxx

# Backoffice Organization
CLERK_INTERNAL_ORG_ID=org_xxx
CLERK_ALLOWED_BACKOFFICE_DOMAINS="@bumara.com,@bumara.co.zm"

# MFA
REQUIRE_MFA=true

# Auth URLs
NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-up
```

### 14.2 Production vs Development

| Feature | Production | Development |
|---------|-----------|-------------|
| MFA Required | `true` | Can be `false` |
| Skip 2FA Button | Hidden | Visible |
| Domain Validation | Strict | Strict |
| Rate Limiting | Active | Active |

---

## 15. Suggestions for Improvements

### 15.1 Security Enhancements

#### A. Add IP Allowlisting for Backoffice

**Current**: Any IP can access backoffice APIs
**Improvement**: Add IP allowlisting for production backoffice access

```typescript
// New middleware: requireAllowedIP
export const requireAllowedIP = createMiddleware<AppBindings>(async (c, next) => {
  const allowedIPs = process.env.BACKOFFICE_ALLOWED_IPS?.split(",") || [];
  const clientIP = c.req.header("x-forwarded-for")?.split(",")[0] || c.req.header("x-real-ip");

  if (allowedIPs.length > 0 && !allowedIPs.includes(clientIP)) {
    logger?.warn({ clientIP, path: c.req.path }, "Access denied: IP not in allowlist");
    return c.json({ error: "Access denied" }, 403);
  }

  return next();
});
```

#### B. Implement Session Binding

**Current**: Sessions can be used from any device/IP
**Improvement**: Bind sessions to device fingerprint or IP

```typescript
// Add to session claims validation
if (session.boundIP && session.boundIP !== requestIP) {
  // Require re-authentication
}
```

#### C. Add Login Anomaly Detection

**Current**: No anomaly detection
**Improvement**: Detect suspicious login patterns

- Login from new geographic location
- Login at unusual hours
- Multiple failed attempts followed by success
- Login from known malicious IPs

#### D. Implement Graduated Rate Limiting

**Current**: Fixed rate limits
**Improvement**: Escalating penalties for repeated violations

```typescript
// After 5 violations in 10 minutes, block for 15 minutes
// After 10 violations in 1 hour, block for 1 hour
// After 20 violations in 24 hours, require admin unlock
```

### 15.2 UX Improvements

#### A. Add "Remember This Device" for MFA

**Current**: MFA required on every login
**Improvement**: Allow trusted devices to skip MFA for 30 days

```typescript
// Store device trust token
interface TrustedDevice {
  deviceId: string;
  userId: string;
  createdAt: Date;
  expiresAt: Date;
  lastUsed: Date;
  deviceInfo: {
    browser: string;
    os: string;
    ip: string;
  };
}
```

#### B. Improve Error Messages

**Current**: Generic error messages
**Improvement**: More specific, actionable error messages

```typescript
// Instead of: "Access denied"
// Use: "Access denied. You need the 'org:compliance:approve' permission to approve filings. Contact your administrator."
```

#### C. Add Concurrent Session Management

**Current**: Unlimited concurrent sessions
**Improvement**: Show active sessions, allow session termination

```typescript
// New UI: /settings/sessions
// - List all active sessions
// - Show last activity, device info
// - "Terminate all other sessions" button
```

### 15.3 Monitoring & Alerting

#### A. Add Real-Time Security Alerts

**Current**: Rely on manual log review
**Improvement**: Automated alerts for:

- 5+ failed login attempts for same user
- Login from new country
- Role escalation (member → admin)
- Staff deactivation
- Bulk data access patterns

#### B. Add Authentication Metrics Dashboard

**Current**: No visibility into auth patterns
**Improvement**: Dashboard showing:

- Login success/failure rates
- MFA adoption rate
- Active sessions by user/role
- Geographic login distribution
- Rate limit hit frequency

### 15.4 Code Quality Improvements

#### A. Centralize Rate Limit Store

**Current**: In-memory store (resets on restart, not distributed)
**Improvement**: Use Redis for distributed rate limiting

```typescript
// Use Redis for rate limit store
import Redis from "ioredis";

const redis = new Redis(process.env.REDIS_URL);

const getRateLimitKey = (ip: string, userId: string) =>
  `rate_limit:${ip}:${userId}`;

// This enables rate limiting across multiple server instances
```

#### B. Add Comprehensive Auth Error Types

**Current**: String-based error codes
**Improvement**: Typed error classes

```typescript
export class AuthenticationError extends Error {
  constructor(
    public code: "NOT_AUTHENTICATED" | "INVALID_TOKEN" | "SESSION_EXPIRED",
    message: string
  ) {
    super(message);
    this.name = "AuthenticationError";
  }
}

export class AuthorizationError extends Error {
  constructor(
    public code: "INSUFFICIENT_ROLE" | "MISSING_PERMISSION" | "WRONG_ORGANIZATION",
    message: string,
    public required?: string[],
    public actual?: string
  ) {
    super(message);
    this.name = "AuthorizationError";
  }
}
```

#### C. Add Request Context Correlation

**Current**: Logs are not correlated across requests
**Improvement**: Add request ID to all logs

```typescript
// Add request ID middleware
export const addRequestId = createMiddleware<AppBindings>(async (c, next) => {
  const requestId = c.req.header("x-request-id") || crypto.randomUUID();
  c.set("requestId", requestId);
  c.header("x-request-id", requestId);
  return next();
});
```

---

## 16. Suggestions for Additions

### 16.1 WebAuthn / Passkeys Support

**What**: Add support for FIDO2/WebAuthn passwordless authentication
**Why**: More secure than passwords, resistant to phishing, better UX

```typescript
// Clerk supports passkeys - enable in Clerk Dashboard
// Update sign-in form to show passkey option

const handlePasskeySignIn = async () => {
  const result = await signIn.authenticateWithPasskey();
  if (result.status === "complete") {
    await setActive({ session: result.createdSessionId });
    router.push("/");
  }
};
```

### 16.2 SSO/SAML Integration

**What**: Add enterprise SSO support
**Why**: Large enterprise clients may require it

- Support SAML 2.0
- Support OIDC with corporate IdPs (Azure AD, Okta, etc.)
- Allow Bumara internal org to use SSO

### 16.3 Temporary Access Grants

**What**: Time-limited elevated access
**Why**: For emergency support scenarios, audits

```typescript
interface TemporaryAccessGrant {
  id: string;
  userId: string;
  grantedBy: string;
  targetOrganizationId: string;
  permissions: string[];
  reason: string;
  startsAt: Date;
  expiresAt: Date;
  revokedAt?: Date;
  revokedBy?: string;
}

// Middleware checks for active grants
// Auto-expires after duration
// Full audit trail
```

### 16.4 API Key Authentication

**What**: Long-lived API keys for automation
**Why**: Enable CI/CD, scripts, integrations

```typescript
interface ApiKey {
  id: string;
  name: string;
  keyHash: string;  // Never store plain key
  organizationId: string;
  createdBy: string;
  permissions: string[];
  expiresAt?: Date;
  lastUsedAt?: Date;
  isActive: boolean;
}

// New middleware: requireApiKeyOrAuth
// Allows either session auth or API key
```

### 16.5 Just-In-Time Provisioning

**What**: Auto-create backoffice records on first login
**Why**: Reduce manual admin work

```typescript
// On first successful login with valid domain:
// 1. Check if back_office_agents record exists
// 2. If not, create with default role (member)
// 3. Notify admin of new user
// 4. User can start with limited access immediately
```

### 16.6 Role Request Workflow

**What**: Self-service role upgrade requests
**Why**: Reduce admin overhead, improve traceability

```typescript
interface RoleRequest {
  id: string;
  requesterId: string;
  requestedRole: string;
  currentRole: string;
  justification: string;
  status: "pending" | "approved" | "rejected";
  reviewedBy?: string;
  reviewedAt?: Date;
  reviewNotes?: string;
}

// UI: "Request Role Upgrade" button
// Admin UI: Pending requests queue
// Auto-notify admin of new requests
```

### 16.7 Security Headers Hardening

**What**: Add comprehensive security headers
**Why**: Defense against XSS, clickjacking, etc.

```typescript
// Add security headers middleware
export const securityHeaders = createMiddleware(async (c, next) => {
  c.header("Strict-Transport-Security", "max-age=31536000; includeSubDomains");
  c.header("X-Content-Type-Options", "nosniff");
  c.header("X-Frame-Options", "DENY");
  c.header("X-XSS-Protection", "1; mode=block");
  c.header("Referrer-Policy", "strict-origin-when-cross-origin");
  c.header("Permissions-Policy", "geolocation=(), microphone=(), camera=()");
  c.header("Content-Security-Policy", "default-src 'self'; ...");
  return next();
});
```

### 16.8 Compliance Reporting

**What**: Generate auth compliance reports
**Why**: SOC2, ISO 27001, regulatory requirements

- User access reports (who has access to what)
- Login history reports
- Permission change audit reports
- MFA compliance reports
- Session duration reports

### 16.9 Break Glass Access

**What**: Emergency access procedure
**Why**: Critical situation recovery

```typescript
interface BreakGlassAccess {
  id: string;
  requesterId: string;
  reason: string;
  approvers: string[];  // Require multiple approvers
  status: "pending" | "approved" | "denied" | "expired" | "used";
  accessGranted: Date;
  accessExpires: Date;  // Short window (e.g., 4 hours)
  actionsPerformed: AuditLog[];
}

// Requires:
// - Multiple admin approvals
// - Detailed reason
// - Time-limited
// - Full action logging
// - Automatic review after use
```

### 16.10 Enhanced Password Security

**What**: Additional password security features
**Why**: Prevent credential stuffing, reuse

- Check passwords against HaveIBeenPwned database
- Prevent reuse of last N passwords
- Password expiry policy (optional, configurable)
- Forced password change after suspicious activity

---

## Summary

The Bumara backoffice authentication system is **production-ready** with:

- Decoupled auth via Clerk (industry-standard)
- Multi-tenancy with strict isolation
- Fine-grained RBAC with permissions
- Mandatory MFA for all staff
- Comprehensive audit logging
- Defense-in-depth with multiple guard layers
- Domain validation for company emails
- Instant deactivation capability

**Key Strengths**:
1. Separation between backoffice and tenant organizations
2. Multiple layers of protection (middleware + guards)
3. Audit trail with actor type distinction
4. Flexible permission system

**Priority Improvements**:
1. Distributed rate limiting (Redis)
2. IP allowlisting for production
3. Login anomaly detection
4. Security alerting

**Future Additions** (by priority):
1. WebAuthn/Passkeys support
2. Temporary access grants
3. API key authentication
4. Enhanced security headers
5. Compliance reporting
