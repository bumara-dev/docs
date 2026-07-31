---
title: "Service Request System Architecture"
description: "This document describes the architecture for handling service request creation in the backoffice catalog, rules engine, and how to easily add new..."
---

## Overview

This document describes the architecture for handling service request creation in the backoffice catalog, rules engine, and how to easily add new regulator service requests following industry best practices.

**Last Updated**: 2026-02-10
**Status**: Architectural Recommendation

---

## Core Design Principles

1. **Template-Driven**: All service requests derive from templates (catalog items)
2. **Regulator-Agnostic Core**: Core system works for any regulator
3. **Extensible via Plugins**: Regulator-specific logic isolated in modules
4. **Rules-Based Activation**: Eligibility determined by declarative rules
5. **Multi-Tenant Safe**: Strict org isolation at all layers
6. **Audit Trail**: All changes logged for compliance

---

## System Layers

### **Layer 1: Catalog (Template Management)**

Templates define **what** can be requested. Managed in backoffice.

**Key Entities**:
- `service_templates` - Template definitions
- `regulators` - Regulator registry
- `regulator_fees` - Pricing rules
- `activation_rules_engine` (new) - Conditional logic

**Example Flow**:
```
Backoffice Admin → Creates "PACRA Change of Directors" Template
  ├── Basic Info (name, description, regulator)
  ├── Intake Fields Schema (JSON: director details, effective date)
  ├── Task Templates (JSON: upload board resolution, fill form CR6)
  ├── Document Requirements (JSON: board resolution, CR6 form, IDs)
  ├── Payment Rules (JSON: feeKey, serviceFee calculation)
  └── Activation Rules (JSON: entityType=company, status=active)
```

---

### **Layer 2: Rules Engine**

Determines **when** a template is available to a tenant.

**Activation Rules Schema** (existing, in `service_templates.activationRules` JSONB):

```typescript
interface ActivationRules {
  // Entity filters
  entityType?: Array<"company" | "business_name" | "partnership">;
  companyType?: Array<"private" | "public">;

  // Connection requirements
  requiresConnection?: boolean; // Must be connected to regulator

  // Plan-based access
  minimumPlanTier?: "start" | "plus" | "pro" | "enterprise";

  // Custom conditions (for complex rules)
  customConditions?: {
    field: string; // e.g., "pacraProfile.registrationStatus"
    operator: "eq" | "neq" | "in" | "nin" | "gt" | "lt";
    value: unknown;
  }[];

  // Feature flags
  requiresFeatureFlag?: string; // e.g., "pacra_annual_returns_enabled"
}
```

**Evaluation Logic** (in `@repo/api-services/domains/compliance/activation.service.ts`):

```typescript
export async function getAvailableServiceTemplates(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  regulatorId: string
) {
  const orgId = requireOrganizationContext(ctx);

  // 1. Get org details (plan, entity type, connections)
  const org = await deps.db.query.organizations.findFirst({
    where: eq(organizations.id, orgId),
    with: {
      subscription: true,
      regulatorConnections: {
        where: eq(regulatorConnections.regulatorId, regulatorId),
      },
    },
  });

  // 2. Get all templates for this regulator
  const templates = await deps.db.query.serviceTemplates.findMany({
    where: and(
      eq(serviceTemplates.regulatorId, regulatorId),
      eq(serviceTemplates.status, "active"),
      isNull(serviceTemplates.organizationId) // Global templates only
    ),
  });

  // 3. Filter by activation rules
  return templates.filter(template =>
    evaluateActivationRules(template.activationRules, {
      organization: org,
      connection: org.regulatorConnections[0],
    })
  );
}

function evaluateActivationRules(
  rules: ActivationRules,
  context: { organization: Org; connection?: Connection }
): boolean {
  // Entity type check
  if (rules.entityType?.length) {
    if (!rules.entityType.includes(context.organization.entityType)) {
      return false;
    }
  }

  // Connection requirement
  if (rules.requiresConnection && !context.connection) {
    return false;
  }

  // Plan tier check
  if (rules.minimumPlanTier) {
    const tierHierarchy = { start: 0, plus: 1, pro: 2, enterprise: 3 };
    const orgTier = tierHierarchy[context.organization.subscription.planTier];
    const requiredTier = tierHierarchy[rules.minimumPlanTier];
    if (orgTier < requiredTier) return false;
  }

  // Custom conditions (extensibility point)
  if (rules.customConditions?.length) {
    for (const condition of rules.customConditions) {
      if (!evaluateCondition(condition, context)) return false;
    }
  }

  return true;
}
```

