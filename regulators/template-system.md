---
title: "Regulator Template System"
description: "Generic template model for regulator integrations in Bumara."
---

This document describes the template-driven workflow system that powers all regulator integrations. It provides a reusable framework for adding new regulators (ZRA, NAPSA, NHIMA, etc.).

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Template Layer (Global)                         │
├─────────────────────────────────────────────────────────────────────────┤
│  obligation_templates    │  service_templates    │  (Seeded by Bumara)  │
│  - template_key          │  - template_key       │                      │
│  - due_date_rule         │  - intake_schema      │                      │
│  - activation_rules      │  - activation_rules   │                      │
│  - task_configs          │  - task_configs       │                      │
│  - doc_requirements      │  - doc_requirements   │                      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       Activation Engine                                 │
├─────────────────────────────────────────────────────────────────────────┤
│  1. Load templates for regulator                                        │
│  2. Evaluate activation rules against tenant config                     │
│  3. Create org_obligations from matching templates                      │
│  4. Generate filings with computed due dates                            │
│  5. Generate tasks from template configs                                │
│  6. Record audit events                                                 │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      Organization Data Layer                            │
├─────────────────────────────────────────────────────────────────────────┤
│  org_obligations  →  filings  →  tasks  →  documents  →  payments       │
│                      (per org, links to templates via templateId)       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Template Types

### 2.1 Obligation Templates

Recurring compliance requirements with a regular schedule.

**Table:** `obligation_templates`

| Column | Type | Description |
|--------|------|-------------|
| `template_key` | TEXT (unique) | Stable identifier, e.g., `PACRA_ANNUAL_RETURN_COMPANY_V1` |
| `template_version` | INTEGER | Version number for upgrade tracking |
| `name` | TEXT | Human-readable name |
| `description` | TEXT | Detailed description |
| `frequency` | ENUM | monthly, quarterly, annually, one_time |
| `regulator` | ENUM | pacra, zra, napsa, nhima |
| `due_date_rule` | JSONB | How to compute due dates |
| `activation_rules` | JSONB | When to activate this template |
| `task_template_configs` | JSONB | Tasks to generate |
| `doc_requirement_configs` | JSONB | Documents required |
| `payment_rule_config` | JSONB | Fee structure |
| `organization_id` | TEXT | NULL for global, set for org-custom |

### 2.2 Service Templates

One-off compliance requests (ad-hoc workflows).

**Table:** `service_templates`

| Column | Type | Description |
|--------|------|-------------|
| `template_key` | TEXT (unique) | Stable identifier |
| `template_version` | INTEGER | Version number |
| `name` | TEXT | Human-readable name |
| `description` | TEXT | Detailed description |
| `what_is_this` | TEXT | Plain language explanation |
| `why_it_matters` | TEXT | Why tenant should care |
| `consequences_of_delay` | TEXT | What happens if delayed |
| `intake_fields_schema` | JSONB | Form fields for request intake |
| `activation_rules` | JSONB | Who can use this template |
| `task_template_configs` | JSONB | Tasks to generate |
| `doc_requirement_configs` | JSONB | Documents required |
| `payment_rule_config` | JSONB | Fee structure |

---

## 3. Activation Rules

Activation rules determine which templates apply to a tenant based on their configuration.

### 3.1 Rule Structure

```typescript
interface ActivationRules {
  entityType?: string[];       // e.g., ["company"]
  companyType?: string[];      // e.g., ["private", "public"]
  managedByBumara?: boolean;   // true = managed submission only
}
```

### 3.2 Evaluation Logic

Rules are evaluated as **AND** conditions:

1. All specified conditions must match
2. Empty rules = always activate
3. Missing fields in config = don't check that rule

```typescript
function evaluateActivationRules(rules, config) {
  if (!rules || isEmpty(rules)) return { shouldActivate: true };
  
  if (rules.entityType && !rules.entityType.includes(config.entityType)) {
    return { shouldActivate: false, reason: "Entity type mismatch" };
  }
  
  // ... check other rules
  
  return { shouldActivate: true };
}
```

### 3.3 Examples

