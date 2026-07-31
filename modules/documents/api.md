---
title: "Documents Module - API Reference"
description: "Complete API endpoint contracts for the Documents module."
---

## Base URL

```
/api/v1/documents
```

All endpoints require authentication via Bearer token (Clerk JWT with `hono` template).

---

## Authentication

All endpoints require:

- **Header:** `Authorization: Bearer <jwt_token>`
- **JWT Claims:** `orgId`, `userId`, `orgRole`

The `orgId` from the JWT is used for all org-scoped queries. Users can only access documents belonging to their organization.

---

## Endpoints Overview

| Method | Path | Description |
|--------|------|-------------|
| POST | `/documents/uploads/init` | Initialize upload, get presigned URL |
| POST | `/documents/uploads/complete` | Confirm upload completion |
| GET | `/documents` | List documents (paginated) |
| GET | `/documents/:id` | Get single document |
| POST | `/documents/:id/download-url` | Get presigned download URL |
| POST | `/documents/:id/link` | Link document to entity |
| DELETE | `/documents/:id/link/:linkId` | Remove document link |
| POST | `/documents/:id/archive` | Archive (soft-delete) document |
| POST | `/documents/:id/lock` | Lock document (backoffice only) |

---

## POST /documents/uploads/init

Initialize a document upload and receive a presigned S3 URL.

### Request

```json
{
  "filename": "invoice-2024-01.pdf",
  "mimeType": "application/pdf",
  "sizeBytes": 245678,
  "kind": "source",
  "regulatorId": "uuid-optional",
  "filingId": "uuid-optional",
  "serviceRequestId": "uuid-optional",
  "metadata": {
    "description": "January invoice",
    "tags": ["vat", "2024"]
  }
}
```

### Request Body Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `filename` | string | Yes | Original filename (1-255 chars) |
| `mimeType` | string | Yes | MIME type (e.g., `application/pdf`) |
| `sizeBytes` | integer | Yes | File size in bytes (max 50MB) |
| `kind` | enum | Yes | Document kind: `source`, `workpaper`, `submission`, `receipt`, `certificate` |
| `regulatorId` | uuid | No | Associated regulator ID |
| `filingId` | uuid | No | Associated filing ID |
| `serviceRequestId` | uuid | No | Associated service request ID |
| `metadata` | object | No | Custom metadata key-value pairs |

### Response (201 Created)

```json
{
  "success": true,
  "data": {
    "documentId": "550e8400-e29b-41d4-a716-446655440000",
    "uploadUrl": "https://bumara-documents.s3.af-south-1.amazonaws.com/org_123/550e8400.../invoice.pdf?X-Amz-Algorithm=...",
    "expiresAt": "2024-01-15T10:30:00.000Z"
  }
}
```

### Client Upload Flow

After receiving the response, the client must:

1. **PUT** the file directly to `uploadUrl` with headers:
   - `Content-Type: <mimeType from request>`
   - `Content-Length: <sizeBytes from request>`
2. Call `POST /documents/uploads/complete` with `documentId`

### Errors

| Status | Code | Description |
|--------|------|-------------|
| 400 | `BAD_REQUEST` | Invalid input (missing fields, invalid MIME type) |
| 400 | `BAD_REQUEST` | File too large (> 50MB) |
| 401 | `UNAUTHORIZED` | Missing or invalid auth token |
| 403 | `FORBIDDEN` | Org access denied |

---

## POST /documents/uploads/complete

Confirm that a document upload has completed successfully.

### Request

```json
{
  "documentId": "550e8400-e29b-41d4-a716-446655440000"
}
```

### Request Body Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `documentId` | uuid | Yes | Document ID from init response |

### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "organizationId": "org_123",
    "regulatorId": null,
    "filingId": null,
    "serviceRequestId": null,
    "module": null,
    "kind": "source",
    "status": "active",
    "storageKey": "org_123/550e8400.../invoice.pdf",
    "filename": "invoice-2024-01.pdf",
    "mimeType": "application/pdf",
    "sizeBytes": 245678,
    "metadata": {
      "description": "January invoice"
    },
    "uploadedAt": "2024-01-15T09:35:00.000Z",
    "lockedAt": null,
    "lockedBy": null,
    "lockedReason": null,
    "archivedAt": null,
    "createdAt": "2024-01-15T09:30:00.000Z",
    "updatedAt": "2024-01-15T09:35:00.000Z"
  },
  "message": "Upload completed successfully"
}
```

### Errors

| Status | Code | Description |
|--------|------|-------------|
| 404 | `NOT_FOUND` | Document not found or wrong org |
| 409 | `CONFLICT` | Document not in `uploading` status |
| 409 | `CONFLICT` | File not found in S3 (upload failed) |

---

## GET /documents

List documents with optional filters and pagination.

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `kind` | enum | - | Filter by kind |
| `status` | enum | - | Filter by status (default excludes `archived`) |
| `regulatorId` | uuid | - | Filter by regulator |
| `filingId` | uuid | - | Filter by filing |
| `serviceRequestId` | uuid | - | Filter by service request |
| `limit` | integer | 20 | Results per page (1-100) |
| `offset` | integer | 0 | Pagination offset |

### Example Request

```
GET /documents?kind=source&limit=10&offset=0
```

### Response (200 OK)

```json
{
  "success": true,
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "organizationId": "org_123",
      "kind": "source",
      "status": "active",
      "filename": "invoice-2024-01.pdf",
      "mimeType": "application/pdf",
      "sizeBytes": 245678,
      "uploadedAt": "2024-01-15T09:35:00.000Z",
      "createdAt": "2024-01-15T09:30:00.000Z",
      "updatedAt": "2024-01-15T09:35:00.000Z"
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

## GET /documents/:id

Get a single document by ID.

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | uuid | Document ID |

### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "organizationId": "org_123",
    "regulatorId": "uuid",
    "filingId": "uuid",
    "serviceRequestId": null,
    "module": "compliance",
    "kind": "submission",
    "status": "active",
    "storageKey": "org_123/550e8400.../submission.pdf",
    "filename": "submission.pdf",
    "mimeType": "application/pdf",
    "sizeBytes": 1024567,
    "metadata": {},
    "uploadedAt": "2024-01-15T09:35:00.000Z",
    "lockedAt": "2024-01-20T14:00:00.000Z",
    "lockedBy": "system",
    "lockedReason": "Auto-locked: Filing accepted by regulator",
    "archivedAt": null,
    "createdAt": "2024-01-15T09:30:00.000Z",
    "updatedAt": "2024-01-20T14:00:00.000Z"
  }
}
```

### Errors

| Status | Code | Description |
|--------|------|-------------|
| 404 | `NOT_FOUND` | Document not found or wrong org |

---

## POST /documents/:id/download-url

Generate a presigned URL for downloading the document.

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | uuid | Document ID |

### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "downloadUrl": "https://bumara-documents.s3.af-south-1.amazonaws.com/org_123/550e8400.../invoice.pdf?X-Amz-Algorithm=...",
    "expiresAt": "2024-01-15T09:40:00.000Z"
  }
}
```