---

### **Layer 3: Service Request Lifecycle**

Manages **execution** of service requests.

**State Machine** (already defined in `service_request_status` enum):

```
pending_data → in_progress → ready_for_submission → awaiting_payment
                                                  ↓
                             submission_in_progress → submitted → accepted
                                                                ↓
                                              needs_correction → [loop back]
                                                                ↓
                                                             waived | cancelled
```

**Transitions** (enforced in `service-requests.service.ts`):

```typescript
const VALID_TRANSITIONS = {
  pending_data: ["in_progress", "cancelled"],
  in_progress: ["ready_for_submission", "awaiting_payment", "cancelled"],
  ready_for_submission: ["submission_in_progress", "cancelled"],
  awaiting_payment: ["ready_for_submission", "cancelled"],
  submission_in_progress: ["submitted", "needs_correction", "cancelled"],
  submitted: ["accepted", "needs_correction"],
  needs_correction: ["in_progress"],
  accepted: [],
  waived: [],
  cancelled: [],
};

export async function transitionServiceRequestStatus(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  requestId: string,
  newStatus: ServiceRequestStatus,
  reason?: string
) {
  const request = await getServiceRequest(ctx, deps, requestId);

  // Validate transition
  if (!VALID_TRANSITIONS[request.status].includes(newStatus)) {
    throw new ServiceError(
      "INVALID_TRANSITION",
      `Cannot transition from ${request.status} to ${newStatus}`
    );
  }

  // Enforce prerequisites
  if (newStatus === "ready_for_submission") {
    await enforceReadyForSubmissionRequirements(ctx, deps, requestId);
  }

  // Update status
  await deps.db.update(serviceRequests)
    .set({ status: newStatus, updatedAt: deps.now() })
    .where(eq(serviceRequests.id, requestId));

  // Audit log
  await recordAuditLog(ctx, deps, {
    action: "update",
    entityType: "service_request",
    entityId: requestId,
    changes: { before: { status: request.status }, after: { status: newStatus } },
    metadata: { reason },
  });
}
```

---

### **Layer 4: Regulator-Specific Handlers** (Plugin Pattern)

For regulators with unique workflows beyond standard templates.

**Directory Structure**:

```
packages/
  backend/src/modules/
    compliance/                    # Core service requests
      service-requests/
        routes.ts
        handlers.ts

    regulators/                    # Regulator-specific logic
      pacra/
        service-handlers/          # PACRA-specific extensions
          name-clearance.handler.ts
          company-registration.handler.ts
          change-of-directors.handler.ts

      zra/
        service-handlers/
          tpin-application.handler.ts
          vat-registration.handler.ts

      napsa/
        service-handlers/
          employer-registration.handler.ts
```

**Handler Interface** (contracts):

