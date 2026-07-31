---
title: "Documents Module - Security & Permissions"
description: "Authorization rules, tenant isolation, and immutability locking for the Documents module."
---

## Table of Contents

1. [Security Principles](#security-principles)
2. [Tenant Isolation](#tenant-isolation)
3. [Role-Based Access Control](#role-based-access-control)
4. [Immutability Locking](#immutability-locking)
5. [Access Control Matrix](#access-control-matrix)
6. [Implementation Patterns](#implementation-patterns)
7. [Audit Trail](#audit-trail)

---

## Security Principles

### Core Rules

1. **Org isolation is mandatory** - Every query filters by `organization_id`
2. **Derive org from session** - Never accept `orgId` from client request body
3. **Least privilege** - Users can only perform actions their role allows
4. **Audit everything** - All state changes create audit events
5. **Locked means locked** - No exceptions for locked documents

### Defense in Depth

```
┌─────────────────────────────────────────────────────────────────┐
│                     SECURITY LAYERS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Authentication (Clerk JWT)                                   │
│     └─ Valid token required for all endpoints                    │
│                                                                  │
│  2. Organization Context                                         │
│     └─ orgId extracted from JWT, not request body                │
│                                                                  │
│  3. Database Query Filtering                                     │
│     └─ WHERE organization_id = ? on every query                  │
│                                                                  │
│  4. Role-Based Access Control                                    │
│     └─ Permissions checked per action                            │
│                                                                  │
│  5. State Guards                                                 │
│     └─ Locked documents block modifications                      │
│                                                                  │
│  6. Audit Logging                                                │
│     └─ Every action recorded with actor + timestamp              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Tenant Isolation

### The Golden Rule

> **Every database query MUST filter by `organization_id` derived from the authenticated session.**

### Implementation Pattern

```typescript
// ✅ CORRECT: Org ID from session context
export async function getDocument(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  documentId: string
) {
  const orgId = ctx.orgId; // From JWT, NOT from request
  
  const document = await deps.db.query.documents.findFirst({
    where: and(
      eq(documents.id, documentId),
      eq(documents.organizationId, orgId) // REQUIRED
    ),
  });
  
  if (!document) {
    throw new NotFoundError("Document not found");
  }
  
  return document;
}

// ❌ WRONG: Accepting orgId from request
export async function getDocumentWRONG(
  deps: ServiceDependencies,
  documentId: string,
  orgId: string // NEVER accept org from request body
) {
  // This allows cross-org access!
}
```

### Query Patterns

Every document query must include org filter:

```typescript
// List documents
const docs = await db
  .select()
  .from(documents)
  .where(
    and(
      eq(documents.organizationId, ctx.orgId), // REQUIRED
      eq(documents.status, "active")
    )
  );

// Update document
await db
  .update(documents)
  .set({ status: "archived" })
  .where(
    and(
      eq(documents.id, documentId),
      eq(documents.organizationId, ctx.orgId) // REQUIRED
    )
  );

// Delete link
await db
  .delete(documentLinks)
  .where(
    and(
      eq(documentLinks.id, linkId),
      eq(documentLinks.organizationId, ctx.orgId) // REQUIRED
    )
  );
```

---

## Role-Based Access Control

### User Roles

From `packages/database/src/schema/enums.ts`:

| Role | Description | Document Access |
|------|-------------|-----------------|
| `admin` | Organization administrator | Full access |
| `manager` | Team manager | Full access |
| `member` | Regular team member | Upload, view, link |
| `backoffice_admin` | Backoffice administrator | Full + lock |
| `backoffice_manager` | Backoffice manager | Full + lock |
| `backoffice_member` | Backoffice agent | View, download |

### Permission Definitions

```typescript
type DocumentPermission =
  | "documents:upload"      // Init + complete uploads
  | "documents:view"        // List and get documents
  | "documents:download"    // Generate download URLs
  | "documents:link"        // Link to entities
  | "documents:unlink"      // Remove links
  | "documents:archive"     // Soft-delete
  | "documents:lock"        // Immutability lock
  | "documents:admin";      // All permissions

const ROLE_PERMISSIONS: Record<string, DocumentPermission[]> = {
  admin: ["documents:admin"],
  manager: ["documents:upload", "documents:view", "documents:download", 
            "documents:link", "documents:unlink", "documents:archive"],
  member: ["documents:upload", "documents:view", "documents:download", 
           "documents:link"],
  
  backoffice_admin: ["documents:admin"],
  backoffice_manager: ["documents:view", "documents:download", 
                       "documents:link", "documents:lock"],
  backoffice_member: ["documents:view", "documents:download"],
};
```

### Permission Check Helper

```typescript
function hasPermission(
  userRole: string, 
  permission: DocumentPermission
): boolean {
  const permissions = ROLE_PERMISSIONS[userRole] || [];
  
  if (permissions.includes("documents:admin")) {
    return true; // Admin has all permissions
  }
  
  return permissions.includes(permission);
}

// Usage in handler
if (!hasPermission(ctx.orgRole, "documents:archive")) {
  throw new ForbiddenError("You do not have permission to archive documents");
}
```

---

## Immutability Locking

### Purpose

Certain documents become **compliance evidence** and must be preserved unchanged:

- Submission screenshots
- Payment receipts
- Regulator certificates

Once a Filing or Service Request is **ACCEPTED** by the regulator, evidence documents are locked.

### Lockable Document Kinds

| Kind | Auto-Lock on ACCEPTED | Reason |
|------|----------------------|--------|
| `submission` | ✅ Yes | Proof of what was submitted |
| `receipt` | ✅ Yes | Payment proof |
| `certificate` | ✅ Yes | Regulator approval |
| `source` | ❌ No | Input documents may be updated |
| `workpaper` | ❌ No | Working documents |

### Lock Trigger

When a Filing or Service Request transitions to `ACCEPTED` status:

```typescript
async function onFilingAccepted(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  filingId: string
) {
  // Find all linked documents
  const links = await deps.db
    .select()
    .from(documentLinks)
    .where(
      and(
        eq(documentLinks.entityType, "filing"),
        eq(documentLinks.entityId, filingId),
        eq(documentLinks.organizationId, ctx.orgId)
      )
    );

  const documentIds = links.map((l) => l.documentId);

  // Lock evidence documents
  const lockableKinds = ["submission", "receipt", "certificate"];
  
  const lockedCount = await deps.db
    .update(documents)
    .set({
      lockedAt: new Date(),
      lockedBy: "system",
      lockedReason: "Auto-locked: Filing accepted by regulator",
    })
    .where(
      and(
        inArray(documents.id, documentIds),
        inArray(documents.kind, lockableKinds),
        isNull(documents.lockedAt) // Not already locked
      )
    );

  // Log events for each locked document
  for (const docId of documentIds) {
    await logDocumentEvent(deps, docId, ctx.orgId, "locked", "system", "system", {
      reason: "Filing accepted",
      filingId,
    });
  }

  return lockedCount;
}
```

### Locked Document Restrictions

Once a document is locked (`lockedAt IS NOT NULL`), the following are **blocked**:

| Action | Allowed? | Error Code |
|--------|----------|------------|
| View metadata | ✅ Yes | - |
| Generate download URL | ✅ Yes | - |
| Add new link | ❌ No | `CONFLICT` |
| Remove link | ❌ No | `CONFLICT` |
| Update metadata | ❌ No | `CONFLICT` |
| Archive | ❌ No | `CONFLICT` |
| Delete | ❌ No | `CONFLICT` |
| Lock again | ❌ No | `CONFLICT` |

### Lock Guard Implementation

```typescript
function assertNotLocked(document: Document): void {
  if (document.lockedAt !== null) {
    throw new ConflictError(
      "Document is locked and cannot be modified",
      {
        hint: "This document was locked because the associated filing was accepted.",
        lockedAt: document.lockedAt,
        lockedBy: document.lockedBy,
        lockedReason: document.lockedReason,
      }
    );
  }
}

// Usage in service functions
export async function archiveDocument(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  documentId: string,
  input: ArchiveDocumentInput
) {
  const document = await getDocument(ctx, deps, documentId);
  
  assertNotLocked(document); // Throws if locked
  
  // Proceed with archive...
}
```

### Manual Lock (Backoffice)

Backoffice users can manually lock documents:

```typescript
export async function lockDocument(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  documentId: string,
  input: LockDocumentInput
) {
  // Check permission
  if (!hasPermission(ctx.orgRole, "documents:lock")) {
    throw new ForbiddenError("Only backoffice users can lock documents");
  }
  
  const document = await getDocument(ctx, deps, documentId);
  
  if (document.lockedAt !== null) {
    throw new ConflictError("Document is already locked");
  }
  
  await deps.db
    .update(documents)
    .set({
      lockedAt: new Date(),
      lockedBy: ctx.userId,
      lockedReason: input.reason,
    })
    .where(eq(documents.id, documentId));
  
  await logDocumentEvent(
    deps, documentId, ctx.orgId, "locked", ctx.userId, "backoffice",
    { reason: input.reason }
  );
}
```

---

## Access Control Matrix

### By Action

| Action | admin | manager | member | backoffice_admin | backoffice_manager | backoffice_member |
|--------|-------|---------|--------|------------------|--------------------|--------------------|
| Upload | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| View/List | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Download | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Link | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| Unlink | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ |
| Archive | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ |
| Lock | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ |

### By Document State

| State | View | Download | Link | Unlink | Archive | Lock |
|-------|------|----------|------|--------|---------|------|
| `uploading` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `active` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `active` + locked | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `archived` | ✅* | ❌ | ❌ | ❌ | ❌ | ❌ |
| `failed` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |

*Archived documents visible only with explicit filter

---

## Implementation Patterns

### Middleware: Require Auth + Org

```typescript
// packages/backend/src/core/middleware/auth.ts

export const requireAuth: MiddlewareHandler = async (c, next) => {
  const token = c.req.header("Authorization")?.replace("Bearer ", "");
  
  if (!token) {
    return c.json({ error: { code: "UNAUTHORIZED" } }, 401);
  }
  
  const claims = await verifyClerkToken(token);
  
  c.set("userId", claims.userId);
  c.set("orgId", claims.orgId);
  c.set("orgRole", claims.orgRole);
  
  await next();
};

export const requireOrg: MiddlewareHandler = async (c, next) => {
  const orgId = c.get("orgId");
  
  if (!orgId) {
    return c.json({ 
      error: { 
        code: "FORBIDDEN", 
        message: "Organization context required" 
      } 
    }, 403);
  }
  
  await next();
};
```

### Service Context Pattern

```typescript
// packages/api-services/src/types.ts

export interface ServiceContext {
  userId: string;
  orgId: string;
  orgRole: string;
  requestId?: string;
}

export interface ServiceDependencies {
  db: DrizzleClient;
  logger: Logger;
}

// Build from Hono context
export function buildServiceContext(c: Context): ServiceContext {
  return {
    userId: c.get("userId"),
    orgId: c.get("orgId"),
    orgRole: c.get("orgRole"),
    requestId: c.get("requestId"),
  };
}
```

### Error Responses

```typescript
// Unauthorized (missing/invalid token)
{
  "success": false,
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Authentication required"
  }
}

// Forbidden (valid auth, wrong permissions)
{
  "success": false,
  "error": {
    "code": "FORBIDDEN",
    "message": "You do not have permission to lock documents"
  }
}

// Not Found (document doesn't exist OR belongs to different org)
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "Document not found"
  }
}

// Conflict (locked document)
{
  "success": false,
  "error": {
    "code": "CONFLICT",
    "message": "Document is locked and cannot be modified",
    "hint": "This document was locked because the filing was accepted."
  }
}
```

---

## Audit Trail

### Event Types

| Event | When | Actor Types |
|-------|------|-------------|
| `upload_initiated` | `POST /uploads/init` | user |
| `uploaded` | `POST /uploads/complete` | user |
| `downloaded` | `POST /download-url` | user, backoffice |
| `linked` | `POST /link` | user, backoffice |
| `unlinked` | `DELETE /link/:id` | user, backoffice |
| `locked` | Auto-lock or manual | system, backoffice |
| `archived` | `POST /archive` | user |
| `metadata_updated` | Future | user |

### Event Schema

```typescript
interface DocumentEvent {
  id: string;
  documentId: string;
  organizationId: string;
  eventType: DocumentEventType;
  actorId: string;      // userId or "system"
  actorType: string;    // "user" | "system" | "backoffice"
  metadata: {
    entityType?: string;  // For link events
    entityId?: string;
    reason?: string;      // For lock/archive
    ipAddress?: string;   // If available
    userAgent?: string;
  };
  createdAt: Date;
}
```

### Logging Best Practices

1. **Log every access** - Even successful reads (for download audit)
2. **Never log sensitive data** - No file contents, credentials, or PII
3. **Include context** - requestId, orgId, userId
4. **Immutable events** - No updates or deletes on event records

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
  
  deps.logger.info({
    event: "document_event",
    documentId,
    eventType,
    actorId,
    actorType,
  });
}
```

