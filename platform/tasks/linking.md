---
title: "Task Linking Guide"
description: "This guide explains how tasks link to specific UI sections and how to configure action anchors in templates."
---

This guide explains how tasks link to specific UI sections and how to configure action anchors in templates.

## Action Anchor Fields

Tasks have two fields for linking:

| Field | Type | Description |
|-------|------|-------------|
| `actionKind` | enum | Type of action: `navigate`, `form_section`, `doc_upload`, `confirmation`, `none` |
| `actionRef` | jsonb | Object with linking details |

### actionRef Structure

```typescript
interface ActionRef {
  href?: string;             // Relative path (e.g., "/regulators/zra/filings/{filingId}")
  anchor?: string;           // Page anchor (e.g., "#employer-details")
  formKey?: string;          // Form identifier (e.g., "ZRA_PAYE_FIGURES")
  section?: string;          // Form section key (e.g., "employer_details")
  docRequirementGroup?: string;  // Document requirement key (e.g., "PAYROLL_SUMMARY")
}
```

## Template Configuration

### In Obligation Templates

```typescript
// packages/database/src/schema/compliance/obligation-templates.ts
{
  taskTemplateConfigs: [
    {
      key: "enter_paye_figures",
      title: "Enter PAYE Figures",
      description: "Enter employee earnings and tax deductions",
      taskType: "fill_form",
      required: true,
      sequence: 1,
      actionKind: "form_section",
      actionRefTemplate: {
        href: "/regulators/zra/filings/{filingId}",
        anchor: "paye-figures",
        formKey: "ZRA_PAYE_FIGURES",
        section: "paye_figures",
      },
    },
  ]
}
```

### In Service Templates

```typescript
// packages/database/src/schema/compliance/service-templates.ts
{
  taskTemplateConfigs: [
    {
      key: "upload_supporting_docs",
      title: "Upload Supporting Documents",
      taskType: "upload_document",
      required: true,
      sequence: 2,
      actionKind: "doc_upload",
      actionRefTemplate: {
        docRequirementGroup: "NAME_CLEARANCE_SUPPORTING",
      },
    },
  ]
}
```

## Placeholder Replacement

When tasks are generated, placeholders in `actionRefTemplate.href` are replaced:

| Placeholder | Replaced With |
|-------------|---------------|
| `{filingId}` | Actual filing UUID |
| `{serviceRequestId}` | Actual service request UUID |

### Example

Template:
```json
{
  "href": "/regulators/zra/filings/{filingId}#paye-figures"
}
```

Generated task for filing `abc-123`:
```json
{
  "href": "/regulators/zra/filings/abc-123#paye-figures"
}
```

## Examples by Regulator

### PACRA Name Clearance

```typescript
{
  key: "enter_name_options",
  title: "Enter Name Options",
  taskType: "fill_form",
  required: true,
  sequence: 1,
  actionKind: "form_section",
  actionRefTemplate: {
    formKey: "PACRA_NAME_CLEARANCE",
    section: "names",
  },
}
```

### ZRA PAYE Filing

```typescript
{
  key: "upload_payroll_summary",
  title: "Upload Payroll Summary",
  taskType: "upload_document",
  required: true,
  sequence: 3,
  actionKind: "doc_upload",
  actionRefTemplate: {
    docRequirementGroup: "PAYE_PAYROLL_SUMMARY",
  },
}
```

### PACRA Annual Return

```typescript
{
  key: "confirm_directors",
  title: "Confirm Directors & Shareholders",
  taskType: "review_approve",
  required: true,
  sequence: 2,
  actionKind: "form_section",
  actionRefTemplate: {
    formKey: "PACRA_ANNUAL_RETURN",
    section: "directors_shareholders",
  },
}
```

## UI Integration

### Rendering Action CTAs

```tsx
function TaskActionButton({ task }: { task: Task }) {
  if (!task.actionKind || task.actionKind === "none") {
    return null;
  }

  const getLabel = () => {
    switch (task.actionKind) {
      case "form_section": return "Go to Form";
      case "doc_upload": return "Upload Document";
      case "navigate": return "View";
      case "confirmation": return "Confirm";
      default: return "Open";
    }
  };

  const getHref = () => {
    if (task.actionRef?.href) {
      return task.actionRef.href;
    }
    // Fallback to parent entity
    if (task.filingId) {
      return `/filings/${task.filingId}`;
    }
    if (task.serviceRequestId) {
      return `/service-requests/${task.serviceRequestId}`;
    }
    return null;
  };

  const href = getHref();
  if (!href) return null;

  return (
    <Button asChild size="sm">
      <Link href={href}>{getLabel()}</Link>
    </Button>
  );
}
```

## Automation Matching

### Simplified Trigger-Based Automation (Recommended)

The task system supports simplified auto-completion using indexed `completion_trigger` columns:

