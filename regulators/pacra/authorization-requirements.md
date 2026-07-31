---
title: "Pacra Authorization Document requirements"
description: "Created by: bumara Created time: March 23, 2026 4:09 PM Category: Feature Last edited by: bumara Last updated time: March 23, 2026 4:19 PM"
---

Created by: bumara
Created time: March 23, 2026 4:09 PM
Category: Feature
Last edited by: bumara
Last updated time: March 23, 2026 4:19 PM

---

---

## 1. Overview

When a tenant connects PACRA through Bumara, PACRA requires a signed **Online User Registration Form** authorizing Bumara to act on the business’s behalf. Bumara must not treat the PACRA connection as complete until this authorization document has been downloaded, signed, and uploaded.

The current PACRA connect flow activates the connection immediately and activates obligation templates without any authorization step. This must change.

The revised flow introduces a required authorization step after PACRA setup. The connection remains in `pending_approval` until the tenant uploads the signed authorization document. Once the document is uploaded, Bumara automatically activates the connection, links the document to the authorization record, activates deferred templates, and emits an asynchronous backoffice review task.

This keeps the tenant flow smooth while preserving the legal requirement for signed authorization.

---

## 2. Product Goal

Add a required authorization-document step to the PACRA connection flow so that:

- Bumara does not operate on behalf of the tenant before authorization is obtained
- PACRA connections remain incomplete until the signed form is uploaded
- obligation/template activation is deferred until authorization is completed
- uploaded authorization documents are reviewable by backoffice asynchronously
- tenants can leave and return later without losing progress

---

## 3. Current Problem

### Current behavior

`connectPacra()` currently:

- creates or updates the PACRA profile
- sets the regulator connection to `active`
- activates obligation templates immediately

### Desired behavior

`connectPacra()` should instead:

- create or update the PACRA profile
- create or update the regulator connection with `registrationStatus = "pending_approval"`
- create or update a PACRA authorization record with `status = "pending"`
- defer template activation
- move the user into an authorization step where they:
    - download a pre-filled DOCX
    - print and sign it
    - upload the signed copy

After successful upload:

- authorization record becomes active
- connection becomes active
- uploaded document is linked to both the authorization record and connection
- deferred template activation runs
- tenant is notified
- backoffice review is queued asynchronously

---

## 4. Existing Infrastructure

The following pieces already exist and should be reused.

### 4.1 `regulatorAuthorizations` table

Already supports:

- representative identity fields
- `documentId`
- `validFrom`, `validUntil`
- `status: pending | active | expired | revoked`
- audit timestamps

### 4.2 `authorizationDocId` on `organization_regulator_connections`

This already exists and should be used to directly link the signed PACRA authorization document to the regulator connection.

### 4.3 Authorization service

Existing service methods already cover most of the lifecycle:

- `getAuthorization`
- `getAuthorizationByRegulatorKey`
- `upsertAuthorization`
- `updateAuthorizationStatus`
- `deleteAuthorization`

### 4.4 Authorization API routes

Existing routes already support:

- get by regulator ID
- get by regulator key
- create/update
- patch status
- soft delete

### 4.5 Document upload infrastructure

Existing document and R2 upload flows already support:

- document metadata creation
- presigned upload URLs
- upload completion
- document download URLs
- reusable upload UI

### 4.6 Connection status enum

`pending_approval` already exists and is the correct intermediate status for this feature.

### 4.7 Document kind enum

Current document kinds do not include `authorization`.

This is optional, but adding it would improve categorization and reporting.

If a migration is not desirable now, `source` can be used initially.

---

## 5. Functional Requirements

### 5.1 PACRA connection must not activate immediately

On initial PACRA connection, the system must not set the connection to `active`.

It must instead:

- create/update the connection as `pending_approval`
- create/update an authorization record as `pending`
- skip template activation

### 5.2 Tenant must be able to download a pre-filled authorization form

The system must provide a DOCX download endpoint that generates a pre-filled PACRA authorization document using organization and initiating user data.

### 5.3 Tenant must upload a signed copy

The tenant must be able to upload a signed PDF or image of the form.

Accepted types:

- PDF
- docx
- docs

Maximum size:

- 10 MB

### 5.4 Upload must lead to a "Pending Approval"
- backoffice review event must be emitted
-The backoffice has to review  and either reject or accept/approves
  If the backoffice approves, the following happens instantly:
authorization record must reference the uploaded document
- authorization status must become `active`
- connection status must become `active`
- `authorizationDocId` must be set on the connection
- PACRA templates must activate
- tenant must receive a success notification


 the connection

After successful upload and completion:

- 

### 5.5 User must be able to resume later

If the user exits before uploading:

- connection remains `pending_approval`
- Regulators Hub shows a “Continue Setup” action
- reopening PACRA setup should land directly on the authorization step

### 5.6 Flow must be idempotent

If the connection is already active, completion should return success without duplicate activation or duplicate task generation.

---

## 6. State Model

### 6.1 Connection state transitions

```
not_connected
  -> pending_approval
  -> active
  -> suspended   (if backoffice rejects uploaded authorization)
```

