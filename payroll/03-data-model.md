---
title: "Payroll Module — Data Model"
description: "Entity relationships and table definitions for payroll settings, employees, compensation, statutory config, payslips, and loans."
---

## Entity Relationship Overview

```
organizations
    │
    ├── payroll_settings (1:1)
    │
    ├── allowances (1:N) ──────────┐
    │                              │
    ├── deductions (1:N) ──────────┤
    │                              │
    ├── statutory_remittance_types (1:N)
    │       ├── percentage_remittances (1:1)
    │       └── tax_bands (1:N)
    │                              │
    ├── departments (1:N) ─────────┤
    │                              │
    ├── designations (1:N) ────────┤
    │                              │
    ├── employees (1:N) ───────────┤
    │       ├── employee_allowances (N:M via allowances)
    │       ├── employee_deductions (N:M via deductions)
    │       ├── employee_loans (1:N)
    │       │       └── loan_repayments (1:N)
    │       ├── leave_transactions (1:N)
    │       ├── gratuity_transactions (1:N)
    │       └── employee_payroll_monthly_inputs (1:N per month)
    │
    ├── payroll_runs (1:N)
    │       └── payslips (1:N per employee)
    │               ├── payslip_allowances (1:N)
    │               ├── payslip_deductions (1:N)
    │               └── payslip_statutory_remittances (1:N)
    │
    ├── napsa_profiles (1:1)
    │       ├── napsa_employees (1:N)
    │       ├── napsa_returns (1:N)
    │       └── napsa_contributions (1:N)
    │
    ├── nhima_profiles (1:1)
    │       ├── nhima_employees (1:N)
    │       └── nhima_returns (1:N)
    │
    └── paye_submissions (1:N)
            └── paye_employees (1:N)
```

---

## Core Tables

### `payroll_settings`

Organization-wide payroll configuration.

| Column | Type | Default | Description |
|--------|------|---------|-------------|
| id | UUID | random | Primary key |
| organizationId | TEXT | — | FK → organizations (cascade delete) |
| leaveCalculationBasis | ENUM | — | `monthly_accrual` or `annual_grant` |
| leaveCarryoverPercentage | DECIMAL(10,2) | — | % of leave that carries over |
| leaveCarryoverMaxDays | DECIMAL(10,2) | — | Max carryover days |
| deductionSequence | JSON | — | Ordered deduction priority list |
| maxDeductionPercentage | DECIMAL(5,2) | 50 | Max % of salary deductible |
| interestCalculationMethod | ENUM | — | `simple` or `compound` |
| maxLoanAmount | DECIMAL(18,2) | — | Max loan principal |
| maxLoanTerm | INT | — | Max loan duration (months) |
| requiresCollateral | BOOLEAN | false | Loan collateral requirement |
| workDays | INT | 26 | Working days per month |
| paymentDayOfMonth | INT | — | Day salaries are paid |
| cutoffDayOfMonth | INT | — | Monthly input cutoff day |
| overtimeMultiplier | DECIMAL(4,2) | 1.5 | Regular overtime rate multiplier |
| holidayOvertimeMultiplier | DECIMAL(4,2) | 2.0 | Holiday overtime rate multiplier |
| currency | TEXT | ZMW | Payroll currency |
| roundingMethod | ENUM | half_up | `half_up`, `half_down`, `floor`, `ceil` |
| isActive | BOOLEAN | true | Active flag |

### `payroll_runs`

