---
title: "Payroll Module — API Reference"
description: "All endpoints require authentication (requireAuth) and organization context (requireOrg). Base path: /payroll (api-payroll worker) and /paye..."
---

All endpoints require authentication (`requireAuth`) and organization context (`requireOrg`).
Base path: `/payroll` (api-payroll worker) and `/paye` (api-regulators worker).

---

## Payroll Settings & Setup

| # | Method | Path | Description |
|---|--------|------|-------------|
| 1 | GET | `/payroll/setup` | Get setup completion status (settings, allowances, employees, departments, designations) |
| 2 | GET | `/payroll/settings` | Get active payroll settings for organization |
| 3 | POST | `/payroll/settings` | Create payroll settings (work days, payment day, cutoff, leave rules, loan config, overtime multipliers) |
| 4 | PUT | `/payroll/settings` | Update payroll settings |

### POST `/payroll/settings` — Request Body
```json
{
  "workDays": 26,
  "paymentDayOfMonth": 25,
  "cutoffDayOfMonth": 20,
  "leaveCalculationBasis": "monthly_accrual",
  "leaveCarryoverPercentage": "50.00",
  "leaveCarryoverMaxDays": "10.00",
  "maxDeductionPercentage": "50.00",
  "interestCalculationMethod": "simple",
  "maxLoanAmount": "50000.00",
  "maxLoanTerm": 24,
  "requiresCollateral": false,
  "overtimeMultiplier": "1.50",
  "holidayOvertimeMultiplier": "2.00",
  "currency": "ZMW",
  "roundingMethod": "half_up"
}
```

---

## Dashboard

| # | Method | Path | Description |
|---|--------|------|-------------|
| 5 | GET | `/payroll/dashboard` | Aggregated dashboard data (KPIs, recent runs, setup status) |

---

## Allowances

| # | Method | Path | Description |
|---|--------|------|-------------|
| 6 | GET | `/payroll/allowances` | List company-wide allowance types |
| 7 | POST | `/payroll/allowances` | Create allowance type (Housing, Transport, Meal, etc.) |
| 8 | PUT | `/payroll/allowances/{id}` | Update allowance type |
| 9 | DELETE | `/payroll/allowances/{id}` | Delete allowance type |

### POST `/payroll/allowances` — Request Body
```json
{
  "name": "Housing Allowance",
  "code": "HOUSING",
  "calculationType": "fixed",
  "payType": "monthly",
  "amount": "2500.00",
  "isTaxable": true,
  "isProRata": true,
  "isAutoCalculated": false
}
```

---

## Deductions

| # | Method | Path | Description |
|---|--------|------|-------------|
| 10 | GET | `/payroll/deductions` | List company-wide deduction types |
| 11 | POST | `/payroll/deductions` | Create deduction type |
| 12 | PUT | `/payroll/deductions/{id}` | Update deduction type |
| 13 | DELETE | `/payroll/deductions/{id}` | Delete deduction type |

### POST `/payroll/deductions` — Request Body
```json
{
  "name": "Salary Advance",
  "code": "ADVANCE",
  "calculationType": "fixed",
  "calculationBase": "gross",
  "payType": "monthly",
  "amount": "0.00",
  "isTaxable": false,
  "employeeContributionRate": "100.00",
  "employerContributionRate": "0.00"
}
```

---

## Statutory Remittances

| # | Method | Path | Description |
|---|--------|------|-------------|
| 14 | GET | `/payroll/statutory-remittances` | List statutory remittance types (PAYE, NAPSA, NHIMA) |
| 15 | POST | `/payroll/statutory-remittances` | Create statutory remittance type with percentage or tax band data |
| 16 | GET | `/payroll/statutory-remittances/{id}` | Get single remittance type with nested config |
| 17 | PUT | `/payroll/statutory-remittances/{id}` | Update remittance type and config |
| 18 | DELETE | `/payroll/statutory-remittances/{id}` | Delete remittance type |

### POST `/payroll/statutory-remittances` — Request Body (Tax Band type)
```json
{
  "code": "PAYE",
  "name": "Pay As You Earn",
  "type": "tax_bands",
  "category": "statutory",
  "frequency": "monthly",
  "calculationBase": "gross",
  "authority": "ZRA",
  "country": "ZM",
  "taxBands": [
    { "bandNumber": 1, "minimumIncome": "0.00", "maximumIncome": "5100.00", "taxRate": "0.00", "fixedAmount": "0.00" },
    { "bandNumber": 2, "minimumIncome": "5100.01", "maximumIncome": "7100.00", "taxRate": "20.00", "fixedAmount": "0.00" },
    { "bandNumber": 3, "minimumIncome": "7100.01", "maximumIncome": "9200.00", "taxRate": "30.00", "fixedAmount": "400.00" },
    { "bandNumber": 4, "minimumIncome": "9200.01", "maximumIncome": "999999999.00", "taxRate": "37.50", "fixedAmount": "1030.00" }
  ]
}
```

