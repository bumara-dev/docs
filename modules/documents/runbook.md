---
title: "Documents Module - Runbook"
description: "Operations guide: cleanup jobs, troubleshooting, and testing checklist."
---

## Table of Contents

1. [Scheduled Jobs](#scheduled-jobs)
2. [Monitoring](#monitoring)
3. [Troubleshooting Guide](#troubleshooting-guide)
4. [Testing Checklist](#testing-checklist)
5. [Common Operations](#common-operations)
6. [Incident Response](#incident-response)

---

## Scheduled Jobs

### Orphan Upload Cleanup

**Purpose:** Clean up documents stuck in `uploading` status.

**Schedule:** Daily at 02:00 UTC

**Logic:**

1. Find documents where `status = 'uploading'` AND `created_at < NOW() - 24 hours`
2. Mark them as `status = 'failed'`
3. Optionally delete the S3 object (may not exist)
4. Log cleanup count for monitoring

**Implementation:**

```typescript
// packages/backend/src/jobs/cleanup-orphan-uploads.ts

import { and, eq, lt } from "drizzle-orm";
import { subHours } from "date-fns";

export async function cleanupOrphanUploads(
  db: DrizzleClient,
  s3Client: S3Client,
  logger: Logger
) {
  const cutoff = subHours(new Date(), 24);
  
  // Find orphan uploads
  const orphans = await db
    .select()
    .from(documents)
    .where(
      and(
        eq(documents.status, "uploading"),
        lt(documents.createdAt, cutoff)
      )
    );
  
  logger.info({ count: orphans.length }, "Found orphan uploads to clean up");
  
  let cleaned = 0;
  let errors = 0;
  
  for (const doc of orphans) {
    try {
      // Mark as failed
      await db
        .update(documents)
        .set({ status: "failed", updatedAt: new Date() })
        .where(eq(documents.id, doc.id));
      
      // Try to delete S3 object (may not exist)
      try {
        await s3Client.send(
          new DeleteObjectCommand({
            Bucket: BUCKET,
            Key: doc.storageKey,
          })
        );
      } catch (s3Error) {
        // Object may not exist - that's fine
        logger.debug({ storageKey: doc.storageKey }, "S3 object not found");
      }
      
      cleaned++;
    } catch (error) {
      errors++;
      logger.error({ documentId: doc.id, error }, "Failed to cleanup orphan");
    }
  }
  
  logger.info({ cleaned, errors, total: orphans.length }, "Orphan cleanup complete");
  
  return { cleaned, errors };
}
```

**Cron Configuration (Cloudflare Workers):**

```typescript
// packages/backend/src/index.ts

export default {
  async scheduled(event: ScheduledEvent, env: Env, ctx: ExecutionContext) {
    if (event.cron === "0 2 * * *") { // Daily at 02:00 UTC
      await cleanupOrphanUploads(db, s3Client, logger);
    }
  },
};
```

**Wrangler Config:**

```jsonc
// packages/backend/wrangler.jsonc
{
  "triggers": {
    "crons": ["0 2 * * *"]
  }
}
```

### Future: Malware Scan Quarantine

**Purpose:** Move suspicious files to quarantine status.

**When implemented:**

1. S3 event triggers Lambda on PutObject
2. Lambda scans file with ClamAV or similar
3. If malicious: update document status to `quarantine`
4. Alert admin via Slack/email

---

## Monitoring

### Key Metrics

| Metric | Alert Threshold | Description |
|--------|-----------------|-------------|
| Upload init rate | > 1000/hour | Potential abuse |
| Upload failure rate | > 10% | Integration issue |
| Orphan uploads | > 100 pending | Cleanup not running |
| Download 403 rate | > 5% | Permission issues |
| S3 error rate | > 1% | AWS issue |
| Avg upload time | > 30s | Performance degradation |

### Logging

**Key log events to monitor:**

```typescript
// Upload initiated
logger.info({ documentId, orgId, filename, sizeBytes }, "document.upload.initiated");

// Upload completed
logger.info({ documentId, orgId, durationMs }, "document.upload.completed");

// Upload failed
logger.error({ documentId, orgId, error }, "document.upload.failed");

// Download URL generated
logger.info({ documentId, orgId, userId }, "document.download.url_generated");

// Document locked
logger.info({ documentId, orgId, reason }, "document.locked");

// Cleanup job
logger.info({ cleaned, errors }, "jobs.orphan_cleanup.complete");
```

### Health Check

```typescript
// GET /health/documents

export async function documentsHealthCheck(db: DrizzleClient, s3Client: S3Client) {
  const checks = {
    database: false,
    s3: false,
  };
  
  // Check database
  try {
    await db.select().from(documents).limit(1);
    checks.database = true;
  } catch (error) {
    // Database unreachable
  }
  
  // Check S3
  try {
    await s3Client.send(
      new HeadBucketCommand({ Bucket: BUCKET })
    );
    checks.s3 = true;
  } catch (error) {
    // S3 unreachable or permission denied
  }
  
  const healthy = checks.database && checks.s3;
  
  return {
    status: healthy ? "healthy" : "unhealthy",
    checks,
  };
}
```

---

## Troubleshooting Guide

### Problem: Upload init works but complete fails

**Symptoms:**

- `POST /uploads/init` returns 201 with presigned URL
- Client uploads file to S3 successfully
- `POST /uploads/complete` returns 409 "File not found in S3"

**Possible Causes:**

1. **Wrong Content-Type:** Client sent different Content-Type than declared
2. **Wrong Content-Length:** File size doesn't match declared size
3. **Eventual consistency:** Very rare, but object may not be immediately visible
4. **Wrong storage key:** Client modified the presigned URL

**Resolution Steps:**

1. Check S3 directly for the object:
   ```bash
   aws s3 ls s3://bumara-documents-production/org_xxx/doc_yyy/
   ```

2. Check document record in database:
   ```sql
   SELECT id, storage_key, status, created_at 
   FROM documents 
   WHERE id = 'xxx';
   ```

3. Compare storage key in DB with actual S3 path

4. If object exists but HEAD fails, check IAM permissions

### Problem: 403 on download URL

**Symptoms:**

- Download URL generates successfully
- Client gets 403 Forbidden when accessing URL

**Possible Causes:**

1. **URL expired:** Default TTL is 5 minutes
2. **Wrong bucket policy:** Public access blocked (correct, but check CORS)
3. **URL modified:** Signature invalid if URL is changed

**Resolution Steps:**

1. Check if URL has expired (compare `expiresAt` with current time)

2. Verify CORS configuration allows the origin:
   ```bash
   aws s3api get-bucket-cors --bucket bumara-documents-production
   ```

3. Test with a fresh download URL

4. Check CloudWatch logs for S3 access errors

### Problem: Wrong org access denied

**Symptoms:**

- User sees "Document not found" for a document that exists
- Document belongs to a different organization

**This is correct behavior!** Org isolation is working.

**If legitimate access is needed:**

1. Verify user is member of the correct organization in Clerk
2. Check `orgId` in the JWT token:
   ```javascript
   // Decode JWT payload
   const payload = JSON.parse(atob(token.split('.')[1]));
   console.log(payload.orgId);
   ```
3. Ensure user switched to the correct org in the app

### Problem: Documents not appearing in list

**Symptoms:**

- User uploaded documents but they don't appear in the list
- No errors in logs

**Possible Causes:**

1. **Status filter:** Documents may be in `uploading` or `failed` status
2. **Archived:** Documents were archived
3. **Different org context:** User switched orgs

**Resolution Steps:**

1. Query all documents for the org (including all statuses):
   ```sql
   SELECT id, filename, status, created_at 
   FROM documents 
   WHERE organization_id = 'org_xxx'
   ORDER BY created_at DESC;
   ```

2. Check for failed uploads:
   ```sql
   SELECT COUNT(*) as failed_count 
   FROM documents 
   WHERE organization_id = 'org_xxx' 
   AND status = 'failed';
   ```

### Problem: Locked document blocking workflow

**Symptoms:**

- User cannot archive or unlink a document
- Error: "Document is locked and cannot be modified"

**This is expected for compliance!**

**If lock was applied incorrectly:**

1. Verify the filing/service request status:
   ```sql
   SELECT f.id, f.status, d.id as doc_id, d.locked_at, d.locked_reason
   FROM documents d
   JOIN filings f ON d.filing_id = f.id
   WHERE d.id = 'xxx';
   ```

2. **If filing is not actually ACCEPTED**, this is a bug - file an issue

3. **Locks cannot be removed** - this is by design for compliance

4. Contact backoffice admin for guidance

---

## Testing Checklist

### Unit Tests

**File:** `packages/api-services/src/domains/documents/__tests__/`

- [ ] `buildStorageKey` generates valid keys
- [ ] `buildStorageKey` sanitizes special characters
- [ ] `buildStorageKey` handles long filenames
- [ ] Permission checks return correct boolean
- [ ] `assertNotLocked` throws for locked documents
- [ ] `assertNotLocked` passes for unlocked documents

### Integration Tests

**File:** `packages/backend/src/modules/documents/__tests__/`

- [ ] **Init upload** returns presigned URL and document ID
- [ ] **Init upload** rejects invalid MIME type
- [ ] **Init upload** rejects file too large
- [ ] **Complete upload** succeeds when S3 object exists
- [ ] **Complete upload** fails when S3 object missing
- [ ] **Complete upload** fails for wrong org
- [ ] **List documents** returns only org's documents
- [ ] **List documents** filters by kind/status correctly
- [ ] **Get document** returns 404 for other org's doc
- [ ] **Download URL** generates valid presigned URL
- [ ] **Link document** creates link record
- [ ] **Link document** fails for locked document
- [ ] **Unlink document** removes link record
- [ ] **Archive document** sets status to archived
- [ ] **Archive document** fails for locked document
- [ ] **Lock document** sets lock fields
- [ ] **Lock document** requires backoffice role

### Manual QA Steps

1. **Upload flow:**
   - [ ] Open app, navigate to Documents
   - [ ] Click Upload, select a PDF file
   - [ ] Verify file uploads (progress indicator)
   - [ ] Verify document appears in list with `active` status
   - [ ] Download the document, verify content matches

2. **Org isolation:**
   - [ ] Upload a document in Org A
   - [ ] Switch to Org B
   - [ ] Verify document is NOT visible
   - [ ] Try to access document by direct URL → should fail

3. **Link flow:**
   - [ ] Create a Filing
   - [ ] Upload a document
   - [ ] Link document to Filing
   - [ ] Verify document appears in Filing's documents list

4. **Lock flow:**
   - [ ] Create a Filing with linked document
   - [ ] Change Filing status to ACCEPTED (via backoffice)
   - [ ] Verify evidence documents are locked
   - [ ] Try to archive locked document → should fail

5. **Archive flow:**
   - [ ] Upload a document
   - [ ] Archive the document
   - [ ] Verify document no longer in main list
   - [ ] Verify document visible with archived filter

---

## Common Operations

### Find documents for a specific filing

```sql
SELECT d.* 
FROM documents d
JOIN document_links dl ON dl.document_id = d.id
WHERE dl.entity_type = 'filing' 
  AND dl.entity_id = 'filing-uuid'
ORDER BY d.created_at DESC;
```

### Count documents by status per org

```sql
SELECT organization_id, status, COUNT(*) as count
FROM documents
GROUP BY organization_id, status
ORDER BY organization_id, status;
```

### Find all locked documents

```sql
SELECT id, organization_id, filename, locked_at, locked_by, locked_reason
FROM documents
WHERE locked_at IS NOT NULL
ORDER BY locked_at DESC;
```

### Check S3 usage per org

```bash
# List all objects for an org with sizes
aws s3 ls s3://bumara-documents-production/org_xxx/ --recursive --summarize
```

### Manually mark upload as failed

```sql
UPDATE documents
SET status = 'failed', updated_at = NOW()
WHERE id = 'document-uuid' 
  AND status = 'uploading';
```

### Get document event history

```sql
SELECT event_type, actor_id, actor_type, metadata, created_at
FROM document_events
WHERE document_id = 'document-uuid'
ORDER BY created_at ASC;
```

---

## Incident Response

### Scenario: Mass upload failure

**Symptoms:** Many uploads failing simultaneously

**Steps:**

1. Check AWS S3 service health: https://status.aws.amazon.com/
2. Check IAM credentials haven't expired
3. Verify bucket exists and permissions are correct
4. Check Cloudflare Worker logs for errors
5. If S3 is down, communicate to users and wait for recovery

### Scenario: Data breach suspected

**Steps:**

1. **Immediate:** Rotate AWS credentials
2. **Immediate:** Revoke all active presigned URLs (change bucket policy temporarily)
3. Check audit logs for suspicious access patterns:
   ```sql
   SELECT * FROM document_events
   WHERE event_type = 'downloaded'
   AND created_at > NOW() - INTERVAL '24 hours'
   ORDER BY created_at DESC;
   ```
4. Identify affected documents and organizations
5. Notify affected organizations per legal requirements
6. Review and tighten IAM permissions

### Scenario: Orphan cleanup not running

**Symptoms:** Growing count of `uploading` status documents

**Steps:**

1. Check cron trigger is configured in `wrangler.jsonc`
2. Check Cloudflare dashboard for scheduled worker runs
3. Run cleanup manually:
   ```bash
   wrangler dev --test-scheduled
   ```
4. Check worker logs for errors during cleanup job

### Scenario: Locked documents blocking legitimate workflow

**This should not happen** - if it does:

1. Verify the Filing/Service Request status
2. If status is not ACCEPTED but docs are locked, this is a bug
3. **Do NOT manually unlock** - this breaks compliance
4. Escalate to engineering with full context

