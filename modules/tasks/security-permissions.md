---
title: "Tasks Module - Security & Permissions"
description: "Authorization rules, tenant isolation, status transition guards, and audit requirements for the Tasks module."
---

## Table of Contents

1. [Security Principles](#security-principles)
2. [Tenant Isolation](#tenant-isolation)
3. [Role-Based Access Control](#role-based-access-control)
4. [Status Transition Guards](#status-transition-guards)
5. [Access Control Matrix](#access-control-matrix)
6. [Implementation Patterns](#implementation-patterns)
7. [Audit Trail](#audit-trail)

---

## Security Principles

### Core Rules

1. **Org isolation is mandatory** — Every query filters by `organization_id`
2. **Derive org from session** — Never accept `orgId` from client request body
3. **Validate transitions server-side** — Don't trust client status changes
4. **Audit sensitive actions** — Skip, reopen, reassign must be logged
5. **Reason required for blocks/skips** — Cannot change to blocked/skipped without explanation

### Defense in Depth

```
┌─────────────────────────────────────────────────────────────────┐
│                     SECURITY LAYERS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Authentication (Clerk JWT)                                   │
│     └─ Valid token required for all endpoints                    │
│                                                                  │
│  2. Organization Context                                         │
│     └─ orgId extracted from JWT, not request body                │
│                                                                  │
│  3. Database Query Filtering                                     │
│     └─ WHERE organization_id = ? on every query                  │
│                                                                  │
│  4. Role-Based Access Control                                    │
│     └─ Permissions checked per action                            │
│                                                                  │
│  5. Status Transition Guards                                     │
│     └─ Server validates from→to transitions                      │
│                                                                  │
│  6. Business Rule Enforcement                                    │
│     └─ Required tasks cannot be skipped                          │
│     └─ Blocked/skipped need reasons                              │
│                                                                  │
│  7. Audit Logging                                                │
│     └─ Sensitive actions recorded with actor + timestamp         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Tenant Isolation

### The Golden Rule

> **Every database query MUST filter by `organization_id` derived from the authenticated session.**

### Implementation Pattern

```typescript
// ✅ CORRECT: Org ID from session context
export async function getTask(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  taskId: string
) {
  const orgId = ctx.orgId; // From JWT, NOT from request

  const task = await deps.db.query.tasks.findFirst({
    where: and(
      eq(tasks.id, taskId),
      eq(tasks.organizationId, orgId) // REQUIRED
    ),
  });

  if (!task) {
    throw new NotFoundError("Task not found");
  }

  return task;
}

// ❌ WRONG: Accepting orgId from request
export async function getTaskWRONG(
  deps: ServiceDependencies,
  taskId: string,
  orgId: string // NEVER accept org from request body
) {
  // This allows cross-org access!
}
```

### Query Patterns

Every task query must include org filter:

```typescript
// List tasks
const taskList = await db
  .select()
  .from(tasks)
  .where(
    and(
      eq(tasks.organizationId, ctx.orgId), // REQUIRED
      eq(tasks.status, "todo")
    )
  );

// Update task
await db
  .update(tasks)
  .set({ status: "doing" })
  .where(
    and(
      eq(tasks.id, taskId),
      eq(tasks.organizationId, ctx.orgId) // REQUIRED
    )
  );

// Delete comment
await db
  .delete(taskComments)
  .where(
    and(
      eq(taskComments.id, commentId),
      eq(taskComments.organizationId, ctx.orgId) // REQUIRED
    )
  );
```

---

## Role-Based Access Control

### User Roles

From `packages/database/src/schema/enums.ts`:

| Role | Description | Task Access |
|------|-------------|-------------|
| `admin` | Organization administrator | Full access |
| `manager` | Team manager | Full access except skip required |
| `member` | Regular team member | Own tasks + view org tasks |
| `backoffice_admin` | Backoffice administrator | Full + create info requests |
| `backoffice_manager` | Backoffice manager | Full + create info requests |
| `backoffice_member` | Backoffice agent | View + create info requests |

### Permission Definitions

```typescript
type TaskPermission =
  | "tasks:view"           // List and get tasks
  | "tasks:view_org"       // View all org tasks (not just own)
  | "tasks:create"         // Create new tasks
  | "tasks:update_status"  // Change task status
  | "tasks:assign"         // Assign/reassign tasks
  | "tasks:skip"           // Skip optional tasks
  | "tasks:skip_required"  // Skip required tasks (special permission)
  | "tasks:reopen"         // Reopen done/skipped tasks
  | "tasks:comment"        // Add comments
  | "tasks:admin";         // All permissions

const ROLE_PERMISSIONS: Record<string, TaskPermission[]> = {
  // Tenant roles
  admin: ["tasks:admin"],
  manager: [
    "tasks:view",
    "tasks:view_org",
    "tasks:create",
    "tasks:update_status",
    "tasks:assign",
    "tasks:skip",
    "tasks:reopen",
    "tasks:comment",
  ],
  member: [
    "tasks:view",
    "tasks:view_org",
    "tasks:update_status",
    "tasks:comment",
  ],

  // Backoffice roles
  backoffice_admin: ["tasks:admin"],
  backoffice_manager: [
    "tasks:view",
    "tasks:view_org",
    "tasks:create",
    "tasks:update_status",
    "tasks:assign",
    "tasks:skip",
    "tasks:reopen",
    "tasks:comment",
  ],
  backoffice_member: [
    "tasks:view",
    "tasks:view_org",
    "tasks:create", // Can create info_request tasks
    "tasks:comment",
  ],
};
```

### Permission Check Helper

```typescript
function hasPermission(
  userRole: string,
  permission: TaskPermission
): boolean {
  const permissions = ROLE_PERMISSIONS[userRole] || [];

  if (permissions.includes("tasks:admin")) {
    return true; // Admin has all permissions
  }

  return permissions.includes(permission);
}

// Usage in handler
if (!hasPermission(ctx.orgRole, "tasks:assign")) {
  throw new ForbiddenError("You do not have permission to assign tasks");
}
```

### Own Task vs Org Task

Members can update status of their own assigned tasks:

```typescript
function canUpdateTaskStatus(ctx: ServiceContext, task: Task): boolean {
  // Admins and managers can update any task
  if (hasPermission(ctx.orgRole, "tasks:assign")) {
    return true;
  }

  // Members can only update tasks assigned to them
  if (task.assignedToMemberId === ctx.userId) {
    return true;
  }

  // Members can update unassigned tasks
  if (task.assignedToMemberId === null) {
    return true;
  }

  return false;
}
```

---

## Status Transition Guards

### Transition Matrix

```
            ┌─────────────────────────────────────────────────────┐
            │                    TO STATUS                         │
            ├─────────┬─────────┬─────────┬─────────┬─────────────┤
            │  todo   │  doing  │ blocked │  done   │  skipped    │
┌───────────┼─────────┼─────────┼─────────┼─────────┼─────────────┤
│ FROM      │         │         │         │         │             │
├───────────┼─────────┼─────────┼─────────┼─────────┼─────────────┤
│ todo      │    -    │    ✓    │    ✓    │    ✓    │ ✓ (optional)│
├───────────┼─────────┼─────────┼─────────┼─────────┼─────────────┤
│ doing     │    ✓    │    -    │    ✓    │    ✓    │ ✓ (optional)│
├───────────┼─────────┼─────────┼─────────┼─────────┼─────────────┤
│ blocked   │    ✓    │    ✓    │    -    │    ✓    │ ✓ (optional)│
├───────────┼─────────┼─────────┼─────────┼─────────┼─────────────┤
│ done      │    ✓*   │    -    │    -    │    -    │      -      │
├───────────┼─────────┼─────────┼─────────┼─────────┼─────────────┤
│ skipped   │    ✓*   │    -    │    -    │    -    │      -      │
└───────────┴─────────┴─────────┴─────────┴─────────┴─────────────┘

✓  = Allowed
✓* = Allowed via /reopen endpoint only (requires admin/manager)
-  = Not allowed
```

### Transition Validation

```typescript
interface TransitionRule {
  to: TaskStatus;
  requiresReason?: boolean;
  requiredPermission?: TaskPermission;
  condition?: (task: Task) => boolean;
}

const ALLOWED_TRANSITIONS: Record<TaskStatus, TransitionRule[]> = {
  todo: [
    { to: "doing" },
    { to: "blocked", requiresReason: true },
    { to: "done" },
    {
      to: "skipped",
      requiresReason: true,
      condition: (task) => !task.required,
    },
  ],
  doing: [
    { to: "todo" },
    { to: "blocked", requiresReason: true },
    { to: "done" },
    {
      to: "skipped",
      requiresReason: true,
      condition: (task) => !task.required,
    },
  ],
  blocked: [
    { to: "todo" },
    { to: "doing" },
    { to: "done" },
    {
      to: "skipped",
      requiresReason: true,
      condition: (task) => !task.required,
    },
  ],
  done: [
    { to: "todo", requiredPermission: "tasks:reopen" },
  ],
  skipped: [
    { to: "todo", requiredPermission: "tasks:reopen" },
  ],
};

function validateTransition(
  ctx: ServiceContext,
  task: Task,
  toStatus: TaskStatus,
  reason?: string
): void {
  const rules = ALLOWED_TRANSITIONS[task.status];
  const rule = rules?.find((r) => r.to === toStatus);

  if (!rule) {
    throw new ConflictError(
      `Cannot transition from ${task.status} to ${toStatus}`
    );
  }

  if (rule.requiresReason && !reason) {
    throw new BadRequestError(`Reason required when setting status to ${toStatus}`);
  }

  if (rule.requiredPermission && !hasPermission(ctx.orgRole, rule.requiredPermission)) {
    throw new ForbiddenError(`You do not have permission to ${toStatus} tasks`);
  }

  if (rule.condition && !rule.condition(task)) {
    if (toStatus === "skipped") {
      throw new ConflictError("Cannot skip required tasks");
    }
    throw new ConflictError(`Transition condition not met`);
  }
}
```

### Skip Required Tasks

Skipping a required task requires elevated permission:

```typescript
function canSkipTask(ctx: ServiceContext, task: Task): boolean {
  if (!task.required) {
    // Optional tasks can be skipped by anyone
    return hasPermission(ctx.orgRole, "tasks:skip");
  }

  // Required tasks need special permission
  return hasPermission(ctx.orgRole, "tasks:skip_required");
}
```

---

## Access Control Matrix

### By Action

| Action | admin | manager | member | backoffice_admin | backoffice_manager | backoffice_member |
|--------|-------|---------|--------|------------------|--------------------|--------------------|
| View own tasks | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| View org tasks | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Create task | ✓ | ✓ | ✗ | ✓ | ✓ | ✓ (info_request only) |
| Update status (own) | ✓ | ✓ | ✓ | ✓ | ✓ | ✗ |
| Update status (any) | ✓ | ✓ | ✗ | ✓ | ✓ | ✗ |
| Assign task | ✓ | ✓ | ✗ | ✓ | ✓ | ✗ |
| Skip optional | ✓ | ✓ | ✗ | ✓ | ✓ | ✗ |
| Skip required | ✓ | ✗ | ✗ | ✓ | ✗ | ✗ |
| Reopen task | ✓ | ✓ | ✗ | ✓ | ✓ | ✗ |
| Add comment | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

### By Task Status

| Status | View | Update | Skip | Reopen | Comment |
|--------|------|--------|------|--------|---------|
| `todo` | ✓ | ✓ | ✓ | - | ✓ |
| `doing` | ✓ | ✓ | ✓ | - | ✓ |
| `blocked` | ✓ | ✓ | ✓ | - | ✓ |
| `done` | ✓ | ✗ | ✗ | ✓ | ✓ |
| `skipped` | ✓ | ✗ | ✗ | ✓ | ✓ |

---

## Implementation Patterns

### Middleware: Require Auth + Org

```typescript
// packages/backend/src/core/middleware/auth.ts

export const requireAuth: MiddlewareHandler = async (c, next) => {
  const token = c.req.header("Authorization")?.replace("Bearer ", "");

  if (!token) {
    return c.json({ error: { code: "UNAUTHORIZED" } }, 401);
  }

  const claims = await verifyClerkToken(token);

  c.set("userId", claims.userId);
  c.set("orgId", claims.orgId);
  c.set("orgRole", claims.orgRole);

  await next();
};

export const requireOrg: MiddlewareHandler = async (c, next) => {
  const orgId = c.get("orgId");

  if (!orgId) {
    return c.json({
      error: {
        code: "FORBIDDEN",
        message: "Organization context required"
      }
    }, 403);
  }

  await next();
};
```

### Service Context Pattern

```typescript
// packages/api-services/src/core/context.ts

export interface ServiceContext {
  userId: string;
  orgId: string;
  orgRole: string;
  requestId?: string;
  ipAddress?: string;
  userAgent?: string;
}

// Build from Hono context
export function buildServiceContext(c: Context): ServiceContext {
  return {
    userId: c.get("userId"),
    orgId: c.get("orgId"),
    orgRole: c.get("orgRole"),
    requestId: c.get("requestId"),
    ipAddress: c.req.header("CF-Connecting-IP"),
    userAgent: c.req.header("User-Agent"),
  };
}
```

### Error Responses

```typescript
// Unauthorized (missing/invalid token)
{
  "success": false,
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Authentication required"
  }
}

// Forbidden (valid auth, wrong permissions)
{
  "success": false,
  "error": {
    "code": "FORBIDDEN",
    "message": "You do not have permission to assign tasks"
  }
}

// Not Found (task doesn't exist OR belongs to different org)
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "Task not found"
  }
}

// Conflict (invalid transition)
{
  "success": false,
  "error": {
    "code": "CONFLICT",
    "message": "Cannot skip required tasks",
    "hint": "Mark the task as done instead"
  }
}
```

---

## Audit Trail

### Actions Requiring Audit

| Action | Audit Required | Reason Storage |
|--------|----------------|----------------|
| Task created | ✓ | - |
| Status changed | ✓ | In metadata if blocked/skipped |
| Task assigned | ✓ | Previous assignee in metadata |
| Task skipped | ✓ | Skip reason in metadata |
| Task reopened | ✓ | Reopen reason in metadata |
| Comment added | ✗ | Comments are self-auditing |

### Audit Log Entry

```typescript
interface TaskAuditEntry {
  organizationId: string;
  userId: string;
  actorType: "TENANT" | "STAFF" | "SYSTEM";
  action: "create" | "update" | "delete";
  entityType: "task";
  entityId: string;
  changes: {
    before?: Partial<Task>;
    after?: Partial<Task>;
  };
  metadata?: {
    reason?: string;
    previousAssignee?: string;
    triggeredBy?: string; // e.g., "filing_status_change"
  };
  timestamp: Date;
}
```

### Audit Logging Implementation

```typescript
// In tasks.service.ts

async function updateTaskStatus(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  taskId: string,
  input: UpdateTaskStatusInput
): Promise<Task> {
  const task = await getTask(ctx, deps, taskId);

  validateTransition(ctx, task, input.status, input.blockedReason);

  const updates: Partial<Task> = {
    status: input.status,
    updatedAt: new Date(),
  };

  if (input.status === "blocked") {
    updates.blockedReason = input.blockedReason;
  }

  if (input.status === "done") {
    updates.completedAt = new Date();
    updates.completedBy = ctx.userId;
  }

  await deps.db
    .update(tasks)
    .set(updates)
    .where(
      and(
        eq(tasks.id, taskId),
        eq(tasks.organizationId, ctx.orgId)
      )
    );

  // Audit log
  await recordAuditLog(ctx, deps, {
    action: "update",
    entityType: "task",
    entityId: taskId,
    changes: {
      before: { status: task.status },
      after: { status: input.status },
    },
    metadata: {
      reason: input.blockedReason || input.skipReason,
    },
  });

  return { ...task, ...updates };
}
```

### Audit Query Examples

```sql
-- All task changes for an org in the last 24 hours
SELECT * FROM audit_logs
WHERE organization_id = ?
  AND entity_type = 'task'
  AND timestamp > NOW() - INTERVAL '24 hours'
ORDER BY timestamp DESC;

-- Who skipped tasks and why
SELECT
  al.entity_id as task_id,
  al.user_id,
  al.metadata->>'reason' as skip_reason,
  al.timestamp
FROM audit_logs al
WHERE al.organization_id = ?
  AND al.entity_type = 'task'
  AND al.changes->'after'->>'status' = 'skipped'
ORDER BY al.timestamp DESC;

-- Task assignment history
SELECT
  al.entity_id as task_id,
  al.changes->'before'->>'assignedToMemberId' as from_assignee,
  al.changes->'after'->>'assignedToMemberId' as to_assignee,
  al.user_id as changed_by,
  al.timestamp
FROM audit_logs al
WHERE al.organization_id = ?
  AND al.entity_type = 'task'
  AND al.changes->'before' ? 'assignedToMemberId'
ORDER BY al.timestamp DESC;
```


