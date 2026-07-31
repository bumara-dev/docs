---
title: "Payroll Module — Technical Architecture"
description: "The payroll module's architecture: technology stack, package structure, multi-tenancy, PII encryption, and atomic payroll runs."
---

## 1. Overview

The Payroll module handles the complete employee compensation lifecycle: employee management, salary configuration, payroll execution, statutory deductions, and regulatory filing for Zambian authorities (ZRA PAYE, NAPSA, NHIMA). It enforces multi-tenancy, field-level encryption for PII, role-based access control, and atomic payroll processing.

## 2. Technology Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 14 (App Router), React, TanStack Query, Tailwind CSS, shadcn/ui |
| API | Hono (OpenAPI), Cloudflare Workers (`api-payroll`, `api-regulators`) |
| Business Logic | `packages/payroll/src/` |
| Database | PostgreSQL via Drizzle ORM (`packages/database`) |
| Auth | Clerk (multi-tenant, org-scoped) |
| Encryption | AES-256-GCM (employee PII at rest) |
| Background Jobs | `apps/api-jobs` (scheduled payroll runs, reminders) |

## 3. Package Structure

Payroll is split by vertical slice, not by layer. Each slice under `packages/payroll/src/`
owns its own `service.ts`, `schema.ts`, `queries.ts`, and colocated `*.test.ts`, and the
worker wires HTTP to it through a matching slice under `apps/api-payroll/src/modules/`.

```
bumara/
├── apps/
│   ├── app/                                # the single Next.js app
│   │   ├── app/payroll/                    # 14 route pages
│   │   └── zones/payroll/                  # the payroll zone
│   │       ├── modules/                    # allowances-deductions, dashboard,
│   │       │                               # employees, loans, monthly-inputs,
│   │       │                               # organization, payruns, reports, settings
│   │       ├── components/                 # zone-wide shared components
│   │       └── lib/                        # hooks, fetchers, query keys
│   ├── api-payroll/                        # Hono payroll worker (thin HTTP edge)
│   │   └── src/modules/<slice>/            # routes.ts, handlers.ts, index.ts
│   └── api-jobs/                           # background job runner
│       └── src/jobs/                       # payroll-run.ts, payroll-reminders.ts
├── packages/
│   ├── payroll/src/                        # payroll domain logic, one dir per slice
│   │   ├── employees/                      # service.ts, schema.ts, queries.ts,
│   │   │                                   # encryption.ts (AES-256-GCM PII)
│   │   ├── payruns/  payslips/  returns/
│   │   ├── allowances/  deductions/
│   │   ├── employee-allowances/  employee-deductions/
│   │   ├── loans/  loan-repayments/  gratuity/  leave/
│   │   ├── departments/  designations/  contract/
│   │   ├── monthly-inputs/  statutory-remittances/
│   │   ├── dashboard/  insights/  settings/
│   │   └── shared/                         # statutory-calculators.ts, pagination.ts
│   └── database/src/
│       ├── schema/payroll/                 # Drizzle table definitions
│       └── repositories/payroll.ts         # typed DB query functions
```

## 4. Multi-Tenancy

Every payroll operation is scoped to an organization:

```
Request → Clerk Auth → requireOrganizationContext(ctx) → organizationId
```

- All database tables include `organizationId` as a foreign key to `organizations`
- `requireOrganizationContext(ctx)` extracts and validates the org from the auth context
- Every query filters by `organizationId` — no cross-tenant data leakage
- Employee numbers (`EMP-0001`) are unique per organization

## 5. Request Flow

```
Browser → Next.js App → API Worker (Hono) → Handler → Service → Repository → PostgreSQL
                                                 ↓
                                          Encrypt/Decrypt PII
                                                 ↓
                                          Access Control Check
```

1. **Frontend**: TanStack Query hook fires fetch to `/payroll/*`
2. **API Worker**: Hono route validates auth + org context, parses Zod schema
3. **Handler**: Creates `ServiceContext` + `ServiceDependencies`, calls service
4. **Service**: Business logic, encryption, access control, calls repository
5. **Repository**: Drizzle ORM query against PostgreSQL
6. **Response**: Decrypt PII fields, filter by permissions, return JSON

## 6. Encryption Architecture

Employee PII is encrypted at rest using AES-256-GCM:

