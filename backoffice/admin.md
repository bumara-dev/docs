---
title: "Admin Module"
description: "Staff management, roles, and system configuration."
---

## Table of Contents

1. [Overview](#1-overview)
2. [Staff Management](#2-staff-management)
3. [Role System](#3-role-system)
4. [Pending Approvals](#4-pending-approvals)
5. [System Configuration](#5-system-configuration)
6. [Audit Logs](#6-audit-logs)

---

## 1. Overview

**Route:** `/admin`  
**Access:** Admin only  
**Purpose:** System administration and staff management.

### 1.1 Admin Dashboard Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│ System Administration                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐         │
│ │ Active Staff    │ │ Pending         │ │ System Health   │         │
│ │      12         │ │ Approvals: 3    │ │    ✓ OK         │         │
│ └─────────────────┘ └─────────────────┘ └─────────────────┘         │
│                                                                      │
│ Quick Actions                                                        │
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ │
│ │ Staff &      │ │ Pending      │ │ Escalations  │ │ Audit        │ │
│ │ Roles        │ │ Approvals    │ │              │ │ Logs         │ │
│ └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘ │
│                                                                      │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ Staff Overview                                                  │ │
│ │                                                                 │ │
│ │ ┌───────────┬────────────┬──────────┬─────────┬───────────────┐│ │
│ │ │ Name      │ Email      │ Role     │ Status  │ Actions       ││ │
│ │ ├───────────┼────────────┼──────────┼─────────┼───────────────┤│ │
│ │ │ Jane Doe  │ jane@...   │ Admin    │ Active  │ [Edit] [...]  ││ │
│ │ │ John Smith│ john@...   │ Manager  │ Active  │ [Edit] [...]  ││ │
│ │ └───────────┴────────────┴──────────┴─────────┴───────────────┘│ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ Pending Approvals                                               │ │
│ │                                                                 │ │
│ │ • High-value payout: ZMW 45,000 - ABC Ltd (awaiting approval)  │ │
│ │ • Access request: newstaff@bumara.com                          │ │
│ └─────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 Access Control

Only users with `admin` role can access `/admin` routes.

```typescript
// apps/backoffice/app/(authenticated)/(home)/(general)/admin/page.tsx

import { requireAdmin } from '@/lib/guards/require-role';

export default async function AdminPage() {
  await requireAdmin(); // Redirects if not admin
  
  return <AdminDashboard />;
}
```

---

## 2. Staff Management

### 2.1 Staff List

| Column | Description |
|--------|-------------|
| Name | Staff member name |
| Email | Work email |
| Role | admin / manager / member |
| Department | Operations, Finance, etc. |
| Status | Active / Inactive |
| Joined | When added to backoffice |
| Last Active | Last login/action |
| Actions | Edit, Disable, Remove |

### 2.2 Staff Actions

| Action | Description | Confirmation |
|--------|-------------|--------------|
| Edit Role | Change role level | Yes |
| Edit Department | Change department | No |
| Disable | Temporarily disable access | Yes |
| Enable | Re-enable disabled account | No |
| Remove | Remove from backoffice org | Yes (with reason) |

### 2.3 Add Staff Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│ Add Staff Member                                                 ✕  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│ Email:                                                               │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ newstaff@bumara.com                                             │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│ ⚠️ Must be @bumara.com email                                        │
│                                                                      │
│ Role:                                                                │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ Member                                                       ▼ │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│ Department:                                                          │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ Operations                                                   ▼ │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│ Specializations (optional):                                          │
│ ☐ ZRA      ☐ PACRA      ☐ NAPSA      ☐ NHIMA                       │
│                                                                      │
│                                             [Cancel] [Send Invite]  │
└─────────────────────────────────────────────────────────────────────┘
```

**Process:**
1. Admin enters staff email
2. System validates email domain
3. System sends invite via Clerk
4. User accepts invite and joins backoffice org
5. System creates `back_office_agents` record

### 2.4 Change Role Flow

```typescript
async function changeStaffRole(
  staffId: string,
  newRole: 'admin' | 'manager' | 'member',
  adminId: string
) {
  // 1. Update Clerk org membership role
  await clerk.organizations.updateMembershipRole({
    organizationId: BACKOFFICE_ORG_ID,
    userId: staffId,
    role: mapRoleToClerkRole(newRole),
  });
  
  // 2. Update back_office_agents record
  await db.backOfficeAgents.update({
    where: { clerkUserId: staffId },
    data: { role: newRole },
  });
  
  // 3. Create audit log
  await createAuditLog({
    resourceType: 'back_office_agent',
    resourceId: staffId,
    action: 'role_changed',
    actorId: adminId,
    beforeState: { role: oldRole },
    afterState: { role: newRole },
  });
}
```

### 2.5 Staff Database Schema

```typescript
// packages/database/src/schema/core/back-office-agents.ts

export const backOfficeAgents = pgTable('back_office_agents', {
  id: uuid('id').primaryKey().defaultRandom(),
  clerkUserId: varchar('clerk_user_id', { length: 50 }).notNull().unique(),
  
  // Role & department
  role: varchar('role', { length: 20 }).notNull().default('member'),
  department: varchar('department', { length: 50 }),
  specializations: jsonb('specializations').$type<string[]>(),
  
  // Capacity
  maxConcurrentTasks: integer('max_concurrent_tasks').default(10),
  currentTaskLoad: integer('current_task_load').default(0),
  
  // Status
  isAvailable: boolean('is_available').default(true),
  isActive: boolean('is_active').default(true),
  
  // Performance (optional)
  avgHandlingTimeMinutes: integer('avg_handling_time_minutes'),
  completedCasesCount: integer('completed_cases_count').default(0),
  
  // Timestamps
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
  disabledAt: timestamp('disabled_at'),
});
```

---

## 3. Role System

### 3.1 Role Hierarchy

```
admin (level 3)
  │
  ├── Full system access
  ├── Staff management
  ├── Configuration changes
  ├── Override any gate
  └── View all audit logs

manager (level 2)
  │
  ├── Team management (within team)
  ├── Reassign cases
  ├── Approve high-value payouts
  ├── Override gates (with reason)
  └── View team audit logs

member (level 1)
  │
  ├── View inbox/queues
  ├── Claim and work cases
  ├── Create tickets
  ├── Upload evidence
  └── Standard operations
```

### 3.2 Permission Matrix

| Permission | member | manager | admin |
|------------|:------:|:-------:|:-----:|
| View all cases | ✅ | ✅ | ✅ |
| Claim cases | ✅ | ✅ | ✅ |
| Work assigned cases | ✅ | ✅ | ✅ |
| Create tickets | ✅ | ✅ | ✅ |
| Upload evidence | ✅ | ✅ | ✅ |
| Verify payments | ✅ | ✅ | ✅ |
| Initiate payouts | ✅ | ✅ | ✅ |
| Reassign cases | ❌ | ✅ | ✅ |
| Override gates | ❌ | ✅ | ✅ |
| Approve high-value payouts | ❌ | ✅ | ✅ |
| View reports | ❌ | ✅ | ✅ |
| Manage staff | ❌ | ❌ | ✅ |
| System configuration | ❌ | ❌ | ✅ |
| View all audit logs | ❌ | ❌ | ✅ |
| Delete/modify records | ❌ | ❌ | ✅ |

### 3.3 Role Guards

**Server-side guards:**

```typescript
// packages/backend/src/core/middleware/auth.ts

export function requireRole(allowedRoles: string[]) {
  return async (c: Context, next: Next) => {
    const staffRole = c.get('staffRole');
    
    if (!allowedRoles.includes(staffRole)) {
      return c.json({
        error: {
          code: 'FORBIDDEN',
          message: 'Insufficient role permissions',
        }
      }, 403);
    }
    
    await next();
  };
}

export const requireAdmin = () => requireRole(['admin']);
export const requireManagerOrAbove = () => requireRole(['manager', 'admin']);
```

**Client-side guards:**

```typescript
// apps/backoffice/lib/guards/require-role.ts

export async function requireAdmin() {
  const role = await getStaffRole();
  
  if (role !== 'admin') {
    redirect('/access-denied?reason=insufficient_permissions');
  }
}

export async function requireManagerOrAbove() {
  const role = await getStaffRole();
  
  if (!['manager', 'admin'].includes(role)) {
    redirect('/access-denied?reason=insufficient_permissions');
  }
}
```

---

## 4. Pending Approvals

### 4.1 Approval Queue

Items requiring admin/manager approval:

| Type | Condition | Approver |
|------|-----------|----------|
| High-value payout | Amount > threshold | Manager+ |
| Gate override | Bypassing readiness | Manager+ |
| Access request | New staff join | Admin |
| Role change | Promotion/demotion | Admin |
| Sensitive action | Configured actions | Admin |

### 4.2 Approval Interface

```
┌─────────────────────────────────────────────────────────────────────┐
│ Pending Approvals                                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│ Payout Approvals (2)                                                 │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ ZMW 45,000 to ZRA for ABC Ltd                                  │ │
│ │ Submitted by: John Smith • 2 hours ago                         │ │
│ │ Evidence: bank_transfer.pdf                      [Review] [↓]  │ │
│ ├─────────────────────────────────────────────────────────────────┤ │
│ │ ZMW 28,000 to PACRA for XYZ Corp                               │ │
│ │ Submitted by: Jane Doe • 30 minutes ago                        │ │
│ │ Evidence: payout_receipt.pdf                     [Review] [↓]  │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│ Access Requests (1)                                                  │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ newstaff@bumara.com                                            │ │
│ │ Requested: Member role • Operations department                 │ │
│ │ Requested: 1 day ago                  [Approve] [Reject] [↓]   │ │
│ └─────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.3 Approval Workflow

```typescript
interface ApprovalRequest {
  id: string;
  type: 'payout' | 'override' | 'access' | 'role_change';
  resourceType: string;
  resourceId: string;
  requestedBy: string;
  requestedAt: Date;
  data: Record<string, any>;
  status: 'pending' | 'approved' | 'rejected';
  reviewedBy?: string;
  reviewedAt?: Date;
  reason?: string;
}

async function handleApproval(
  requestId: string,
  decision: 'approve' | 'reject',
  reason: string | undefined,
  adminId: string
) {
  const request = await getApprovalRequest(requestId);
  
  // Execute the approved action
  if (decision === 'approve') {
    await executeApprovedAction(request);
  }
  
  // Update request status
  await updateApprovalRequest(requestId, {
    status: decision === 'approve' ? 'approved' : 'rejected',
    reviewedBy: adminId,
    reviewedAt: new Date(),
    reason,
  });
  
  // Notify requester
  await notifyStaff(request.requestedBy, {
    type: 'approval_decision',
    requestId,
    decision,
    reason,
  });
  
  // Audit log
  await createAuditLog({
    resourceType: 'approval_request',
    resourceId: requestId,
    action: `request_${decision}`,
    actorId: adminId,
    metadata: { reason },
  });
}
```

---

## 5. System Configuration

### 5.1 Configuration Options

| Setting | Description | Default |
|---------|-------------|---------|
| Payout approval threshold | Amount requiring approval | ZMW 5,000 |
| SLA first response | Hours for first response | 4 |
| SLA resolution | Hours for resolution | 24 |
| Allowed email domains | Domains for staff | bumara.com |
| MFA required | Enforce MFA for staff | true |
| Session timeout | Idle timeout minutes | 60 |

### 5.2 Configuration UI

**MVP:** Configuration via environment variables or database settings table.

**Phase 2:** Admin UI for configuration changes with audit logging.

### 5.3 Feature Flags

Some features are gated by feature flags:

```typescript
// config/feature-flags.ts

export const featureFlags = {
  backoffice_reports: false,    // Reports module
  backoffice_catalog: false,    // Catalog management
  dual_approval: true,          // Require dual approval
  auto_assignment: false,       // Auto-assign cases
};
```

---

## 6. Audit Logs

### 6.1 Global Audit View

Admins can view all audit events:

```
┌─────────────────────────────────────────────────────────────────────┐
│ System Audit Log                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│ Filters: [Date Range] [Actor] [Action] [Resource Type] [Search]    │
│                                                                      │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ Timestamp          │ Actor       │ Action       │ Resource      │ │
│ ├────────────────────┼─────────────┼──────────────┼───────────────┤ │
│ │ Jan 3, 2:30 PM     │ jane@...    │ verified     │ Payment #123  │ │
│ │ Jan 3, 2:15 PM     │ john@...    │ claimed      │ Case FIL-456  │ │
│ │ Jan 3, 2:00 PM     │ system      │ created      │ Filing #789   │ │
│ │ Jan 3, 1:45 PM     │ admin@...   │ role_changed │ Staff #101    │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│ [Export CSV]                                        [<] 1 2 3 [>]   │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 Audit Event Details

Clicking an event shows full details:

```
┌─────────────────────────────────────────────────────────────────────┐
│ Audit Event Details                                              ✕  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│ Event ID: aud_abc123                                                 │
│ Timestamp: January 3, 2026 at 2:30:45 PM                            │
│                                                                      │
│ Actor: Jane Doe (jane@bumara.com)                                   │
│ Actor Type: STAFF                                                    │
│                                                                      │
│ Action: payment_verified                                             │
│ Resource: payment_request #pay_xyz789                               │
│                                                                      │
│ Before State:                                                        │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ {                                                               │ │
│ │   "status": "PAID_UNVERIFIED",                                  │ │
│ │   "verifiedAt": null,                                           │ │
│ │   "verifiedBy": null                                            │ │
│ │ }                                                               │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│ After State:                                                         │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ {                                                               │ │
│ │   "status": "PAID_VERIFIED",                                    │ │
│ │   "verifiedAt": "2026-01-03T14:30:45Z",                         │ │
│ │   "verifiedBy": "usr_jane123"                                   │ │
│ │ }                                                               │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│ Evidence: payment_proof.pdf                            [View]       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.3 Audit API

| Endpoint | Method | Description | Access |
|----------|--------|-------------|--------|
| `GET /audit/global` | GET | List all events | Admin |
| `GET /audit/:id` | GET | Get event detail | Admin |
| `GET /audit/export` | GET | Export CSV | Admin |

---

## Catalog & Rules (Phase 2)

**Route:** `/catalog`  
**Status:** Placeholder / Admin only

### Future Features

- Obligation templates
- Task templates
- SLA policy configuration
- Fee schedules
- Regulator rules

---

## Reports & Analytics (Phase 2)

**Route:** `/reports`  
**Status:** Feature-flagged

### Future Features

- Case volume trends
- SLA compliance metrics
- Staff performance
- Payment summaries
- Regulator breakdown

---

## File Locations

**Current Implementation:**

| Component | Location |
|-----------|----------|
| Admin page | `apps/backoffice/app/(authenticated)/(home)/(general)/admin/page.tsx` |
| Role guards | `apps/backoffice/lib/guards/require-role.ts` |
| Backoffice service | `packages/api-services/src/domains/backoffice/service.ts` |
| Staff schema | `packages/database/src/schema/core/back-office-agents.ts` |
| Audit schema | `packages/database/src/schema/system/audit-logs.ts` |

**To Create:**

| Component | Location |
|-----------|----------|
| Staff management UI | `apps/backoffice/components/admin/staff-management.tsx` |
| Role change modal | `apps/backoffice/components/admin/role-change-modal.tsx` |
| Approval queue | `apps/backoffice/components/admin/approval-queue.tsx` |
| Audit viewer | `apps/backoffice/components/admin/audit-viewer.tsx` |

---

## Related Documentation

- [Architecture](/backoffice/architecture) - Auth model details
- [Specification](/backoffice/specification) - Role definitions
- [Implementation Plan](/backoffice/implementation-plan) - Build steps

