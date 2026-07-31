---
title: "NAPSA Database Integration - Completion Guide"
description: "This document outlines the completed NAPSA (National Pension Scheme Authority) database integration and provides the SQL seed data needed to register..."
---

## Overview

This document outlines the completed NAPSA (National Pension Scheme Authority) database integration and provides the SQL seed data needed to register NAPSA as an active regulator in the system.

## Completed Work

### 1. Schema Files ✅
All NAPSA tables have been created:
- `napsa_profiles` - Employer profiles
- `napsa_employees` - Employee records
- `napsa_contributions` - Monthly contributions per employee
- `napsa_returns` - Aggregate monthly returns
- `napsa_contribution_schedule` - Detailed contribution breakdowns

### 2. Schema Exports ✅
- Created `packages/database/src/schema/napsa/index.ts` to export all NAPSA tables
- Updated `packages/database/src/schema/index.ts` to include NAPSA exports

### 3. Relations Defined ✅
Added comprehensive Drizzle ORM relations in `packages/database/src/schema/napsa/napsa-relations.ts`:
- `napsaProfilesRelations` - Links to organization, employees, contributions, returns
- `napsaEmployeesRelations` - Links to profile, organization, tax profiles, contributions
- `napsaContributionsRelations` - Links to profile, employee, payslip, organization
- `napsaReturnsRelations` - Links to profile, organization, member, schedules
- `napsaContributionScheduleRelations` - Links to return and contribution

### 4. Documentation Updated ✅
Updated `docs/ARCHITECTURE/DATABASE_SCHEMA.md` with:
- Complete NAPSA table structures
- Relationship diagrams including NAPSA
- Cross-regulator integration details (ZRA ↔ NAPSA)
- Business requirement alignment

## Next Steps

### Register NAPSA in the Regulators Table

To complete the integration, insert the NAPSA regulator record into the `regulators` table:

```sql
INSERT INTO regulators (
  code,
  name,
  short_name,
  category,
  industry,
  minimum_plan_required,
  requires_add_on,
  is_public_access,
  is_available_in_trial,
  has_specialized_schema,
  has_api_integration,
  config,
  available_task_types,
  is_active,
  is_national,
  description,
  website
) VALUES (
  'napsa',
  'National Pension Scheme Authority',
  'NAPSA',
  'social_security',
  'pension',
  'starter',
  false,
  true,
  true,
  true,
  false,
  '{
    "portalUrl": "https://www.napsa.co.zm",
    "supportEmail": "info@napsa.co.zm",
    "supportPhone": "+260-211-256600",
    "requiresRegistration": true,
    "registrationFields": ["employer_number", "employer_name", "contact_person", "contact_email"],
    "defaultComplianceSchedule": "monthly"
  }'::jsonb,
  '[
    {
      "code": "monthly_contributions",
      "name": "Monthly Contributions Return",
      "description": "Submit monthly NAPSA contributions for all employees",
      "category": "contribution_filing",
      "isRecurring": true,
      "defaultPriority": "high",
      "defaultDocuments": ["contribution_schedule", "payment_proof"],
      "serviceFee": 150,
      "regulatoryFee": 0,
      "totalFee": 150
    },
    {
      "code": "employee_registration",
      "name": "Employee Registration",
      "description": "Register new employee with NAPSA",
      "category": "registration",
      "isRecurring": false,
      "defaultPriority": "normal",
      "defaultDocuments": ["employment_contract", "nrc_copy"],
      "serviceFee": 50,
      "regulatoryFee": 0,
      "totalFee": 50
    },
    {
      "code": "employer_registration",
      "name": "Employer Registration",
      "description": "Register organization as NAPSA employer",
      "category": "registration",
      "isRecurring": false,
      "defaultPriority": "high",
      "defaultDocuments": ["business_registration", "tpin_certificate"],
      "serviceFee": 200,
      "regulatoryFee": 0,
      "totalFee": 200
    },
    {
      "code": "contribution_amendment",
      "name": "Contribution Amendment",
      "description": "Amend previously submitted contributions",
      "category": "amendment",
      "isRecurring": false,
      "defaultPriority": "normal",
      "serviceFee": 100,
      "regulatoryFee": 0,
      "totalFee": 100
    }
  ]'::jsonb,
  true,
  true,
  'The National Pension Scheme Authority (NAPSA) is the statutory body responsible for the management of the National Pension Scheme in Zambia. Employers are required to submit monthly contributions (5% employee + 5% employer = 10% total) for all employees.',
  'https://www.napsa.co.zm'
);
```

### Create Organization-Regulator Connections

When organizations onboard with NAPSA profiles, create corresponding connections:

```sql
-- Example: When a napsa_profile is created, also create the connection
INSERT INTO organization_regulator_connections (
  organization_id,
  regulator_id,
  registration_number,
  registration_date,
  registration_status,
  compliance_status,
  is_active
)
SELECT 
  np.organization_id,
  (SELECT id FROM regulators WHERE code = 'napsa'),
  np.employer_number,
  np.registration_date,
  CASE 
    WHEN np.status = 'active' THEN 'active'::registration_connection_status
    ELSE 'not_started'::registration_connection_status
  END,
  'not_required'::compliance_status,
  true
FROM napsa_profiles np
WHERE NOT EXISTS (
  SELECT 1 FROM organization_regulator_connections orc
  WHERE orc.organization_id = np.organization_id
  AND orc.regulator_id = (SELECT id FROM regulators WHERE code = 'napsa')
);
```

