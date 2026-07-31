---
title: "Documents Module"
description: "Documents & Evidence documentation."
---

## Table of Contents

1. [Overview](#1-overview)
2. [Document Model](#2-document-model)
3. [Upload Flow](#3-upload-flow)
4. [Evidence Requirements](#4-evidence-requirements)
5. [Tagging System](#5-tagging-system)
6. [Immutability Rules](#6-immutability-rules)

---

## 1. Overview

**Route:** `/documents`  
**Purpose:** Manage all documents and evidence files.

### 1.1 Storage Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      DOCUMENT STORAGE                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Browser                    API                        AWS S3      │
│      │                        │                          │          │
│      │ ── Get upload URL ────►│                          │          │
│      │                        │ ─── Generate presigned ─►│          │
│      │ ◄── Presigned URL ─────│ ◄─────────────────────────│          │
│      │                        │                          │          │
│      │ ────────────── Direct upload ────────────────────►│          │
│      │                        │                          │          │
│      │ ── Confirm upload ────►│                          │          │
│      │                        │ ─── Verify exists ──────►│          │
│      │                        │ ─── Save metadata ──────►│ (DB)     │
│      │ ◄── Document record ───│                          │          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 Page Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│ Documents & Evidence                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐         │
│ │ Total Documents │ │ Pending Review  │ │ Evidence Files  │         │
│ │     1,234       │ │     15          │ │     456         │         │
│ └─────────────────┘ └─────────────────┘ └─────────────────┘         │
│                                                                      │
│ ┌───────────────────────────────────────────────────────┐ [Upload]  │
│ │ 🔍 Search documents...                                │           │
│ └───────────────────────────────────────────────────────┘           │
│                                                                      │
│ Filters: [Organization ▼] [Tag ▼] [Date Range ▼] [Case ▼]          │
│                                                                      │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ Name              │ Org      │ Tag     │ Case    │ Date │ Size │ │
│ ├───────────────────┼──────────┼─────────┼─────────┼──────┼──────┤ │
│ │ payment_proof.pdf │ ABC Ltd  │ RECEIPT │ FIL-123 │ Jan 3│ 245KB│ │
│ │ submission.png    │ XYZ Corp │ SUBMIS..│ SRQ-456 │ Jan 2│ 1.2MB│ │
│ └─────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Document Model

### 2.1 Database Schema

```typescript
// packages/database/src/schema/compliance/documents.ts

export const documents = pgTable('documents', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id),
  
  // Optional case linking
  filingId: uuid('filing_id').references(() => filings.id),
  serviceRequestId: uuid('service_request_id').references(() => serviceRequests.id),
  
  // Classification
  module: varchar('module', { length: 50 }),  // 'compliance', 'invoicing', etc.
  kind: documentKindEnum('kind').notNull(),
  regulatorId: uuid('regulator_id').references(() => regulators.id),
  
  // Storage
  storageKey: varchar('storage_key', { length: 500 }).notNull(),
  filename: varchar('filename', { length: 255 }).notNull(),
  mimeType: varchar('mime_type', { length: 100 }).notNull(),
  sizeBytes: integer('size_bytes').notNull(),
  checksum: varchar('checksum', { length: 64 }),
  
  // Metadata
  metadata: jsonb('metadata'),
  uploadedById: varchar('uploaded_by_id', { length: 50 }),
  uploadedByType: actorTypeEnum('uploaded_by_type'),
  
  // Timestamps
  uploadedAt: timestamp('uploaded_at').defaultNow().notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
  deletedAt: timestamp('deleted_at'),
});
```

### 2.2 Document Interface

```typescript
interface Document {
  id: string;
  organizationId: string;
  filingId?: string;
  serviceRequestId?: string;
  
  module: string;
  kind: DocumentKind;
  regulatorId?: string;
  
  storageKey: string;
  filename: string;
  mimeType: string;
  sizeBytes: number;
  checksum?: string;
  
  metadata?: Record<string, any>;
  uploadedById: string;
  uploadedByType: 'STAFF' | 'TENANT' | 'SYSTEM';
  
  uploadedAt: Date;
}
```

---

## 3. Upload Flow

### 3.1 Presigned URL Flow

```typescript
// Step 1: Request upload URL
const { uploadUrl, documentId, storageKey } = await api.documents.getUploadUrl({
  organizationId,
  filename: 'payment_proof.pdf',
  mimeType: 'application/pdf',
  sizeBytes: 245000,
  kind: 'PAYMENT_PROOF',
  caseId: 'fil_123',
  caseType: 'filing',
});

// Step 2: Upload directly to S3/R2
await fetch(uploadUrl, {
  method: 'PUT',
  body: file,
  headers: {
    'Content-Type': file.type,
  },
});

// Step 3: Confirm upload
const document = await api.documents.confirmUpload({
  documentId,
  checksum: calculatedChecksum,
});
```

### 3.2 Upload Dropzone Component

```tsx
// apps/backoffice/components/documents/upload-dropzone.tsx

interface UploadDropzoneProps {
  organizationId: string;
  caseId?: string;
  caseType?: 'filing' | 'service_request';
  kind?: DocumentKind;
  onUploadComplete: (doc: Document) => void;
  accept?: string[];
  maxSize?: number;
}

export function UploadDropzone({
  organizationId,
  caseId,
  caseType,
  kind,
  onUploadComplete,
  accept = ['application/pdf', 'image/png', 'image/jpeg'],
  maxSize = 10 * 1024 * 1024, // 10MB
}: UploadDropzoneProps) {
  // Drag and drop implementation
  // Progress indicator
  // Error handling
}
```

### 3.3 Evidence Upload Modal

Used when uploading evidence for payment/payout verification:

```
┌─────────────────────────────────────────────────────────────────────┐
│ Upload Evidence                                                  ✕  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│ Document Type:                                                       │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ Select evidence type...                                      ▼ │ │
│ │ ○ Payment Proof                                                │ │
│ │ ○ Payout Proof                                                 │ │
│ │ ○ Submission Proof                                             │ │
│ │ ○ Outcome Proof                                                │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │                                                                 │ │
│ │     📎 Drop file here or click to browse                       │ │
│ │                                                                 │ │
│ │     Accepted: PDF, PNG, JPG (max 10MB)                         │ │
│ │                                                                 │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│ ┌──────────────────────────────────────────────────────────────┐    │
│ │ ████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  45%    │    │
│ │ payment_proof.pdf (245KB)                                    │    │
│ └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│                                            [Cancel] [Upload]        │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.4 API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `POST /documents/upload-url` | POST | Get presigned upload URL |
| `POST /documents/confirm` | POST | Confirm upload complete |
| `POST /documents/:id/attach` | POST | Attach to case |
| `GET /documents` | GET | List with filters |
| `GET /documents/:id` | GET | Get document |
| `GET /documents/:id/download-url` | GET | Get presigned download URL |

---

## 4. Evidence Requirements

### 4.1 Evidence Gates

Certain actions require evidence documents:

| Action | Required Evidence |
|--------|-------------------|
| Verify tenant payment | `PAYMENT_PROOF` |
| Confirm payout | `PAYOUT_PROOF` |
| Mark as submitted | `SUBMISSION_PROOF` |
| Mark as accepted | `OUTCOME_PROOF` |

### 4.2 Evidence Validation

Before allowing status transition:

```typescript
async function validateEvidenceGate(
  caseId: string,
  requiredTag: DocumentKind
): Promise<{ passed: boolean; documents: Document[] }> {
  const docs = await db.documents.findMany({
    where: {
      OR: [
        { filingId: caseId },
        { serviceRequestId: caseId },
      ],
      kind: requiredTag,
    },
  });
  
  return {
    passed: docs.length > 0,
    documents: docs,
  };
}
```

### 4.3 Evidence in Case Detail

Display evidence status in readiness checklist:

```
Documents & Evidence
────────────────────────────────────────
✓ Source documents (3)
  └─ tax_return.pdf, financials.xlsx, support_letter.pdf
✓ Payment proof (1)
  └─ payment_receipt.pdf
✗ Submission proof
  └─ Required before marking as submitted
────────────────────────────────────────
[Upload Evidence]
```

---

## 5. Tagging System

### 5.1 Document Kinds (Tags)

| Tag | Description | Uploaded By |
|-----|-------------|-------------|
| `SOURCE` | Input documents from tenant | Tenant |
| `WORKPAPER` | Internal working documents | Staff |
| `PAYMENT_PROOF` | Tenant payment evidence | Staff |
| `PAYOUT_PROOF` | Regulator payout evidence | Staff |
| `SUBMISSION_PROOF` | Submission screenshot/PDF | Staff |
| `OUTCOME_PROOF` | Acceptance/rejection proof | Staff |
| `RECEIPT` | Payment receipts | Either |
| `CERTIFICATE` | Approval letters/certificates | Staff |

### 5.2 Tag Selection UI

```tsx
<DocumentTagSelect
  value={selectedTag}
  onChange={setSelectedTag}
  allowedTags={['PAYMENT_PROOF', 'PAYOUT_PROOF', 'SUBMISSION_PROOF']}
/>
```

### 5.3 Auto-Tagging

Some documents are auto-tagged based on context:

| Context | Auto-Tag |
|---------|----------|
| Uploaded during payment verification | `PAYMENT_PROOF` |
| Uploaded during payout confirmation | `PAYOUT_PROOF` |
| Uploaded when marking submitted | `SUBMISSION_PROOF` |
| Uploaded by tenant via task | `SOURCE` |

---

## 6. Immutability Rules

### 6.1 Protection Rules

| State | Modification Allowed | Delete Allowed |
|-------|---------------------|----------------|
| Case open | ✅ Yes | ✅ Yes (soft delete) |
| Case closed | ❌ No | ❌ No |
| Evidence attached | ❌ No | ❌ No |

### 6.2 Admin Override

Admins can modify protected documents with:
- Override reason (required)
- Audit log entry
- Notification to relevant parties

```typescript
async function modifyProtectedDocument(
  documentId: string,
  action: 'update' | 'delete',
  reason: string,
  staffId: string,
  isAdmin: boolean
): Promise<Result> {
  if (!isAdmin) {
    return { error: 'Admin role required for protected documents' };
  }
  
  if (!reason) {
    return { error: 'Override reason required' };
  }
  
  // Perform action
  // Create audit log with override details
}
```

### 6.3 Soft Delete

Documents are never hard-deleted:

```sql
UPDATE documents 
SET deleted_at = NOW()
WHERE id = $documentId;
```

Soft-deleted documents:
- Hidden from normal queries
- Preserved for audit purposes
- Can be restored by admin

---

## Documents Table Columns

| Column | Description |
|--------|-------------|
| Filename | Original filename with icon |
| Organization | Tenant name |
| Tag | Document kind badge |
| Case | Linked case (if any) |
| Uploaded By | Staff or tenant name |
| Date | Upload date |
| Size | File size |
| Actions | View, Download, Attach |

---

## Filters

| Filter | Options |
|--------|---------|
| Organization | Search/autocomplete |
| Tag | Multi-select document kinds |
| Case | Case ID search |
| Date Range | Upload date range |
| Uploaded By | Staff filter |
| Module | Compliance, Invoicing, etc. |

---

## File Locations

**Current Implementation:**

| Component | Location |
|-----------|----------|
| Documents page | `apps/backoffice/app/(authenticated)/(modules)/documents/page.tsx` |
| Document schema | `packages/database/src/schema/compliance/documents.ts` |
| Document kind enum | `packages/database/src/schema/enums.ts` |

**To Create:**

| Component | Location |
|-----------|----------|
| Upload dropzone | `apps/backoffice/components/documents/upload-dropzone.tsx` |
| Evidence modal | `apps/backoffice/components/documents/evidence-upload-modal.tsx` |
| Tag select | `apps/backoffice/components/documents/document-tag-select.tsx` |
| API routes | `packages/backend/src/modules/documents/routes.ts` |
| Storage service | `packages/api-services/src/domains/documents/storage.service.ts` |

---

## Related Documentation

- [Operations Module](/backoffice/modules/operations) - Cases that use documents
- [Finance Module](/backoffice/modules/finance) - Payment evidence
- [Implementation Plan](/backoffice/implementation-plan) - Build steps

