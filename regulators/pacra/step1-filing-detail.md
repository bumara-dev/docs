---
title: "PACRA Filing Detail - Step 1 Implementation"
description: "Step 1 Complete: Tenant can prepare an Annual Return filing to a \"Ready (prepared)\" state."
---

## Overview

This document describes the implementation of the PACRA Filing Detail page where tenants can:
- See all tasks generated for a filing
- Complete confirmation and form tasks inline
- See required document requirements
- Upload files to satisfy document requirements
- See real-time progress updates (Tasks X/Y, Documents A/B)
- See a "Blockers" panel listing exactly what is missing

**Not included in Step 1:** Request Submission, Payment, Backoffice integration.

---

## Endpoints

### GET `/filings/{id}/view`

Returns comprehensive filing view including tasks, documents, requirements, and blockers.

**Response:**
```json
{
  "success": true,
  "data": {
    "filing": {
      "id": "uuid",
      "status": "pending_data",
      "periodLabel": "FY 2025",
      "dueOn": "2025-03-31T00:00:00Z",
      "obligation": { "id": "uuid", "name": "Annual Return" }
    },
    "tasks": [
      {
        "id": "uuid",
        "title": "Review company details",
        "description": "Verify registered address and directors",
        "taskType": "review_approve",
        "required": true,
        "status": "todo",
        "payload": null,
        "sequence": 1
      }
    ],
    "docRequirements": [
      {
        "key": "board_resolution",
        "name": "Board Resolution",
        "description": "Resolution authorizing filing",
        "kind": "source",
        "required": true,
        "satisfied": false,
        "satisfiedByDocId": null,
        "satisfiedByDocFilename": null
      }
    ],
    "documents": [
      {
        "id": "uuid",
        "filename": "resolution.pdf",
        "kind": "source",
        "requirementKey": "board_resolution",
        "uploadedAt": "2025-01-08T10:00:00Z"
      }
    ],
    "progress": {
      "tasksDone": 2,
      "tasksTotal": 5,
      "docsDone": 1,
      "docsTotal": 3
    },
    "blockers": {
      "isReady": false,
      "blockedTasks": [],
      "pendingRequiredTasks": [
        { "id": "uuid", "title": "Review company details", "status": "todo" }
      ],
      "missingRequiredDocs": [
        { "key": "financial_statements", "name": "Financial Statements" }
      ]
    }
  }
}
```

### PATCH `/tasks/{id}/status`

Updates task status with optional payload (for form-type tasks).

**Request:**
```json
{
  "status": "done",
  "payload": { "notes": "Verified all details are correct" }
}
```

### PATCH `/tasks/{id}/payload`

Updates task payload only without changing status (save progress).

**Request:**
```json
{
  "payload": { "notes": "In progress..." }
}
```

### POST `/documents/uploads/complete`

Completes a document upload after file is uploaded to Vercel Blob.

**Request:**
```json
{
  "storageKey": "documents/filing-uuid/1234567890-file.pdf",
  "blobUrl": "https://blob.vercel-storage.com/...",
  "filingId": "uuid",
  "requirementKey": "board_resolution",
  "kind": "source",
  "filename": "resolution.pdf",
  "mimeType": "application/pdf",
  "sizeBytes": 102400
}
```

---

## Task Types Supported

| Type | UI | Completion |
|------|-----|------------|
| `review_approve` | Checkbox + description | Toggle checkbox marks done |
| `info_request` | Checkbox + description | Toggle checkbox marks done |
| `fill_form` | Expandable form with textarea | Save progress or Save & Complete |
| `custom` | Expandable form with textarea | Save progress or Save & Complete |
| `upload_document` | Link to Documents section | Auto-completes when doc uploaded |
| `payment_action` | Display only | Handled in Payments tab |

---

## Document Requirement Rules

1. **Requirements from Template**: Defined in `obligationTemplates.docRequirementConfigs`
2. **Satisfaction Tracking**: Document with matching `requirementKey` satisfies requirement
3. **Required vs Optional**: Required docs show "Required" badge and must be uploaded for readiness
4. **Replacement**: Can upload new file to replace existing satisfaction
5. **Unlinked Documents**: Documents without `requirementKey` shown in "Other Uploaded Documents"

---

## Components

### `FilingBlockersPanel`
- Shows "Ready for Submission" when `blockers.isReady === true`
- Otherwise lists pending tasks, blocked tasks, and missing documents
- Updates live as tasks complete and documents upload

### `FilingTasksSection`
- Renders all tasks with type-specific UI
- Confirmation tasks: inline checkbox
- Form tasks: expandable with save/complete buttons
- Status indicators: todo, doing, blocked, done, skipped

