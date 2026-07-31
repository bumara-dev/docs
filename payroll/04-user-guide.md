---
title: "Payroll Module — User Guide"
description: "The Payroll module handles everything related to paying your employees: setting up compensation structures, tracking attendance, calculating salaries..."
---

## What This Module Does

The Payroll module handles everything related to paying your employees: setting up compensation structures, tracking attendance, calculating salaries with Zambian statutory deductions (PAYE, NAPSA, NHIMA), generating payslips, managing loans, and filing regulatory returns.

---

## Getting Started

### First-Time Setup (5 steps)

The dashboard shows a setup checklist when you first access Payroll. Complete these in order:

1. **Configure Payroll Settings**
   - Set working days per month (default: 26)
   - Set payment day (e.g., 25th)
   - Set cutoff day for monthly inputs (e.g., 20th)
   - Configure overtime multipliers (1.5x regular, 2.0x holiday)
   - Set leave and loan policies

2. **Create Departments**
   - Add your company's departments (Finance, Operations, HR, etc.)
   - Each employee will be assigned to a department

3. **Create Job Positions**
   - Add designations/titles (Accountant, Manager, Driver, etc.)

4. **Set Up Allowances & Deductions**
   - System auto-creates: Housing, Transport, Meal, Leave Pay allowances + Advance deduction
   - Add custom allowances (e.g., Fuel, Communication)
   - Add custom deductions (e.g., Uniform, Social Fund)

5. **Add Employees**
   - Enter employee details (name, NRC, TPIN, bank details)
   - All sensitive data is encrypted at rest
   - System generates employee numbers (EMP-0001, EMP-0002, ...)
   - Assign salary, department, position, allowances, deductions

### Registering with Regulators

Before running payroll, register your organization with:

- **NAPSA**: Enter your NAPSA employer number and portal credentials
- **NHIMA**: Enter your NHIMA employer number and portal credentials
- **ZRA**: Your TPIN is already in organization settings

Each regulator page has a registration form that guides you through the process.

---

## Monthly Payroll Process

### Step 1: Submit Monthly Inputs (by cutoff day)

Navigate to **Payroll → Monthly Inputs** and enter for each employee:

| Field | Description |
|-------|-------------|
| Days Absent | Working days missed (unpaid leave, unauthorized absence) |
| Overtime Hours | Regular overtime hours worked |
| Holiday Overtime | Holiday/weekend overtime hours |
| Leave Days Pay | Leave days to be compensated |
| Advance | Salary advance requested |
| Other Deductions | Any ad-hoc deductions |

You can upload inputs in bulk using the CSV template.

### Step 2: Preview Payroll

Click **Preview Payroll** to see calculations before committing:
- Verify gross pay per employee
- Check statutory deduction amounts
- Review loan installments being deducted
- Confirm net pay figures

No data is saved during preview.

### Step 3: Run Payroll

Click **Process Payroll** with the period (e.g., "2026-05"). The system:

1. Calculates daily rate = salary / working days
2. Adds allowances (Housing, Transport, etc.)
3. Adds overtime pay (hours × rate × multiplier)
4. Calculates statutory deductions:
   - **NHIMA**: 1% of basic salary
   - **NAPSA**: 5.5% of gross (with ceiling)
   - **PAYE**: Progressive tax bands applied to gross
5. Applies employee deductions (advances, loan installments, etc.)
6. Deducts for absent days
7. Generates payslips for every employee

The payroll run starts in **Draft** status.

### Step 4: Approve & Process

| Status | Action | Who |
|--------|--------|-----|
| Draft | Review calculations | Payroll Officer |
| Approved | Authorized for payment | HR Manager / Finance |
| Processed | Payslips finalized | System |
| Paid | Bank transfers complete | Finance |
| Reconciled | Bank statement matched | Finance |
| Locked | Permanently sealed | System |

### Step 5: Distribute Payslips

From the payroll run detail page:
- **Email individual**: Click email icon on a payslip row
- **Email all**: Click "Email All Payslips" to send to every employee
- **Download PDF**: Download individual or bulk payslip PDFs

---

## Employee Management

### Adding an Employee

Navigate to **Payroll → Employees → Add Employee** and fill in:

**Personal Information:**
- Name, date of birth, gender, marital status
- NRC number, passport number

**Contact:**
- Email, phone, alternative phone, address
- Emergency contact (name, phone, relationship)

**Employment:**
- Department, designation, branch
- Employment type (full-time, part-time, contract, casual)
- Hire date, probation end date
- Basic salary

**Regulatory IDs:**
- TPIN (for ZRA PAYE)
- NAPSA number
- NHIMA number

**Banking:**
- Bank name, branch, account number, sort code, SWIFT code

