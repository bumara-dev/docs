---
title: "ZRA MVP Implementation — Codebase Scan"
description: "Scan Date: 2026-01-15 Goal: Document existing architecture to inform ZRA MVP implementation"
---

## 1. PACRA Activation Engine Reference

### 1.1 Core Activation Service

**Location:** activation.service.ts

**Main Entry Point:** `activateRegulatorTemplates(ctx, deps, input)`

Flow:
1. Load global templates for regulator → `loadObligationTemplates()`, `loadServiceTemplates()`
2. Evaluate activation rules against tenant config → `filterActivatableTemplates()`
3. For each matching template → `activateObligationTemplate()`
   - Create org obligation (idempotent via upsert)
   - Compute next period via `computeNextPeriod()`
   - Create filing (idempotent via unique constraint)
   - Generate tasks from template configs
4. Record audit logs for all created records

**Key Functions:**
| Function | Purpose |
|----------|---------|
| `activateRegulatorTemplates()` | Main entry, orchestrates full activation |
| `loadObligationTemplates()` | Loads global templates for regulator |
| `upsertOrgObligation()` | Creates/updates org obligation (idempotent) |
| `upsertFiling()` | Creates/updates filing for period (idempotent) |
| `generateTasksForFiling()` | Creates tasks from template configs |

### 1.2 Due Date Rules

**Location:** due-date-rules.ts

Supports:
- `FY_END_PLUS_MONTHS` — Annual filings
- `PERIOD_END_PLUS_DAYS` — Monthly/quarterly filings
- `FIXED_DATE` — Fixed calendar dates

**Key Function:** `computeNextPeriod(rule, frequency, config)` returns:
```typescript
interface ComputedPeriod {
  periodKey: string;      // "2025-01", "FY2024"
  periodLabel: string;    // "January 2025"
  periodStart: Date;
  periodEnd: Date;
  dueOn: Date;
}
```

### 1.3 Activation Rules

**Location:** activation-rules.ts

Evaluates:
- `entityType` — Match company/business_name
- `companyType` — Match specific company types
- `managedByBumara` — Whether managed submission

---

## 2. Template Definitions

### 2.1 Template Schema

**Location:** obligation-templates.ts

**Key Interfaces:**

```typescript
interface ObligationTemplateSeed {
  templateKey: string;           // "PACRA_ANNUAL_RETURN_COMPANY_V1"
  templateVersion: number;
  name: string;
  description: string;
  frequency: "monthly" | "quarterly" | "annually" | "one_time";
  regulator: "pacra" | "zra" | "napsa" | "other";
  dueDateRule: DueDateRule;
  activationRules: ActivationRules;
  taskTemplateConfigs: TaskTemplateConfig[];
  docRequirementConfigs: DocRequirementConfig[];
  paymentRuleConfig: PaymentRuleConfig;
  billingTag: "included" | "overage";
}

interface TaskTemplateConfig {
  key: string;
  title: string;
  description?: string;
  taskType: "upload_document" | "fill_form" | "review_approve" | ...;
  required: boolean;
  isBlocking?: boolean;
  sequence: number;
  dueDaysBeforeFiling?: number;
}
```

### 2.2 PACRA Template Examples

**Location:** pacra-templates.ts

Templates defined:
- `PACRA_ANNUAL_RETURN_COMPANY_V1` — Annual, FY_END_PLUS_MONTHS(3)
- `PACRA_ANNUAL_RETURN_BUSINESS_NAME_V1` — Annual
- Services: Name Clearance, Name Reservation, Change of Directors, Company Registration

### 2.3 Template Seeding

**Location:** seed-pacra-templates.ts

---

## 3. Schema and Enums

### 3.1 Compliance Tables

**Location:** `packages/database/src/schema/compliance/`

| Table | Purpose | Key Fields |
|-------|---------|------------|
| `obligation_templates` | Global template definitions | regulatorId, templateKey, dueDateRule, taskTemplateConfigs |
| `org_obligations` | Org-specific obligation instances | organizationId, templateId, status |
| `filings` | Period instances of obligations | obligationId, periodKey, dueOn, status |
| `tasks` | Individual tasks for filings | filingId, templateKey, status |
| `documents` | Files attached to filings | filingId, kind |

### 3.2 Regulator Enum

**Location:** enums.ts:175

```typescript
export const regulatorEnum = pgEnum("regulator", [
  "zra",
  "napsa",
  "pacra",
  "other",
]);
```

> ✅ ZRA already exists in regulator enum

