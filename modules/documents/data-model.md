---
title: "Documents Module - Data Model"
description: "Database tables, enums, indexes, and relationships for the Documents module."
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
│                       DOCUMENTS DATA MODEL                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  organizations ◄───────────────────────────────────────────────────┐│
│       │                                                             ││
│       │ 1:N                                                         ││
│       ▼                                                             ││
│  ┌─────────────┐     1:N     ┌────────────────┐                    ││
│  │  documents  │────────────►│ document_links │                    ││
│  │             │             │                │                    ││
│  │  id         │             │ entityType     │                    ││
│  │  orgId      │             │ entityId       │───► filings        ││
│  │  kind       │             └────────────────┘     service_reqs   ││
│  │  status     │                                    tickets        ││
│  │  storageKey │     1:N     ┌─────────────────┐    payments       ││
│  │  lockedAt   │────────────►│ document_events │    payouts        ││
│  │  ...        │             │                 │                   ││
│  └─────────────┘             │ eventType       │                   ││
│                              │ actorId         │                   ││
│                              │ metadata        │                   ││
│                              └─────────────────┘                   ││
│                                                                     ││
└─────────────────────────────────────────────────────────────────────┘│
```

---

## Enums

### document_kind (existing)

Document classification for compliance purposes.

| Value | Description |
|-------|-------------|
| `source` | Input documents from tenant (invoices, statements) |
| `workpaper` | Working documents during preparation |
| `submission` | Screenshots/forms submitted to regulator |
| `receipt` | Payment proof (bank receipts, mobile money) |
| `certificate` | Approval letters from regulator |

**File:** `packages/database/src/schema/enums.ts`

```typescript
export const documentKindEnum = pgEnum("document_kind", [
  "source",
  "workpaper",
  "submission",
  "receipt",
  "certificate",
]);
```

### document_status (new)

Document lifecycle status.

| Value | Description |
|-------|-------------|
| `uploading` | Presigned URL issued, upload in progress |
| `active` | Upload complete, document available |
| `archived` | Soft-deleted, not visible in lists |
| `failed` | Upload failed or timed out |
| `quarantine` | Flagged by malware scan (future) |

**File:** `packages/database/src/schema/enums.ts`

```typescript
export const documentStatusEnum = pgEnum("document_status", [
  "uploading",
  "active",
  "archived",
  "failed",
  "quarantine",
]);
```

### document_link_entity_type (new)

Types of entities that can be linked to documents.

| Value | Description |
|-------|-------------|
| `filing` | Recurring obligation filing |
| `service_request` | One-off service request |
| `ticket` | Support/issue ticket |
| `payment_request` | Tenant payment request |
| `regulator_payout` | Payment to regulator |

**File:** `packages/database/src/schema/enums.ts`

```typescript
export const documentLinkEntityTypeEnum = pgEnum("document_link_entity_type", [
  "filing",
  "service_request",
  "ticket",
  "payment_request",
  "regulator_payout",
]);
```

### document_event_type (new)

Types of audit events for documents.

| Value | Description |
|-------|-------------|
| `upload_initiated` | Presigned URL generated |
| `uploaded` | Upload confirmed complete |
| `downloaded` | Download URL generated |
| `linked` | Document linked to entity |
| `unlinked` | Document unlinked from entity |
| `locked` | Document locked for immutability |
| `archived` | Document archived |
| `restored` | Document restored from archive |
| `metadata_updated` | Document metadata changed |

**File:** `packages/database/src/schema/enums.ts`

```typescript
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
```

---

## Tables

### documents (updated)

Main documents table storing file metadata.

**File:** `packages/database/src/schema/compliance/documents.ts`

```typescript
import { 
  index, 
  integer, 
  jsonb, 
  pgTable, 
  text, 
  timestamp, 
  uuid 
} from "drizzle-orm/pg-core";
import { timestamps } from "../../helpers/timestamps";
import { documentKindEnum, documentStatusEnum } from "../enums";
import { organizations } from "../core/organizations";
import { regulators } from "../core/regulators";
import { filings } from "./filings";
import { serviceRequests } from "./service-requests";