```typescript
// When a form section is saved
await completeTasksOnFormSave(ctx, deps, {
  parentType: "service_request",
  parentId: serviceRequestId,
  formKey: "PACRA_BUSINESS_NAME",
  section: "business_details",
});
// Completes tasks with trigger "form:PACRA_BUSINESS_NAME:business_details"

// When a document is uploaded
await completeTasksOnDocUpload(ctx, deps, {
  parentType: "service_request",
  parentId: serviceRequestId,
  requirementKey: "owner_nrc_copies",
});
// Completes tasks with trigger "doc:owner_nrc_copies"
```

### Legacy Action-Based Automation

The original `completeTasksForAction` still works but uses JSONB matching:

```typescript
await completeTasksForAction(ctx, deps, {
  parentType: "filing",
  parentId: filingId,
  actionKind: "form_section",
  match: {
    section: "employer_details",
    formKey: "ZRA_PAYE_FIGURES",
  },
});
```

### Document Upload Automation

Automatically triggered in `createDocument` when `requirementKey` is provided:

```typescript
await completeTasksForAction(ctx, deps, {
  parentType: "filing",
  parentId: filingId,
  actionKind: "doc_upload",
  match: {
    docRequirementGroup: requirementKey,
  },
});
```

## Complete Template Examples

### PACRA Business Name Registration

```typescript
// packages/database/src/seeds/pacra-templates.ts
{
  taskTemplateConfigs: [
    {
      key: "provide_business_details",
      title: "Provide Business Details",
      taskType: "fill_form",
      required: true,
      isBlocking: true,
      sequence: 1,
      actionKind: "form_section",
      actionRefTemplate: {
        href: "/regulators/pacra/service-requests/{serviceRequestId}",
        anchor: "business-details",
        formKey: "PACRA_BUSINESS_NAME",
        section: "business_details",
      },
    },
    {
      key: "provide_owner_details",
      title: "Provide Owner Details",
      taskType: "fill_form",
      required: true,
      isBlocking: true,
      sequence: 2,
      actionKind: "form_section",
      actionRefTemplate: {
        href: "/regulators/pacra/service-requests/{serviceRequestId}",
        anchor: "owners",
        formKey: "PACRA_BUSINESS_NAME",
        section: "owners",
      },
    },
    {
      key: "upload_required_documents",
      title: "Upload Required Documents",
      taskType: "upload_document",
      required: true,
      isBlocking: true,
      sequence: 3,
      actionKind: "doc_upload",
      actionRefTemplate: {
        href: "/regulators/pacra/service-requests/{serviceRequestId}",
        anchor: "documents",
        docRequirementGroup: "owner_nrc_copies",
      },
    },
    {
      key: "review_confirm_submission",
      title: "Review and Confirm Submission",
      taskType: "review_approve",
      required: true,
      isBlocking: true,
      sequence: 4,
      actionKind: "confirmation",
      actionRefTemplate: {
        href: "/regulators/pacra/service-requests/{serviceRequestId}",
        anchor: "submit",
      },
    },
  ],
}
```

### ZRA PAYE Registration

```typescript
// packages/database/src/seeds/zra-service-templates.ts
{
  taskTemplateConfigs: [
    {
      key: "provide_employer_details",
      title: "Provide Employer Details",
      taskType: "fill_form",
      required: true,
      isBlocking: true,
      sequence: 1,
      actionKind: "form_section",
      actionRefTemplate: {
        href: "/regulators/zra/service-requests/{serviceRequestId}",
        anchor: "employer-details",
        formKey: "ZRA_PAYE_REGISTRATION",
        section: "employer_details",
      },
    },
    {
      key: "upload_supporting_docs",
      title: "Upload Supporting Documents",
      taskType: "upload_document",
      required: true,
      isBlocking: true,
      sequence: 2,
      actionKind: "doc_upload",
      actionRefTemplate: {
        href: "/regulators/zra/service-requests/{serviceRequestId}",
        anchor: "documents",
        docRequirementGroup: "tpin_certificate",
      },
    },
    {
      key: "confirm_submission",
      title: "Review & Confirm",
      taskType: "review_approve",
      required: true,
      isBlocking: true,
      sequence: 3,
      actionKind: "confirmation",
      actionRefTemplate: {
        href: "/regulators/zra/service-requests/{serviceRequestId}",
        anchor: "submit",
      },
    },
  ],
}
```

## Generated Task Example

When a service request is created from the PACRA Business Name template:

```json
{
  "id": "task-uuid-123",
  "organizationId": "org-uuid-456",
  "serviceRequestId": "sr-uuid-789",
  "templateKey": "provide_business_details",
  "title": "Provide Business Details",
  "taskType": "fill_form",
  "status": "todo",
  "required": true,
  "isBlocking": true,
  "sequence": 1,
  "actionKind": "form_section",
  "actionRef": {
    "href": "/regulators/pacra/service-requests/sr-uuid-789",
    "anchor": "business-details",
    "formKey": "PACRA_BUSINESS_NAME",
    "section": "business_details"
  },
  "regulatorKey": "PACRA",
  "completionTrigger": "form:PACRA_BUSINESS_NAME:business_details"
}
```
