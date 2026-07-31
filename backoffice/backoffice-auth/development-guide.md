---
title: "Development Guide"
description: "How to protect routes, check permissions, and implement secure backoffice features."
---

This guide provides practical code examples for common authorization patterns in the backoffice.

---

## Protecting API Routes

### Basic Backoffice Route

Any backoffice staff member can access:

```typescript
// packages/backend/src/modules/my-module/routes.ts
import { Hono } from "hono";
import {
  clerkAuth,
  attachAuthToContext,
  requireAuth,
  requireBackofficeOrg,
  requireActiveStaff,
  requireBackoffice,
} from "@/core/middleware/auth";
import type { AppBindings } from "@/types";

const app = new Hono<AppBindings>();

app.use("*", clerkAuth);
app.use("*", attachAuthToContext);

app.get(
  "/my-resource",
  requireAuth,
  requireBackofficeOrg,
  requireActiveStaff,
  requireBackoffice,
  async (c) => {
    const userId = c.get("userId");
    const orgId = c.get("orgId");
    
    // Your handler logic
    return c.json({ success: true });
  }
);

export default app;
```

### Route Requiring Approval Permission

Only managers and admins can access:

```typescript
import { requireBackofficeApprover } from "@/core/middleware/auth";

app.post(
  "/approve-payout/:id",
  requireAuth,
  requireBackofficeOrg,
  requireActiveStaff,
  requireBackofficeApprover,  // Only admin/manager
  async (c) => {
    const payoutId = c.req.param("id");
    
    // Approval logic
    return c.json({ success: true });
  }
);
```

### Route with Specific Permission

```typescript
import { requirePermission, Permissions } from "@/core/middleware/auth";

app.post(
  "/process-payment",
  requireAuth,
  requireBackofficeOrg,
  requireActiveStaff,
  requirePermission(Permissions.PAYMENTS_PROCESS),
  async (c) => {
    // Only users with PAYMENTS_PROCESS permission
    return c.json({ success: true });
  }
);
```

### Route with Multiple Possible Permissions

```typescript
import { requireAnyPermission, Permissions } from "@/core/middleware/auth";

app.get(
  "/financial-report",
  requireAuth,
  requireOrg,
  requireAnyPermission([
    Permissions.PAYMENTS_READ,
    Permissions.PAYROLL_READ,
  ]),
  async (c) => {
    // Users with either permission can access
    return c.json({ success: true });
  }
);
```

---

## Adding Server Component Guards

### Protect a Page

```tsx
// apps/backoffice/app/(authenticated)/(home)/admin/page.tsx
import { auth } from "@repo/auth/server";
import { isBackofficeAdmin } from "@repo/auth/helpers";
import { env } from "@/env";
import { redirect } from "next/navigation";

export default async function AdminPage() {
  const { orgId, orgRole } = await auth();
  
  // Check if user is in backoffice org
  if (orgId !== env.CLERK_INTERNAL_ORG_ID) {
    redirect("/access-denied");
  }
  
  // Check if user is admin
  if (!isBackofficeAdmin(orgRole)) {
    redirect("/access-denied?reason=insufficient_role");
  }
  
  return (
    <div>
      <h1>Admin Dashboard</h1>
      {/* Admin-only content */}
    </div>
  );
}
```

### Create a Reusable Guard

```tsx
// apps/backoffice/components/guards/admin-guard.tsx
import { auth } from "@repo/auth/server";
import { isBackofficeAdmin } from "@repo/auth/helpers";
import { redirect } from "next/navigation";
import type { ReactNode } from "react";

export async function AdminGuard({ children }: { children: ReactNode }) {
  const { orgRole } = await auth();
  
  if (!isBackofficeAdmin(orgRole)) {
    redirect("/access-denied?reason=admin_required");
  }
  
  return <>{children}</>;
}
```

Usage:

```tsx
// apps/backoffice/app/(authenticated)/(home)/admin/layout.tsx
import { AdminGuard } from "@/components/guards/admin-guard";

export default function AdminLayout({ children }) {
  return <AdminGuard>{children}</AdminGuard>;
}
```