export const documents = pgTable(
  "documents",
  {
    // Primary key
    id: uuid("id").primaryKey().defaultRandom(),
    
    // Organization (required, tenant isolation)
    organizationId: text("organization_id")
      .notNull()
      .references(() => organizations.id, { onDelete: "cascade" }),
    
    // Optional associations (legacy direct links)
    regulatorId: uuid("regulator_id").references(() => regulators.id, {
      onDelete: "set null",
    }),
    filingId: uuid("filing_id").references(() => filings.id, {
      onDelete: "set null",
    }),
    serviceRequestId: uuid("service_request_id").references(
      () => serviceRequests.id,
      { onDelete: "set null" }
    ),
    
    // Classification
    module: text("module"), // "compliance", "invoicing", "payroll"
    kind: documentKindEnum("kind").notNull(),
    status: documentStatusEnum("status").notNull().default("uploading"),
    
    // S3 storage reference
    storageKey: text("storage_key").notNull(), // {org_id}/{doc_id}/{filename}
    
    // File metadata
    filename: text("filename"),
    mimeType: text("mime_type"),
    sizeBytes: integer("size_bytes"),
    metadata: jsonb("metadata").$type<Record<string, unknown>>(),
    
    // Upload tracking
    uploadedAt: timestamp("uploaded_at", { mode: "date" }),
    
    // Immutability locking
    lockedAt: timestamp("locked_at", { mode: "date" }),
    lockedBy: text("locked_by"), // userId or "system"
    lockedReason: text("locked_reason"),
    
    // Soft delete
    archivedAt: timestamp("archived_at", { mode: "date" }),
    archivedBy: text("archived_by"),
    
    // Standard timestamps
    ...timestamps,
  },
  (table) => [
    // Org + kind for listing by type
    index("idx_documents_org_kind").on(table.organizationId, table.kind),
    
    // Org + status for active documents list
    index("idx_documents_org_status").on(table.organizationId, table.status),
    
    // Parent entity lookups (legacy direct links)
    index("idx_documents_parents").on(
      table.organizationId,
      table.filingId,
      table.serviceRequestId
    ),
    
    // Storage key for S3 operations
    index("idx_documents_storage_key").on(table.storageKey),
    
    // Regulator filtering
    index("idx_documents_org_regulator").on(table.organizationId, table.regulatorId),
  ]
);

export type Document = typeof documents.$inferSelect;
export type NewDocument = typeof documents.$inferInsert;
```

### document_links (new)

Polymorphic linking table for connecting documents to various entities.

**File:** `packages/database/src/schema/compliance/document-links.ts`

```typescript
import { 
  index, 
  pgTable, 
  text, 
  timestamp, 
  uuid,
  unique,
} from "drizzle-orm/pg-core";
import { timestamps } from "../../helpers/timestamps";
import { documentLinkEntityTypeEnum } from "../enums";
import { documents } from "./documents";
import { organizations } from "../core/organizations";

export const documentLinks = pgTable(
  "document_links",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    
    // Document reference
    documentId: uuid("document_id")
      .notNull()
      .references(() => documents.id, { onDelete: "cascade" }),
    
    // Organization (denormalized for efficient filtering)
    organizationId: text("organization_id")
      .notNull()
      .references(() => organizations.id, { onDelete: "cascade" }),
    
    // Polymorphic entity reference
    entityType: documentLinkEntityTypeEnum("entity_type").notNull(),
    entityId: uuid("entity_id").notNull(),
    
    // Link metadata
    linkedBy: text("linked_by").notNull(), // userId who created link
    linkedAt: timestamp("linked_at", { mode: "date" }).notNull().defaultNow(),
    
    ...timestamps,
  },
  (table) => [
    // Find all links for a document
    index("idx_document_links_document").on(table.documentId),
    
    // Find all documents for an entity
    index("idx_document_links_entity").on(table.entityType, table.entityId),
    
    // Org filtering
    index("idx_document_links_org").on(table.organizationId),
    
    // Prevent duplicate links
    unique("uq_document_links_doc_entity").on(
      table.documentId, 
      table.entityType, 
      table.entityId
    ),
  ]
);

export type DocumentLink = typeof documentLinks.$inferSelect;
export type NewDocumentLink = typeof documentLinks.$inferInsert;
```

### document_events (new)

Audit log for document lifecycle events.

**File:** `packages/database/src/schema/compliance/document-events.ts`

```typescript
import { 
  index, 
  jsonb, 
  pgTable, 
  text, 
  timestamp, 
  uuid 
} from "drizzle-orm/pg-core";
import { documentEventTypeEnum } from "../enums";
import { documents } from "./documents";
import { organizations } from "../core/organizations";