### 6.2 Authorization state transitions

```
none
  -> pending
  -> active
  -> revoked     (if rejected)
  -> expired     (future extension)
```

### 6.3 Activation rule

PACRA obligations, filings, and task templates must only activate when the connection becomes `active`.

---

## 7. User Flows

## 7.1 Happy path

1. Tenant opens PACRA connect modal
2. Tenant completes PACRA classification/review steps
3. Backend creates:
    - PACRA profile
    - regulator connection with `pending_approval`
    - PACRA authorization record with `pending`
4. Backend does **not** activate templates
5. Modal advances to **Authorization Step**
6. Tenant downloads pre-filled DOCX
7. Tenant prints, signs, scans or photographs the form
8. Tenant uploads the signed file
9. System:
    - stores document
    - links it to authorization and connection
    - activates authorization
    - activates connection
    - runs deferred template activation
    - notifies tenant
    - emits backoffice review event
10. Regulators Hub shows PACRA as connected

---

## 7.2 Interrupted flow

1. Tenant completes PACRA setup but closes before upload
2. Connection remains `pending_approval`
3. Regulators Hub shows PACRA as “Linking” or equivalent pending badge
4. User clicks “Continue Setup”
5. Modal opens directly on authorization step
6. User uploads signed file
7. Connection completes normally

---

## 7.3 Backoffice review flow

1. System emits `PACRA_AUTHORIZATION_REVIEW_REQUIRED`
2. Backoffice staff reviews uploaded document
3. Staff may:
    - approve: no change to connection state
    - reject:
        - connection becomes `suspended`
        - authorization becomes `revoked`
        - tenant is notified to re-upload

---

## 8. DOCX Pre-fill Requirements

The PACRA authorization form should be pre-filled from existing tenant and initiating-user data.

| Form Field | Source |
| --- | --- |
| Date | Current date |
| Full name | Initiating admin user |
| Address | Organization address |
| Mobile number | User phone, fallback org phone |
| Email address | User email, fallback org email |
| Business name | Organization legal name |
| Registration number | Organization incorporation number |

### Fields intentionally left blank

These should remain blank for the user to fill manually:

- NRC / Passport number
- role in business
- director / partner signatory details
- signatures
- any other sensitive or handwritten fields

### Important template note

If placeholder replacement is done by editing `word/document.xml` directly, placeholders in the DOCX template must remain intact in single text runs. If Word splits placeholders across runs, replacement will fail. If that becomes unreliable, move to a DOCX templating library later.

---

## 9. Backend Design

## 9.1 Update `connectPacra()`

**File:** `packages/api-services/src/domains/pacra/pacra-connect.service.ts`

### Required changes

1. Change connection status from `active` to `pending_approval`
2. Create or update PACRA authorization record after connection creation
3. Defer template activation for new connections
4. Preserve existing active reconnection behavior if applicable

### Expected behavior

For a new PACRA connection:

- profile is created
- connection is stored as `pending_approval`
- authorization record is stored as `pending`
- activation is deferred

For an already active reconnection scenario:

- existing behavior may continue where appropriate

---

## 9.2 Add `completePacraAuthorization()`

This service should:

1. verify the uploaded document belongs to the current organization
2. load the PACRA connection
3. return success immediately if already active
4. validate the connection is in `pending_approval`
5. update the authorization record with the uploaded document
6. set authorization status to `active`
7. set connection status to `active`
8. set `authorizationDocId`
9. activate deferred PACRA templates
10. write audit log
11. emit tenant notification
12. emit backoffice review event

### Important implementation note

Do **not** upsert authorization with placeholder values like `"updated"` for `representativeName`.

Instead, either:

- fetch the existing authorization first and preserve representative fields, or
- extend the service with a targeted document-link/status update path

The completion flow should not overwrite representative details with dummy data.

---

## 9.3 Add DOCX download endpoint

**New route:** `GET /pacra/authorization-document/download`

Responsibilities:

- authenticate user and org
- fetch org and initiating user data
- load DOCX template from storage
- replace placeholders
- return generated DOCX as a file download

### Error handling

Return a clear failure when:

- template is missing
- template is invalid
- org or user cannot be resolved

---

## 9.4 Add authorization completion endpoint

**New route:** `POST /pacra/authorization/complete`

Request body:

```json
{
  "documentId": "uuid"
}
```

Responsibilities:

- call `completePacraAuthorization()`
- return success payload
- remain safe under retries

---

## 9.5 Register routes

Add both new PACRA routes to the PACRA router and ensure they are mounted correctly in the compliance API.

---

## 10. Frontend Design

## 10.1 Add PACRA authorization step component

**New file:**

`apps/app/features/pacra/components/connect/pacra-authorization-step.tsx`

### Responsibilities

- explain why authorization is required
- allow DOCX download
- accept signed file upload
- show upload success state
- allow completion after upload
- support “I’ll do this later”

### Validation

- accepted mime types: PDF, JPG, JPEG, PNG
- max size: 10 MB

### UX expectations

