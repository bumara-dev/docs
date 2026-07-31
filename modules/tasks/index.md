---
title: "Tasks Module"
description: "Backend documentation for the Bumara Tasks module — checklist items for Filings and Service Requests."
---

The Tasks module provides workflow management for compliance preparation. Tasks are the actionable checklist items that tenants and backoffice must complete before a Filing or Service Request can be submitted to regulators.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                          TASKS MODULE                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌──────────────────────────────────────────────────────────────┐  │
│   │                    Parent Entities                            │  │
│   │  ┌─────────────┐              ┌────────────────────┐         │  │
│   │  │   Filing    │              │  Service Request   │         │  │
│   │  │  (Recurring │              │  (One-off workflow)│         │  │
│   │  │  obligation)│              │                    │         │  │
│   │  └──────┬──────┘              └─────────┬──────────┘         │  │
│   │         │                               │                     │  │
│   │         │ 1:N                           │ 1:N                 │  │
│   │         ▼                               ▼                     │  │
│   │  ┌─────────────────────────────────────────────────────────┐ │  │
│   │  │                       TASKS                              │ │  │
│   │  │  id, title, status, required, taskType, dueOn            │ │  │
│   │  │  assignedToMemberId, sequence, isBlocking                │ │  │
│   │  └────────────────────────────┬────────────────────────────┘ │  │
│   │                               │                               │  │
│   └───────────────────────────────┼───────────────────────────────┘  │
│                                   │                                  │
│   ┌───────────────────────────────┼───────────────────────────────┐  │
│   │  Task Dependencies            │                               │  │
│   │                               │                               │  │
│   │   ┌───────────────┐   ┌──────┴───────┐   ┌───────────────┐   │  │
│   │   │ task_comments │   │ document_    │   │  audit_logs   │   │  │
│   │   │               │   │    links     │   │               │   │  │
│   │   │ Discussion    │   │ (via entity  │   │ All state     │   │  │
│   │   │ threads       │   │  type=task)  │   │ changes       │   │  │
│   │   └───────────────┘   └──────────────┘   └───────────────┘   │  │
│   │                                                               │  │
│   └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Core Principle:** Tasks are preparation checklist items. Required tasks must be completed before a Filing or Service Request can become `READY_FOR_SUBMISSION`.

---

## Module Scope

### MVP (Phase 1)

- [x] Tasks table with basic fields (title, status, assignee, due date)
- [x] Link tasks to Filings and Service Requests
- [x] Status workflow: TODO → DOING → DONE | BLOCKED | SKIPPED
- [ ] Required vs optional tasks
- [ ] Task types (upload_document, fill_form, info_request, etc.)
- [ ] Filing readiness gating on required tasks
- [ ] Task comments/notes
- [ ] Document attachments via document_links
- [ ] Status transition validation (server-side)
- [ ] Audit logging for task state changes
- [ ] Multi-tenant isolation in all queries
- [ ] Tenant app: My Tasks, Task drawer, Filing-level task list
- [ ] Backoffice: info-request task creation

### Later (Phase 2+)

- [ ] Task dependencies (block until predecessor done)
- [ ] Task templates per Obligation/Service template
- [ ] Auto-generate tasks when Filing/Request created
- [ ] SLA tracking per task type
- [ ] Due soon / overdue notifications
- [ ] Task assignment notifications
- [ ] Bulk task operations
- [ ] Task analytics dashboard

---

## Documentation Index

| Document | Description |
|----------|-------------|
| [data-model.md](/modules/tasks/data-model) | Database tables, enums, indexes, relationships |
| [api.md](/modules/tasks/api) | API endpoint contracts (Hono RPC) |
| [ui-flows.md](/modules/tasks/ui-flows) | Tenant and backoffice UI workflows |
| [security-permissions.md](/modules/tasks/security-permissions) | Authorization rules, RBAC, audit |
| [jobs-notifications.md](/modules/tasks/jobs-notifications) | Background jobs and notification events |
| [runbook.md](/modules/tasks/runbook) | Operations, troubleshooting, testing |

### Related Documentation

| Document | Description |
|----------|-------------|
| [Documents Module](/modules/documents) | File attachments for tasks |
| [Database Schema](/ARCHITECTURE/DATABASE_SCHEMA) | Full database architecture |
| [API Setup](/API-SETUP) | Hono backend setup guide |

---

## Quick Reference

### Task Statuses

| Status | Description |
|--------|-------------|
| `todo` | Not yet started |
| `doing` | Work in progress |
| `blocked` | Cannot proceed (requires blockedReason) |
| `done` | Completed successfully |
| `skipped` | Intentionally not done (requires skipReason) |

### Task Types

| Type | Description | Example |
|------|-------------|---------|
| `upload_document` | Tenant must upload a file | Financial statements, ID copy |
| `fill_form` | Tenant must provide data | Director details, address info |
| `review_approve` | Tenant reviews and confirms | Declaration, terms acceptance |
| `payment_action` | Payment-related task | Pay regulator fee |
| `info_request` | Backoffice requests info | Clarification question |
| `custom` | Free-form task | Any other preparation step |

### Task Parent Types

| Scope | Parent Field | Description |
|-------|--------------|-------------|
| Filing | `filingId` | Task for a recurring obligation period |
| Service Request | `serviceRequestId` | Task for a one-off request |

> **Invariant:** Every task must have exactly one parent — either `filingId` OR `serviceRequestId`, never both, never neither.

### Key Flows

1. **Create Filing:** Tasks generated from template → tasks appear as TODO
2. **Tenant Works:** Updates tasks TODO → DOING → DONE
3. **Readiness Check:** All required tasks DONE? → Filing becomes READY_FOR_SUBMISSION
4. **Backoffice Info Request:** Creates info_request task → Tenant sees new TODO
5. **Skip Task:** Only with reason, only for optional tasks

---

## Definitions

### Filing Readiness Gating

A Filing or Service Request can only transition to `READY_FOR_SUBMISSION` when:

```typescript
const allRequiredTasksDone = tasks
  .filter(t => t.required)
  .every(t => t.status === 'done');
```

Optional tasks may remain in any status.

### Blocking Tasks

Tasks with `isBlocking = true` prevent the parent (Filing/ServiceRequest) from progressing past `IN_PROGRESS` until the task is `done`.

### Required Tasks

Tasks with `required = true` must be completed before submission. They cannot be skipped (or skipping requires elevated permission + reason).

---

## Getting Started

1. Read [data-model.md](/modules/tasks/data-model) to understand the schema
2. Review [api.md](/modules/tasks/api) for endpoint contracts
3. Check [security-permissions.md](/modules/tasks/security-permissions) for access rules
4. Use [runbook.md](/modules/tasks/runbook) for testing and troubleshooting