**Encrypted Fields (11):**
- `nrcNumberEncrypted`, `passportNumberEncrypted`, `tpinEncrypted`
- `napsaNumberEncrypted`, `nhimaNumberEncrypted`
- `emailEncrypted`, `phoneEncrypted`, `alternativePhoneEncrypted`, `addressEncrypted`
- `bankDetailsEncrypted` (JSON: accountNumber, sortCode, swiftCode, bankName, bankBranch)
- `emergencyContactEncrypted` (JSON: name, phone, relationship)

**Flow:**
- **Write**: `encryptEmployeeFields(input)` → encrypted data → DB insert
- **Read**: DB fetch → `decryptEmployeeFields(record)` → plaintext to caller
- **Export**: `redactSensitiveFields(employee, permissions)` → strip unauthorized fields
- **Audit**: `sanitizeForAuditLog(employee)` → strip all PII before logging

## 7. Access Control (FLAC)

Field-Level Access Control enforces who can see what:

| Role | ID Numbers | Bank Details | Salary | Contact | Export | Tax Reports |
|------|-----------|-------------|--------|---------|--------|------------|
| HR Manager | ✓ | ✓ | ✓ | ✓ | ✓ | ✗ |
| Payroll Officer | ✓ | ✓ | ✓ | ✗ | ✓ | ✗ |
| Finance Manager | ✗ | ✓ | ✓ | ✗ | ✓ | ✓ |
| Dept Manager | ✗ | ✗ | ✓ | ✓ | ✗ | ✗ |
| Employee | ✗ | ✗ | Own only | ✗ | ✗ | ✗ |
| Admin | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

Functions: `hasPermission()`, `ensureHasPermission()`, `filterEmployeeDataByPermissions()`, `isAccessingOwnData()`

## 8. Payroll Calculation Engine

The `runPayrunService()` function processes all employees in a single atomic transaction:

```
For each active employee:

  dailyRate = basicSalary / workingDaysInMonth
  hourlyRate = dailyRate / 8

  GROSS = basicSalary
        + Σ(employee allowances)
        + (hourlyRate × overtimeHours × 1.5)
        + (hourlyRate × holidayOvertimeHours × 2.0)
        + leavePay

  STATUTORY DEDUCTIONS:
    NHIMA  = basicSalary × 1%
    NAPSA  = calculateZambianNAPSA(GROSS)   → 5.5% employee contribution
    PAYE   = calculateZambianPAYE(GROSS)    → progressive tax bands

  OTHER DEDUCTIONS:
    absentism = basicSalary × (daysAbsent / workingDays)
    + employee-configured deductions
    + advance repayments
    + loan installments

  NET PAY = GROSS - all deductions
```

All writes (payslips, loan repayments, leave/gratuity transactions) are wrapped in `db.transaction()`.

## 9. Regulatory Integration Points

```
                    ┌──────────────┐
                    │  Payroll Run  │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌─────────┐  ┌─────────┐  ┌─────────┐
        │ZRA PAYE │  │  NAPSA  │  │  NHIMA  │
        │(Income  │  │(Pension │  │(Health  │
        │  Tax)   │  │  Fund)  │  │  Ins.)  │
        └─────────┘  └─────────┘  └─────────┘
```

- **ZRA PAYE**: Calculates PAYE per tax bands, files monthly returns with employee TPIN + tax deducted
- **NAPSA**: 5.5% employee + employer match, filed via contribution returns
- **NHIMA**: 1% contribution, filed via monthly returns

Each regulator has: profile registration → employee enrollment → periodic filing

## 10. Background Jobs

| Job | Schedule | Purpose |
|-----|----------|---------|
| `payroll-run.ts` | Configurable | Auto-run payroll for orgs with `nextPayrollDue <= today` |
| `payroll-reminders.ts` | Daily | Send reminders before payroll cutoff dates |

## 11. Key Design Decisions

1. **Slice-per-concern with a shared transaction**: payroll logic lives in one directory per slice under `packages/payroll/src/`, but a payroll run still executes inside a single transaction so payslips, loan repayments, and leave entries commit or roll back together
2. **Encryption at service layer**: DB stores ciphertext; decryption happens only when data leaves the service
3. **Atomic payroll runs**: Single transaction ensures payslips + loan repayments + leave entries are all-or-nothing
4. **Role-based field filtering**: Rather than separate endpoints, a single employee endpoint returns different field sets based on caller's role
5. **Tax bands as data**: PAYE brackets stored in DB, not hardcoded — supports regulatory changes without code deploys