```typescript
// packages/api-services/src/domains/regulators/handler-interface.ts

export interface ServiceRequestHandler {
  /**
   * Unique handler key (matches template key)
   */
  readonly handlerKey: string;

  /**
   * Regulator this handler belongs to
   */
  readonly regulatorCode: string;

  /**
   * Custom validation beyond standard template validation
   */
  validateIntakeData?(
    ctx: ServiceContext,
    deps: ServiceDependencies,
    data: Record<string, unknown>
  ): Promise<ValidationResult>;

  /**
   * Transform intake data before saving (e.g., normalize phone numbers)
   */
  transformIntakeData?(
    data: Record<string, unknown>
  ): Record<string, unknown>;

  /**
   * Generate dynamic tasks based on intake data
   * (e.g., for PACRA company registration, generate director tasks dynamically)
   */
  generateDynamicTasks?(
    ctx: ServiceContext,
    deps: ServiceDependencies,
    request: ServiceRequest,
    intakeData: Record<string, unknown>
  ): Promise<TaskTemplateConfig[]>;

  /**
   * Calculate dynamic fees (complex pricing logic)
   */
  calculateFees?(
    ctx: ServiceContext,
    deps: ServiceDependencies,
    intakeData: Record<string, unknown>
  ): Promise<{ serviceFee: number; regulatorFee: number; total: number }>;

  /**
   * Pre-submission hook (e.g., call regulator API for name availability)
   */
  beforeSubmission?(
    ctx: ServiceContext,
    deps: ServiceDependencies,
    requestId: string
  ): Promise<void>;

  /**
   * Post-submission hook (e.g., poll regulator API for status)
   */
  afterSubmission?(
    ctx: ServiceContext,
    deps: ServiceDependencies,
    requestId: string,
    submissionResult: unknown
  ): Promise<void>;
}
```

**Example Handler Implementation**:

```typescript
// packages/backend/src/modules/regulators/pacra/service-handlers/name-clearance.handler.ts

export class PacraNameClearanceHandler implements ServiceRequestHandler {
  readonly handlerKey = "PACRA_NAME_CLEARANCE_V1";
  readonly regulatorCode = "pacra";

  async validateIntakeData(
    ctx: ServiceContext,
    deps: ServiceDependencies,
    data: Record<string, unknown>
  ): Promise<ValidationResult> {
    // Custom validation: ensure at least one proposed name
    if (!data.proposedName1) {
      return {
        valid: false,
        errors: [{ field: "proposedName1", message: "At least one name is required" }],
      };
    }

    // Check against PACRA restricted words API (if available)
    const hasRestrictedWords = await checkRestrictedWords(data.proposedName1);
    if (hasRestrictedWords) {
      return {
        valid: false,
        errors: [{
          field: "proposedName1",
          message: "Name contains restricted words. Please choose another."
        }],
      };
    }

    return { valid: true };
  }

  async beforeSubmission(
    ctx: ServiceContext,
    deps: ServiceDependencies,
    requestId: string
  ): Promise<void> {
    // Call PACRA API to check real-time name availability
    const request = await getServiceRequest(ctx, deps, requestId);
    const intakeData = request.payload as { proposedName1: string };

    const availability = await pacraApi.checkNameAvailability(intakeData.proposedName1);

    if (!availability.available) {
      throw new ServiceError(
        "NAME_UNAVAILABLE",
        `The name "${intakeData.proposedName1}" is not available. ${availability.reason}`
      );
    }
  }
}
```

**Handler Registry** (auto-discovery):

```typescript
// packages/api-services/src/domains/regulators/handler-registry.ts

const HANDLER_REGISTRY = new Map<string, ServiceRequestHandler>();

export function registerHandler(handler: ServiceRequestHandler) {
  HANDLER_REGISTRY.set(handler.handlerKey, handler);
}

export function getHandler(templateKey: string): ServiceRequestHandler | null {
  return HANDLER_REGISTRY.get(templateKey) ?? null;
}

// Auto-register all handlers on startup
import { PacraNameClearanceHandler } from "../modules/regulators/pacra/service-handlers/name-clearance.handler";
import { PacraCompanyRegistrationHandler } from "../modules/regulators/pacra/service-handlers/company-registration.handler";

registerHandler(new PacraNameClearanceHandler());
registerHandler(new PacraCompanyRegistrationHandler());
// ... register all handlers
```

**Integration in Service Creation**:

```typescript
// Modified createServiceRequest in service-requests.service.ts

export async function createServiceRequest(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  input: CreateServiceRequestInput
) {
  const template = await getTemplate(input.templateId);

  // Check if custom handler exists
  const handler = getHandler(template.templateKey);

  // 1. Validate intake data
  if (handler?.validateIntakeData) {
    const validation = await handler.validateIntakeData(ctx, deps, input.intakeData);
    if (!validation.valid) {
      throw new ServiceError("VALIDATION_ERROR", validation.errors);
    }
  }

  // 2. Transform data if needed
  let intakeData = input.intakeData;
  if (handler?.transformIntakeData) {
    intakeData = handler.transformIntakeData(intakeData);
  }

  // 3. Create service request
  const request = await createRequestRecord(ctx, deps, template, intakeData);

  // 4. Generate tasks (standard + dynamic)
  let taskConfigs = template.taskTemplateConfigs;
  if (handler?.generateDynamicTasks) {
    const dynamicTasks = await handler.generateDynamicTasks(ctx, deps, request, intakeData);
    taskConfigs = [...taskConfigs, ...dynamicTasks];
  }

  await createTasksFromConfigs(ctx, deps, request.id, taskConfigs);

  // 5. Calculate fees if dynamic
  if (handler?.calculateFees) {
    const fees = await handler.calculateFees(ctx, deps, intakeData);
    await createPaymentRequest(ctx, deps, request.id, fees);
  }

  return request;
}
```

---

## Adding a New Regulator Service Request

### **Step 1: Define Service Template** (Backoffice or Seed)

**Option A: Via Backoffice UI** (for non-technical staff):
1. Navigate to `/backoffice/catalog/service-templates/new`
2. Fill in form:
   - **Basic Info**: Name, Regulator, Description
   - **Plain Language**: What, Why, Consequences
   - **Intake Fields**: Define form schema (JSON)
   - **Tasks**: Define workflow steps
   - **Documents**: Define required uploads
   - **Payment Rules**: Link to regulator fee or specify amount

**Option B: Via Seed File** (for developers, version-controlled):

```typescript
// packages/database/src/seeds/{regulator}-templates.ts

export const ZRA_VAT_REGISTRATION_V1: ServiceTemplateSeed = {
  templateKey: "ZRA_VAT_REGISTRATION_V1",
  templateVersion: 1,
  name: "VAT Registration",
  description: "Register for Value Added Tax with ZRA",
  regulator: "zra",

  whatIsThis: "VAT registration allows you to charge VAT on sales...",
  whyItMatters: "Required for businesses with turnover > ZMW 800,000...",
  consequencesOfDelay: "Penalties and backdated liability...",

  expectedDueInDays: "14",
  defaultPriority: "high",

  intakeFieldsSchema: [
    {
      key: "tpin",
      label: "TPIN",
      type: "text",
      required: true,
      placeholder: "1000000000",
    },
    {
      key: "estimatedAnnualTurnover",
      label: "Estimated Annual Turnover (ZMW)",
      type: "number",
      required: true,
    },
  ],

  activationRules: {
    requiresConnection: true,
    customConditions: [
      {
        field: "zraConnection.tpin",
        operator: "neq",
        value: null,
      },
    ],
  },

  taskTemplateConfigs: [
    {
      key: "upload_bank_statement",
      title: "Upload Bank Statement",
      description: "Upload latest 3 months bank statement",
      taskType: "upload_document",
      required: true,
      sequence: 1,
      actionKind: "doc_upload",
      actionRefTemplate: {
        docRequirementGroup: "bank_statement",
      },
    },
    {
      key: "fill_vat_form",
      title: "Complete VAT Registration Form",
      description: "Fill in all required VAT registration details",
      taskType: "fill_form",
      required: true,
      sequence: 2,
      actionKind: "form_section",
      actionRefTemplate: {
        formKey: "zra_vat_registration",
        section: "business_details",
      },
    },
  ],

  docRequirementConfigs: [
    {
      key: "bank_statement",
      name: "Bank Statement",
      description: "Last 3 months bank statement",
      kind: "source",
      required: true,
    },
    {
      key: "certificate_of_incorporation",
      name: "Certificate of Incorporation",
      kind: "source",
      required: true,
      conditions: {
        entityType: ["company"],
      },
    },
  ],

  paymentRuleConfig: {
    paymentRequired: false, // ZRA VAT registration is free
  },

  billingTag: "included",
};
```

Run seed:
```bash
pnpm --filter @repo/database seed:zra-templates
```

---

### **Step 2: (Optional) Create Custom Handler**

If the service requires custom logic beyond templates:

