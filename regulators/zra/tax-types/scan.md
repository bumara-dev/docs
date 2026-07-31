---
title: "ZRA Tax Types - Repository Scan"
description: "Codebase scan of the ZRA connection model, activation engine, and template patterns behind tax type registration and configuration."
---

## 1. ZRA Connection Model

### Database Schema
**Table: `organization_regulator_connections`**
- Location: `packages/database/src/schema/core/organization-regulator-connections.ts`
- Key fields:
  - `organizationId` - FK to organizations
  - `regulatorId` - FK to regulators
  - `registrationNumber` - Stores TPIN
  - `registrationStatus` - Enum (pending, active, etc.)
  - `metadata` - JSONB storing `{ tpin, businessName, selectedObligations }`
  - `isActive` - Boolean
- Unique constraint: one connection per (org_id, regulator_id)

**Table: `zra_tax_profiles`**
- Has boolean flags for each tax type:
  - `isPayeRegistered`
  - `isTurnOverTaxRegistered`
  - `isWhtRegistered`

### Tax Type Mapping
**Location:** `packages/api-services/src/domains/zra/zra-connect.service.ts`
```typescript
const ZRA_OBLIGATION_TEMPLATE_MAP = {
  paye: "ZRA_PAYE_MONTHLY_V1",
  turnover_tax: "ZRA_TURNOVER_TAX_MONTHLY_V1",
  withholding_tax: "ZRA_WHT_MONTHLY_V1",
} as const;
```

### Connect Service
**Function:** `connectZraWithActivation()`
- Location: `packages/api-services/src/domains/zra/zra-connect.service.ts`
- Takes `ZraConnectWithActivationConfig` with `selectedObligations: string[]`
- Creates/updates ZRA tax profile
- Creates/updates regulator connection (stores `selectedObligations` in metadata)
- Calls `activateRegulatorTemplates()` with filtered template keys
- Records audit logs for profile + connection creation

---

## 2. ZRA Connect UI

### Connect Form
**Location:** `apps/app/features/zra/components/connect/zra-connect-form.tsx`
- Multi-section form:
  - ZRA Portal Credentials (TPIN, password, business name)
  - **Tax Obligations Selection** - Checkboxes for PAYE, TOT, WHT
  - Consent checkboxes
- Validation: requires at least one obligation
- Uses `ZRA_OBLIGATION_OPTIONS` from `@/lib/queries/zra/zra-connect`

### Connect Modal
**Location:** `apps/app/features/zra/components/connect/zra-connect-modal.tsx`

---

## 3. Activation Engine

### Main Entry Point
**Function:** `activateRegulatorTemplates()`
- Location: `packages/api-services/src/domains/activation/activation.service.ts`
- Takes `{ orgId, regulatorKey, regulatorId, config }`
- `config.templateKeys` filters which templates to activate
- Loads obligation templates for regulator
- Creates org obligations, filings, and tasks

### Template Activation
**Function:** `activateObligationTemplate()`
- Creates org obligation from template
- Computes next period and due date
- Creates filing for period
- Generates tasks from template configs
- Uses upsert for idempotency

### Key Insight
The activation engine already supports:
- **Selective activation** via `templateKeys` filter
- **Idempotent** creation (ON CONFLICT handling)
- **Template-based** task and doc requirement generation

---

## 4. Obligation Templates

### Schema
**Table: `obligation_templates`**
- Location: `packages/database/src/schema/compliance/obligation-templates.ts`
- Key fields: `key`, `regulatorId`, `name`, `frequency`, `dueDateRule`, `taskTemplateConfigs`, `docRequirementConfigs`

### ZRA Templates (Expected)
Based on `ZRA_OBLIGATION_TEMPLATE_MAP`:
- `ZRA_PAYE_MONTHLY_V1`
- `ZRA_TURNOVER_TAX_MONTHLY_V1`
- `ZRA_WHT_MONTHLY_V1`

---

## 5. Service Templates + Dedicated Pages

### Schema
**Table: `service_templates`**
- Location: `packages/database/src/schema/compliance/service-templates.ts`
- Key fields:
  - `templateKey` (unique)
  - `intakeFieldsSchema` (JSONB)
  - `taskTemplateConfigs`, `docRequirementConfigs`, `paymentRuleConfig`
  - `activationRules`