Individual payroll execution cycles.

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| organizationId | TEXT | FK → organizations |
| periodStart | DATE | Payroll period start |
| periodEnd | DATE | Payroll period end |
| paymentDate | DATE | When salaries are/were paid |
| paymentReference | TEXT | Bank transfer reference |
| totalEmployees | DECIMAL(10,0) | Employee count |
| totalGross | DECIMAL(18,2) | Sum of gross salaries |
| totalAllowances | DECIMAL(18,2) | Sum of all allowances |
| totalDeductions | DECIMAL(18,2) | Sum of all deductions |
| totalNetPay | DECIMAL(18,2) | Gross minus deductions |
| skillDevLevy | DECIMAL(18,2) | Skills development levy |
| totalPAYE | DECIMAL(18,2) | Total PAYE tax withheld |
| totalNapsa | DECIMAL(18,2) | Total NAPSA contributions |
| totalNhima | DECIMAL(18,2) | Total NHIMA contributions |
| totalLoansDeducted | DECIMAL(18,2) | Loan repayments deducted |
| status | ENUM | `draft` → `approved` → `processed` → `paid` → `reconciled` → `locked` |
| approvalNotes | TEXT | Notes from approver |
| createdByUserId | TEXT | Who created |
| approvedByUserId | TEXT | Who approved |
| processedByUserId | TEXT | Who processed |
| paidByUserId | TEXT | Who marked paid |
| approvalDate | TIMESTAMP | When approved |
| processingDate | TIMESTAMP | When processed |
| paidDate | TIMESTAMP | When paid |
| reconciliationDate | TIMESTAMP | When reconciled |
| isLocked | BOOLEAN | Prevents edits after processing |
| reconciliationNotes | TEXT | Reconciliation notes |
| attachments | JSON | `[{name, url, type}]` |
| notes | TEXT | General notes |

**Indexes:** `organizationId`, `status`, `periodStart/periodEnd`, `organizationId + status`

### Payroll Run Status Lifecycle

```
draft → approved → processed → paid → reconciled → locked
  │                                                    │
  └────────────── (no reverse transitions) ────────────┘
```

- **draft**: Payroll calculated, awaiting review
- **approved**: Reviewed and approved by authorized user
- **processed**: Payslips finalized, ready for payment
- **paid**: Bank transfers executed
- **reconciled**: Bank statement matched
- **locked**: Permanently sealed, no further changes

---

## Employee Tables

### `employees`

Employee master record with encrypted PII.

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| organizationId | TEXT | FK → organizations |
| employeeNumber | TEXT | Auto-generated `EMP-XXXX` (unique per org) |
| firstName | TEXT | First name |
| lastName | TEXT | Last name |
| middleName | TEXT | Middle name |
| dateOfBirth | DATE | Date of birth |
| gender | ENUM | Gender |
| maritalStatus | ENUM | Marital status |
| **Encrypted Fields** | | |
| nrcNumberEncrypted | TEXT | National Registration Card |
| passportNumberEncrypted | TEXT | Passport number |
| tpinEncrypted | TEXT | Taxpayer Identification Number |
| napsaNumberEncrypted | TEXT | NAPSA member number |
| nhimaNumberEncrypted | TEXT | NHIMA member number |
| emailEncrypted | TEXT | Email address |
| phoneEncrypted | TEXT | Phone number |
| alternativePhoneEncrypted | TEXT | Alternative phone |
| addressEncrypted | TEXT | Physical address |
| bankDetailsEncrypted | TEXT | JSON: &#123;accountNumber, sortCode, swiftCode, bankName, bankBranch&#125; |
| emergencyContactEncrypted | TEXT | JSON: &#123;name, phone, relationship&#125; |
| **Employment** | | |
| departmentId | UUID | FK → departments |
| designationId | UUID | FK → designations |
| branchId | UUID | FK → branches |
| basicSalary | DECIMAL(18,2) | Monthly base salary |
| employmentType | ENUM | `full_time`, `part_time`, `contract`, `casual` |
| employmentStatus | ENUM | `active`, `on_leave`, `suspended`, `terminated` |
| hireDate | DATE | Start date |
| probationEndDate | DATE | End of probation |
| confirmationDate | DATE | Confirmation date |
| terminationDate | DATE | Termination date |
| isActive | BOOLEAN | Active flag |

### `employee_payroll_monthly_inputs`

Monthly attendance and overtime data per employee.

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| employeeId | UUID | FK → employees |
| organizationId | TEXT | FK → organizations |
| month | INT | 1-12 |
| year | INT | e.g. 2026 |
| workingDaysAbsent | DECIMAL | Days absent |
| overtimeHours | DECIMAL | Regular overtime hours |
| holidayOvertimeHours | DECIMAL | Holiday overtime hours |
| leaveDaysPay | DECIMAL | Leave days to pay |
| advance | DECIMAL | Salary advance amount |
| otherDeductions | DECIMAL | Miscellaneous deductions |
| loanPayments | DECIMAL | Manual loan payments |
| notes | TEXT | Notes |