export const documentEvents = pgTable(
  "document_events",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    
    // Document reference
    documentId: uuid("document_id")
      .notNull()
      .references(() => documents.id, { onDelete: "cascade" }),
    
    // Organization (denormalized for efficient filtering)
    organizationId: text("organization_id")
      .notNull()
      .references(() => organizations.id, { onDelete: "cascade" }),
    
    // Event type
    eventType: documentEventTypeEnum("event_type").notNull(),
    
    // Actor information
    actorId: text("actor_id").notNull(),
    actorType: text("actor_type").notNull(), // "user" | "system" | "backoffice"
    
    // Event-specific metadata
    metadata: jsonb("metadata").$type<DocumentEventMetadata>(),
    
    // Timestamp (no updated_at - events are immutable)
    createdAt: timestamp("created_at", { mode: "date" }).notNull().defaultNow(),
  },
  (table) => [
    // Document timeline
    index("idx_document_events_document").on(table.documentId),
    
    // Org-wide event log
    index("idx_document_events_org").on(table.organizationId),
    
    // Filter by event type
    index("idx_document_events_type").on(table.eventType),
    
    // Time-based queries
    index("idx_document_events_created").on(table.createdAt),
    
    // Combined for paginated org event log
    index("idx_document_events_org_created").on(
      table.organizationId, 
      table.createdAt
    ),
  ]
);

// Metadata type for different event types
interface DocumentEventMetadata {
  // For linked/unlinked events
  entityType?: string;
  entityId?: string;
  
  // For locked/archived events
  reason?: string;
  
  // For status changes
  previousStatus?: string;
  newStatus?: string;
  
  // Request context (optional)
  ipAddress?: string;
  userAgent?: string;
  
  // Additional context
  [key: string]: unknown;
}

export type DocumentEvent = typeof documentEvents.$inferSelect;
export type NewDocumentEvent = typeof documentEvents.$inferInsert;
```

---

## Indexes

### Index Strategy

| Table | Index | Purpose |
|-------|-------|---------|
| `documents` | `idx_documents_org_kind` | List documents by kind per org |
| `documents` | `idx_documents_org_status` | List active/archived documents |
| `documents` | `idx_documents_parents` | Find documents for filing/service request |
| `documents` | `idx_documents_storage_key` | S3 key lookups |
| `documents` | `idx_documents_org_regulator` | Filter by regulator |
| `document_links` | `idx_document_links_document` | Get all links for a document |
| `document_links` | `idx_document_links_entity` | Get all documents for an entity |
| `document_links` | `idx_document_links_org` | Org-scoped link queries |
| `document_links` | `uq_document_links_doc_entity` | Prevent duplicate links |
| `document_events` | `idx_document_events_document` | Document audit trail |
| `document_events` | `idx_document_events_org` | Org-wide audit log |
| `document_events` | `idx_document_events_type` | Filter events by type |
| `document_events` | `idx_document_events_created` | Time-based event queries |

### Query Patterns Covered

1. **List org's documents with filters:**
   ```sql
   SELECT * FROM documents 
   WHERE organization_id = ? AND status = 'active' AND kind = ?
   ORDER BY created_at DESC
   LIMIT 20 OFFSET 0;
   ```

2. **Get documents for a filing:**
   ```sql
   SELECT d.* FROM documents d
   JOIN document_links dl ON dl.document_id = d.id
   WHERE dl.entity_type = 'filing' AND dl.entity_id = ?;
   ```

3. **Document audit trail:**
   ```sql
   SELECT * FROM document_events
   WHERE document_id = ?
   ORDER BY created_at ASC;
   ```

4. **Orphan uploads (cleanup job):**
   ```sql
   SELECT * FROM documents
   WHERE status = 'uploading' 
   AND created_at < NOW() - INTERVAL '24 hours';
   ```

---

## Relations

### Drizzle Relations

**File:** `packages/database/src/schema/compliance/compliance-relations.ts`

Add these relations:

```typescript
import { relations } from "drizzle-orm";
import { documents } from "./documents";
import { documentLinks } from "./document-links";
import { documentEvents } from "./document-events";
import { organizations } from "../core/organizations";
import { regulators } from "../core/regulators";
import { filings } from "./filings";
import { serviceRequests } from "./service-requests";