- disable completion until upload is complete
- show loading states during download, upload, and completion
- show actionable error toasts on failure

---

## 10.2 Update PACRA connect modal

**File:**

`apps/app/features/pacra/components/connect/pacra-connect-modal.tsx`

### Required changes

- add `authorization` as a modal step
- store connection details from connect response
- move to authorization step after PACRA setup succeeds
- close only after:
    - completion succeeds, or
    - user explicitly chooses to continue later

---

## 10.3 Update Regulators Hub

**File:**

`apps/app/features/regulators/components/regulators-hub-page.tsx`

When PACRA is `pending_approval`:

- show pending/linking badge
- show “Continue Setup”
- reopen PACRA flow directly at authorization step

---

## 10.4 Add React Query hooks

**New file:**

`apps/app/lib/queries/pacra/pacra-authorization.ts`

Hooks needed:

- download PACRA authorization form
- complete PACRA authorization
- fetch PACRA authorization status

### Cache invalidation after completion

Invalidate all PACRA-related data that can change after activation, including:

- connections
- regulators
- PACRA profile
- obligations
- filings
- tasks
- onboarding
- authorization

---

## 11. Data Model Impact

## 11.1 No required migrations

The current schema already supports the feature through existing tables and columns.

### Already available

- `regulator_authorizations`
- `regulator_authorizations.documentId`
- `organization_regulator_connections.authorizationDocId`
- `organization_regulator_connections.registrationStatus`
- `documents`

## 11.2 Optional migration

Add `authorization` to `documentKindEnum` for cleaner modeling.

## 11.3 Required enum/event addition

Add a notification or outbox event type for:

```tsx
PACRA_AUTHORIZATION_REVIEW_REQUIRED
```

---

## 12. Acceptance Criteria

The feature is complete when all of the following are true:

1. New PACRA connections are created in `pending_approval`, not `active`
2. Authorization record is created during PACRA connection
3. Template activation does not run before authorization completion
4. Tenant can download a pre-filled DOCX successfully
5. Tenant can upload a signed file successfully
6. Upload completion activates the PACRA connection
7. Upload completion links the document to both authorization and connection
8. Deferred PACRA templates activate after completion
9. Completion flow is idempotent
10. Tenant can leave and resume later
11. Regulators Hub reflects pending vs active state correctly
12. Backoffice receives a review task/event after completion
13. Reject flow can suspend the connection and revoke authorization
14. Tenant isolation is preserved across all document and connection operations

---

## 13. Edge Cases

| Scenario | Expected Behavior |
| --- | --- |
| User closes modal before upload | Connection remains `pending_approval`; user can resume later |
| User downloads form multiple times | Allowed |
| User uploads invalid type | Rejected client-side and server-side |
| User uploads after already active | Return success without re-running activation |
| Multiple admins upload simultaneously | First successful completion activates; later attempts return idempotent success |
| DOCX template missing | Return clear failure and log operational alert |
| Org data changes after DOCX generation | User may re-download updated form |
| Missing optional fields like phone or address | Leave blanks in generated document |
| Backoffice rejects document | Connection suspended, authorization revoked, tenant notified |

---

## 14. Security Requirements

- all reads and writes must be scoped by `organizationId`
- no sensitive document contents should be logged
- document ownership must be verified before completion
- signed uploads must use scoped R2 keys and presigned URLs
- file type and size must be validated
- completion flow must be safe under retries and concurrent requests

---

## 15. Testing Requirements

### Backend

- `connectPacra()` creates pending connection
- authorization record is created on connect
- activation is deferred
- `completePacraAuthorization()` activates correctly
- completion is idempotent
- wrong-org document usage is rejected
- invalid state transitions are rejected

### Frontend

- PACRA modal advances to authorization step
- download action works
- invalid upload types are blocked
- completion button stays disabled until upload completes
- “I’ll do this later” exits cleanly
- “Continue Setup” reopens authorization step

### End-to-end

- connect → download → upload → active
- connect → close → resume → upload → active
- reject in backoffice → suspended + revoked + tenant notified

---

## 16. File Impact Summary

### New files

- `apps/api-compliance/src/routes/pacra/authorization-document.ts`
- `apps/app/features/pacra/components/connect/pacra-authorization-step.tsx`
- `apps/app/lib/queries/pacra/pacra-authorization.ts`
- PACRA DOCX template in storage
- optional route-definition split file if desired

### Modified files

- `packages/api-services/src/domains/pacra/pacra-connect.service.ts`
- `apps/app/features/pacra/components/connect/pacra-connect-modal.tsx`
- `apps/app/features/regulators/components/regulators-hub-page.tsx`
- `apps/api-compliance/src/routes/pacra/index.ts`
- route mounting file if needed
- notification event definitions
- optional enum file for document kind

---

## 17. Implementation Notes

### Recommended rollout

Build this in the following order:

1. backend status and authorization changes in `connectPacra()`
2. completion service
3. download endpoint
4. completion endpoint
5. frontend authorization step
6. modal integration
7. hub resume flow
8. review event wiring
9. tests