---

## Compensation Configuration

### `allowances`

Company-wide allowance type definitions.

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| organizationId | TEXT | FK → organizations |
| name | TEXT | e.g. "Housing Allowance" |
| code | TEXT | e.g. "HOUSING" |
| calculationType | ENUM | `fixed`, `percentage` |
| payType | ENUM | `monthly`, `one_time` |
| amount | DECIMAL(18,2) | Default amount |
| isTaxable | BOOLEAN | Subject to PAYE |
| isProRata | BOOLEAN | Pro-rated for partial months |
| isAutoCalculated | BOOLEAN | System-managed |
| isSystemGenerated | BOOLEAN | Cannot be deleted (Housing, Transport, Meal, Leave Pay) |

### `deductions`

Company-wide deduction type definitions.

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| organizationId | TEXT | FK → organizations |
| name | TEXT | e.g. "Salary Advance" |
| code | TEXT | e.g. "ADVANCE" |
| calculationType | ENUM | `fixed`, `percentage` |
| calculationBase | ENUM | `gross`, `net` |
| payType | ENUM | `monthly`, `one_time` |
| amount | DECIMAL(18,2) | Default amount |
| employeeContributionRate | DECIMAL(5,2) | Employee rate |
| employerContributionRate | DECIMAL(5,2) | Employer rate |
| isSystemGenerated | BOOLEAN | Cannot be deleted |

### `employee_allowances` / `employee_deductions`

Join tables assigning allowances/deductions to specific employees with custom amounts and effective date ranges.

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| employeeId | UUID | FK → employees |
| allowanceId / deductionId | UUID | FK → allowances / deductions |
| amount | DECIMAL(18,2) | Custom amount for this employee |
| isTaxable | BOOLEAN | Override taxable flag |
| frequency | ENUM | `one_time`, `monthly`, `quarterly`, `annual` (deductions only) |
| effectiveFrom | DATE | Start date |
| effectiveTo | DATE | End date (null = indefinite) |

---

## Statutory Configuration

### `statutory_remittance_types`

Configurable tax/contribution types.

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| organizationId | TEXT | FK → organizations |
| code | TEXT | `PAYE`, `NAPSA`, `NHIMA` |
| name | TEXT | Display name |
| type | ENUM | `percentage` or `tax_bands` |
| category | ENUM | `statutory` or `voluntary` |
| frequency | ENUM | `monthly`, `quarterly`, `annual` |
| calculationBase | TEXT | `gross`, `basic`, etc. |
| authority | TEXT | `ZRA`, `NAPSA`, `NHIMA` |
| country | TEXT | `ZM` |

### `percentage_remittances`

For percentage-based contributions (NAPSA, NHIMA).

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| remittanceTypeId | UUID | FK → statutory_remittance_types |
| employeeContributionRate | DECIMAL(5,2) | Employee % |
| employerContributionRate | DECIMAL(5,2) | Employer % |
| ceilingAmount | DECIMAL(18,2) | Max salary subject to contribution |
| thresholdAmount | DECIMAL(18,2) | Minimum salary for contribution |
| exclusions | JSON | Exempt categories |

### `tax_bands`

For progressive tax brackets (PAYE).

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| remittanceTypeId | UUID | FK → statutory_remittance_types |
| bandNumber | INT | Order (1, 2, 3, 4) |
| minimumIncome | DECIMAL(18,2) | Band floor |
| maximumIncome | DECIMAL(18,2) | Band ceiling |
| taxRate | DECIMAL(5,2) | Marginal rate % |
| fixedAmount | DECIMAL(18,2) | Fixed amount for band |
| effectiveFrom | DATE | Start date |
| effectiveTo | DATE | End date |

**Current Zambian PAYE Bands (2026):**