---

## Checking Permissions in Components

### Server Component

```tsx
import { auth } from "@repo/auth/server";
import { isBackofficeManager } from "@repo/auth/helpers";

export default async function ActionBar() {
  const { orgRole } = await auth();
  const canApprove = isBackofficeManager(orgRole);
  
  return (
    <div className="flex gap-2">
      <Button>View</Button>
      {canApprove && <Button>Approve</Button>}
    </div>
  );
}
```

### Client Component

```tsx
"use client";

import { useOrganization } from "@repo/auth/client";

export function ActionBar() {
  const { membership } = useOrganization();
  const role = membership?.role;
  
  const canApprove = 
    role === "org:admin" || 
    role === "org:manager" ||
    role === "org:backoffice_admin" ||
    role === "org:backoffice_manager";
  
  return (
    <div className="flex gap-2">
      <Button>View</Button>
      {canApprove && <Button>Approve</Button>}
    </div>
  );
}
```

---

## Adding Audit Logs

### Basic Audit Log

```typescript
import { recordAuditLog } from "@repo/api-services";

// In your handler
await recordAuditLog(ctx, deps, {
  action: "approve",
  entityType: "payout",
  entityId: payoutId,
  actorType: "STAFF",  // Important for backoffice actions
  changes: {
    before: { status: "pending" },
    after: { status: "approved" },
  },
});
```

### Audit Log with Metadata

```typescript
await recordAuditLog(ctx, deps, {
  action: "create",
  entityType: "filing",
  entityId: filingId,
  actorType: "STAFF",
  changes: {
    after: {
      status: "draft",
      regulator: "ZRA",
      type: "VAT",
    },
  },
  metadata: {
    requestId: c.get("requestId"),
    ipAddress: c.req.header("x-forwarded-for"),
    userAgent: c.req.header("user-agent"),
  },
});
```

### State Transition Audit

```typescript
async function transitionFilingStatus(
  filingId: string,
  fromStatus: string,
  toStatus: string,
  reason?: string
) {
  // Update status
  await db
    .update(filings)
    .set({ status: toStatus, updatedAt: new Date() })
    .where(eq(filings.id, filingId));
  
  // Audit log
  await recordAuditLog(ctx, deps, {
    action: "transition",
    entityType: "filing",
    entityId: filingId,
    actorType: "STAFF",
    fromStatus,
    toStatus,
    reason,
    changes: {
      before: { status: fromStatus },
      after: { status: toStatus },
    },
  });
}
```

---

## Working with Staff Records

### Check Staff Status

```typescript
import db from "@repo/database";
import { backOfficeAgents } from "@repo/database/schema";
import { eq } from "drizzle-orm";

async function getStaffRecord(clerkUserId: string) {
  const [staff] = await db
    .select()
    .from(backOfficeAgents)
    .where(eq(backOfficeAgents.clerkUserId, clerkUserId))
    .limit(1);
  
  return staff;
}
```

### Deactivate Staff Member

```typescript
async function deactivateStaff(staffId: string, reason: string) {
  await db
    .update(backOfficeAgents)
    .set({ isActive: false, updatedAt: new Date() })
    .where(eq(backOfficeAgents.id, staffId));
  
  await recordAuditLog(ctx, deps, {
    action: "deactivate",
    entityType: "staff",
    entityId: staffId,
    actorType: "STAFF",
    reason,
    changes: {
      before: { isActive: true },
      after: { isActive: false },
    },
  });
}
```

---

## Handling Cross-Org Access

Backoffice staff can access data across all tenant organizations:

```typescript
app.get(
  "/tenants/:tenantId/filings",
  requireAuth,
  requireBackofficeOrg,
  requireActiveStaff,
  requireBackoffice,
  async (c) => {
    const tenantId = c.req.param("tenantId");
    
    // Backoffice can query any tenant's data
    const filings = await db
      .select()
      .from(filingsTable)
      .where(eq(filingsTable.orgId, tenantId));
    
    return c.json({ filings });
  }
);
```

> **Important:** Regular tenant routes should always filter by the user's `orgId` from context.

---

## Error Response Format