### POST `/payroll/statutory-remittances` — Request Body (Percentage type)
```json
{
  "code": "NAPSA",
  "name": "National Pension",
  "type": "percentage",
  "category": "statutory",
  "frequency": "monthly",
  "calculationBase": "gross",
  "authority": "NAPSA",
  "country": "ZM",
  "percentageData": {
    "employeeContributionRate": "5.50",
    "employerContributionRate": "5.50",
    "ceilingAmount": "28938.00",
    "thresholdAmount": "0.00"
  }
}
```

---

## Employees

| # | Method | Path | Description |
|---|--------|------|-------------|
| 19 | GET | `/payroll/employees` | List employees (filter: status, department, branch, search, pagination) |
| 20 | POST | `/payroll/employees` | Create employee (auto-generates EMP-XXXX number, encrypts PII) |
| 21 | GET | `/payroll/employees/{id}` | Get employee detail (decrypts PII, filters by role permissions) |
| 22 | PUT | `/payroll/employees/{id}` | Update employee (re-encrypts changed fields) |
| 23 | DELETE | `/payroll/employees/{id}` | Soft-delete employee |
| 24 | POST | `/payroll/bulk-employees` | Bulk create employees |
| 25 | GET | `/payroll/employees/template/download` | Download employee import CSV template |
| 26 | POST | `/payroll/employees/bulk/upload` | Upload bulk employee CSV |

### Query Parameters for GET `/payroll/employees`
| Param | Type | Description |
|-------|------|-------------|
| `status` | enum | `active`, `on_leave`, `suspended`, `terminated` |
| `isActive` | string | `"true"` or `"false"` |
| `departmentId` | uuid | Filter by department |
| `branchId` | uuid | Filter by branch |
| `search` | string | Free-text search (name, employee number) |
| `limit` | number | Page size (1-100, default 20) |
| `page` | number | Page number (default 1) |

---

## Employee Allowances & Deductions

| # | Method | Path | Description |
|---|--------|------|-------------|
| 27 | GET | `/payroll/employees/:employeeId/allowances` | List allowances assigned to employee |
| 28 | POST | `/payroll/employees/:employeeId/allowances` | Assign allowance to employee with amount + effective dates |
| 29 | GET | `/payroll/employees/:employeeId/deductions` | List deductions assigned to employee |
| 30 | POST | `/payroll/employees/:employeeId/deductions` | Assign deduction to employee with frequency |

---

## Payroll Runs

| # | Method | Path | Description |
|---|--------|------|-------------|
| 31 | GET | `/payroll/runs` | List payroll runs (filter by status) |
| 32 | POST | `/payroll/runs/preview` | Preview payroll calculation without committing |
| 33 | POST | `/payroll/runs/process` | Execute payroll run for period (atomic transaction) |
| 34 | GET | `/payroll/runs/:payrollRunId` | Get payroll run detail |

### POST `/payroll/runs/process` — Request Body
```json
{
  "period": "2026-05",
  "workDaysOverride": 22
}
```

### Response (PayrollRunResult)
```json
{
  "payrollRun": {
    "id": "uuid",
    "periodStart": "2026-05-01",
    "periodEnd": "2026-05-31",
    "totalEmployees": "15",
    "totalGross": "450000.00",
    "totalAllowances": "75000.00",
    "totalDeductions": "125000.00",
    "totalNetPay": "325000.00",
    "totalPAYE": "67500.00",
    "totalNapsa": "24750.00",
    "totalNhima": "4500.00",
    "totalLoansDeducted": "8000.00",
    "status": "draft"
  }
}
```

---

## Payslips

| # | Method | Path | Description |
|---|--------|------|-------------|
| 35 | GET | `/payroll/runs/:payrollRunId/payslips` | List payslips for a payroll run |
| 36 | GET | `/payroll/employees/:employeeId/payslips` | List payslips for an employee |
| 37 | GET | `/payroll/payslips/:payslipId/download` | Download payslip as PDF |
| 38 | GET | `/payroll/employees/:employeeId/payslips/download-bulk` | Bulk download employee payslips |
| 39 | GET | `/payroll/runs/:payrollRunId/payslips/download-bulk` | Bulk download all payslips for a run |
| 40 | POST | `/payroll/payslips/{id}/email` | Email individual payslip to employee |
| 41 | POST | `/payroll/payslips/email-bulk` | Bulk email payslips for a run |
| 42 | POST | `/payroll/runs/{id}/email` | Email entire payroll run summary |

---

## Loans