| Band | Income Range (ZMW) | Rate | Fixed |
|------|-------------------|------|-------|
| 1 | 0 – 5,100 | 0% | 0 |
| 2 | 5,100.01 – 7,100 | 20% | 0 |
| 3 | 7,100.01 – 9,200 | 30% | K400 |
| 4 | 9,200.01+ | 37.5% | K1,030 |

---

## Payslip Tables

### `payslips`

Individual employee pay records per payroll run.

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| payrollRunId | UUID | FK → payroll_runs |
| employeeId | UUID | FK → employees |
| periodStart | DATE | Pay period start |
| periodEnd | DATE | Pay period end |
| basicSalary | DECIMAL(18,2) | Base salary |
| workDays | INT | Days worked |
| overtimeHours | DECIMAL | Regular OT hours |
| overtimeAmount | DECIMAL(18,2) | Regular OT pay |
| holidayOvertimeHours | DECIMAL | Holiday OT hours |
| holidayOvertimeAmount | DECIMAL(18,2) | Holiday OT pay |
| grossPay | DECIMAL(18,2) | Total gross |
| payeAmount | DECIMAL(18,2) | PAYE deducted |
| napsaAmount | DECIMAL(18,2) | NAPSA deducted |
| nhimaAmount | DECIMAL(18,2) | NHIMA deducted |
| totalAllowances | DECIMAL(18,2) | Sum of allowances |
| totalDeductions | DECIMAL(18,2) | Sum of deductions |
| netPay | DECIMAL(18,2) | Take-home pay |
| loanDeductions | DECIMAL(18,2) | Loan repayments |

### `payslip_allowances` / `payslip_deductions` / `payslip_statutory_remittances`

Line-item breakdowns on each payslip linking back to the allowance/deduction/remittance type with the calculated amount.

---

## Loan Tables

### `employee_loans`

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| employeeId | UUID | FK → employees |
| organizationId | TEXT | FK → organizations |
| principalAmount | DECIMAL(18,2) | Loan principal |
| interestRate | DECIMAL(5,2) | Annual interest rate |
| numberOfInstallments | INT | Total installments |
| installmentAmount | DECIMAL(18,2) | Per-installment amount |
| startDate | DATE | Repayment start |
| endDate | DATE | Repayment end |
| status | ENUM | `pending`, `active`, `completed`, `defaulted`, `cancelled` |
| installmentsPaid | INT | Count of installments paid |
| totalPaid | DECIMAL(18,2) | Total repaid |
| remainingBalance | DECIMAL(18,2) | Outstanding balance |

### `loan_repayments`

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| loanId | UUID | FK → employee_loans |
| payrollRunId | UUID | FK → payroll_runs (null if manual) |
| installmentNumber | INT | Sequence number |
| amount | DECIMAL(18,2) | Repayment amount |
| repaymentDate | DATE | When repaid |
| notes | TEXT | Notes |

---

## Regulator Tables

### `napsa_profiles` / `nhima_profiles`

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| organizationId | TEXT | FK → organizations |
| employerNumber | TEXT | Regulator employer registration # |
| registrationDate | DATE | When registered |
| status | ENUM | `active`, `inactive`, `suspended` |
| portalCredentialsEncrypted | TEXT | Encrypted portal login |

### `napsa_employees` / `nhima_employees`

Employee enrollment status per regulator.

### `napsa_returns` / `nhima_returns`

Monthly/quarterly contribution filing records.

### `paye_submissions`

ZRA PAYE monthly filing records.

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| organizationId | TEXT | FK → organizations |
| filingPeriod | TEXT | "2026-05" |
| isNilReturn | BOOLEAN | No employees to report |
| submissionMethod | TEXT | `online`, `manual` |
| status | ENUM | `draft`, `submitted`, `accepted`, `rejected` |
| grossEmoluments | DECIMAL | Total gross |
| chargeableEmoluments | DECIMAL | Taxable amount |
| totalTaxCredit | DECIMAL | Tax credits |
| taxDeducted | DECIMAL | PAYE withheld |
| submittedAt | TIMESTAMP | When submitted to ZRA |