### 3.3 Regulator Records

Regulators table seeded with records including code/name. Need to verify ZRA regulator record exists.

---

## 4. PACRA Connect Flow

### 4.1 Connect Service

**Location:** pacra-connect.service.ts

Flow:
1. Validate input via `pacraConnectConfigSchema`
2. Get regulator record (`regulators.code = 'pacra'`)
3. Upsert PACRA profile (`pacra_profiles` table)
4. Upsert regulator connection (`connections` table)
5. Call `activateRegulatorTemplates()` with config
6. Return activation summary

### 4.2 Connect Handler

**Location:** handlers.ts

HTTP handler calls `connectPacra()` service.

### 4.3 Connect Schema

**Location:** activation.schema.ts

Exports `pacraConnectConfigSchema` with entity type, company type, financial year end date, etc.

---

## 5. Tenant UI Patterns

### 5.1 PACRA Workspace Structure

**Location:** `apps/app/app/(authenticated)/(general)/regulators/pacra/`

Subdirectories:
- `filings/` - Filings list and detail pages
- `tasks/` - Tasks filtered by regulator
- `service-requests/` - Service requests
- `obligations/` - Obligations list
- `documents/`, `timeline/`, `payments/`, `reports/`

### 5.2 Filings Page Pattern

**Location:** page.tsx/(general)/regulators/pacra/filings/page.tsx)

Uses reusable components:
- `RegulatorWorkspaceLayout` — Layout wrapper
- `FilingsPageContent` — Paginated filing list with config

Config pattern:
```typescript
<FilingsPageContent
  config={PACRA_CONFIG}
  regulatorId={pacraConnection.regulatorId}
/>
```

### 5.3 Regulator Config Pattern

**Location:** `apps/app/features/regulators/config/`

Each regulator has a config object defining UI metadata.

---

## 6. Existing ZRA Implementation

### 6.1 ZRA Domain Services

**Location:** `packages/api-services/src/domains/zra/`

| Service | Purpose |
|---------|---------|
| `zra-paye.service.ts` | PAYE return logic |
| `zra-tot.service.ts` | Turnover Tax logic |
| `zra-wht.service.ts` | Withholding Tax logic |
| `zra-tax-profile.service.ts` | Tax profile management |
| `zra-provisional-tax.service.ts` | Provisional tax |
| `zra-tax-clearance.service.ts` | Tax clearance requests |
| `zra.schema.ts` | ZRA-specific Zod schemas |

> ⚠️ These services exist but are NOT integrated with the activation engine

### 6.2 ZRA UI

**Location:** `apps/app/app/(authenticated)/(general)/regulators/zra/`

Currently only has a basic `page.tsx`:
```typescript
// Simple placeholder page
export default function ZraPage() { ... }
```

### 6.3 ZRA Connect Modal

**Location:** zra-connect-modal.tsx

Exists but needs to be connected to activation engine.

---

## 7. Existing Tests

**Location:** `packages/api-services/src/domains/activation/__tests__/`

| Test File | Coverage |
|-----------|----------|
| `activation-rules.test.ts` | Activation rule evaluation |
| `due-date-rules.test.ts` | Due date computations |
| `activation.service.test.ts` | Main activation service |

---

## 8. Key Findings Summary

### ✅ Reusable Components
- Activation engine (`activateRegulatorTemplates`) is regulator-agnostic
- Template system supports monthly obligations (via PERIOD_END_PLUS_DAYS)
- UI components (`FilingsPageContent`, `RegulatorWorkspaceLayout`) are reusable
- ZRA regulator enum value already exists

### ⚠️ Gaps to Address
1. **No ZRA templates** — Need to create PAYE, Turnover Tax, WHT templates
2. **No ZRA connect service** — Need `connectZra()` equivalent to `connectPacra()`
3. **No ZRA profile schema** — May need ZRA tax profile upsert
4. **ZRA regulator record** — Verify seeded in `regulators` table
5. **ZRA workspace UI** — Minimal, needs filings/tasks pages

### 📋 Implementation Pattern
Follow PACRA pattern:
1. Define templates in `packages/database/src/seeds/zra-templates.ts`
2. Create seed script `seed-zra-templates.ts`
3. Create `connectZra()` service in `packages/api-services/src/domains/zra/`
4. Add backend routes in `packages/backend/src/modules/zra/connect/`
5. Create ZRA config in `apps/app/features/regulators/config/`
6. Build ZRA workspace pages using reusable components