// Update existing documentsRelations
export const documentsRelations = relations(documents, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [documents.organizationId],
    references: [organizations.id],
  }),
  regulator: one(regulators, {
    fields: [documents.regulatorId],
    references: [regulators.id],
  }),
  filing: one(filings, {
    fields: [documents.filingId],
    references: [filings.id],
  }),
  serviceRequest: one(serviceRequests, {
    fields: [documents.serviceRequestId],
    references: [serviceRequests.id],
  }),
  // New relations
  links: many(documentLinks),
  events: many(documentEvents),
}));

// New relations for document_links
export const documentLinksRelations = relations(documentLinks, ({ one }) => ({
  document: one(documents, {
    fields: [documentLinks.documentId],
    references: [documents.id],
  }),
  organization: one(organizations, {
    fields: [documentLinks.organizationId],
    references: [organizations.id],
  }),
}));

// New relations for document_events
export const documentEventsRelations = relations(documentEvents, ({ one }) => ({
  document: one(documents, {
    fields: [documentEvents.documentId],
    references: [documents.id],
  }),
  organization: one(organizations, {
    fields: [documentEvents.organizationId],
    references: [organizations.id],
  }),
}));
```

---

## Migration Guide

### Step 1: Add New Enums

```sql
-- Add new enums
CREATE TYPE document_status AS ENUM (
  'uploading', 'active', 'archived', 'failed', 'quarantine'
);

CREATE TYPE document_link_entity_type AS ENUM (
  'filing', 'service_request', 'ticket', 'payment_request', 'regulator_payout'
);

CREATE TYPE document_event_type AS ENUM (
  'upload_initiated', 'uploaded', 'downloaded', 'linked', 
  'unlinked', 'locked', 'archived', 'restored', 'metadata_updated'
);
```

### Step 2: Alter Documents Table

```sql
-- Add new columns to documents table
ALTER TABLE documents 
  ADD COLUMN status document_status NOT NULL DEFAULT 'active',
  ADD COLUMN locked_at TIMESTAMP,
  ADD COLUMN locked_by TEXT,
  ADD COLUMN locked_reason TEXT,
  ADD COLUMN archived_at TIMESTAMP,
  ADD COLUMN archived_by TEXT;

-- Add new indexes
CREATE INDEX idx_documents_org_status ON documents(organization_id, status);
CREATE INDEX idx_documents_storage_key ON documents(storage_key);
CREATE INDEX idx_documents_org_regulator ON documents(organization_id, regulator_id);
```

### Step 3: Create New Tables

```sql
-- Create document_links table
CREATE TABLE document_links (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  document_id UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
  organization_id TEXT NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  entity_type document_link_entity_type NOT NULL,
  entity_id UUID NOT NULL,
  linked_by TEXT NOT NULL,
  linked_at TIMESTAMP NOT NULL DEFAULT NOW(),
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMP,
  UNIQUE(document_id, entity_type, entity_id)
);

CREATE INDEX idx_document_links_document ON document_links(document_id);
CREATE INDEX idx_document_links_entity ON document_links(entity_type, entity_id);
CREATE INDEX idx_document_links_org ON document_links(organization_id);

-- Create document_events table
CREATE TABLE document_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  document_id UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
  organization_id TEXT NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  event_type document_event_type NOT NULL,
  actor_id TEXT NOT NULL,
  actor_type TEXT NOT NULL,
  metadata JSONB,
  created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_document_events_document ON document_events(document_id);
CREATE INDEX idx_document_events_org ON document_events(organization_id);
CREATE INDEX idx_document_events_type ON document_events(event_type);
CREATE INDEX idx_document_events_created ON document_events(created_at);
CREATE INDEX idx_document_events_org_created ON document_events(organization_id, created_at);
```

### Step 4: Drizzle Migration

Generate and apply migration using Drizzle:

```bash
cd packages/database
pnpm db:generate
pnpm db:migrate
```

### Step 5: Backfill Existing Documents

```sql
-- Set all existing documents to 'active' status
UPDATE documents SET status = 'active' WHERE status IS NULL;

-- Set uploaded_at for existing documents
UPDATE documents SET uploaded_at = created_at WHERE uploaded_at IS NULL;
```

