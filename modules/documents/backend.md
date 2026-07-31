---
title: "Documents Module - Backend Implementation"
description: "Step-by-step guide for implementing the Documents module backend."
---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Implementation Checklist](#implementation-checklist)
3. [Step 1: Database Schema](#step-1-database-schema)
4. [Step 2: API Services Layer](#step-2-api-services-layer)
5. [Step 3: Hono Routes](#step-3-hono-routes)
6. [Step 4: AWS S3 Integration](#step-4-aws-s3-integration)
7. [Step 5: Entity Integration](#step-5-entity-integration)
8. [Step 6: Audit Events](#step-6-audit-events)
9. [Testing](#testing)

---

## Prerequisites

Before implementing, ensure you have:

- [ ] AWS account with S3 access
- [ ] IAM user/role with S3 permissions (see [s3-storage.md](/modules/documents/s3-storage))
- [ ] Environment variables configured
- [ ] Familiarity with existing patterns in `packages/backend/src/modules/`

---

## Implementation Checklist

```
[ ] 1. Add database schema changes
    [ ] Add documentStatusEnum to enums.ts
    [ ] Add new columns to documents table
    [ ] Create document_links table
    [ ] Create document_events table
    [ ] Generate and run migration

[ ] 2. Create API services layer
    [ ] Create documents.schema.ts (Zod schemas)
    [ ] Create documents.service.ts (business logic)
    [ ] Export from @repo/api-services

[ ] 3. Create Hono routes
    [ ] Create documents/ module folder
    [ ] Implement documents.routes.ts
    [ ] Implement documents.handlers.ts
    [ ] Register routes in modules/index.ts

[ ] 4. Configure AWS S3
    [ ] Install @aws-sdk/client-s3 and @aws-sdk/s3-request-presigner
    [ ] Create S3 client utility
    [ ] Implement presigned URL generation
    [ ] Add environment variables

[ ] 5. Integrate with Filings/Service Requests
    [ ] Add document linking logic
    [ ] Implement auto-lock on ACCEPTED status

[ ] 6. Add audit events
    [ ] Log UPLOAD_INITIATED, UPLOADED, DOWNLOADED, LINKED, LOCKED, ARCHIVED

[ ] 7. Test implementation
    [ ] Unit tests for service functions
    [ ] Integration tests for routes
    [ ] Manual upload/download verification
```

---

## Step 1: Database Schema

### 1.1 Add Document Status Enum

**File:** `packages/database/src/schema/enums.ts`

```typescript
// Add after documentKindEnum
export const documentStatusEnum = pgEnum("document_status", [
  "uploading",    // Presigned URL issued, waiting for upload
  "active",       // Upload complete, document available
  "archived",     // Soft-deleted
  "failed",       // Upload failed or timed out
  "quarantine",   // Flagged by malware scan (future)
]);
```

### 1.2 Update Documents Table

**File:** `packages/database/src/schema/compliance/documents.ts`

Add these columns to the existing `documents` table:

```typescript
import { documentStatusEnum } from "../enums";

export const documents = pgTable(
  "documents",
  {
    // ... existing fields ...
    
    // New fields
    status: documentStatusEnum("status").notNull().default("uploading"),
    lockedAt: timestamp("locked_at", { mode: "date" }),
    lockedBy: text("locked_by"),  // userId or "system"
    lockedReason: text("locked_reason"),
    archivedAt: timestamp("archived_at", { mode: "date" }),
    archivedBy: text("archived_by"),
    
    // Existing timestamps from helpers
    ...timestamps,
  },
  (table) => [
    // Existing indexes...
    index("idx_documents_org_kind").on(table.organizationId, table.kind),
    index("idx_documents_parents").on(
      table.organizationId,
      table.filingId,
      table.serviceRequestId
    ),
    // New indexes
    index("idx_documents_org_status").on(table.organizationId, table.status),
    index("idx_documents_storage_key").on(table.storageKey),
  ]
);
```

### 1.3 Create Document Links Table

**File:** `packages/database/src/schema/compliance/document-links.ts`

```typescript
import { index, pgTable, text, timestamp, uuid } from "drizzle-orm/pg-core";
import { timestamps } from "../../helpers/timestamps";
import { documents } from "./documents";

export const documentLinkEntityTypeEnum = pgEnum("document_link_entity_type", [
  "filing",
  "service_request",
  "ticket",
  "payment_request",
  "regulator_payout",
]);

export const documentLinks = pgTable(
  "document_links",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    documentId: uuid("document_id")
      .notNull()
      .references(() => documents.id, { onDelete: "cascade" }),
    organizationId: text("organization_id").notNull(),
    
    // Polymorphic link
    entityType: documentLinkEntityTypeEnum("entity_type").notNull(),
    entityId: uuid("entity_id").notNull(),
    
    // Link metadata
    linkedBy: text("linked_by").notNull(),
    linkedAt: timestamp("linked_at", { mode: "date" }).defaultNow(),
    
    ...timestamps,
  },
  (table) => [
    index("idx_document_links_document").on(table.documentId),
    index("idx_document_links_entity").on(table.entityType, table.entityId),
    index("idx_document_links_org").on(table.organizationId),
  ]
);
```

### 1.4 Create Document Events Table

**File:** `packages/database/src/schema/compliance/document-events.ts`

```typescript
import { index, jsonb, pgTable, text, timestamp, uuid } from "drizzle-orm/pg-core";
import { documents } from "./documents";

export const documentEventTypeEnum = pgEnum("document_event_type", [
  "upload_initiated",
  "uploaded",
  "downloaded",
  "linked",
  "unlinked",
  "locked",
  "archived",
  "restored",
  "metadata_updated",
]);

export const documentEvents = pgTable(
  "document_events",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    documentId: uuid("document_id")
      .notNull()
      .references(() => documents.id, { onDelete: "cascade" }),
    organizationId: text("organization_id").notNull(),
    
    eventType: documentEventTypeEnum("event_type").notNull(),
    
    // Actor info
    actorId: text("actor_id").notNull(),
    actorType: text("actor_type").notNull(), // "user" | "system" | "backoffice"
    
    // Event details
    metadata: jsonb("metadata").$type<{
      entityType?: string;
      entityId?: string;
      reason?: string;
      ipAddress?: string;
      userAgent?: string;
      previousStatus?: string;
      newStatus?: string;
    }>(),
    
    createdAt: timestamp("created_at", { mode: "date" }).notNull().defaultNow(),
  },
  (table) => [
    index("idx_document_events_document").on(table.documentId),
    index("idx_document_events_org").on(table.organizationId),
    index("idx_document_events_type").on(table.eventType),
    index("idx_document_events_created").on(table.createdAt),
  ]
);
```

### 1.5 Generate Migration

```bash
cd packages/database
pnpm db:generate
pnpm db:migrate
```

---

## Step 2: API Services Layer

### 2.1 Create Zod Schemas

**File:** `packages/api-services/src/domains/documents/documents.schema.ts`

```typescript
import { z } from "zod";

// Enums
export const documentKindSchema = z.enum([
  "source",
  "workpaper",
  "submission",
  "receipt",
  "certificate",
]);

export const documentStatusSchema = z.enum([
  "uploading",
  "active",
  "archived",
  "failed",
  "quarantine",
]);

export const documentLinkEntityTypeSchema = z.enum([
  "filing",
  "service_request",
  "ticket",
  "payment_request",
  "regulator_payout",
]);

// Input schemas
export const initUploadInputSchema = z.object({
  filename: z.string().min(1).max(255),
  mimeType: z.string().min(1),
  sizeBytes: z.number().int().positive().max(52428800), // 50MB max
  kind: documentKindSchema,
  regulatorId: z.string().uuid().optional(),
  filingId: z.string().uuid().optional(),
  serviceRequestId: z.string().uuid().optional(),
  metadata: z.record(z.unknown()).optional(),
});

export const completeUploadInputSchema = z.object({
  documentId: z.string().uuid(),
});

export const listDocumentsQuerySchema = z.object({
  kind: documentKindSchema.optional(),
  status: documentStatusSchema.optional(),
  regulatorId: z.string().uuid().optional(),
  filingId: z.string().uuid().optional(),
  serviceRequestId: z.string().uuid().optional(),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  offset: z.coerce.number().int().min(0).default(0),
});

export const linkDocumentInputSchema = z.object({
  entityType: documentLinkEntityTypeSchema,
  entityId: z.string().uuid(),
});

export const archiveDocumentInputSchema = z.object({
  reason: z.string().min(1).max(500).optional(),
});

export const lockDocumentInputSchema = z.object({
  reason: z.string().min(1).max(500),
});

// Response schemas
export const documentResponseSchema = z.object({
  id: z.string().uuid(),
  organizationId: z.string(),
  regulatorId: z.string().uuid().nullable(),
  filingId: z.string().uuid().nullable(),
  serviceRequestId: z.string().uuid().nullable(),
  module: z.string().nullable(),
  kind: documentKindSchema,
  status: documentStatusSchema,
  storageKey: z.string(),
  filename: z.string().nullable(),
  mimeType: z.string().nullable(),
  sizeBytes: z.number().nullable(),
  metadata: z.record(z.unknown()).nullable(),
  uploadedAt: z.date().nullable(),
  lockedAt: z.date().nullable(),
  lockedBy: z.string().nullable(),
  lockedReason: z.string().nullable(),
  archivedAt: z.date().nullable(),
  createdAt: z.date(),
  updatedAt: z.date(),
});

export const initUploadResponseSchema = z.object({
  documentId: z.string().uuid(),
  uploadUrl: z.string().url(),
  expiresAt: z.date(),
});

export const downloadUrlResponseSchema = z.object({
  downloadUrl: z.string().url(),
  expiresAt: z.date(),
});

// Type exports
export type InitUploadInput = z.infer<typeof initUploadInputSchema>;
export type CompleteUploadInput = z.infer<typeof completeUploadInputSchema>;
export type ListDocumentsQuery = z.infer<typeof listDocumentsQuerySchema>;
export type LinkDocumentInput = z.infer<typeof linkDocumentInputSchema>;
export type ArchiveDocumentInput = z.infer<typeof archiveDocumentInputSchema>;
export type LockDocumentInput = z.infer<typeof lockDocumentInputSchema>;
export type DocumentResponse = z.infer<typeof documentResponseSchema>;
```

### 2.2 Create Documents Service

**File:** `packages/api-services/src/domains/documents/documents.service.ts`

```typescript
import type { ServiceContext, ServiceDependencies } from "../../types";
import type {
  InitUploadInput,
  ListDocumentsQuery,
  LinkDocumentInput,
  ArchiveDocumentInput,
  LockDocumentInput,
} from "./documents.schema";

// Service functions follow the pattern from organization.service.ts

export async function initUpload(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  input: InitUploadInput
) {
  // 1. Validate org access
  // 2. Generate document ID and storage key
  // 3. Create document record with status="uploading"
  // 4. Generate presigned PUT URL
  // 5. Log UPLOAD_INITIATED event
  // 6. Return { documentId, uploadUrl, expiresAt }
}

export async function completeUpload(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  documentId: string
) {
  // 1. Fetch document, verify org ownership
  // 2. Verify status is "uploading"
  // 3. HEAD request to S3 to verify object exists
  // 4. Update status to "active", set uploadedAt
  // 5. Log UPLOADED event
  // 6. Return document
}

export async function listDocuments(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  query: ListDocumentsQuery
) {
  // 1. Build query with org filter
  // 2. Apply kind/status/regulator/filing/serviceRequest filters
  // 3. Exclude archived unless explicitly requested
  // 4. Return paginated results
}

export async function getDocument(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  documentId: string
) {
  // 1. Fetch document with org filter
  // 2. Throw NOT_FOUND if missing
  // 3. Return document
}

export async function getDownloadUrl(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  documentId: string
) {
  // 1. Fetch document, verify org ownership
  // 2. Verify status is "active"
  // 3. Generate presigned GET URL
  // 4. Log DOWNLOADED event
  // 5. Return { downloadUrl, expiresAt }
}

export async function linkDocument(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  documentId: string,
  input: LinkDocumentInput
) {
  // 1. Fetch document, verify org ownership
  // 2. Verify document is not locked
  // 3. Verify target entity exists and belongs to org
  // 4. Create document_links record
  // 5. Log LINKED event
  // 6. Return link
}

export async function unlinkDocument(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  documentId: string,
  linkId: string
) {
  // 1. Fetch document, verify org ownership
  // 2. Verify document is not locked
  // 3. Fetch link, verify it belongs to this document
  // 4. Delete link
  // 5. Log UNLINKED event
}

export async function archiveDocument(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  documentId: string,
  input: ArchiveDocumentInput
) {
  // 1. Fetch document, verify org ownership
  // 2. Verify document is not locked
  // 3. Update status to "archived", set archivedAt, archivedBy
  // 4. Log ARCHIVED event
  // 5. Return document
}

export async function lockDocument(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  documentId: string,
  input: LockDocumentInput
) {
  // 1. Fetch document, verify org ownership
  // 2. Verify caller has lock permission (backoffice or system)
  // 3. Update lockedAt, lockedBy, lockedReason
  // 4. Log LOCKED event
  // 5. Return document
}

export async function lockDocumentsForEntity(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  entityType: "filing" | "service_request",
  entityId: string,
  reason: string
) {
  // Called when Filing/ServiceRequest becomes ACCEPTED
  // 1. Find all documents linked to entity
  // 2. Filter to lockable kinds (submission, receipt, certificate)
  // 3. Lock each document
  // 4. Return count of locked documents
}
```

---

## Step 3: Hono Routes

### 3.1 Create Module Structure

```
packages/backend/src/modules/documents/
├── index.ts
├── documents.routes.ts
└── documents.handlers.ts
```

### 3.2 Define Routes

**File:** `packages/backend/src/modules/documents/documents.routes.ts`

```typescript
import { createRoute, z } from "@hono/zod-openapi";
import {
  initUploadInputSchema,
  completeUploadInputSchema,
  listDocumentsQuerySchema,
  linkDocumentInputSchema,
  archiveDocumentInputSchema,
  lockDocumentInputSchema,
  documentResponseSchema,
  initUploadResponseSchema,
  downloadUrlResponseSchema,
} from "@repo/api-services";
import * as HttpStatusCodes from "stoker/http-status-codes";
import { jsonContent, jsonContentRequired } from "stoker/openapi/helpers";
import { errorResponseSchema, successResponseSchema } from "@repo/backend/middleware/auth";";
import { requireAuth, requireOrg } from "@repo/backend/middleware/auth";

const tags = ["Documents"];

export const initUploadRoute = createRoute({
  tags,
  method: "post",
  path: "/documents/uploads/init",
  summary: "Initialize document upload",
  description: "Get a presigned URL for uploading a document to S3",
  middleware: [requireAuth, requireOrg],
  request: {
    body: jsonContentRequired(initUploadInputSchema, "Upload initialization data"),
  },
  responses: {
    [HttpStatusCodes.CREATED]: jsonContent(
      successResponseSchema.extend({ data: initUploadResponseSchema }),
      "Upload initialized"
    ),
    [HttpStatusCodes.BAD_REQUEST]: jsonContent(errorResponseSchema, "Invalid input"),
    [HttpStatusCodes.UNAUTHORIZED]: jsonContent(errorResponseSchema, "Auth required"),
  },
});

export const completeUploadRoute = createRoute({
  tags,
  method: "post",
  path: "/documents/uploads/complete",
  summary: "Complete document upload",
  description: "Confirm upload completion and activate document",
  middleware: [requireAuth, requireOrg],
  request: {
    body: jsonContentRequired(completeUploadInputSchema, "Document ID"),
  },
  responses: {
    [HttpStatusCodes.OK]: jsonContent(
      successResponseSchema.extend({ data: documentResponseSchema }),
      "Upload completed"
    ),
    [HttpStatusCodes.NOT_FOUND]: jsonContent(errorResponseSchema, "Document not found"),
    [HttpStatusCodes.CONFLICT]: jsonContent(errorResponseSchema, "Invalid state"),
  },
});

export const listDocumentsRoute = createRoute({
  tags,
  method: "get",
  path: "/documents",
  summary: "List documents",
  description: "List documents with filters and pagination",
  middleware: [requireAuth, requireOrg],
  request: {
    query: listDocumentsQuerySchema,
  },
  responses: {
    [HttpStatusCodes.OK]: jsonContent(
      successResponseSchema.extend({
        data: z.array(documentResponseSchema),
        pagination: z.object({
          limit: z.number(),
          offset: z.number(),
          total: z.number(),
        }),
      }),
      "Documents list"
    ),
  },
});

export const getDocumentRoute = createRoute({
  tags,
  method: "get",
  path: "/documents/{id}",
  summary: "Get document",
  description: "Get document by ID",
  middleware: [requireAuth, requireOrg],
  request: {
    params: z.object({ id: z.string().uuid() }),
  },
  responses: {
    [HttpStatusCodes.OK]: jsonContent(
      successResponseSchema.extend({ data: documentResponseSchema }),
      "Document retrieved"
    ),
    [HttpStatusCodes.NOT_FOUND]: jsonContent(errorResponseSchema, "Not found"),
  },
});

export const getDownloadUrlRoute = createRoute({
  tags,
  method: "post",
  path: "/documents/{id}/download-url",
  summary: "Get download URL",
  description: "Get a presigned URL for downloading the document",
  middleware: [requireAuth, requireOrg],
  request: {
    params: z.object({ id: z.string().uuid() }),
  },
  responses: {
    [HttpStatusCodes.OK]: jsonContent(
      successResponseSchema.extend({ data: downloadUrlResponseSchema }),
      "Download URL generated"
    ),
    [HttpStatusCodes.NOT_FOUND]: jsonContent(errorResponseSchema, "Not found"),
  },
});

export const linkDocumentRoute = createRoute({
  tags,
  method: "post",
  path: "/documents/{id}/link",
  summary: "Link document to entity",
  description: "Link a document to a filing, service request, or other entity",
  middleware: [requireAuth, requireOrg],
  request: {
    params: z.object({ id: z.string().uuid() }),
    body: jsonContentRequired(linkDocumentInputSchema, "Link data"),
  },
  responses: {
    [HttpStatusCodes.CREATED]: jsonContent(successResponseSchema, "Document linked"),
    [HttpStatusCodes.NOT_FOUND]: jsonContent(errorResponseSchema, "Not found"),
    [HttpStatusCodes.CONFLICT]: jsonContent(errorResponseSchema, "Document locked"),
  },
});

export const unlinkDocumentRoute = createRoute({
  tags,
  method: "delete",
  path: "/documents/{id}/link/{linkId}",
  summary: "Unlink document from entity",
  description: "Remove a document link",
  middleware: [requireAuth, requireOrg],
  request: {
    params: z.object({
      id: z.string().uuid(),
      linkId: z.string().uuid(),
    }),
  },
  responses: {
    [HttpStatusCodes.OK]: jsonContent(successResponseSchema, "Link removed"),
    [HttpStatusCodes.NOT_FOUND]: jsonContent(errorResponseSchema, "Not found"),
    [HttpStatusCodes.CONFLICT]: jsonContent(errorResponseSchema, "Document locked"),
  },
});

export const archiveDocumentRoute = createRoute({
  tags,
  method: "post",
  path: "/documents/{id}/archive",
  summary: "Archive document",
  description: "Soft-delete a document (archive)",
  middleware: [requireAuth, requireOrg],
  request: {
    params: z.object({ id: z.string().uuid() }),
    body: jsonContentRequired(archiveDocumentInputSchema, "Archive reason"),
  },
  responses: {
    [HttpStatusCodes.OK]: jsonContent(
      successResponseSchema.extend({ data: documentResponseSchema }),
      "Document archived"
    ),
    [HttpStatusCodes.NOT_FOUND]: jsonContent(errorResponseSchema, "Not found"),
    [HttpStatusCodes.CONFLICT]: jsonContent(errorResponseSchema, "Document locked"),
  },
});

export const lockDocumentRoute = createRoute({
  tags,
  method: "post",
  path: "/documents/{id}/lock",
  summary: "Lock document",
  description: "Lock a document to prevent modifications (backoffice/system only)",
  middleware: [requireAuth, requireOrg],
  request: {
    params: z.object({ id: z.string().uuid() }),
    body: jsonContentRequired(lockDocumentInputSchema, "Lock reason"),
  },
  responses: {
    [HttpStatusCodes.OK]: jsonContent(
      successResponseSchema.extend({ data: documentResponseSchema }),
      "Document locked"
    ),
    [HttpStatusCodes.NOT_FOUND]: jsonContent(errorResponseSchema, "Not found"),
    [HttpStatusCodes.FORBIDDEN]: jsonContent(errorResponseSchema, "Not authorized"),
  },
});

// Export route types
export type InitUploadRoute = typeof initUploadRoute;
export type CompleteUploadRoute = typeof completeUploadRoute;
export type ListDocumentsRoute = typeof listDocumentsRoute;
export type GetDocumentRoute = typeof getDocumentRoute;
export type GetDownloadUrlRoute = typeof getDownloadUrlRoute;
export type LinkDocumentRoute = typeof linkDocumentRoute;
export type UnlinkDocumentRoute = typeof unlinkDocumentRoute;
export type ArchiveDocumentRoute = typeof archiveDocumentRoute;
export type LockDocumentRoute = typeof lockDocumentRoute;
```

### 3.3 Register Routes

**File:** `packages/backend/src/modules/documents/index.ts`

```typescript
import { createRouter } from "@/core/config/router";
import * as routes from "./documents.routes";
import * as handlers from "./documents.handlers";

export const documentsRouter = createRouter()
  .openapi(routes.initUploadRoute, handlers.initUpload)
  .openapi(routes.completeUploadRoute, handlers.completeUpload)
  .openapi(routes.listDocumentsRoute, handlers.listDocuments)
  .openapi(routes.getDocumentRoute, handlers.getDocument)
  .openapi(routes.getDownloadUrlRoute, handlers.getDownloadUrl)
  .openapi(routes.linkDocumentRoute, handlers.linkDocument)
  .openapi(routes.unlinkDocumentRoute, handlers.unlinkDocument)
  .openapi(routes.archiveDocumentRoute, handlers.archiveDocument)
  .openapi(routes.lockDocumentRoute, handlers.lockDocument);
```

**Update:** `packages/backend/src/modules/index.ts`

```typescript
import { documentsRouter } from "./documents";

// Add to the main router:
.route("/", documentsRouter)
```

---

## Step 4: AWS S3 Integration

### 4.1 Install Dependencies

```bash
cd packages/backend
pnpm add @aws-sdk/client-s3 @aws-sdk/s3-request-presigner
```

### 4.2 Create S3 Client

**File:** `packages/backend/src/core/storage/s3-client.ts`

```typescript
import { S3Client, HeadObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
import { PutObjectCommand, GetObjectCommand } from "@aws-sdk/client-s3";

const s3Client = new S3Client({
  region: process.env.AWS_REGION || "af-south-1",
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID!,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!,
  },
});

const BUCKET = process.env.S3_DOCUMENTS_BUCKET!;
const UPLOAD_EXPIRY = parseInt(process.env.S3_UPLOAD_URL_EXPIRY_SECONDS || "3600");
const DOWNLOAD_EXPIRY = parseInt(process.env.S3_DOWNLOAD_URL_EXPIRY_SECONDS || "300");

export function buildStorageKey(orgId: string, documentId: string, filename: string): string {
  // Sanitize filename
  const sanitized = filename.replace(/[^a-zA-Z0-9._-]/g, "_");
  return `${orgId}/${documentId}/${sanitized}`;
}

export async function generateUploadUrl(
  storageKey: string,
  contentType: string,
  contentLength: number
): Promise<{ url: string; expiresAt: Date }> {
  const command = new PutObjectCommand({
    Bucket: BUCKET,
    Key: storageKey,
    ContentType: contentType,
    ContentLength: contentLength,
  });

  const url = await getSignedUrl(s3Client, command, { expiresIn: UPLOAD_EXPIRY });
  const expiresAt = new Date(Date.now() + UPLOAD_EXPIRY * 1000);

  return { url, expiresAt };
}

export async function generateDownloadUrl(
  storageKey: string,
  filename: string
): Promise<{ url: string; expiresAt: Date }> {
  const command = new GetObjectCommand({
    Bucket: BUCKET,
    Key: storageKey,
    ResponseContentDisposition: `attachment; filename="${filename}"`,
  });

  const url = await getSignedUrl(s3Client, command, { expiresIn: DOWNLOAD_EXPIRY });
  const expiresAt = new Date(Date.now() + DOWNLOAD_EXPIRY * 1000);

  return { url, expiresAt };
}

export async function verifyObjectExists(storageKey: string): Promise<boolean> {
  try {
    await s3Client.send(
      new HeadObjectCommand({
        Bucket: BUCKET,
        Key: storageKey,
      })
    );
    return true;
  } catch (error: any) {
    if (error.name === "NotFound") {
      return false;
    }
    throw error;
  }
}

export { s3Client, BUCKET };
```

---

## Step 5: Entity Integration

### 5.1 Auto-Lock on Filing/ServiceRequest ACCEPTED

When a Filing or Service Request transitions to `ACCEPTED` status, automatically lock all linked evidence documents.

**Add to filing status transition handler:**

```typescript
import { lockDocumentsForEntity } from "@repo/api-services";

// In the status change handler for Filing/ServiceRequest:
if (newStatus === "accepted") {
  const lockedCount = await lockDocumentsForEntity(
    ctx,
    deps,
    "filing", // or "service_request"
    entityId,
    "Auto-locked: Filing accepted by regulator"
  );
  
  logger.info({ entityId, lockedCount }, "Documents locked for accepted filing");
}
```

### 5.2 Lockable Document Kinds

Only these document kinds are auto-locked on acceptance:

- `submission` - Proof of submission to regulator
- `receipt` - Payment receipts
- `certificate` - Approval certificates

The `source` and `workpaper` kinds are NOT auto-locked.

---

## Step 6: Audit Events

### 6.1 Event Types Reference

| Event | Trigger | Metadata |
|-------|---------|----------|
| `upload_initiated` | `POST /uploads/init` | filename, mimeType, sizeBytes |
| `uploaded` | `POST /uploads/complete` | - |
| `downloaded` | `POST /download-url` | ipAddress, userAgent |
| `linked` | `POST /link` | entityType, entityId |
| `unlinked` | `DELETE /link/:id` | entityType, entityId |
| `locked` | `POST /lock` or auto-lock | reason, lockedBy |
| `archived` | `POST /archive` | reason |
| `restored` | Future feature | - |

### 6.2 Event Logging Helper

```typescript
async function logDocumentEvent(
  deps: ServiceDependencies,
  documentId: string,
  organizationId: string,
  eventType: DocumentEventType,
  actorId: string,
  actorType: "user" | "system" | "backoffice",
  metadata?: Record<string, unknown>
) {
  await deps.db.insert(documentEvents).values({
    documentId,
    organizationId,
    eventType,
    actorId,
    actorType,
    metadata,
  });
}
```

---

## Testing

### Unit Tests

**File:** `packages/api-services/src/domains/documents/__tests__/documents.service.test.ts`

```typescript
describe("documents.service", () => {
  describe("buildStorageKey", () => {
    it("should create valid storage key", () => {
      const key = buildStorageKey("org_123", "doc_456", "invoice.pdf");
      expect(key).toBe("org_123/doc_456/invoice.pdf");
    });

    it("should sanitize special characters", () => {
      const key = buildStorageKey("org_123", "doc_456", "my file (1).pdf");
      expect(key).toBe("org_123/doc_456/my_file__1_.pdf");
    });
  });

  describe("lockDocument", () => {
    it("should prevent locking already locked document", async () => {
      // ... test implementation
    });

    it("should require backoffice role", async () => {
      // ... test implementation
    });
  });

  describe("archiveDocument", () => {
    it("should fail for locked documents", async () => {
      // ... test implementation
    });
  });
});
```

### Integration Tests

**File:** `packages/backend/src/modules/documents/__tests__/documents.integration.test.ts`

```typescript
describe("Documents API", () => {
  describe("POST /documents/uploads/init", () => {
    it("should return presigned URL", async () => {
      // ... test implementation
    });

    it("should enforce org isolation", async () => {
      // ... test implementation
    });
  });

  describe("POST /documents/uploads/complete", () => {
    it("should verify S3 object exists", async () => {
      // ... test implementation with S3 mock
    });
  });
});
```

### Manual QA Checklist

- [ ] Upload a file and verify it appears in S3
- [ ] Complete upload and verify document status changes to `active`
- [ ] Generate download URL and verify file downloads correctly
- [ ] Link document to a filing
- [ ] Lock document and verify archive/unlink blocked
- [ ] Verify org isolation (cannot access other org's documents)
- [ ] Verify audit events are logged correctly

