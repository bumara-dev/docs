---
title: "Authorization"
description: "Role-based and permission-based access control for the backoffice."
---

The backoffice uses a comprehensive authorization system with roles, permissions, and middleware to control access to features and operations.

---

## Role Model

### Two Role Contexts

Roles exist in two contexts but share the same hierarchy:

| Context | Organization | Users |
|---------|--------------|-------|
| **Tenant** | Customer orgs | Business users managing their compliance |
| **Backoffice** | Internal org | Bumara staff managing all tenants |

### Role Hierarchy

```
Level 3 (Admin):   org:admin = org:backoffice_admin     (full access)
Level 2 (Manager): org:manager = org:backoffice_manager (approve + submit)
Level 1 (Member):  org:member = org:backoffice_member   (submit only)
```

### Tenant Roles

| Role | Clerk Key | Description |
|------|-----------|-------------|
| Admin | `org:admin` | Full organization access, manage members |
| Manager | `org:manager` | Approve filings, manage submissions |
| Member | `org:member` | View and create, no approval rights |

### Backoffice Roles

The backoffice accepts both standard Clerk roles and backoffice-specific roles:

**Option A: Standard Clerk Roles (Recommended)**

| Role | Clerk Key | Description |
|------|-----------|-------------|
| Admin | `org:admin` | Full backoffice access, manage staff |
| Manager | `org:manager` | Approve operations, manage submissions |
| Member | `org:member` | Submit only, no approval rights |

**Option B: Backoffice-Specific Roles**

| Role | Clerk Key | Description |
|------|-----------|-------------|
| Backoffice Admin | `org:backoffice_admin` | Full backoffice access |
| Backoffice Manager | `org:backoffice_manager` | Approve operations |
| Backoffice Member | `org:backoffice_member` | Submit only |

> Both role types are treated identically at the same level.

### Valid Backoffice Roles

**File:** `packages/auth/helpers.ts`

```typescript
export const BACKOFFICE_ROLES = [
  // Standard roles
  "org:admin",
  "org:manager",
  "org:member",
  // Backoffice-specific roles
  "org:backoffice_admin",
  "org:backoffice_manager",
  "org:backoffice_member",
] as const;
```

---

## Role Groups

**File:** `packages/backend/src/core/middleware/auth.ts`

```typescript
export const RoleGroups = {
  /** All admin-level roles */
  ADMINS: ["org:admin", "org:backoffice_admin"],
  
  /** All manager-level roles (can approve) */
  MANAGERS: [
    "org:admin",
    "org:manager",
    "org:backoffice_admin",
    "org:backoffice_manager",
  ],
  
  /** All backoffice roles */
  BACKOFFICE: [
    "org:backoffice_admin",
    "org:backoffice_manager",
    "org:backoffice_member",
  ],
  
  /** All organization (tenant) roles */
  ORGANIZATION: [
    "org:admin",
    "org:manager",
    "org:member",
  ],
};
```

---

## Permission System

### Permission Format

Permissions follow the pattern: `org:feature:action`

