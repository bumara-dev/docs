---
title: "Tasks Module - Data Model"
description: "Database tables, enums, indexes, and relationships for the Tasks module."
---

## Table of Contents

1. [Schema Overview](#schema-overview)
2. [Enums](#enums)
3. [Tables](#tables)
4. [Indexes](#indexes)
5. [Relations](#relations)
6. [Migration Guide](#migration-guide)

---

## Schema Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         TASKS DATA MODEL                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  organizations ◄───────────────────────────────────────────────────┐│
│       │                                                             ││
│       │ 1:N                                                         ││
│       ▼                                                             ││
│  ┌─────────────┐     1:N     ┌────────────────┐                    ││
│  │    tasks    │────────────►│ task_comments  │                    ││
│  │             │             │                │                    ││
│  │  id         │             │ taskId         │                    ││
│  │  orgId      │             │ authorId       │                    ││
│  │  filingId   │             │ content        │                    ││
│  │  srId       │             └────────────────┘                    ││
│  │  status     │                                                   ││
│  │  taskType   │     N:1     ┌─────────────────┐                   ││
│  │  required   │────────────►│ document_links  │                   ││
│  │  ...        │             │ (entityType=    │                   ││
│  └─────────────┘             │  'task')        │                   ││
│       │                      └─────────────────┘                   ││
│       │ N:1                                                        ││
│       ▼                                                            ││
│  ┌─────────────────────────────────────────────────────────────┐   ││
│  │    filings / service_requests                                │   ││
│  │    (exactly one parent per task)                             │   ││
│  └─────────────────────────────────────────────────────────────┘   ││
│                                                                     ││
└─────────────────────────────────────────────────────────────────────┘│
```

---

## Enums

### compliance_task_status (existing)

Task lifecycle status.

| Value | Description |
|-------|-------------|
| `todo` | Not yet started |
| `doing` | Work in progress |
| `blocked` | Cannot proceed, needs intervention |
| `done` | Completed successfully |
| `skipped` | Intentionally not completed |

**File:** `packages/database/src/schema/enums.ts`

```typescript
export const complianceTaskStatusEnum = pgEnum("compliance_task_status", [
  "todo",
  "doing",
  "blocked",
  "done",
  "skipped",
]);
```

### compliance_task_type (new)

Classification of task action required.

| Value | Description |
|-------|-------------|
| `upload_document` | Tenant must upload a file |
| `fill_form` | Tenant must provide data/answers |
| `review_approve` | Tenant must review and confirm |
| `payment_action` | Payment-related action required |
| `info_request` | Backoffice requests additional info |
| `custom` | Free-form task |

**File:** `packages/database/src/schema/enums.ts`

```typescript
export const complianceTaskTypeEnum = pgEnum("compliance_task_type", [
  "upload_document",
  "fill_form",
  "review_approve",
  "payment_action",
  "info_request",
  "custom",
]);
```

---

## Tables

### tasks (updated)

Main tasks table storing checklist items for Filings and Service Requests.

**File:** `packages/database/src/schema/compliance/tasks.ts`

```typescript
import {
  boolean,
  index,
  integer,
  pgTable,
  text,
  timestamp,
  uuid,
} from "drizzle-orm/pg-core";
import { timestamps } from "../../helpers/timestamps";
import { complianceTaskStatusEnum, complianceTaskTypeEnum } from "../enums";
import { organizations } from "../core/organizations";
import { organizationMembers } from "../core/organization-members";
import { filings } from "./filings";
import { serviceRequests } from "./service-requests";

export const tasks = pgTable(
  "tasks",
  {
    // Primary key
    id: uuid("id").primaryKey().defaultRandom(),

    // Organization (required, tenant isolation)
    organizationId: text("organization_id")
      .notNull()
      .references(() => organizations.id, { onDelete: "cascade" }),

    // Parent entity (exactly one must be set)
    filingId: uuid("filing_id").references(() => filings.id, {
      onDelete: "set null",
    }),
    serviceRequestId: uuid("service_request_id").references(
      () => serviceRequests.id,
      { onDelete: "set null" }
    ),

    // Task details
    title: text("title").notNull(),
    description: text("description"),

    // Classification
    taskType: complianceTaskTypeEnum("task_type").default("custom").notNull(),
    required: boolean("required").default(false).notNull(),

    // Status
    status: complianceTaskStatusEnum("status").default("todo").notNull(),

    // Reason fields (required when status = blocked/skipped)
    blockedReason: text("blocked_reason"),
    skipReason: text("skip_reason"),

    // Assignment
    assignedToMemberId: text("assigned_to_member_id").references(
      () => organizationMembers.id,
      { onDelete: "set null" }
    ),

    // Ordering and gating
    sequence: integer("sequence"),
    isBlocking: boolean("is_blocking").default(false).notNull(),

    // Due date
    dueOn: timestamp("due_on", { mode: "date" }),

    // Completion tracking
    completedAt: timestamp("completed_at", { mode: "date" }),
    completedBy: text("completed_by").references(
      () => organizationMembers.id,
      { onDelete: "set null" }
    ),

    // Standard timestamps
    ...timestamps,
  },
  (table) => [
    // Org + status for listing by status per org
    index("idx_tasks_org_status").on(table.organizationId, table.status),

    // Parent entity lookups
    index("idx_tasks_parent").on(table.filingId, table.serviceRequestId),

    // Org + due date for overdue queries
    index("idx_tasks_org_due").on(table.organizationId, table.dueOn),

    // Assignee for "my tasks" view
    index("idx_tasks_assignee").on(table.assignedToMemberId, table.status),

    // Org + required for readiness calculation
    index("idx_tasks_org_required").on(table.organizationId, table.required),
  ]
);

export type Task = typeof tasks.$inferSelect;
export type NewTask = typeof tasks.$inferInsert;
```

### task_comments (new)

Comments/notes on tasks for discussion threads.

**File:** `packages/database/src/schema/compliance/task-comments.ts`

```typescript
import {
  index,
  pgTable,
  text,
  timestamp,
  uuid,
} from "drizzle-orm/pg-core";
import { tasks } from "./tasks";
import { organizations } from "../core/organizations";
import { organizationMembers } from "../core/organization-members";

export const taskComments = pgTable(
  "task_comments",
  {
    id: uuid("id").primaryKey().defaultRandom(),

    // Task reference
    taskId: uuid("task_id")
      .notNull()
      .references(() => tasks.id, { onDelete: "cascade" }),

    // Organization (denormalized for efficient filtering)
    organizationId: text("organization_id")
      .notNull()
      .references(() => organizations.id, { onDelete: "cascade" }),

    // Author
    authorId: text("author_id")
      .notNull()
      .references(() => organizationMembers.id, { onDelete: "set null" }),

    // Content
    content: text("content").notNull(),

    // Timestamp (no updated_at - comments are immutable after creation)
    createdAt: timestamp("created_at", { mode: "date" }).notNull().defaultNow(),
  },
  (table) => [
    // Task timeline (comments on a task)
    index("idx_task_comments_task").on(table.taskId),

    // Org filtering
    index("idx_task_comments_org").on(table.organizationId),

    // Author lookup
    index("idx_task_comments_author").on(table.authorId),

    // Time-ordered retrieval
    index("idx_task_comments_created").on(table.taskId, table.createdAt),
  ]
);

export type TaskComment = typeof taskComments.$inferSelect;
export type NewTaskComment = typeof taskComments.$inferInsert;
```

---

## Indexes

### Index Strategy

| Table | Index | Purpose |
|-------|-------|---------|
| `tasks` | `idx_tasks_org_status` | List tasks by status per org |
| `tasks` | `idx_tasks_parent` | Find tasks for filing/service request |
| `tasks` | `idx_tasks_org_due` | Overdue and due-soon queries |
| `tasks` | `idx_tasks_assignee` | "My tasks" view per user |
| `tasks` | `idx_tasks_org_required` | Readiness calculation |
| `task_comments` | `idx_task_comments_task` | Comments on a task |
| `task_comments` | `idx_task_comments_org` | Org-scoped comment queries |
| `task_comments` | `idx_task_comments_created` | Time-ordered retrieval |

### Query Patterns Covered

1. **List org's tasks with filters:**
   ```sql
   SELECT * FROM tasks
   WHERE organization_id = ? AND status = ?
   ORDER BY due_on ASC NULLS LAST
   LIMIT 20 OFFSET 0;
   ```

2. **Get tasks for a filing:**
   ```sql
   SELECT * FROM tasks
   WHERE filing_id = ? AND organization_id = ?
   ORDER BY sequence ASC NULLS LAST;
   ```

3. **My tasks (assigned to me):**
   ```sql
   SELECT * FROM tasks
   WHERE assigned_to_member_id = ? AND status NOT IN ('done', 'skipped')
   ORDER BY due_on ASC NULLS LAST;
   ```

4. **Check filing readiness:**
   ```sql
   SELECT COUNT(*) as incomplete_required
   FROM tasks
   WHERE filing_id = ?
     AND required = true
     AND status NOT IN ('done');
   -- If count = 0, filing is ready for submission
   ```

5. **Overdue tasks:**
   ```sql
   SELECT * FROM tasks
   WHERE organization_id = ?
     AND due_on < NOW()
     AND status NOT IN ('done', 'skipped')
   ORDER BY due_on ASC;
   ```

6. **Task comments timeline:**
   ```sql
   SELECT * FROM task_comments
   WHERE task_id = ?
   ORDER BY created_at ASC;
   ```

---

## Relations

### Drizzle Relations

**File:** `packages/database/src/schema/compliance/compliance-relations.ts`

Update existing relations:

```typescript
import { relations } from "drizzle-orm";
import { tasks } from "./tasks";
import { taskComments } from "./task-comments";
import { organizations } from "../core/organizations";
import { organizationMembers } from "../core/organization-members";
import { filings } from "./filings";
import { serviceRequests } from "./service-requests";

// Update existing tasksRelations
export const tasksRelations = relations(tasks, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [tasks.organizationId],
    references: [organizations.id],
  }),
  filing: one(filings, {
    fields: [tasks.filingId],
    references: [filings.id],
  }),
  serviceRequest: one(serviceRequests, {
    fields: [tasks.serviceRequestId],
    references: [serviceRequests.id],
  }),
  assignee: one(organizationMembers, {
    fields: [tasks.assignedToMemberId],
    references: [organizationMembers.id],
    relationName: "taskAssignee",
  }),
  completedByMember: one(organizationMembers, {
    fields: [tasks.completedBy],
    references: [organizationMembers.id],
    relationName: "taskCompleter",
  }),
  // New relation
  comments: many(taskComments),
}));

// New relations for task_comments
export const taskCommentsRelations = relations(taskComments, ({ one }) => ({
  task: one(tasks, {
    fields: [taskComments.taskId],
    references: [tasks.id],
  }),
  organization: one(organizations, {
    fields: [taskComments.organizationId],
    references: [organizations.id],
  }),
  author: one(organizationMembers, {
    fields: [taskComments.authorId],
    references: [organizationMembers.id],
  }),
}));
```

---

## Migration Guide

### Step 1: Add New Enum

```sql
-- Add task type enum
CREATE TYPE compliance_task_type AS ENUM (
  'upload_document', 'fill_form', 'review_approve',
  'payment_action', 'info_request', 'custom'
);
```

### Step 2: Alter Tasks Table

```sql
-- Add new columns to tasks table
ALTER TABLE tasks
  ADD COLUMN task_type compliance_task_type NOT NULL DEFAULT 'custom',
  ADD COLUMN required BOOLEAN NOT NULL DEFAULT false,
  ADD COLUMN blocked_reason TEXT,
  ADD COLUMN skip_reason TEXT,
  ADD COLUMN completed_at TIMESTAMP,
  ADD COLUMN completed_by TEXT REFERENCES organization_members(id) ON DELETE SET NULL;

-- Add new indexes
CREATE INDEX idx_tasks_org_due ON tasks(organization_id, due_on);
CREATE INDEX idx_tasks_assignee ON tasks(assigned_to_member_id, status);
CREATE INDEX idx_tasks_org_required ON tasks(organization_id, required);
```

### Step 3: Create Task Comments Table

```sql
CREATE TABLE task_comments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  task_id UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
  organization_id TEXT NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  author_id TEXT NOT NULL REFERENCES organization_members(id) ON DELETE SET NULL,
  content TEXT NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_task_comments_task ON task_comments(task_id);
CREATE INDEX idx_task_comments_org ON task_comments(organization_id);
CREATE INDEX idx_task_comments_author ON task_comments(author_id);
CREATE INDEX idx_task_comments_created ON task_comments(task_id, created_at);
```

### Step 4: Drizzle Migration

Generate and apply migration using Drizzle:

```bash
cd packages/database
pnpm db:generate
pnpm db:migrate
```

### Step 5: Backfill Existing Tasks

```sql
-- Set default task type for existing tasks
UPDATE tasks SET task_type = 'custom' WHERE task_type IS NULL;

-- Set required = false for existing tasks (can be updated manually)
UPDATE tasks SET required = false WHERE required IS NULL;
```

---

## Document Linking

Tasks can have document attachments via the existing `document_links` table.

**Linking a document to a task:**

```typescript
// When uploading a document for a task
await db.insert(documentLinks).values({
  documentId: uploadedDoc.id,
  organizationId: ctx.orgId,
  entityType: 'task', // Note: may need to add 'task' to documentLinkEntityTypeEnum
  entityId: taskId,
  linkedBy: ctx.userId,
});
```

**Note:** The `document_link_entity_type` enum may need to be extended to include `'task'` if not already present.

```typescript
// Add to enums.ts if not present
export const documentLinkEntityTypeEnum = pgEnum("document_link_entity_type", [
  "filing",
  "service_request",
  "ticket",
  "payment_request",
  "regulator_payout",
  "task", // Add this
]);
```