```typescript
// packages/backend/src/modules/regulators/zra/service-handlers/vat-registration.handler.ts

export class ZraVatRegistrationHandler implements ServiceRequestHandler {
  readonly handlerKey = "ZRA_VAT_REGISTRATION_V1";
  readonly regulatorCode = "zra";

  async validateIntakeData(
    ctx: ServiceContext,
    deps: ServiceDependencies,
    data: Record<string, unknown>
  ): Promise<ValidationResult> {
    // Validate TPIN format
    const tpin = data.tpin as string;
    if (!/^\d{10}$/.test(tpin)) {
      return {
        valid: false,
        errors: [{ field: "tpin", message: "TPIN must be 10 digits" }],
      };
    }

    // Check if already VAT registered via ZRA API
    const vatStatus = await zraApi.checkVatStatus(tpin);
    if (vatStatus.registered) {
      return {
        valid: false,
        errors: [{ field: "tpin", message: "This TPIN is already VAT registered" }],
      };
    }

    return { valid: true };
  }

  async calculateFees(
    ctx: ServiceContext,
    deps: ServiceDependencies,
    intakeData: Record<string, unknown>
  ): Promise<FeeCalculation> {
    // VAT registration is free from ZRA
    // But Bumara charges a service fee
    return {
      serviceFee: 50000, // ZMW 500
      regulatorFee: 0,
      total: 50000,
    };
  }
}

// Register handler
registerHandler(new ZraVatRegistrationHandler());
```

---

### **Step 3: Add Routes (if needed)**

Most service requests use the generic `/service-requests` routes. But for regulator-specific UIs:

```typescript
// packages/backend/src/modules/regulators/zra/routes.ts

import { createRoute } from "@hono/zod-openapi";

export const initiateVatRegistrationRoute = createRoute({
  method: "post",
  path: "/regulators/zra/vat-registration/initiate",
  middleware: [requireAuth, requireZraConnection],
  request: {
    body: {
      content: {
        "application/json": {
          schema: vatRegistrationIntakeSchema,
        },
      },
    },
  },
  responses: {
    200: {
      description: "VAT registration service request created",
      content: {
        "application/json": {
          schema: serviceRequestResponseSchema,
        },
      },
    },
  },
});
```

---

### **Step 4: Frontend Integration**

**Catalog Display** (auto-generated from templates):

```tsx
// apps/app/features/regulators/components/service-catalog.tsx

export function ServiceCatalog({ regulatorId }: { regulatorId: string }) {
  const { data: templates } = useAvailableServiceTemplates(regulatorId);

  return (
    <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
      {templates?.map(template => (
        <ServiceTemplateCard
          key={template.id}
          template={template}
          onSelect={() => initiateServiceRequest(template.id)}
        />
      ))}
    </div>
  );
}
```

**Intake Form** (dynamic, driven by `intakeFieldsSchema`):

```tsx
// apps/app/features/compliance/components/dynamic-intake-form.tsx

export function DynamicIntakeForm({
  templateId,
  onSubmit
}: DynamicIntakeFormProps) {
  const { data: template } = useServiceTemplate(templateId);

  // Build dynamic Zod schema from intakeFieldsSchema
  const schema = useMemo(() =>
    buildSchemaFromFields(template.intakeFieldsSchema),
    [template]
  );

  const form = useForm({ resolver: zodResolver(schema) });

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)}>
        {template.intakeFieldsSchema.map(field => (
          <DynamicField key={field.key} field={field} form={form} />
        ))}
        <Button type="submit">Create Request</Button>
      </form>
    </Form>
  );
}
```

---

## Best Practices Summary

### ✅ **DO**

1. **Use templates for 80% of cases** - Don't over-engineer. Most service requests fit the template model.
2. **Isolate regulator logic** - Keep regulator-specific code in `modules/regulators/{code}/`.
3. **Version templates** - Use `templateVersion` field for breaking changes.
4. **Validate at boundaries** - Handler validation + Zod schemas.
5. **Audit everything** - Use `recordAuditLog` for all state changes.
6. **Enforce multi-tenancy** - Every query must filter by `organizationId`.
7. **Test activation rules** - Write tests for complex eligibility logic.

