---
title: "Task System Overview"
description: "The task system: checklist items tied to Filings and Service Requests that gate submission to regulators."
---

The Bumara Task System provides a structured checklist framework for tracking work required on **Filings** and **Service Requests**. Tasks ensure that all required steps are completed before submission to regulators.

## Key Concepts

### What is a Task?

A **Task** is a checklist item tied to a single parent entity:
- **Filing** — A period instance from an obligation (e.g., "PAYE Return — Nov 2025")
- **Service Request** — A one-off workflow (e.g., "Name Clearance #12345")

Tasks are **never free-floating**. Every task must belong to exactly one parent entity.

### Task Types

| Type | Description | Example |
|------|-------------|---------|
| `upload_document` | Upload a required document | Payroll summary PDF |
| `fill_form` | Complete form fields | Employer details |
| `review_approve` | Acknowledge/confirm information | Confirm directors |
| `payment_action` | Payment required | Filing fee |
| `info_request` | Gather information | Employee count |
| `custom` | General task | Any other step |

### Task Lifecycle

```
TODO → DOING → DONE
  ↓      ↓      ↓
BLOCKED  ↓   (reopen)
  ↓      ↓      ↓
  └──→ SKIPPED ←┘
```

| Status | Description |
|--------|-------------|
| `todo` | Not started |
| `doing` | In progress |
| `blocked` | Cannot proceed (reason required) |
| `done` | Completed |
| `skipped` | Skipped (optional only, reason required) |

## How Tasks Are Used

### Template-Generated Tasks

When a filing or service request is created, tasks are auto-generated from **template configs**:

```typescript
// In obligation_templates or service_templates
{
  taskTemplateConfigs: [
    {
      key: "confirm_company_details",
      title: "Confirm Company Details",
      taskType: "review_approve",
      required: true,
      sequence: 1,
      actionKind: "form_section",
      actionRefTemplate: {
        formKey: "PACRA_ANNUAL_RETURN",
        section: "company_details",
      },
    },
    // ... more tasks
  ]
}
```

### Action Anchors

Tasks can deep-link to specific UI sections via **action anchors**:

| ActionKind | Purpose | Example |
|------------|---------|---------|
| `navigate` | Link to page | Filing details page |
| `form_section` | Link to form section | Employer details section |
| `doc_upload` | Link to upload area | Payroll docs upload |
| `confirmation` | Ack step | Terms acceptance |
| `none` | No link | Generic task |

### Readiness Gates

A parent entity becomes **ready for submission** when:

1. ✅ All **required tasks** are `done` or `skipped`
2. ✅ No tasks are `blocked`
3. ✅ All **required documents** are uploaded

## Automation

### Document Upload Automation

When a document with `requirementKey` is uploaded, any matching `doc_upload` tasks are auto-completed:

```typescript
// Triggered in completeDocumentUpload
completeTasksForAction(ctx, deps, {
  parentType: "filing",
  parentId: filingId,
  actionKind: "doc_upload",
  match: { docRequirementGroup: requirementKey },
});
```

### Auto-Transition

When a task is marked `done`:
1. System checks if parent is ready for submission
2. If ready, auto-transitions to `ready_for_submission`
3. Audit log recorded with `trigger: "auto_task_completion"`

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/tasks` | GET | List tasks (paginated, filtered) |
| `/tasks/{id}` | GET | Get single task |
| `/tasks` | POST | Create task |
| `/tasks/{id}/status` | PATCH | Update status |
| `/tasks/{id}/skip` | POST | Skip task |
| `/tasks/{id}/reopen` | POST | Reopen task |
| `/filings/{id}/tasks` | GET | Get filing tasks |
| `/filings/{id}/readiness` | GET | Check readiness |

## Related Documentation

- [Task System Specification](/platform/tasks/spec)
- [Task Linking Guide](/platform/tasks/linking)
- [Progress & Readiness](/platform/tasks/progress-and-readiness)