| Template | Activation Rules | Activates For |
|----------|------------------|---------------|
| Company Annual Return | `entityType: ["company"]` | All companies |
| BN Annual Return | `entityType: ["business_name"]` | All business names |
| Public Company Filing | `entityType: ["company"], companyType: ["public"]` | Public companies only |
| Universal Template | `{}` or `null` | Everyone |

---

## 4. Due Date Rules

Due date rules define how to compute filing due dates from tenant configuration.

### 4.1 Rule Types

| Type | Description | Parameters |
|------|-------------|------------|
| `FY_END_PLUS_MONTHS` | Due X months after financial year end | `months` |
| `PERIOD_END_PLUS_DAYS` | Due X days after period end | `days` |
| `FIXED_DATE` | Due on a fixed date each year | `fixedMonth`, `fixedDay` |

### 4.2 Rule Structure

```typescript
interface DueDateRule {
  type: "FY_END_PLUS_MONTHS" | "PERIOD_END_PLUS_DAYS" | "FIXED_DATE";
  months?: number;      // For FY_END_PLUS_MONTHS
  days?: number;        // For PERIOD_END_PLUS_DAYS
  fixedMonth?: number;  // For FIXED_DATE (1-12)
  fixedDay?: number;    // For FIXED_DATE
}
```

### 4.3 Examples

| Rule | Config | Due Date |
|------|--------|----------|
| `{ type: "FY_END_PLUS_MONTHS", months: 3 }` | FY end: Dec 31, 2024 | March 31, 2025 |
| `{ type: "PERIOD_END_PLUS_DAYS", days: 14 }` | Period end: Jan 31 | Feb 14 |
| `{ type: "FIXED_DATE", fixedMonth: 6, fixedDay: 30 }` | Any | June 30 |

---

## 5. Task Template Configs

Tasks are auto-generated when a filing is created.

### 5.1 Config Structure

```typescript
interface TaskTemplateConfig {
  key: string;              // Unique key within template
  title: string;            // Display title
  description?: string;     // Help text
  taskType: TaskType;       // upload_document, fill_form, review_approve, etc.
  required: boolean;        // Required for submission?
  isBlocking?: boolean;     // Blocks other tasks?
  sequence: number;         // Display order
  dueDaysBeforeFiling?: number;  // Task due date offset
}
```

### 5.2 Task Types

| Type | Description |
|------|-------------|
| `upload_document` | Tenant must upload a file |
| `fill_form` | Tenant must complete a form |
| `review_approve` | Tenant must review and approve |
| `payment_action` | Tenant must make a payment |
| `info_request` | Tenant must provide information |
| `custom` | Custom task type |

---

## 6. Document Requirement Configs

Defines documents that may be required for a filing.

### 6.1 Config Structure

```typescript
interface DocRequirementConfig {
  key: string;           // Unique key
  name: string;          // Document name
  description?: string;  // Help text
  kind: DocumentKind;    // source, workpaper, submission, receipt, certificate
  required: boolean;     // Always required?
  conditions?: {         // Conditional requirements
    entityType?: string[];
  };
}
```

---

## 7. Payment Rule Configs

Defines fee structure for a filing.

### 7.1 Config Structure

```typescript
interface PaymentRuleConfig {
  paymentRequired: boolean;
  feeKey?: string;      // Key for looking up regulator fees from regulator_fees table
  serviceFee?: number;  // Bumara's service fee (in minor units)
  notes?: string;       // Notes for ops
}
```

### 7.2 Centralized Fee Management

Regulator fees are stored in the `regulator_fees` table, managed by backoffice:

**Table:** `regulator_fees`

| Column | Type | Description |
|--------|------|-------------|
| `regulator_id` | UUID | FK to regulators |
| `fee_key` | TEXT | Lookup key (e.g., `PACRA_ANNUAL_RETURN_COMPANY`) |
| `name` | TEXT | Human-readable name |
| `amount` | INTEGER | Fee in minor units (e.g., ngwee for ZMW) |
| `currency` | TEXT | ISO 4217 code (default: ZMW) |
| `effective_from` | TIMESTAMP | When fee became effective |
| `effective_until` | TIMESTAMP | NULL = currently active |
| `conditions` | JSONB | Entity type, company type conditions |

**Fee Lookup Flow:**

1. Template defines `feeKey` in `paymentRuleConfig`
2. On payment request creation, lookup fee from `regulator_fees` where `effective_until IS NULL`
3. Apply conditions if defined (e.g., different fees for company types)
4. Calculate total: `regulatorFee + serviceFee`

