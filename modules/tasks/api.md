---
title: "Tasks Module - API Reference"
description: "Complete API endpoint contracts for the Tasks module."
---

## Base URL

```
/api/v1/tasks
```

All endpoints require authentication via Bearer token (Clerk JWT with `hono` template).

---

## Authentication

All endpoints require:

- **Header:** `Authorization: Bearer <jwt_token>`
- **JWT Claims:** `orgId`, `userId`, `orgRole`

The `orgId` from the JWT is used for all org-scoped queries. Users can only access tasks belonging to their organization.

---

## Endpoints Overview

| Method | Path | Description |
|--------|------|-------------|
| GET | `/tasks` | List tasks (paginated, filtered) |
| GET | `/tasks/:id` | Get single task with details |
| POST | `/tasks` | Create a new task |
| PATCH | `/tasks/:id/status` | Update task status |
| PATCH | `/tasks/:id/assign` | Assign task to member |
| POST | `/tasks/:id/comments` | Add comment to task |
| GET | `/tasks/:id/comments` | List comments for task |
| POST | `/tasks/:id/skip` | Skip an optional task |
| POST | `/tasks/:id/reopen` | Reopen a done/skipped task |
| GET | `/filings/:id/tasks` | List tasks for a filing |
| GET | `/service-requests/:id/tasks` | List tasks for a service request |
| GET | `/filings/:id/readiness` | Check filing task readiness |

---

## GET /tasks

List tasks with optional filters and pagination.

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `status` | enum | - | Filter by status: `todo`, `doing`, `blocked`, `done`, `skipped` |
| `taskType` | enum | - | Filter by type: `upload_document`, `fill_form`, etc. |
| `required` | boolean | - | Filter by required flag |
| `assignedToMe` | boolean | - | Filter to current user's tasks |
| `assignedToMemberId` | string | - | Filter by specific assignee |
| `filingId` | uuid | - | Filter by parent filing |
| `serviceRequestId` | uuid | - | Filter by parent service request |
| `overdue` | boolean | - | Filter to overdue tasks only |
| `dueSoon` | boolean | - | Filter to tasks due within 7 days |
| `limit` | integer | 20 | Results per page (1-100) |
| `offset` | integer | 0 | Pagination offset |
| `sortBy` | string | `dueOn` | Sort field: `dueOn`, `createdAt`, `updatedAt`, `sequence` |
| `sortOrder` | string | `asc` | Sort direction: `asc`, `desc` |

### Example Request

```
GET /tasks?status=todo&assignedToMe=true&limit=10&offset=0
```

### Response (200 OK)

```json
{
  "success": true,
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "organizationId": "org_123",
      "filingId": "661f9500-f39c-52e5-b827-557766551111",
      "serviceRequestId": null,
      "title": "Upload Annual Return Documents",
      "description": "Please upload the signed annual return form",
      "taskType": "upload_document",
      "required": true,
      "status": "todo",
      "blockedReason": null,
      "skipReason": null,
      "assignedToMemberId": "user_abc",
      "sequence": 1,
      "isBlocking": true,
      "dueOn": "2025-12-15T00:00:00.000Z",
      "completedAt": null,
      "completedBy": null,
      "createdAt": "2025-11-01T09:00:00.000Z",
      "updatedAt": "2025-11-01T09:00:00.000Z"
    }
  ],
  "pagination": {
    "limit": 10,
    "offset": 0,
    "total": 42
  }
}
```

---

## GET /tasks/:id

Get a single task by ID with full details.

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | uuid | Task ID |

### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "organizationId": "org_123",
    "filingId": "661f9500-f39c-52e5-b827-557766551111",
    "serviceRequestId": null,
    "title": "Upload Annual Return Documents",
    "description": "Please upload the signed annual return form",
    "taskType": "upload_document",
    "required": true,
    "status": "todo",
    "blockedReason": null,
    "skipReason": null,
    "assignedToMemberId": "user_abc",
    "sequence": 1,
    "isBlocking": true,
    "dueOn": "2025-12-15T00:00:00.000Z",
    "completedAt": null,
    "completedBy": null,
    "createdAt": "2025-11-01T09:00:00.000Z",
    "updatedAt": "2025-11-01T09:00:00.000Z",
    "assignee": {
      "id": "user_abc",
      "firstName": "John",
      "lastName": "Doe",
      "email": "john@example.com"
    },
    "filing": {
      "id": "661f9500-f39c-52e5-b827-557766551111",
      "periodStart": "2025-01-01",
      "periodEnd": "2025-12-31",
      "status": "in_progress"
    },
    "commentsCount": 3,
    "documentsCount": 0
  }
}
```

### Errors

| Status | Code | Description |
|--------|------|-------------|
| 404 | `NOT_FOUND` | Task not found or wrong org |

---

## POST /tasks

Create a new task. Typically called by backoffice for info requests.

### Request

```json
{
  "filingId": "661f9500-f39c-52e5-b827-557766551111",
  "title": "Please provide updated bank statements",
  "description": "We need the last 3 months of bank statements to verify...",
  "taskType": "info_request",
  "required": false,
  "dueOn": "2025-12-20T00:00:00.000Z",
  "assignedToMemberId": "user_abc"
}
```

### Request Body Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `filingId` | uuid | One of | Parent filing ID |
| `serviceRequestId` | uuid | One of | Parent service request ID |
| `title` | string | Yes | Task title (1-255 chars) |
| `description` | string | No | Task description |
| `taskType` | enum | Yes | Task type |
| `required` | boolean | No | Is this task required? (default: false) |
| `isBlocking` | boolean | No | Does this block parent progress? (default: false) |
| `dueOn` | timestamp | No | Due date |
| `assignedToMemberId` | string | No | Assignee member ID |
| `sequence` | integer | No | Display order |

**Invariant:** Exactly one of `filingId` or `serviceRequestId` must be provided.

### Response (201 Created)

```json
{
  "success": true,
  "data": {
    "id": "772g0500-g40d-63f6-c938-668877662222",
    "title": "Please provide updated bank statements",
    "status": "todo",
    "taskType": "info_request",
    "createdAt": "2025-12-10T14:30:00.000Z"
  },
  "message": "Task created successfully"
}
```

### Errors

| Status | Code | Description |
|--------|------|-------------|
| 400 | `BAD_REQUEST` | Missing required fields |
| 400 | `BAD_REQUEST` | Both filingId and serviceRequestId provided |
| 403 | `FORBIDDEN` | User cannot create tasks (role check) |
| 404 | `NOT_FOUND` | Parent filing/service request not found |

---

## PATCH /tasks/:id/status

Update task status with transition validation.

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | uuid | Task ID |

### Request

```json
{
  "status": "doing"
}
```

For blocked status, include reason:

```json
{
  "status": "blocked",
  "blockedReason": "Waiting for client to provide missing information"
}
```

### Request Body Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `status` | enum | Yes | New status: `todo`, `doing`, `blocked`, `done` |
| `blockedReason` | string | Conditional | Required if status = `blocked` |

### Status Transition Rules

| From | Allowed To |
|------|------------|
| `todo` | `doing`, `done`, `blocked` |
| `doing` | `todo`, `done`, `blocked` |
| `blocked` | `todo`, `doing`, `done` |
| `done` | (use /reopen endpoint) |
| `skipped` | (use /reopen endpoint) |

### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "doing",
    "updatedAt": "2025-12-10T15:00:00.000Z"
  },
  "message": "Task status updated"
}
```

### Errors

| Status | Code | Description |
|--------|------|-------------|
| 400 | `BAD_REQUEST` | Invalid status value |
| 400 | `BAD_REQUEST` | Missing blockedReason for blocked status |
| 404 | `NOT_FOUND` | Task not found |
| 409 | `CONFLICT` | Invalid status transition |

---

## PATCH /tasks/:id/assign

Assign or reassign a task to an organization member.

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | uuid | Task ID |

### Request

```json
{
  "assignedToMemberId": "user_xyz"
}
```

To unassign:

```json
{
  "assignedToMemberId": null
}
```

### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "assignedToMemberId": "user_xyz",
    "updatedAt": "2025-12-10T15:30:00.000Z"
  },
  "message": "Task assigned"
}
```

### Errors

| Status | Code | Description |
|--------|------|-------------|
| 403 | `FORBIDDEN` | User cannot assign tasks (not admin/manager) |
| 404 | `NOT_FOUND` | Task or member not found |

---

## POST /tasks/:id/comments

Add a comment to a task.

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | uuid | Task ID |

### Request

```json
{
  "content": "I've uploaded the first two statements. Still waiting on December's statement from the bank."
}
```

### Response (201 Created)

```json
{
  "success": true,
  "data": {
    "id": "883h1600-h51e-74g7-d049-779988773333",
    "taskId": "550e8400-e29b-41d4-a716-446655440000",
    "authorId": "user_abc",
    "content": "I've uploaded the first two statements...",
    "createdAt": "2025-12-10T16:00:00.000Z"
  },
  "message": "Comment added"
}
```

---

## GET /tasks/:id/comments

List comments for a task.

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | uuid | Task ID |

### Response (200 OK)

```json
{
  "success": true,
  "data": [
    {
      "id": "883h1600-h51e-74g7-d049-779988773333",
      "taskId": "550e8400-e29b-41d4-a716-446655440000",
      "authorId": "user_abc",
      "content": "I've uploaded the first two statements...",
      "createdAt": "2025-12-10T16:00:00.000Z",
      "author": {
        "id": "user_abc",
        "firstName": "John",
        "lastName": "Doe"
      }
    }
  ]
}
```

---

## POST /tasks/:id/skip

Skip an optional task with required reason.

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | uuid | Task ID |

### Request

```json
{
  "skipReason": "Not applicable for this filing period - no changes to report"
}
```

### Request Body Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `skipReason` | string | Yes | Reason for skipping (1-500 chars) |

### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "skipped",
    "skipReason": "Not applicable for this filing period...",
    "updatedAt": "2025-12-10T17:00:00.000Z"
  },
  "message": "Task skipped"
}
```

### Errors

| Status | Code | Description |
|--------|------|-------------|
| 400 | `BAD_REQUEST` | Missing skip reason |
| 409 | `CONFLICT` | Cannot skip required tasks |
| 409 | `CONFLICT` | Task already done/skipped |

---

## POST /tasks/:id/reopen

Reopen a done or skipped task.

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | uuid | Task ID |

### Request

```json
{
  "reason": "Additional information needed after review"
}
```

### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "todo",
    "completedAt": null,
    "completedBy": null,
    "skipReason": null,
    "updatedAt": "2025-12-10T18:00:00.000Z"
  },
  "message": "Task reopened"
}
```

### Errors

| Status | Code | Description |
|--------|------|-------------|
| 403 | `FORBIDDEN` | Only admin/manager can reopen |
| 409 | `CONFLICT` | Task not in done/skipped status |

---

## GET /filings/:id/tasks

List all tasks for a specific filing.

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | uuid | Filing ID |

### Response (200 OK)

Same format as `GET /tasks` but filtered to the filing.

---

## GET /filings/:id/readiness

Check if a filing is ready for submission based on task completion.

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | uuid | Filing ID |

### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "isReady": false,
    "totalTasks": 5,
    "completedTasks": 3,
    "totalRequired": 3,
    "completedRequired": 2,
    "blockedTasks": [],
    "pendingRequired": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "title": "Upload Annual Return Documents",
        "status": "todo"
      }
    ]
  }
}
```

---

## Error Response Format

All errors follow a consistent format:

```json
{
  "success": false,
  "error": {
    "code": "CONFLICT",
    "message": "Cannot skip required tasks",
    "hint": "Mark the task as done instead, or contact an admin to change the required flag",
    "requestId": "req_abc123"
  }
}
```

### Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `BAD_REQUEST` | 400 | Invalid input data |
| `UNAUTHORIZED` | 401 | Missing or invalid auth |
| `FORBIDDEN` | 403 | Permission denied |
| `NOT_FOUND` | 404 | Resource not found |
| `CONFLICT` | 409 | State conflict (invalid transition, skip required) |
| `INTERNAL` | 500 | Server error |

---

## Rate Limits

| Endpoint | Limit |
|----------|-------|
| List/Get | 1000/hour per org |
| Create/Update | 200/hour per org |
| Comments | 500/hour per org |

When rate limited, the API returns `429 Too Many Requests` with a `Retry-After` header.


