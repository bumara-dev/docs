---
title: "ADR-0001: Documents Storage with S3 Presigned URLs"
description: "Status: Accepted Date: 2025-12-31 Authors: Engineering Team Reviewers: CTO"
---

**Status:** Accepted  
**Date:** 2025-12-31  
**Authors:** Engineering Team  
**Reviewers:** CTO

---

## Context

The Bumara platform requires a document storage solution for the Documents module. Users need to upload, store, and retrieve various file types (PDFs, images, spreadsheets) associated with compliance workflows like Filings and Service Requests.

### Requirements

1. **Secure uploads:** Files must be uploaded securely without exposing cloud credentials
2. **Tenant isolation:** Organizations must only access their own documents
3. **Compliance:** Evidence documents must be immutable once filings are accepted
4. **Performance:** Large files (up to 50MB) must upload efficiently, especially on slow networks
5. **Cost-effective:** Storage costs should be manageable for a startup
6. **Audit trail:** All document access must be logged

### Constraints

- Backend runs on Cloudflare Workers (serverless, 128KB response size limit)
- Frontend is Next.js deployed on Vercel
- Database is PostgreSQL (Neon)
- Budget constraints favor pay-as-you-go pricing

---

## Decision

We will use **AWS S3** for blob storage with **presigned URLs** for both uploads and downloads.

### Architecture

```
┌──────────┐     ┌─────────┐     ┌──────────────┐     ┌────────┐
│  Client  │────►│  API    │────►│  PostgreSQL  │     │   S3   │
│  (Next)  │     │ (Hono)  │     │  (metadata)  │     │ (blob) │
└────┬─────┘     └────┬────┘     └──────────────┘     └────▲───┘
     │                │                                     │
     │                │ Presigned URL                       │
     │◄───────────────┘                                     │
     │                                                      │
     │ Direct upload/download                               │
     └──────────────────────────────────────────────────────┘
```

**Flow:**

1. Client requests upload URL from API
2. API creates document record in DB, generates presigned PUT URL
3. Client uploads file directly to S3
4. Client confirms upload complete
5. API verifies object exists, marks document active

---

## Consequences

### Positive

1. **Scalable:** S3 handles unlimited concurrent uploads
2. **Cost-effective:** Pay only for storage and requests
3. **Secure:** Credentials never exposed to client
4. **Fast:** Direct S3 upload, no proxy overhead
5. **Flexible:** Full control over bucket policies, lifecycle, encryption
6. **Regional:** `af-south-1` provides low latency for target market
7. **Audit-ready:** All access goes through API, logged in database

### Negative

1. **Two-step upload:** Client must call API before and after upload
2. **AWS dependency:** Adds AWS to infrastructure alongside Cloudflare/Vercel
3. **Credential management:** Need to securely store and rotate AWS keys
4. **CORS configuration:** Required for cross-origin uploads
5. **Orphan objects:** Uploads started but not completed need cleanup

### Mitigations

| Risk | Mitigation |
|------|------------|
| Orphan objects | Daily cleanup job marks as failed after 24h |
| Credential exposure | Environment variables, never in code |
| CORS issues | Explicit CORS configuration per environment |
| Two-step complexity | Clear client SDK/hooks abstract the flow |

---

## Implementation Notes

### S3 Bucket Configuration

- **Region:** `af-south-1` (Cape Town)
- **Encryption:** SSE-S3 (AES-256)
- **Versioning:** Enabled for production
- **Public access:** Blocked
- **CORS:** Configured for app domains

### Key Naming

```
{organization_id}/{document_id}/{sanitized_filename}
```

Org prefix enables efficient listing and prevents cross-org access even if presigned URL is leaked (URL is scoped to specific key).

### Presigned URL Expiry

| Operation | Expiry | Reason |
|-----------|--------|--------|
| PUT (upload) | 1 hour | Allow time for large file uploads |
| GET (download) | 5 minutes | Short window, user can request again |

### IAM Permissions

Principle of least privilege:

```json
{
  "Effect": "Allow",
  "Action": ["s3:PutObject", "s3:GetObject", "s3:HeadObject", "s3:DeleteObject"],
  "Resource": "arn:aws:s3:::bumara-documents-*/*"
}
```

### Database Schema

Metadata stored in PostgreSQL:

- `documents` table: id, org_id, storage_key, filename, mime_type, size, status, locked_at
- `document_links` table: Polymorphic links to filings/requests
- `document_events` table: Audit trail

---

## Related Decisions

- **ADR-XXXX:** Cloudflare Workers for API (drives presigned URL approach)
- **ADR-XXXX:** PostgreSQL for metadata (Drizzle ORM)

---

## References

- [AWS S3 Presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [AWS SDK v3 for JavaScript](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/)
- [@aws-sdk/s3-request-presigner](https://www.npmjs.com/package/@aws-sdk/s3-request-presigner)
- [S3 CORS Configuration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/cors.html)

