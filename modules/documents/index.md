---
title: "Documents Module"
description: "Backend documentation for the Bumara Documents module with AWS S3 storage."
---

The Documents module provides secure file storage and retrieval for compliance-related documents. Files are stored in AWS S3 with metadata in PostgreSQL, accessed via presigned URLs.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         DOCUMENTS MODULE                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌──────────┐     ┌──────────────┐     ┌─────────────────────┐    │
│   │  Client  │────►│  Hono API    │────►│  PostgreSQL (Meta)  │    │
│   │  (App)   │     │  /documents  │     │  documents table    │    │
│   └────┬─────┘     └──────────────┘     └─────────────────────┘    │
│        │                                                             │
│        │ Presigned URL                                               │
│        ▼                                                             │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                     AWS S3 Bucket                            │   │
│   │  bumara-documents-{env}                                      │   │
│   │  ├── {org_id}/{document_id}/{filename}                       │   │
│   │  └── Private, encrypted, versioning optional                 │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Core Principle:** Blob in S3, metadata in DB, access via presigned URLs.

---

## Module Scope

### MVP (Phase 1)

- [x] Direct-to-S3 uploads via presigned PUT URLs
- [x] Signed download URLs with short TTL
- [x] Document metadata stored in PostgreSQL
- [x] Link documents to Filings and Service Requests
- [x] Document audit events (upload, download, link, archive, lock)
- [x] Immutability locking for compliance evidence
- [x] Org-scoped access control

### Later (Phase 2+)

- [ ] Thumbnail/preview generation (Lambda trigger)
- [ ] Malware scanning integration (quarantine status)
- [ ] S3 Object Lock for legal hold
- [ ] Full-text search (OpenSearch/Algolia)
- [ ] Bulk upload/download (ZIP)
- [ ] Document versioning UI
- [ ] OCR extraction for invoices/receipts

---

## Documentation Index

| Document | Description |
|----------|-------------|
| [backend.md](/modules/documents/backend) | Implementation steps and responsibilities |
| [api.md](/modules/documents/api) | API endpoint contracts |
| [data-model.md](/modules/documents/data-model) | Database tables, enums, indexes |
| [s3-storage.md](/modules/documents/s3-storage) | S3 bucket configuration and presigned URL flows |
| [security-permissions.md](/modules/documents/security-permissions) | Authorization rules and immutability locking |
| [runbook.md](/modules/documents/runbook) | Operations, troubleshooting, and testing |

### Related Documentation

| Document | Description |
|----------|-------------|
| [ADR: S3 Presigned URLs](/adr/0001-documents-s3-presigned-urls) | Architecture decision record |
| [Database Schema](/ARCHITECTURE/DATABASE_SCHEMA) | Full database architecture |
| [API Setup](/API-SETUP) | Hono backend setup guide |

---

## Quick Reference

### Document Kinds

| Kind | Description | Example |
|------|-------------|---------|
| `source` | Input documents from tenant | Financial statements, invoices |
| `workpaper` | Working documents during preparation | Calculations, notes |
| `submission` | Screenshots/forms submitted to regulator | Portal submission proof |
| `receipt` | Payment proof | Bank receipts, mobile money confirmations |
| `certificate` | Approval letters from regulator | Tax clearance, registration certificates |

### Document Statuses

| Status | Description |
|--------|-------------|
| `uploading` | Presigned URL issued, upload in progress |
| `active` | Upload complete, document available |
| `archived` | Soft-deleted, not visible in lists |
| `failed` | Upload failed or timed out |
| `quarantine` | Flagged by malware scan (future) |

### Key Flows

1. **Upload**: Client → Init → Presigned PUT → S3 → Complete → DB
2. **Download**: Client → Request URL → Presigned GET → S3
3. **Link**: Attach document to Filing/ServiceRequest
4. **Lock**: When Filing/Case becomes ACCEPTED, lock evidence docs

---

## Environment Variables

```bash
# AWS S3 Configuration
AWS_REGION=af-south-1
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
S3_DOCUMENTS_BUCKET=bumara-documents-production

# Presigned URL Settings
S3_UPLOAD_URL_EXPIRY_SECONDS=3600      # 1 hour for uploads
S3_DOWNLOAD_URL_EXPIRY_SECONDS=300     # 5 minutes for downloads
S3_MAX_FILE_SIZE_BYTES=52428800        # 50MB max
```

---

## Getting Started

1. Read [data-model.md](/modules/documents/data-model) to understand the schema
2. Follow [backend.md](/modules/documents/backend) for implementation steps
3. Use [api.md](/modules/documents/api) as the endpoint reference
4. Configure S3 per [s3-storage.md](/modules/documents/s3-storage)
5. Implement security rules from [security-permissions.md](/modules/documents/security-permissions)
6. Use [runbook.md](/modules/documents/runbook) for testing and troubleshooting