### Dedicated Service Page Pattern
**Example:** `apps/app/app/(authenticated)/(general)/regulators/pacra/services/name-clearance/page.tsx`
- Uses `TEMPLATE_KEY` constant to identify template
- Fetches template via `useServiceTemplates(regulatorId)`
- Renders wizard component (`NameClearanceWizard`)
- Shows info panel explaining the service
- Back navigation to service requests list
- Loading skeleton and error states

### Existing ZRA Service Pages
**Location:** `apps/app/app/(authenticated)/(general)/regulators/zra/services/`
- `tax-clearance/` - Exists (Tax Clearance Certificate)

---

## 6. Submission Job + Payment Flow

### Submission Jobs Table
- Created when tenant requests submission
- Status flow: `queued → assigned → in_progress → submitted → accepted`

### Payment Requests Table
- Created when payment required for submission
- Status flow: `required_pending → pending_gateway → paid_platform_verified`

### Request Submission Flow
**Function:** `requestSubmission()`
- Location: `packages/api-services/src/domains/submissions/submissions.service.ts`
- Checks readiness (tasks, docs, payment, PAYE data)
- Creates `submission_job` if ready
- Updates source status to `submission_in_progress`

---

## 7. Audit Log Helper

### Main Function
**Function:** `recordAuditLog()`
- Location: `packages/api-services/src/domains/audit/audit-log.service.ts`
- Parameters:
  ```typescript
  {
    action: "create" | "update" | "delete" | ...,
    entityType: string,
    entityId: string,
    changes: { before?: object, after?: object },
    metadata?: object,
    actorType?: "USER" | "STAFF" | "SYSTEM",
  }
  ```

### Usage Pattern
```typescript
await recordAuditLog(ctx, deps, {
  action: "create",
  entityType: "regulator_connection",
  entityId: connection.id,
  changes: {
    after: { regulatorKey: "zra", selectedObligations },
  },
});
```

---

## 8. Key Decisions for Implementation

### Approach 1: New `zra_tax_config` Table (Recommended)
**Pros:**
- Clean separation of concerns
- Explicit `enabled_tax_types` vs `registered_tax_types`
- Easy to extend with more tax types
- Matches spec requirements

**Cons:**
- New table/migration
- Need to sync with existing `zra_tax_profiles` flags

### Approach 2: Extend Existing Structures
**Option A:** Use `organization_regulator_connections.metadata`
- Already stores `selectedObligations`
- Add `registeredTaxTypes` to metadata

**Option B:** Use `zra_tax_profiles` boolean flags
- Already has `isPayeRegistered`, etc.
- Add `isPaye*Enabled*` flags separately

### Recommendation
Use **Approach 2A** (connection metadata) for MVP:
- No new table needed
- `metadata.enabledTaxTypes` = what tenant wants
- `metadata.registeredTaxTypes` = what's confirmed on ZRA
- Update `zra_tax_profiles` flags to mirror registered state

---

## 9. Files to Modify / Create

### Data Model
- [ ] Add Zod schema for `TaxTypeKey` and `ZraTaxConfig`
- [ ] Add helper to read/update tax config from connection metadata

### Services
- [ ] Create `zra-tax-config.service.ts` with CRUD operations
- [ ] Modify `activateSelectedObligations()` to use config
- [ ] Add `addZraTaxType()` service for SR completion

### Routes/Handlers
- [ ] Add `GET /zra/tax-config` endpoint
- [ ] Add `PATCH /zra/tax-config` endpoint

### Service Request Templates
- [ ] Seed `ZRA_REGISTER_PAYE_V1`, `ZRA_REGISTER_WHT_V1`, `ZRA_REGISTER_TOT_V1`

### UI Pages
- [ ] `regulators/zra/services/register-paye/page.tsx`
- [ ] `regulators/zra/services/register-wht/page.tsx`
- [ ] `regulators/zra/services/register-tot/page.tsx`
- [ ] `regulators/zra/settings/tax-types/page.tsx`

---

## 10. Related Files

| File | Purpose |
|------|---------|
| `packages/database/src/schema/core/organization-regulator-connections.ts` | Connection schema |
| `packages/api-services/src/domains/zra/zra-connect.service.ts` | Connect + activation |
| `packages/api-services/src/domains/activation/activation.service.ts` | Activation engine |
| `packages/database/src/schema/compliance/service-templates.ts` | SR templates |
| `apps/app/features/zra/components/connect/zra-connect-form.tsx` | Tax type selection UI |
| `apps/app/.../regulators/pacra/services/name-clearance/page.tsx` | Dedicated SR page pattern |
