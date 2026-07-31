---
title: "Service Request System Design Patterns - Comparison"
description: "A side-by-side comparison of architectural patterns for managing service requests across regulators, and the trade-offs of each."
---

This document compares different architectural patterns for managing service requests across multiple regulators, evaluating their trade-offs for the Bumara platform.

**Last Updated**: 2026-02-10

---

## Evaluation Criteria

1. **Extensibility**: How easy is it to add new regulators/services?
2. **Maintainability**: How complex is the codebase to maintain?
3. **Performance**: Impact on query/response times
4. **Developer Experience**: Learning curve and productivity
5. **Flexibility**: Can handle complex, regulator-specific logic?
6. **Multi-Tenancy**: Supports org-specific customization?

---

## Pattern 1: Hardcoded Routes (Anti-Pattern)

### Architecture

```
/api/pacra/name-clearance          ← Hardcoded route
/api/pacra/company-registration    ← Hardcoded route
/api/zra/vat-registration          ← Hardcoded route
/api/zra/paye-registration         ← Hardcoded route
/api/napsa/employer-registration   ← Hardcoded route
... (100+ routes for all services)
```

Each service is a separate route with custom handlers, validation, and database logic.

### Example Code

```typescript
// packages/backend/src/modules/pacra/name-clearance.ts
app.post("/api/pacra/name-clearance", async (c) => {
  const { proposedName1, proposedName2, entityType } = c.req.body;

  // Custom validation
  if (!proposedName1) throw new Error("Name required");

  // Custom DB insert
  const request = await db.insert(pacraNameClearanceRequests).values({
    organizationId: c.get("userId"),
    proposedName1,
    proposedName2,
    entityType,
    status: "pending",
  });

  // Custom task creation
  await db.insert(tasks).values([
    { requestId: request.id, title: "Upload TPIN" },
    { requestId: request.id, title: "Fill Form" },
  ]);

  return c.json({ success: true, requestId: request.id });
});
```

### Pros ✅
- Simple to understand initially
- Full control over each service
- Easy to debug specific service issues

### Cons ❌
- **Massive code duplication** (validation, task creation, status management repeated 100+ times)
- **Schema explosion** (separate table for each service type: `pacra_name_clearance_requests`, `zra_vat_registration_requests`, etc.)
- **Hard to refactor** (changing status flow requires touching 100+ files)
- **No shared business logic** (status transitions, payment handling, task management all duplicated)
- **Slow to add new services** (requires migrations, route creation, handler creation, tests for each)

### Verdict: ❌ **Not Recommended**

---

## Pattern 2: Entity-Attribute-Value (EAV)

### Architecture

```
service_requests (generic table)
  ├── id
  ├── organization_id
  ├── regulator
  └── service_type

service_request_attributes (key-value pairs)
  ├── request_id (FK)
  ├── attribute_key   ("proposedName1")
  └── attribute_value ("Bumara Ltd")
```

All service data stored as key-value pairs in a separate table.

### Example Code

```typescript
// Create service request
const request = await db.insert(serviceRequests).values({
  organizationId: orgId,
  regulator: "pacra",
  serviceType: "name_clearance",
});

// Store attributes as key-value pairs
await db.insert(serviceRequestAttributes).values([
  { requestId: request.id, key: "proposedName1", value: "Bumara Ltd" },
  { requestId: request.id, key: "proposedName2", value: "Bumara Inc" },
  { requestId: request.id, key: "entityType", value: "company" },
]);

// Retrieve (horrible query)
const attrs = await db
  .select()
  .from(serviceRequestAttributes)
  .where(eq(serviceRequestAttributes.requestId, requestId));

const data = Object.fromEntries(attrs.map(a => [a.key, a.value]));
```

### Pros ✅
- Extremely flexible (any service can have any fields)
- No schema migrations for new services
- Easy to add custom fields per org