### Available Permissions

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

  // Invoicing
  INVOICE_READ: "org:invoice:read",
  INVOICE_CREATE: "org:invoice:create",
  INVOICE_SEND: "org:invoice:send",
  INVOICE_MANAGE: "org:invoice:manage",

  // Documents
  DOCUMENTS_READ: "org:documents:read",
  DOCUMENTS_UPLOAD: "org:documents:upload",
  DOCUMENTS_DELETE: "org:documents:delete",
  DOCUMENTS_MANAGE: "org:documents:manage",

  // Payments
  PAYMENTS_READ: "org:payments:read",
  PAYMENTS_CREATE: "org:payments:create",
  PAYMENTS_PROCESS: "org:payments:process",
  PAYMENTS_APPROVE: "org:payments:approve",
  PAYMENTS_REFUND: "org:payments:refund",

  // AI
  AI_USE: "org:ai:use",
  AI_INSIGHTS: "org:ai:insights",
};
```

### Permission Groups

```typescript
export const PermissionGroups = {
  VIEWER: [
    Permissions.COMPLIANCE_READ,
    Permissions.PAYROLL_READ,
    Permissions.INVOICE_READ,
    Permissions.DOCUMENTS_READ,
    Permissions.PAYMENTS_READ,
  ],

  APPROVER: [
    Permissions.COMPLIANCE_APPROVE,
    Permissions.PAYROLL_APPROVE,
    Permissions.PAYMENTS_APPROVE,
  ],

  COMPLIANCE_FULL: [
    Permissions.COMPLIANCE_READ,
    Permissions.COMPLIANCE_CREATE,
    Permissions.COMPLIANCE_SUBMIT,
    Permissions.COMPLIANCE_APPROVE,
  ],
  
  // ... more groups
};
```

---

## API Middleware Reference

### Authentication Middleware

| Middleware | Purpose | Error |
|------------|---------|-------|
| `clerkAuth` | Validates Clerk JWT | 401 |
| `attachAuthToContext` | Extracts user/org from claims | - |
| `requireAuth` | Verifies user exists in DB | 401/404 |
| `requireOrg` | Verifies org exists in DB | 403/404 |

### Backoffice Middleware

| Middleware | Purpose | Error |
|------------|---------|-------|
| `requireBackofficeOrg` | Validates internal org membership | 403 |
| `requireActiveStaff` | Checks `is_active` flag | 403 |
| `requireBackoffice` | Validates any backoffice role | 403 |
| `requireBackofficeApprover` | Requires admin or manager | 403 |

### Role Middleware

| Middleware | Purpose |
|------------|---------|
| `requireRole([roles])` | User must have one of specified roles |
| `requireAdmin` | Shorthand for `org:admin` only |
| `requireAnyAdmin` | Any admin role (org or backoffice) |
| `requireManager` | Any manager or admin role |

### Permission Middleware

| Middleware | Purpose |
|------------|---------|
| `requirePermission(perm)` | User must have specific permission |
| `requireAnyPermission([perms])` | User must have at least one permission |
| `requireAllPermissions([perms])` | User must have all permissions |

---

## Standard Middleware Chains

### Backoffice Route (Any Role)

```typescript
middleware: [
  clerkAuth,
  attachAuthToContext,
  requireAuth,
  requireBackofficeOrg,
  requireActiveStaff,
  requireBackoffice,
]
```

### Backoffice Route (Approval Required)

```typescript
middleware: [
  clerkAuth,
  attachAuthToContext,
  requireAuth,
  requireBackofficeOrg,
  requireActiveStaff,
  requireBackofficeApprover,
]
```

### Backoffice Route (Admin Only)

```typescript
middleware: [
  clerkAuth,
  attachAuthToContext,
  requireAuth,
  requireBackofficeOrg,
  requireActiveStaff,
  requireAnyAdmin,
]
```

### Tenant Route with Permission

```typescript
middleware: [
  clerkAuth,
  attachAuthToContext,
  requireAuth,
  requireOrg,
  requirePermission(Permissions.COMPLIANCE_APPROVE),
]
```

---

## Server Component Guards

### Guard Components

| Guard | Location | Purpose |
|-------|----------|---------|
| `AuthGuard` | `components/shared/auth/auth-guard.tsx` | Ensures signed in |
| `ServerOrganizationGuard` | `components/shared/organization/server-organization-guard.tsx` | Ensures org context |
| `BackofficeOrgGuard` | `components/shared/organization/backoffice-org-guard.tsx` | Verifies internal org |
| `MfaGuard` | `components/shared/auth/mfa-guard.tsx` | Requires MFA enabled |

### Layout Composition

```tsx
// apps/backoffice/app/(authenticated)/layout.tsx
const AppLayout = ({ children }) => (
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

---

## Helper Functions

### Role Helpers

**File:** `packages/auth/helpers.ts`

```typescript
// Check if role is valid backoffice role
export function isBackofficeRole(role: string | null): boolean

// Check role hierarchy
export function hasRoleOrHigher(
  userRole: BackofficeRole | null,
  requiredRole: BackofficeRole
): boolean

// Check if admin
export function isBackofficeAdmin(role: string | null): boolean

// Check if manager or higher
export function isBackofficeManager(role: string | null): boolean
```

### Context Helpers

**File:** `packages/backend/src/core/middleware/auth.ts`

```typescript
// Get user ID from context
export const getAuthUserId = (c: Context): string | undefined

// Get org ID from context
export const getAuthOrgId = (c: Context): string | undefined

// Get role from context
export const getAuthOrgRole = (c: Context): string | undefined

// Check if admin
export const isOrgAdmin = (c: Context): boolean
export const isAnyAdmin = (c: Context): boolean

// Check if manager or higher
export const isManager = (c: Context): boolean

// Check if backoffice member
export const isBackoffice = (c: Context): boolean

// Check if can approve
export const canApprove = (c: Context): boolean

// Check specific permission
export const hasPermission = (c: Context, permission: string): boolean
```

---

## Realm Scope Helpers

**File:** `packages/auth/helpers.ts`

```typescript
export type Realm = "backoffice" | "tenant";

// Check if in backoffice scope
export function isBackofficeScope(
  currentOrgId: string | null,
  backofficeOrgId: string | null
): boolean

// Get realm from org context
export function getRealm(
  currentOrgId: string | null,
  backofficeOrgId: string | null
): Realm | null

// Assert backoffice scope (throws if not)
export function assertBackofficeScope(
  currentOrgId: string | null,
  backofficeOrgId: string | null
): void
```

---

## Staff Deactivation

The `back_office_agents` table provides instant revocation:

```sql
-- Deactivate a staff member
UPDATE back_office_agents 
SET is_active = false 
WHERE clerk_user_id = 'user_xxx';
```

The `requireActiveStaff` middleware immediately blocks deactivated users without requiring Clerk changes.

---

## Audit Logging with Actor Type

All authorization-sensitive actions should be logged:

```typescript
await recordAuditLog(ctx, deps, {
  action: 'update_role',
  entityType: 'staff',
  entityId: staffId,
  actorType: 'STAFF',  // Identifies backoffice action
  changes: {
    before: { role: 'org:member' },
    after: { role: 'org:manager' },
  },
});
```

---

## Next Steps

- [Environment Setup](/backoffice/backoffice-auth/environment-setup) - Configure Clerk and env vars
- [Development Guide](/backoffice/backoffice-auth/development-guide) - Protect new routes