### ❌ **DON'T**

1. **Don't hardcode regulator IDs** - Always look up by `code` field.
2. **Don't skip status validation** - Use `VALID_TRANSITIONS` map.
3. **Don't allow arbitrary status jumps** - Enforce prerequisites (tasks done, payment verified).
4. **Don't log PII** - No intake data in logs.
5. **Don't create unbounded JSONB** - Keep configs structured (use TypeScript types).
6. **Don't bypass handlers** - Always check `getHandler()` before executing.

---

## Migration Path for Existing Regulators

### **Phase 1**: Catalog Cleanup
- [ ] Audit existing templates in `service_templates`
- [ ] Ensure all have proper `activationRules`
- [ ] Add missing `intakeFieldsSchema` for dynamic forms

### **Phase 2**: Handler Migration
- [ ] Identify regulator-specific logic in monolithic handlers
- [ ] Extract into `ServiceRequestHandler` implementations
- [ ] Register in handler registry

### **Phase 3**: Rules Engine
- [ ] Migrate hardcoded eligibility checks to `activationRules`
- [ ] Add `customConditions` support for complex logic

### **Phase 4**: Backoffice UI
- [ ] Enable template editing in backoffice (already exists)
- [ ] Add activation rules UI editor
- [ ] Add template testing/preview mode

---

## Performance Considerations

1. **Cache active templates** - Use Redis to cache `getAvailableServiceTemplates()` results (5 min TTL)
2. **Index JSONB fields** - Add GIN indexes on `activationRules` for fast filtering
3. **Batch task creation** - Use `db.insert().values([...])` for multiple tasks
4. **Lazy load handlers** - Only instantiate handlers when needed

---

## Security Considerations

1. **Template authorization** - Only backoffice admins can create/edit templates
2. **Org-scoped templates** - Support org-specific templates (for white-label)
3. **Rate limiting** - Limit service request creation (e.g., 10/hour per org)
4. **Input sanitization** - Sanitize `intakeData` before storage
5. **RBAC on submission** - Require `manager` role to submit to regulator

---

## Monitoring & Observability

```typescript
// Track template usage
metrics.increment("service_request.created", {
  regulator: template.regulator,
  templateKey: template.templateKey,
});

// Track handler execution
const duration = await measureAsync(
  () => handler.beforeSubmission(ctx, deps, requestId)
);
metrics.timing("service_request.handler.beforeSubmission", duration, {
  handlerKey: handler.handlerKey,
});
```

---

## Example: Adding NHIMA Employer Registration

```typescript
// 1. Seed template
export const NHIMA_EMPLOYER_REGISTRATION_V1: ServiceTemplateSeed = {
  templateKey: "NHIMA_EMPLOYER_REGISTRATION_V1",
  templateVersion: 1,
  name: "Employer Registration",
  description: "Register as an employer with NHIMA",
  regulator: "nhima",
  // ... rest of config
};

// 2. (Optional) Create handler if needed
export class NhimaEmployerRegistrationHandler implements ServiceRequestHandler {
  readonly handlerKey = "NHIMA_EMPLOYER_REGISTRATION_V1";
  readonly regulatorCode = "nhima";

  async validateIntakeData(ctx, deps, data) {
    // Validate employee count, industry code, etc.
  }
}

// 3. Register handler
registerHandler(new NhimaEmployerRegistrationHandler());

// 4. Seed to database
pnpm --filter @repo/database seed:nhima-templates

// Done! Template now appears in catalog for orgs that meet activation rules.
```

---

## Conclusion

This architecture provides:

✅ **Scalability** - Add regulators via config, not code
✅ **Maintainability** - Separation of concerns (templates vs. handlers)
✅ **Flexibility** - JSONB configs + handler plugins
✅ **Auditability** - All changes logged
✅ **Multi-tenancy** - Org isolation enforced
✅ **Developer Experience** - Clear contracts, easy to extend

The key insight: **90% of service requests can be template-driven. The remaining 10% (complex workflows) use handlers.** This prevents over-engineering while maintaining extensibility.