### Cons ❌
- **Terrible query performance** (N+1 queries to fetch a single request)
- **No type safety** (attributes are just strings)
- **Complex validation** (can't use DB constraints)
- **Reporting nightmare** (can't efficiently query across attributes)
- **Lost indexing** (can't index on specific attributes like "tpin")

### Verdict: ❌ **Not Recommended** (only for highly dynamic user-generated schemas)

---

## Pattern 3: Polymorphic Service Tables

### Architecture

```
service_requests (base table)
  ├── id
  ├── organization_id
  ├── regulator
  ├── service_type (enum)
  ├── status
  └── metadata (JSONB, service-specific data)

tasks (polymorphic)
  ├── id
  ├── service_request_id (FK)
  ├── title
  └── status

documents (polymorphic)
  ├── id
  ├── service_request_id (FK)
  └── file_key
```

Service-specific data stored in JSONB `metadata` column.

### Example Code

```typescript
const request = await db.insert(serviceRequests).values({
  organizationId: orgId,
  regulator: "pacra",
  serviceType: "name_clearance",
  status: "pending_data",
  metadata: {
    proposedName1: "Bumara Ltd",
    proposedName2: "Bumara Inc",
    entityType: "company",
  },
});
```

### Pros ✅
- Unified schema (one table for all services)
- Better performance than EAV
- Flexible data storage (JSONB)
- Easy to add new services (no migrations)

### Cons ❌
- **Weak typing** (JSONB is unvalidated)
- **Limited indexing** (can't efficiently index JSONB nested keys)
- **No reuse** (every service is unique, no template pattern)
- **Hard to version** (changing a service structure affects all existing requests)

### Verdict: ⚠️ **Partial Solution** (good for simple cases, but lacks governance)

---

## Pattern 4: Template-Driven with JSONB Configs ✅ **RECOMMENDED**

### Architecture

```
service_templates (catalog)
  ├── id
  ├── template_key ("PACRA_NAME_CLEARANCE_V1")
  ├── regulator_id (FK)
  ├── intake_fields_schema (JSONB, defines form)
  ├── task_template_configs (JSONB, defines workflow)
  ├── doc_requirement_configs (JSONB, defines documents)
  ├── payment_rule_config (JSONB, defines fees)
  └── activation_rules (JSONB, defines eligibility)

service_requests (instances)
  ├── id
  ├── organization_id
  ├── template_id (FK)
  ├── payload (JSONB, intake form data)
  └── status

tasks (workflow steps, generated from template)
  ├── id
  ├── service_request_id (FK)
  ├── template_key (maps to task in template)
  ├── title
  ├── status
  └── payload (JSONB, task-specific data)
```

### Example Code

**1. Define Template (once)**:
```typescript
const template = {
  templateKey: "PACRA_NAME_CLEARANCE_V1",
  regulatorId: "pacra-uuid",
  intakeFieldsSchema: [
    { key: "proposedName1", label: "First Choice", type: "text", required: true },
    { key: "proposedName2", label: "Second Choice", type: "text", required: false },
  ],
  taskTemplateConfigs: [
    { key: "upload_tpin", title: "Upload TPIN", taskType: "upload_document", sequence: 1 },
    { key: "fill_form", title: "Fill Form", taskType: "fill_form", sequence: 2 },
  ],
  docRequirementConfigs: [
    { key: "tpin_cert", name: "TPIN Certificate", required: true },
  ],
  paymentRuleConfig: { feeKey: "PACRA_NAME_CLEARANCE", serviceFee: 20000 },
};
```

**2. Create Service Request (from template)**:
```typescript
const request = await createServiceRequest(ctx, deps, {
  templateId: "template-uuid",
  intakeData: {
    proposedName1: "Bumara Ltd",
    proposedName2: "Bumara Inc",
    entityType: "company",
  },
});

// Tasks auto-generated from template:
// ✅ Task 1: Upload TPIN (seq: 1, status: todo)
// ✅ Task 2: Fill Form (seq: 2, status: todo)
```

**3. (Optional) Custom Handler**:
```typescript
class PacraNameClearanceHandler implements ServiceRequestHandler {
  async validateIntakeData(ctx, deps, data) {
    // Custom validation beyond template schema
    const hasRestrictedWords = await checkRestrictedWords(data.proposedName1);
    if (hasRestrictedWords) return { valid: false, errors: [...] };
    return { valid: true };
  }

  async beforeSubmission(ctx, deps, requestId) {
    // Call PACRA API to check real-time availability
    await pacraApi.checkNameAvailability(...);
  }
}
```

### Pros ✅
- **High reusability** (one template → many requests)
- **Versioned** (`templateVersion` allows evolution without breaking existing requests)
- **Governed** (templates managed in backoffice, requires approval)
- **Type-safe** (Zod schemas validate JSONB configs)
- **Extensible** (handlers for complex logic, templates for standard logic)
- **Performance** (no N+1 queries, proper indexes on FK relationships)
- **Multi-tenant safe** (template can be global or org-specific)
- **Fast to add new services** (just create a template, no code changes)
- **Audit-friendly** (template changes are versioned, requests reference immutable templates)

### Cons ❌
- Requires understanding template model (learning curve)
- JSONB configs can become complex (mitigated by TypeScript types)
- Need tooling to edit templates (mitigated by backoffice UI)

### Verdict: ✅ **Recommended** (best balance of flexibility, maintainability, and performance)

---

## Pattern 5: Microservices per Regulator

### Architecture

```
pacra-service (separate microservice)
  ├── POST /name-clearance
  ├── POST /company-registration
  └── Database: pacra_requests

zra-service (separate microservice)
  ├── POST /vat-registration
  ├── POST /paye-registration
  └── Database: zra_requests

napsa-service (separate microservice)
  ├── POST /employer-registration
  └── Database: napsa_requests
```

Each regulator has a separate service with its own database.

### Pros ✅
- Complete isolation (one regulator can't affect another)
- Independent scaling (scale PACRA service separately)
- Technology flexibility (use different languages/frameworks per regulator)
- Clear boundaries

### Cons ❌
- **Operational complexity** (deploy/monitor 20+ services)
- **Data fragmentation** (cross-regulator reporting is hard)
- **Code duplication** (status management, payment handling duplicated across services)
- **Distributed transactions** (hard to maintain consistency)
- **Network overhead** (inter-service communication latency)
- **Over-engineering** (Bumara doesn't need this level of isolation yet)

### Verdict: ⚠️ **Overkill for now** (consider if Bumara scales to 1M+ requests/month)

---

## Pattern 6: Plugin Architecture

### Architecture

```
core/
  ├── service-requests.service.ts (base logic)
  └── plugin-loader.ts

plugins/
  ├── pacra-plugin/
  │   ├── manifest.json
  │   ├── name-clearance.handler.ts
  │   └── company-registration.handler.ts
  ├── zra-plugin/
  │   ├── manifest.json
  │   └── vat-registration.handler.ts
  └── napsa-plugin/
      ├── manifest.json
      └── employer-registration.handler.ts
```

Regulators implemented as plugins that register handlers dynamically.

### Pros ✅
- Clear boundaries (one folder per regulator)
- Extensible (add plugins without touching core)
- Testable (test plugins in isolation)
- Reusable (publish plugins as npm packages)

### Cons ❌
- Complexity (plugin loader, dependency injection)
- Over-abstraction (adds layers for no clear benefit)
- Versioning challenges (plugin compatibility with core)
- Debugging harder (indirect control flow)

### Verdict: ⚠️ **Interesting, but premature** (consider if building a marketplace for third-party regulator integrations)

---

## Recommended Approach for Bumara

### **Hybrid: Template-Driven (Pattern 4) + Optional Handlers (Pattern 6 Lite)**

**Core Design**:
1. **90% of services** use **templates only** (no custom code)
2. **10% of services** use **templates + handlers** (for complex logic)

**Example**:

| Service | Complexity | Approach |
|---------|------------|----------|
| PACRA Annual Return | Simple | Template only |
| NAPSA Monthly Contribution | Simple | Template only |
| PACRA Name Clearance | Medium | Template + Handler (API call for validation) |
| PACRA Company Registration | High | Template + Handler (dynamic tasks, complex fees) |
| ZRA VAT Registration | High | Template + Handler (integration with ZRA portal) |

**Benefits**:
- ✅ **Fast to add simple services** (template in backoffice, no code)
- ✅ **Flexible for complex services** (handler for custom logic)
- ✅ **Single database** (easier reporting, transactions)
- ✅ **Moderate complexity** (not too simple, not over-engineered)
- ✅ **Scales well** (can handle 100+ regulators, 1000+ service types)

---

## Implementation Checklist

- [x] Template schema defined (`service_templates` table)
- [x] Service request lifecycle (`service_requests`, `tasks`, `documents`)
- [x] Activation rules engine (eligibility checks)
- [x] Handler interface (`ServiceRequestHandler`)
- [x] Handler registry (auto-discovery)
- [x] Template seeding (TypeScript seed files)
- [ ] Backoffice UI for template management (partially done)
- [ ] Frontend dynamic form generator (partially done)
- [ ] Handler plugin system (optional, future)
- [ ] Template marketplace (optional, future)

---

## Conclusion

After evaluating 6 patterns, the **Template-Driven + Optional Handlers** approach is the best fit for Bumara because:

1. **Pragmatic**: Solves 90% of cases with zero code (templates)
2. **Extensible**: Handles 10% complex cases with handlers
3. **Performant**: No N+1 queries, proper relational structure
4. **Maintainable**: Clear separation of concerns, versioned templates
5. **Developer-friendly**: Add new services in minutes, not days

This pattern is used by successful products like:
- **Stripe**: Product catalog (products → prices → subscriptions)
- **Shopify**: Theme templates (themes → stores)
- **Zapier**: Workflow templates (templates → zaps)
- **Salesforce**: Process Builder (templates → processes)

By following this proven pattern, Bumara can scale to support hundreds of regulators while keeping the codebase maintainable and the developer experience excellent.