## Database Relationships

### NAPSA → Organizations
```
napsa_profiles.organization_id → organizations.id
napsa_employees.organization_id → organizations.id
napsa_contributions.organization_id → organizations.id
napsa_returns.organization_id → organizations.id
```

### NAPSA → ZRA (Cross-Regulator Integration)
```
napsa_employees.employee_tax_profile_id → employee_tax_profiles.id
napsa_contributions.payslip_id → payslips.id
```

### NAPSA Internal Relationships
```
napsa_profiles (1) ──► (N) napsa_employees
napsa_profiles (1) ──► (N) napsa_contributions
napsa_profiles (1) ──► (N) napsa_returns

napsa_employees (1) ──► (N) napsa_contributions

napsa_returns (1) ──► (N) napsa_contribution_schedule
napsa_contributions (1) ──► (1) napsa_contribution_schedule
```

## Task Integration

NAPSA tasks can now be created using the polymorphic task system:

```typescript
// Example: Create a monthly contribution task
const task = await db.insert(obligationTasks).values({
  organizationId: 'org_123',
  regulatorId: napsaRegulatorId,
  taskType: 'monthly_contributions', // From availableTaskTypes
  category: 'contribution_filing',
  title: 'NAPSA Monthly Contributions - October 2024',
  priority: 'high',
  dueDate: new Date('2024-11-15'),
  regulatorDeadline: new Date('2024-11-20'),
  status: 'draft',
  relatedEntityType: 'napsa_return',
  relatedEntityId: returnId,
  serviceFee: 150.00,
  isRecurring: true,
  recurringSchedule: 'monthly'
});
```

## API Usage Examples

### Query NAPSA Data with Relations

```typescript
// Get organization's NAPSA profile with all employees
const napsaProfile = await db.query.napsaProfiles.findFirst({
  where: eq(napsaProfiles.organizationId, orgId),
  with: {
    employees: {
      where: eq(napsaEmployees.isActive, true),
      orderBy: [asc(napsaEmployees.lastName)]
    },
    returns: {
      orderBy: [desc(napsaReturns.contributionMonth)],
      limit: 12
    }
  }
});

// Get monthly contribution summary
const monthlyContributions = await db.query.napsaContributions.findMany({
  where: and(
    eq(napsaContributions.organizationId, orgId),
    eq(napsaContributions.contributionMonth, '2024-10')
  ),
  with: {
    employee: true,
    payslip: true
  }
});

// Get compliance status from connections
const napsaConnection = await db.query.organizationRegulatorConnections.findFirst({
  where: and(
    eq(organizationRegulatorConnections.organizationId, orgId),
    eq(organizationRegulatorConnections.regulatorId, napsaRegulatorId)
  ),
  with: {
    regulator: true
  }
});
```

## Compliance Tracking

### Update Compliance Status After Submission

```typescript
// After successful NAPSA return submission
await db.update(organizationRegulatorConnections)
  .set({
    lastSubmissionDate: new Date(),
    complianceStatus: 'compliant',
    nextDueDate: calculateNextDueDate('2024-10') // Next month
  })
  .where(and(
    eq(organizationRegulatorConnections.organizationId, orgId),
    eq(organizationRegulatorConnections.regulatorId, napsaRegulatorId)
  ));
```

## Business Requirements Alignment

✅ **Multi-tenant architecture** - All NAPSA tables include `organization_id` with cascade deletes

✅ **Flexible regulator management** - NAPSA registered in `regulators` table with dynamic task types

✅ **Plan-based access control** - NAPSA set to 'starter' plan with public access

✅ **Compliance tracking** - Connected through `organization_regulator_connections`

✅ **Task management** - NAPSA tasks supported through `obligation_tasks` with polymorphic relations

✅ **Specialized schemas** - 5 dedicated tables for complex contribution management

✅ **Generic connections** - Universal interface through `organization_regulator_connections`

✅ **Cross-regulator integration** - Links to ZRA employee tax profiles and payslips

## Testing Checklist

- [ ] Run database migration to ensure all NAPSA tables are created
- [ ] Insert NAPSA regulator record (see SQL above)
- [ ] Create test organization with NAPSA profile
- [ ] Verify organization_regulator_connection is created
- [ ] Test employee registration
- [ ] Test monthly contribution calculation
- [ ] Test return generation with contribution schedule
- [ ] Verify cross-integration with ZRA (payslips → contributions)
- [ ] Test task creation for NAPSA compliance
- [ ] Verify compliance status updates correctly

## Migration Command

If using Drizzle migrations:

```bash
cd packages/backend
pnpm drizzle-kit generate
pnpm drizzle-kit migrate
```

## Summary

The NAPSA database integration is complete and follows the same architectural patterns as ZRA and PACRA:
- Specialized schema with 5 interconnected tables
- Full Drizzle ORM relations defined
- Integration with the universal regulator system
- Cross-regulator integration with ZRA
- Comprehensive documentation

Once the NAPSA regulator is seeded in the database, the system will be ready to handle NAPSA compliance workflows end-to-end.