| # | Method | Path | Description |
|---|--------|------|-------------|
| 43 | GET | `/payroll/loans` | List all loans (filter by status) |
| 44 | GET | `/payroll/employees/:employeeId/loans` | List loans for an employee |
| 45 | GET | `/payroll/loans/:loanId` | Get loan detail |
| 46 | POST | `/payroll/employees/:employeeId/loans` | Create loan for employee |
| 47 | PUT | `/payroll/loans/:loanId` | Update loan |
| 48 | DELETE | `/payroll/loans/:loanId` | Delete loan |

---

## Loan Repayments

| # | Method | Path | Description |
|---|--------|------|-------------|
| 49 | GET | `/payroll/loans/:loanId/repayments` | List repayments for a loan |
| 50 | POST | `/payroll/loans/:loanId/repayments` | Record manual repayment |
| 51 | GET | `/payroll/employees/:employeeId/repayments` | List all repayments for employee |
| 52 | GET | `/payroll/loans/:loanId/repayment-summary` | Loan repayment summary (paid, remaining, schedule) |
| 53 | GET | `/payroll/repayments/:repaymentId` | Get single repayment detail |
| 54 | PUT | `/payroll/repayments/:repaymentId` | Update repayment |
| 55 | DELETE | `/payroll/repayments/:repaymentId` | Delete repayment |

---

## Regulatory Returns Export

| # | Method | Path | Description |
|---|--------|------|-------------|
| 56 | GET | `/payroll/runs/:payrollRunId/returns/napsa/download` | Download NAPSA return file for a payroll run |
| 57 | GET | `/payroll/runs/:payrollRunId/returns/nhima/download` | Download NHIMA return file for a payroll run |
| 58 | GET | `/payroll/runs/:payrollRunId/returns/paye/download` | Download PAYE return file for a payroll run |
| 59 | GET | `/payroll/runs/:payrollRunId/export-payroll` | Export complete payroll data (Excel/CSV) |

---

## Departments & Designations

| # | Method | Path | Description |
|---|--------|------|-------------|
| 60 | GET | `/payroll/departments` | List departments |
| 61 | POST | `/payroll/departments` | Create department |
| 62 | PUT | `/payroll/departments/{id}` | Update department |
| 63 | DELETE | `/payroll/departments/{id}` | Delete department |
| 64 | GET | `/payroll/designations` | List job designations/positions |
| 65 | POST | `/payroll/designations` | Create designation |
| 66 | PUT | `/payroll/designations/{id}` | Update designation |
| 67 | DELETE | `/payroll/designations/{id}` | Delete designation |

---

## Monthly Inputs

| # | Method | Path | Description |
|---|--------|------|-------------|
| 68 | GET | `/payroll/monthly-inputs` | List monthly attendance/overtime inputs |
| 69 | GET | `/payroll/monthly-inputs/:monthlyInputId/template/download` | Download monthly input template |
| 70 | POST | `/payroll/monthly-inputs/:monthlyInputId/upload` | Upload monthly input data |
| 71 | POST | `/payroll/monthly-inputs/{monthlyInputId}/batch-upload` | Batch upload monthly inputs |

---

## ZRA PAYE Endpoints (api-regulators worker)

| # | Method | Path | Description |
|---|--------|------|-------------|
| 72 | GET | `/paye/submissions` | List PAYE submissions |
| 73 | POST | `/paye/submissions` | Create PAYE submission (filing period, nil return flag) |
| 74 | GET | `/paye/submissions/{id}` | Get PAYE submission detail |
| 75 | PATCH | `/paye/submissions/{id}/status` | Update submission status (submitted/accepted/rejected) |
| 76 | POST | `/paye/submissions/{id}/employees` | Save employee PAYE details |
| 77 | GET | `/paye/submissions/{submissionId}/employees/{employeeId}` | Get employee PAYE record |
| 78 | PUT | `/paye/submissions/{submissionId}/employees/{employeeId}` | Update employee PAYE record |
| 79 | POST | `/paye/prefill-data` | Pre-fill PAYE data from payroll run |
| 80 | GET | `/paye/by-filing/{filingId}` | Get PAYE submission by filing ID |
| 81 | POST | `/paye/submissions/{id}/payroll` | Save payroll totals to submission |
| 82 | POST | `/paye/submissions/{id}/employees` | Save employee list to submission |
| 83 | POST | `/paye/submissions/{id}/confirm` | Confirm and finalize PAYE submission |

**Total: 83 endpoints**

---

## Common Response Envelope

All responses follow:
```json
{
  "success": true,
  "data": { ... },
  "meta": {
    "total": 150,
    "page": 1,
    "limit": 20,
    "totalPages": 8
  }
}
```

Error responses:
```json
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "Employee not found"
  }
}
```