**Note:** Download URLs expire in 5 minutes by default. The client should use the URL immediately.

### Errors

| Status | Code | Description |
|--------|------|-------------|
| 404 | `NOT_FOUND` | Document not found or wrong org |
| 409 | `CONFLICT` | Document not in `active` status |

---

## POST /documents/:id/link

Link a document to an entity (filing, service request, etc.).

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | uuid | Document ID |

### Request

```json
{
  "entityType": "filing",
  "entityId": "uuid-of-filing"
}
```

### Request Body Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `entityType` | enum | Yes | `filing`, `service_request`, `ticket`, `payment_request`, `regulator_payout` |
| `entityId` | uuid | Yes | ID of the target entity |

### Response (201 Created)

```json
{
  "success": true,
  "data": {
    "linkId": "link-uuid",
    "documentId": "550e8400-e29b-41d4-a716-446655440000",
    "entityType": "filing",
    "entityId": "uuid-of-filing",
    "linkedBy": "user_abc",
    "linkedAt": "2024-01-15T10:00:00.000Z"
  },
  "message": "Document linked successfully"
}
```

### Errors

| Status | Code | Description |
|--------|------|-------------|
| 404 | `NOT_FOUND` | Document or entity not found |
| 409 | `CONFLICT` | Document is locked (cannot add links) |
| 409 | `CONFLICT` | Link already exists |

---

## DELETE /documents/:id/link/:linkId

Remove a document link.

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | uuid | Document ID |
| `linkId` | uuid | Link ID to remove |

### Response (200 OK)

```json
{
  "success": true,
  "message": "Link removed successfully"
}
```

### Errors

| Status | Code | Description |
|--------|------|-------------|
| 404 | `NOT_FOUND` | Document or link not found |
| 409 | `CONFLICT` | Document is locked (cannot remove links) |

---

## POST /documents/:id/archive

Archive (soft-delete) a document.

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | uuid | Document ID |

### Request

```json
{
  "reason": "Replaced with updated version"
}
```

### Request Body Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `reason` | string | No | Reason for archiving (1-500 chars) |

### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "archived",
    "archivedAt": "2024-01-15T11:00:00.000Z"
  },
  "message": "Document archived successfully"
}
```

### Errors

| Status | Code | Description |
|--------|------|-------------|
| 404 | `NOT_FOUND` | Document not found |
| 409 | `CONFLICT` | Document is locked (cannot archive) |

---

## POST /documents/:id/lock

Lock a document to prevent modifications. **Backoffice/system only.**

### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | uuid | Document ID |

### Request

```json
{
  "reason": "Filing accepted by regulator"
}
```

### Request Body Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `reason` | string | Yes | Reason for locking (1-500 chars) |

### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "active",
    "lockedAt": "2024-01-15T12:00:00.000Z",
    "lockedBy": "agent_xyz",
    "lockedReason": "Filing accepted by regulator"
  },
  "message": "Document locked successfully"
}
```

### Errors

| Status | Code | Description |
|--------|------|-------------|
| 404 | `NOT_FOUND` | Document not found |
| 403 | `FORBIDDEN` | Only backoffice users can lock documents |
| 409 | `CONFLICT` | Document already locked |

---

## Error Response Format

All errors follow a consistent format:

```json
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "Document not found",
    "hint": "Verify the document ID and ensure it belongs to your organization",
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
| `CONFLICT` | 409 | State conflict (locked, wrong status) |
| `LIMIT_REACHED` | 429 | Plan limit exceeded |
| `INTERNAL` | 500 | Server error |

---

## Rate Limits

| Endpoint | Limit |
|----------|-------|
| Upload init | 100/hour per org |
| Download URL | 500/hour per org |
| List/Get | 1000/hour per org |

When rate limited, the API returns `429 Too Many Requests` with a `Retry-After` header.

