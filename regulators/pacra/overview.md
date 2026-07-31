---
title: "PACRA Integration - Current State Overview"
description: "Pre-implementation analysis for the PACRA Template System and Activation Engine"
---

This document summarizes the existing codebase state before implementing the template-driven workflow activation system for PACRA.

---

## 1. Existing Infrastructure

### 1.1 Database Schema

#### Core Compliance Tables (`packages/database/src/schema/compliance/`)

| Table | File | Status | Notes |
|-------|------|--------|-------|
| `obligation_templates` | `obligation-templates.ts` | Exists | Basic structure: name, frequency, regulator, billingTag, firstDueOn |
| `org_obligations` | `org-obligations.ts` | Exists | Links org to templates via `templateId` FK |
| `filings` | `filings.ts` | Exists | Full status workflow, SLA tracking, submission fields |
| `service_templates` | `service-templates.ts` | Exists | Basic structure: name, regulator, expectedDueInDays |
| `service_requests` | `service-requests.ts` | Exists | Links to templates via `templateId` FK |
| `tasks` | `tasks.ts` | Exists | Task checklist with types, status, required/blocking flags |
| `documents` | `documents.ts` | Exists | Document metadata with `kind` enum |
| `submission_jobs` | `submission-jobs.ts` | Exists | Backoffice job queue with priority |
| `payment_requests` | `payment-requests.ts` | Exists | Tenant → Bumara payments with fee tracking |
| `regulator_payouts` | `regulator-payouts.ts` | Exists | Bumara → Regulator payments with evidence |
| `tickets` | `tickets.ts` | Exists | Issue tracking with type/status enums |
| `timeline_events` | `timeline-events.ts` | Exists | Event timeline per filing/request |

#### PACRA-Specific Tables (`packages/database/src/schema/pacra/`)

| Table | File | Status | Notes |
|-------|------|--------|-------|
| `pacra_profiles` | `profiles.ts` | Exists | entityType, financialYearEnd, registrationNumber |
| `annual_returns` | `annual-returns/` | Exists | Detailed annual return with financial data |
| `name_clearance` | `name-clearance.ts` | Exists | Name clearance request data |
| `name_reservation` | `name-reservation.ts` | Exists | Name reservation request data |
| `company_registration` | `company-registration/` | Exists | Full company registration workflow |
| `business_name_registration` | `business-name-registration/` | Exists | Business name registration workflow |

#### Core Tables (`packages/database/src/schema/core/`)

| Table | File | Status | Notes |
|-------|------|--------|-------|
| `organizations` | `organizations.ts` | Exists | Has `financialYearEnd` field (MM-DD format) |
| `regulators` | `regulators.ts` | Exists | Has `availableTaskTypes` JSONB, seeded with PACRA |
| `organization_regulator_connections` | `organization-regulator-connections.ts` | Exists | Has `metadata` JSONB for regulator-specific config |

#### Enums (`packages/database/src/schema/enums.ts`)

All required enums exist:
- `filingStatusEnum` - Full workflow: pending_data → in_progress → ready_for_submission → submitted → accepted
- `serviceRequestStatusEnum` - Mirrors filing status
- `complianceTaskStatusEnum` - todo, doing, blocked, done, skipped
- `complianceTaskTypeEnum` - upload_document, fill_form, review_approve, payment_action, info_request, custom
- `frequencyEnum` - monthly, quarterly, annually, one_time
- `entityTypeEnum` - company, business_name, partnership, trust, ngo
- `billingTagEnum` - included, overage
- `documentKindEnum` - source, workpaper, submission, receipt, certificate

### 1.2 Backend Services

#### API Services (`packages/api-services/src/domains/`)

| Domain | Status | Notes |
|--------|--------|-------|
| `regulator-connections/` | Exists | Full CRUD with audit logging, encryption |
| `pacra/` | Exists | Profile CRUD, annual returns, name clearance, etc. |
| `audit/` | Exists | Audit log recording service |
| `organizations/` | Exists | Org management |

#### Backend RPC (`packages/backend/src/modules/`)

| Module | Status | Notes |
|--------|--------|-------|
| `regulator-connections/` | Exists | List, get, create, update, delete connections |
| `pacra/` | Exists | Profile, annual-returns, name-clearance, name-reservation, company/business registration |
| `audit-logs/` | Exists | Audit log queries |

### 1.3 Seed Data

The `packages/database/seed.ts` file seeds regulators including PACRA with:
- Basic regulator info (code: 'pacra', name, website)
- `availableTaskTypes` JSONB with task type definitions

---

## 2. What's Missing for Template-Driven Activation

### 2.1 Template System Gaps

#### `obligation_templates` table is missing:

| Field | Purpose |
|-------|---------|
| `template_key` | Stable unique ID (e.g., `PACRA_ANNUAL_RETURN_COMPANY_V1`) |
| `template_version` | Integer version for upgrades |
| `due_date_rule` | JSONB config: `{ "type": "FY_END_PLUS_MONTHS", "months": 3 }` |
| `activation_rules` | JSONB conditions: `{ "entityType": ["company"] }` |
| `task_template_configs` | JSONB array of task definitions |
| `doc_requirement_configs` | JSONB array of required documents |
| `payment_rule_config` | JSONB for fee configuration |

Current table has `organizationId` FK which suggests org-specific templates, but we need **global templates** for system-wide regulator workflows.

#### `service_templates` table is missing:

Same fields as above, plus:
| Field | Purpose |
|-------|---------|
| `intake_fields_schema` | JSONB Zod-like schema for request intake |

#### `filings` table is missing:

| Field | Purpose |
|-------|---------|
| `period_key` | Unique period identifier (e.g., `FY2024`) for idempotency |
| `template_id` | Reference to obligation_templates for runbook lookup |