### `FilingDocumentsSection`
- Shows required documents first, then optional
- Each requirement shows satisfied status
- Upload dialog with drag-drop and progress
- Uploads to Vercel Blob then calls `completeDocumentUpload`

---

## Files Changed/Created

### Database
- `packages/database/src/schema/compliance/tasks.ts` - Added `payload` column
- `packages/database/src/schema/compliance/documents.ts` - Added `requirementKey` column
- `packages/database/drizzle/0024_add_task_payload_and_doc_requirement.sql` - Migration

### Backend
- `packages/api-services/src/domains/tasks/tasks.schema.ts` - Added payload schemas
- `packages/api-services/src/domains/tasks/tasks.service.ts` - Added `updateTaskPayload`
- `packages/api-services/src/domains/compliance/filings.schema.ts` - Added `filingViewResponseSchema`
- `packages/api-services/src/domains/compliance/filings.service.ts` - Added `getFilingView`
- `packages/api-services/src/domains/compliance/documents.schema.ts` - Added upload schemas
- `packages/api-services/src/domains/compliance/documents.service.ts` - Added `completeDocumentUpload`
- `packages/backend/src/modules/tasks/tasks.routes.ts` - Added payload route
- `packages/backend/src/modules/tasks/tasks.handlers.ts` - Added payload handler
- `packages/backend/src/modules/compliance/filings/routes.ts` - Added view route
- `packages/backend/src/modules/compliance/filings/handlers.ts` - Added view handler
- `packages/backend/src/modules/compliance/documents/routes.ts` - Added upload complete route
- `packages/backend/src/modules/compliance/documents/handlers.ts` - Added upload complete handler

### Frontend
- `apps/app/lib/queries/regulators/types.ts` - Added filing view types
- `apps/app/lib/queries/regulators/fetchers/filings.ts` - Added `fetchFilingView`
- `apps/app/lib/queries/regulators/fetchers/documents.ts` - Added `completeDocumentUpload`
- `apps/app/lib/queries/regulators/hooks/use-filings.ts` - Added `useFilingView`
- `apps/app/lib/queries/regulators/hooks/use-documents.ts` - Added `useCompleteDocumentUpload`
- `apps/app/lib/queries/tasks/types.ts` - Added payload types
- `apps/app/lib/queries/tasks/fetchers/tasks.ts` - Added `updateTaskPayload`
- `apps/app/lib/queries/tasks/hooks/use-tasks.ts` - Added `useUpdateTaskPayload`
- `apps/app/features/filings/components/filing-blockers-panel.tsx` - **NEW**
- `apps/app/features/filings/components/filing-tasks-section.tsx` - **NEW**
- `apps/app/features/filings/components/filing-documents-section.tsx` - **NEW**
- `apps/app/features/filings/components/index.ts` - **NEW**
- `apps/app/app/(authenticated)/(general)/regulators/pacra/filings/[filingId]/page.tsx` - Refactored

---

## Manual QA Steps

1. **Navigate to PACRA Filing Detail**
   - Go to `/regulators/pacra/filings`
   - Click on any filing to open detail page

2. **Verify Header**
   - Check filing title, status badge, due date are shown
   - Check progress chips show "Tasks X/Y" and "Docs A/B"

3. **Verify Blockers Panel**
   - For filings in `pending_data` or `in_progress`:
     - Should show amber "What to do next" panel
     - Lists pending tasks and missing docs
   - For filings in `ready_for_submission`:
     - Should show green "Ready for Submission" panel

4. **Test Task Completion**
   - **Confirmation Task**: Check the checkbox, verify status changes
   - **Form Task**: Expand, enter notes, click "Save Progress", verify saved
   - **Form Task Complete**: Click "Save & Complete", verify marked done

5. **Test Document Upload**
   - Click "Upload" on a required document requirement
   - Select a file (PDF, image, etc.)
   - Verify progress bar during upload
   - Verify document appears as "satisfied" after upload
   - Check progress counts update

6. **Verify Progress Updates**
   - After completing a task, verify "Tasks X/Y" chip updates
   - After uploading a document, verify "Docs A/B" chip updates
   - Verify blockers panel updates (items removed from list)

7. **Verify Edge Cases**
   - Filing with no tasks: Should show "No tasks for this filing"
   - Filing with no doc requirements: Should show empty state with upload option
   - Blocked task: Should show blocked status with reason

---

## Known Gaps for Step 2

1. **Request Submission**: Button shown but not functional
2. **Payment Integration**: Payment tab shows placeholder
3. **Backoffice Handoff**: No submission job creation
4. **Authorization Gate**: Not enforced (display only if data exists)
5. **Document Delete**: Can view but not delete uploaded documents
6. **Task Reopen**: Reopening done tasks available for admins only