**Fee Updates:**

- Backoffice updates fees when regulators publish new fee schedules
- Old fees remain for historical records (with `effective_until` set)
- New fees have `effective_until = NULL` (active)

---

## 8. Adding a New Regulator

### Step 1: Create Template Seed File

```typescript
// packages/database/src/seeds/{regulator}-templates.ts

export const ZRA_VAT_RETURN_V1: ObligationTemplateSeed = {
  templateKey: "ZRA_VAT_RETURN_V1",
  templateVersion: 1,
  name: "ZRA VAT Return",
  frequency: "monthly",
  regulator: "zra",
  dueDateRule: {
    type: "PERIOD_END_PLUS_DAYS",
    days: 18, // VAT due 18th of following month
  },
  activationRules: {
    // VAT applies to all
  },
  taskTemplateConfigs: [...],
  docRequirementConfigs: [...],
  paymentRuleConfig: {
    paymentRequired: true,
    feeKey: "ZRA_VAT",
    serviceFee: 5000,  // ZMW 50.00 in minor units
  },
};
```

### Step 2: Create Connect Service

```typescript
// packages/api-services/src/domains/{regulator}/{regulator}-connect.service.ts

export async function connectZra(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  input: ZraConnectConfig
): Promise<ZraConnectResponse> {
  // 1. Validate input
  // 2. Create/update regulator profile
  // 3. Create/update connection
  // 4. Call activateRegulatorTemplates({ regulatorKey: "zra", ... })
  // 5. Return summary
}
```

### Step 3: Add RPC Endpoint

```typescript
// packages/backend/src/modules/{regulator}/connect/

// routes.ts
export const connectZraRoute = createRoute({...});

// handlers.ts
export const connectZraHandler = async (c) => {
  const result = await connectZra(ctx, deps, body);
  return c.json({ success: true, data: result });
};
```

### Step 4: Seed Templates

```bash
pnpm db:seed:zra
```

### Step 5: Test

```bash
# Run unit tests
pnpm test

# Manual QA
POST /api/zra/connect
```

---

## 9. Template Versioning

### 9.1 Version Strategy

- `template_version` is an integer starting at 1
- Increment version when template definition changes
- Old version records remain linked via `templateId`

### 9.2 Migration Path

When upgrading templates:

1. Update template seed with new version
2. Re-run seeder (uses ON CONFLICT DO UPDATE)
3. Existing org_obligations keep their `template_version`
4. New activations get new version

### 9.3 Template Key Format

```
{REGULATOR}_{OBLIGATION_TYPE}_{VARIANT}_V{VERSION}
```

Examples:

- `PACRA_ANNUAL_RETURN_COMPANY_V1`
- `ZRA_VAT_MONTHLY_V1`
- `NAPSA_CONTRIBUTION_V1`

---

## 10. Idempotency

The template system is fully idempotent:

| Entity | Unique Constraint | Behavior |
|--------|-------------------|----------|
| `org_obligations` | `(org_id, template_key)` | Update on conflict |
| `filings` | `(org_id, obligation_id, period_key)` | Update on conflict |
| `tasks` | `(filing_id, template_key)` | Skip on conflict |

This ensures:

- Re-connecting a regulator doesn't create duplicates
- Re-running activation is safe
- Upgrades can be applied without data loss

---

## 11. Best Practices

### Template Design

- Keep templates focused and single-purpose
- Use descriptive template keys
- Include helpful descriptions for tenants
- Define all required tasks and documents

### Activation Rules

- Start with broad rules, add specificity as needed
- Test rules with different entity types
- Document rule logic in template description

### Due Date Rules

- Use FY_END_PLUS_MONTHS for annual filings
- Use PERIOD_END_PLUS_DAYS for monthly/quarterly
- Consider grace periods and penalties

### Testing

- Write unit tests for new due date rules
- Write unit tests for activation rules
- Test idempotency with multiple connect calls

---

## 12. Related Documentation

- [PACRA Specification](/regulators/pacra/spec) - PACRA-specific details
- [PACRA Overview](/regulators/pacra/overview) - Current state analysis
- [Tasks Module](/modules/tasks) - Task system details