All auth errors follow a consistent format:

```typescript
interface AuthError {
  success: false;
  error: string;      // HTTP status phrase
  message: string;    // Human-readable message
}

// Example responses
return c.json({
  success: false,
  error: "Unauthorized",
  message: "Authentication required. Please login.",
}, 401);

return c.json({
  success: false,
  error: "Forbidden",
  message: "Access denied. Backoffice organization membership required.",
}, 403);
```

---

## Testing Authorization

### Unit Test for Middleware

```typescript
// packages/backend/src/core/middleware/auth.test.ts
import { describe, it, expect, vi } from "vitest";
import { requireBackofficeOrg } from "./auth";

describe("requireBackofficeOrg", () => {
  it("allows request when orgId matches backoffice org", async () => {
    process.env.CLERK_INTERNAL_ORG_ID = "org_backoffice";
    
    const c = createMockContext({
      orgId: "org_backoffice",
    });
    
    const next = vi.fn();
    await requireBackofficeOrg(c, next);
    
    expect(next).toHaveBeenCalled();
  });
  
  it("rejects request when orgId does not match", async () => {
    process.env.CLERK_INTERNAL_ORG_ID = "org_backoffice";
    
    const c = createMockContext({
      orgId: "org_tenant",
    });
    
    const next = vi.fn();
    const result = await requireBackofficeOrg(c, next);
    
    expect(next).not.toHaveBeenCalled();
    expect(result.status).toBe(403);
  });
});
```

### Integration Test

```typescript
describe("POST /backoffice/approve-payout", () => {
  it("rejects non-manager roles", async () => {
    const res = await app.request("/backoffice/approve-payout/123", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${memberToken}`,  // member role
      },
    });
    
    expect(res.status).toBe(403);
  });
  
  it("allows manager role", async () => {
    const res = await app.request("/backoffice/approve-payout/123", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${managerToken}`,  // manager role
      },
    });
    
    expect(res.status).toBe(200);
  });
});
```

---

## Best Practices

### 1. Always Use the Full Middleware Chain

Don't skip middleware for convenience:

```typescript
// ❌ BAD - Missing security middleware
app.get("/sensitive-data", requireAuth, async (c) => {});

// ✅ GOOD - Complete chain
app.get("/sensitive-data",
  requireAuth,
  requireBackofficeOrg,
  requireActiveStaff,
  requireBackoffice,
  async (c) => {}
);
```

### 2. Check Permissions at the Lowest Level

Don't rely only on route-level checks:

```typescript
// ❌ BAD - Only route-level check
app.post("/approve/:id", requireBackofficeApprover, async (c) => {
  await approveItem(id);  // No additional check
});

// ✅ GOOD - Additional business logic check
app.post("/approve/:id", requireBackofficeApprover, async (c) => {
  const item = await getItem(id);
  
  if (item.status !== "pending_approval") {
    return c.json({ error: "Cannot approve item in current state" }, 400);
  }
  
  await approveItem(id);
});
```

### 3. Audit All State Changes

```typescript
// ❌ BAD - No audit trail
await db.update(payouts).set({ status: "approved" });

// ✅ GOOD - With audit log
await db.update(payouts).set({ status: "approved" });
await recordAuditLog(ctx, deps, {
  action: "approve",
  entityType: "payout",
  entityId: payoutId,
  actorType: "STAFF",
  changes: { before: { status: "pending" }, after: { status: "approved" } },
});
```

### 4. Never Trust Client Input for Auth

```typescript
// ❌ BAD - Trusting client-provided org ID
app.get("/data", async (c) => {
  const orgId = c.req.query("orgId");  // Client provides this!
  const data = await getData(orgId);
});

// ✅ GOOD - Use authenticated context
app.get("/data", requireAuth, requireOrg, async (c) => {
  const orgId = c.get("orgId");  // From verified JWT
  const data = await getData(orgId);
});
```

---

## Next Steps

- [Troubleshooting](/backoffice/backoffice-auth/troubleshooting) - Common issues and solutions
- [Architecture](/backoffice/backoffice-auth/architecture) - Review security layers