#### `org_obligations` table is missing:

| Field | Purpose |
|-------|---------|
| `template_key` | Denormalized for easy lookup |
| `template_version` | Track which version activated |

#### `tasks` table is missing:

| Field | Purpose |
|-------|---------|
| `template_key` | Reference to task template for idempotency |

### 2.2 Missing Unique Constraints

For idempotent activation, we need:
- `(organization_id, template_key)` on `org_obligations`
- `(organization_id, obligation_id, period_key)` on `filings`
- `(filing_id, template_key)` on `tasks`

### 2.3 Missing Activation Engine

No centralized logic exists to:
1. Load templates for a regulator
2. Evaluate activation rules against tenant config
3. Create org_obligations from matching templates
4. Generate initial filings with computed due dates
5. Generate baseline tasks and document requirements
6. Audit all created records

### 2.4 Missing PACRA Connect Flow

Current `createConnection` in `regulator-connections/connections.service.ts`:
- Creates connection record with metadata
- Does NOT create PACRA profile
- Does NOT activate obligations
- Does NOT generate filings/tasks

---

## 3. Risks and Alignment Issues

### 3.1 Template Ownership Ambiguity

Current `obligation_templates.organizationId` suggests per-org templates. For the template system:
- **Global templates** (no org) should be seeded by Bumara
- **Org-specific templates** could be custom obligations (future)

**Resolution:** Make `organizationId` nullable. NULL = global template, non-NULL = org-custom.

### 3.2 Due Date Computation

Current `firstDueOn` is a static timestamp. We need:
- Rule-based computation from `financialYearEndDate`
- Support for different rule types (FY_END + X months, fixed date, etc.)

**Resolution:** Add `due_date_rule` JSONB and compute on activation.

### 3.3 Task Generation

No existing linkage between templates and task definitions. Current tasks are created ad-hoc.

**Resolution:** Add `task_template_configs` JSONB to templates, generate tasks on filing creation.

### 3.4 Existing PACRA Annual Returns Table

`packages/database/src/schema/pacra/annual-returns/annual-returns.ts` is a detailed PACRA-specific table. This is **separate** from the generic `filings` table.

**Resolution:** Use generic `filings` for compliance workflow tracking. The PACRA-specific `annual_returns` table stores the detailed return data (financial info, director changes, etc.). Link via `filingId` on annual_returns if needed.

---

## 4. Implementation Dependencies

### 4.1 Required Before Template System

All required tables and enums already exist. No blocking dependencies.

### 4.2 Schema Changes Required

1. Enhance `obligation_templates` with new fields
2. Enhance `service_templates` with new fields
3. Enhance `filings` with `period_key`, `template_id`
4. Enhance `org_obligations` with `template_key`, `template_version`
5. Enhance `tasks` with `template_key`
6. Add unique constraints for idempotency

### 4.3 New Components Required

1. PACRA template seed data
2. Activation engine (`activateRegulatorTemplates`)
3. Due date rule computation
4. PACRA connect service
5. PACRA connect RPC endpoint

---

## 5. File Reference

### Key Files to Modify

```
packages/database/src/schema/compliance/
├── obligation-templates.ts  # Add template_key, version, rules, configs
├── service-templates.ts     # Add template_key, version, rules, intake schema
├── org-obligations.ts       # Add template_key, template_version
├── filings.ts               # Add period_key, template_id
└── tasks.ts                 # Add template_key
```

### Key Files to Create

```
packages/database/src/seeds/
└── pacra-templates.ts       # PACRA obligation + service templates

packages/api-services/src/domains/activation/
├── index.ts                 # Exports
├── activation.schema.ts     # Zod schemas
├── activation.service.ts    # Core activation engine
├── due-date-rules.ts        # Due date computation
└── activation-rules.ts      # Rule evaluation

packages/api-services/src/domains/pacra/
└── pacra-connect.service.ts # PACRA connect flow

packages/backend/src/modules/pacra/connect/
├── connect.routes.ts        # RPC route definitions
└── connect.handlers.ts      # Request handlers
```

---

## 6. Summary

| Category | Status |
|----------|--------|
| Core compliance tables | ✅ Enhanced with template fields |
| PACRA-specific tables | ✅ Exists (profiles, annual returns) |
| Enums | ✅ Complete |
| Regulator seed data | ✅ Exists for PACRA |
| Connection CRUD | ✅ Exists |
| Template system | ✅ Implemented |
| Activation engine | ✅ Implemented |
| PACRA connect flow | ✅ Implemented |
| Centralized fee management | ✅ Implemented (`regulator_fees` table) |

All template system components have been implemented. See [Template System](/regulators/template-system) for details.

### Key Implementation Files

```
packages/database/src/schema/compliance/
├── obligation-templates.ts  # Template key, version, rules, configs
├── service-templates.ts     # Template key, version, intake schema
├── org-obligations.ts       # Template key, version, activated_at
├── filings.ts               # Period key, readiness tracking
├── tasks.ts                 # Template key for idempotency
└── regulator-fees.ts        # Centralized regulator fee management

packages/database/src/seeds/
├── pacra-templates.ts       # PACRA obligation + service templates
└── pacra-fees.ts            # PACRA regulator fees

packages/api-services/src/domains/activation/
├── activation.service.ts    # Core activation engine
├── due-date-rules.ts        # Due date computation
└── activation-rules.ts      # Rule evaluation

packages/api-services/src/domains/pacra/
└── pacra-connect.service.ts # PACRA connect flow

packages/backend/src/modules/pacra/connect/
├── routes.ts                # RPC route definitions
└── handlers.ts              # Request handlers
```