All sensitive fields (NRC, TPIN, bank details, etc.) are encrypted before storage.

### Bulk Import

1. Download the CSV template from **Employees → Template Download**
2. Fill in employee data
3. Upload via **Employees → Bulk Upload**

### Assigning Allowances & Deductions

From an employee's profile:
1. Go to the **Allowances** tab
2. Click **Add Allowance** → select type, enter amount, set effective dates
3. Go to the **Deductions** tab
4. Click **Add Deduction** → select type, enter amount, set frequency

---

## Loan Management

### Creating a Loan

Navigate to **Payroll → Loans → Create Loan**:

| Field | Description |
|-------|-------------|
| Employee | Select employee |
| Principal Amount | Loan amount |
| Interest Rate | Annual interest rate (%) |
| Number of Installments | How many payments |
| Start Date | First repayment date |
| End Date | Final repayment date |

The system calculates the installment amount and automatically deducts from payroll each month.

### Loan Lifecycle

```
Pending → Active → Completed
                 → Defaulted
          → Cancelled
```

- **Active** loans have installments deducted during payroll runs
- When all installments are paid, status changes to **Completed**
- View repayment history and remaining balance on the loan detail page

---

## Regulatory Filing

### ZRA PAYE Returns

Navigate to **ZRA → PAYE Filing**:

1. **Create Submission**: Select filing period (month), choose normal or nil return
2. **Pre-fill from Payroll**: System populates employee data from the completed payroll run
3. **Review Employee Data**: Verify per-employee gross emoluments, chargeable emoluments, tax credits, PAYE deducted
4. **Confirm & Submit**: Finalize the return

**Key fields per employee:**
- TPIN
- Gross Emoluments (total earnings)
- Chargeable Emoluments (taxable portion)
- Total Tax Credit
- PAYE Tax Deducted

PAYE returns are due by the **10th of the following month**.

### NAPSA Returns

Navigate to **NAPSA → Returns**:

1. System calculates contributions from payroll data:
   - Employee contribution: 5.5% of gross (up to ceiling)
   - Employer contribution: 5.5% of gross (up to ceiling)
2. Download the NAPSA return file
3. Upload to NAPSA portal

### NHIMA Returns

Navigate to **NHIMA → Returns**:

1. System calculates contributions from payroll data:
   - Employee contribution: 1% of basic salary
   - Employer contribution: 1% of basic salary
2. Download the NHIMA return file
3. Upload to NHIMA portal

### Downloading Return Files

From any payroll run detail page, click:
- **Download NAPSA Return** — generates NAPSA-formatted file
- **Download NHIMA Return** — generates NHIMA-formatted file
- **Download PAYE Return** — generates ZRA PAYE-formatted file
- **Export Payroll** — complete payroll data export (Excel/CSV)

---

## Reports & Analytics

### Dashboard KPIs

The payroll dashboard displays:
- **Active Employees**: Current headcount
- **Total Gross Pay**: Current month's gross
- **Total Net Pay**: Current month's net
- **Next Payrun Date**: Countdown to next payroll
- **Payroll Cost Trend**: 6-month gross pay chart
- **Status Distribution**: Draft/Approved/Paid breakdown

### Available Reports

| Report | Description |
|--------|-------------|
| Payroll Summary | Totals per run (gross, deductions, net) |
| Employee Earnings | Per-employee breakdown |
| Statutory Deductions | PAYE, NAPSA, NHIMA per employee |
| Loan Report | Outstanding loans and repayment status |
| Department Summary | Payroll cost by department |

---

## Access Control

Different roles see different data:

| What you can see | HR Manager | Payroll Officer | Finance | Dept Manager | Employee |
|-----------------|------------|-----------------|---------|--------------|----------|
| Employee names & departments | Yes | Yes | Yes | Own dept | Own only |
| NRC, Passport, TPIN | Yes | Yes | No | No | No |
| Bank details | Yes | Yes | Yes | No | No |
| Salary & payslips | Yes | Yes | Yes | Own dept | Own only |
| Export data | Yes | Yes | Yes | No | No |
| Run payroll | Yes | Yes | No | No | No |
| File regulatory returns | No | No | Yes | No | No |

---

## Tips

- **Cutoff Day**: Ensure all monthly inputs are submitted before the cutoff day. Late inputs require re-running payroll.
- **Preview First**: Always preview before processing. Once processed, a payroll run cannot be easily reversed.
- **Regulatory Deadlines**: PAYE is due by 10th of following month. NAPSA and NHIMA are due monthly.
- **Loan Limits**: Loans respect the max amount and max term configured in settings. Total deductions cannot exceed the max deduction percentage (default 50%) of gross.
- **Bulk Operations**: Use CSV templates for large employee imports and monthly input submissions to save time.
