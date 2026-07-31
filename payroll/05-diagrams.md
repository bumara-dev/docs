---
title: "Payroll Module — Diagrams"
description: "System architecture and process flows for payroll runs, salary calculation, encryption, regulator integration, and PAYE filing."
---

## 1. System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        BROWSER (Next.js App)                        │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────────┐ │
│  │  Payroll      │  │  Employee     │  │  Regulator Dashboards    │ │
│  │  Dashboard    │  │  Management   │  │  (ZRA, NAPSA, NHIMA)     │ │
│  └──────┬───────┘  └──────┬───────┘  └───────────┬───────────────┘ │
│         │                  │                       │                 │
│         └──────────────────┼───────────────────────┘                 │
│                            │  TanStack Query Hooks                   │
└────────────────────────────┼────────────────────────────────────────┘
                             │ HTTPS
                ┌────────────┼────────────┐
                ▼            ▼            ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │api-payroll│  │api-regs  │  │api-jobs  │
        │  (Hono)  │  │  (Hono)  │  │(Cron/BG) │
        └────┬─────┘  └────┬─────┘  └────┬─────┘
             │              │              │
             └──────────────┼──────────────┘
                            │
                    ┌───────┴────────┐
                    │  Service Layer  │
                    │  (api-services) │
                    ├────────────────┤
                    │ • Encryption   │
                    │ • Access Ctrl  │
                    │ • Calculations │
                    │ • Audit Log    │
                    └───────┬────────┘
                            │
                    ┌───────┴────────┐
                    │  Repository    │
                    │  (Drizzle ORM) │
                    └───────┬────────┘
                            │
                    ┌───────┴────────┐
                    │  PostgreSQL    │
                    │  (Encrypted    │
                    │   PII at rest) │
                    └────────────────┘
```

## 2. Payroll Run Execution Flow

```
┌──────────┐
│  Admin    │
│  clicks   │
│ "Process" │
└────┬─────┘
     │
     ▼
┌────────────────────────────────────────────┐
│           VALIDATE INPUTS                   │
│  • Period format "YYYY-MM"                  │
│  • No duplicate run for this period         │
│  • Settings exist                           │
└────┬───────────────────────────────────────┘
     │
     ▼
┌────────────────────────────────────────────┐
│           FETCH DATA (parallel)             │
│  • Active employees                         │
│  • Company allowances & deductions          │
│  • Employee allowances & deductions         │
│  • Monthly inputs for period                │
│  • Active loans                             │
│  • Statutory remittance config              │
└────┬───────────────────────────────────────┘
     │
     ▼
┌────────────────────────────────────────────┐
│     BEGIN TRANSACTION (atomic)              │
│                                             │
│  FOR EACH EMPLOYEE:                         │
│  ┌────────────────────────────────────┐    │
│  │ 1. Calculate daily/hourly rate     │    │
│  │ 2. Sum allowances                  │    │
│  │ 3. Calculate overtime pay          │    │
│  │ 4. Calculate leave pay             │    │
│  │ 5. GROSS = basic + allowances +    │    │
│  │          overtime + leave           │    │
│  │ 6. NHIMA = basic × 1%             │    │
│  │ 7. NAPSA = calculateNAPSA(gross)   │    │
│  │ 8. PAYE = calculatePAYE(gross)     │    │
│  │ 9. Absentism deduction             │    │
│  │ 10. Employee deductions            │    │
│  │ 11. Loan installments              │    │
│  │ 12. NET = gross - all deductions   │    │
│  │ 13. Create payslip record          │    │
│  │ 14. Create loan repayment records  │    │
│  │ 15. Create leave/gratuity entries  │    │
│  └────────────────────────────────────┘    │
│                                             │
│  Create payroll_run with totals             │
│  Status = "draft"                           │
│                                             │
│     COMMIT TRANSACTION                      │
└────┬───────────────────────────────────────┘
     │
     ▼
┌────────────────────────────────────────────┐
│  Return PayrollRunResult                    │
│  { payrollRun, payslips[], totals }         │
└────────────────────────────────────────────┘
```

## 3. Employee Salary Calculation Breakdown

```
┌─────────────────────────────────────────────────────────────────┐
│                    GROSS PAY CALCULATION                         │
│                                                                  │
│  Basic Salary                                   K 8,000.00      │
│  + Housing Allowance                            K 2,500.00      │
│  + Transport Allowance                          K 1,000.00      │
│  + Meal Allowance                               K   500.00      │
│  + Regular Overtime (20hrs × K38.46 × 1.5)      K 1,153.85      │
│  + Holiday Overtime (8hrs × K38.46 × 2.0)        K   615.38      │
│  + Leave Pay                                    K     0.00      │
│  ─────────────────────────────────────────────────────────       │
│  = GROSS PAY                                    K13,769.23      │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                    DEDUCTIONS                                    │
│                                                                  │
│  NHIMA (1% of basic)                            K    80.00      │
│  NAPSA (5.5% of gross, cap K28,938)              K   757.31      │
│  PAYE (tax bands on gross):                                      │
│    Band 1: K0-5,100 @ 0%                       K     0.00      │
│    Band 2: K5,100-7,100 @ 20%                  K   400.00      │
│    Band 3: K7,100-9,200 @ 30% + K400           K 1,030.00      │
│    Band 4: K9,200+ @ 37.5% + K1,030            K 2,743.46      │
│  Total PAYE                                     K 2,743.46      │
│  Absentism (2 days absent)                      K   615.38      │
│  Salary Advance                                 K   500.00      │
│  Loan Installment                               K   833.33      │
│  ─────────────────────────────────────────────────────────       │
│  = TOTAL DEDUCTIONS                             K 5,529.48      │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  NET PAY = K13,769.23 - K5,529.48              = K 8,239.75     │
└─────────────────────────────────────────────────────────────────┘
```

## 4. Payroll Run Status Lifecycle

```
                    ┌──────────┐
                    │  DRAFT   │ ← Payroll calculated
                    └────┬─────┘
                         │ Approve
                         ▼
                    ┌──────────┐
                    │ APPROVED │ ← Manager signs off
                    └────┬─────┘
                         │ Process
                         ▼
                    ┌──────────┐
                    │PROCESSED │ ← Payslips finalized
                    └────┬─────┘
                         │ Mark Paid
                         ▼
                    ┌──────────┐
                    │   PAID   │ ← Bank transfers done
                    └────┬─────┘
                         │ Reconcile
                         ▼
                    ┌──────────────┐
                    │ RECONCILED   │ ← Statement matched
                    └────┬─────────┘
                         │ Lock
                         ▼
                    ┌──────────┐
                    │  LOCKED  │ ← Permanent seal
                    └──────────┘

    * No reverse transitions allowed
    * Each transition records userId + timestamp
    * Locked runs cannot be modified
```

## 5. Employee Data Encryption Flow

```
                    CREATE / UPDATE EMPLOYEE
                    ========================

    Input (plaintext)                    Database (encrypted)
    ─────────────────                    ────────────────────
    nrcNumber: "123456/78/1"    ──►     nrcNumberEncrypted: "aes256:iv:ct:tag"
    tpin: "1234567890"          ──►     tpinEncrypted: "aes256:iv:ct:tag"
    bankAccount: "001234567"    ──►     bankDetailsEncrypted: "aes256:iv:ct:tag"
    email: "john@co.zm"        ──►     emailEncrypted: "aes256:iv:ct:tag"
    phone: "+260977123456"      ──►     phoneEncrypted: "aes256:iv:ct:tag"
    emergencyContact: {...}     ──►     emergencyContactEncrypted: "aes256:iv:ct:tag"

                         encryptEmployeeFields()
                              AES-256-GCM


                    READ EMPLOYEE
                    =============

    Database (encrypted)                 API Response (plaintext)
    ────────────────────                 ────────────────────────
    nrcNumberEncrypted   ──►            nrcNumber: "123456/78/1"
    tpinEncrypted        ──►            tpin: "1234567890"
    bankDetailsEncrypted ──►            bankAccountNumber: "001234567"
                                         bankSortCode: "..."
                                         bankName: "..."

                         decryptEmployeeFields()
                              AES-256-GCM


                    UNAUTHORIZED ACCESS
                    ===================

    Database (encrypted)                 API Response (redacted)
    ────────────────────                 ───────────────────────
    nrcNumberEncrypted   ──►            (field removed)
    tpinEncrypted        ──►            (field removed)
    bankDetailsEncrypted ──►            (field removed)

                         filterEmployeeDataByPermissions()
```

## 6. Regulatory Integration Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      PAYROLL RUN (completed)                     │
│  totalGross | totalPAYE | totalNapsa | totalNhima | payslips[]  │
└──────────┬───────────────────┬────────────────────┬─────────────┘
           │                   │                    │
           ▼                   ▼                    ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│    ZRA PAYE      │ │     NAPSA        │ │     NHIMA        │
│ ──────────────── │ │ ──────────────── │ │ ──────────────── │
│                  │ │                  │ │                  │
│ 1. Create        │ │ 1. Calculate     │ │ 1. Calculate     │
│    Submission    │ │    contributions │ │    contributions │
│                  │ │                  │ │                  │
│ 2. Prefill from  │ │ 2. Generate      │ │ 2. Generate      │
│    Payroll Data  │ │    return file   │ │    return file   │
│                  │ │                  │ │                  │
│ 3. Review per-   │ │ 3. Upload to     │ │ 3. Upload to     │
│    employee      │ │    NAPSA portal  │ │    NHIMA portal  │
│    PAYE details  │ │                  │ │                  │
│                  │ │ Rates:           │ │ Rates:           │
│ 4. Confirm &     │ │  Employee: 5.5%  │ │  Employee: 1%    │
│    Submit        │ │  Employer: 5.5%  │ │  Employer: 1%    │
│                  │ │  Ceiling: K28,938│ │                  │
│ Due: 10th of     │ │                  │ │                  │
│ following month  │ │ Due: Monthly     │ │ Due: Monthly     │
└──────────────────┘ └──────────────────┘ └──────────────────┘
```

## 7. PAYE Filing Wizard Flow

```
┌────────────────┐     ┌────────────────┐     ┌────────────────┐
│   STEP 1       │     │   STEP 2       │     │   STEP 3       │
│ Filing Setup   │────►│ Payroll Summary│────►│ Employee       │
│                │     │                │     │ Details        │
│ • Period       │     │ • Total gross  │     │ • TPIN         │
│ • Nil return?  │     │ • Chargeable   │     │ • Gross        │
│ • Submission   │     │ • Tax credits  │     │ • Chargeable   │
│   method       │     │ • Tax deducted │     │ • Tax credit   │
│                │     │                │     │ • PAYE deducted│
└────────────────┘     └────────────────┘     └───────┬────────┘
                                                       │
                                                       ▼
                                              ┌────────────────┐
                                              │   STEP 4       │
                                              │ Confirm &      │
                                              │ Submit         │
                                              │                │
                                              │ Status:        │
                                              │ draft →        │
                                              │ submitted →    │
                                              │ accepted /     │
                                              │ rejected       │
                                              └────────────────┘
```

## 8. Monthly Payroll Process Flow

```
 Day 1                     Cutoff Day              Payment Day        10th Next Month
  │                            │                       │                    │
  ▼                            ▼                       ▼                    ▼
┌──────┐  ┌──────────────┐  ┌──────────┐  ┌────────┐  ┌──────┐  ┌────────────────┐
│Month │  │Submit Monthly │  │Preview & │  │Approve │  │Pay   │  │File Regulatory │
│Starts│─►│Inputs:       │─►│Process   │─►│Payroll │─►│Staff │─►│Returns:        │
│      │  │• Attendance  │  │Payroll   │  │Run     │  │      │  │• PAYE (ZRA)    │
│      │  │• Overtime    │  │          │  │        │  │      │  │• NAPSA         │
│      │  │• Advances    │  │          │  │        │  │      │  │• NHIMA         │
│      │  │• Leave       │  │          │  │        │  │      │  │                │
└──────┘  └──────────────┘  └──────────┘  └────────┘  └──────┘  └────────────────┘

  Example timeline with settings: cutoff=20th, payment=25th
  ────────────────────────────────────────────────────────────
  May 1          May 20           May 25           Jun 10
  Month starts   Inputs due       Salaries paid    PAYE/NAPSA/NHIMA due
```

## 9. Access Control Decision Flow

```
                    ┌──────────────────┐
                    │ API Request for  │
                    │ Employee Data    │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │ Authenticate     │
                    │ (Clerk)          │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │ Get User Role    │──── hr_manager / payroll_officer /
                    │ & Permissions    │     finance_manager / dept_manager /
                    └────────┬─────────┘     employee / admin
                             │
                    ┌────────▼─────────┐     Yes ┌────────────────────┐
                    │ Accessing own    │────────►│ Return own salary  │
                    │ data?            │         │ details only       │
                    └────────┬─────────┘         └────────────────────┘
                             │ No
                    ┌────────▼─────────┐
                    │ Decrypt all      │
                    │ employee fields  │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────────────┐
                    │ filterEmployeeData       │
                    │ ByPermissions()          │
                    │                          │
                    │ Remove unauthorized      │
                    │ fields based on role:    │
                    │ • ID numbers?            │
                    │ • Bank details?          │
                    │ • Salary info?           │
                    │ • Contact info?          │
                    └────────┬─────────────────┘
                             │
                    ┌────────▼─────────┐
                    │ Log sensitive    │
                    │ data access      │
                    │ (audit trail)    │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │ Return filtered  │
                    │ employee data    │
                    └──────────────────┘
```

## 10. Loan Lifecycle & Payroll Deduction

```
    ┌──────────────┐
    │ Create Loan  │
    │ K10,000      │
    │ 12 months    │
    │ 10% interest │
    └──────┬───────┘
           │
    ┌──────▼───────┐
    │   PENDING    │ ← Awaiting approval
    └──────┬───────┘
           │ Approve
    ┌──────▼───────┐
    │   ACTIVE     │ ← Deductions begin
    └──────┬───────┘
           │
           │  Each Payroll Run:
           │  ┌──────────────────────────────────────┐
           │  │ installmentsPaid < numberOfInstalls?  │
           │  │     YES → Deduct installmentAmount    │
           │  │            Create loan_repayment      │
           │  │            installmentsPaid += 1       │
           │  │            remainingBalance -= amount  │
           │  │     NO  → Skip                        │
           │  └──────────────────────────────────────┘
           │
    ┌──────▼───────┐
    │  COMPLETED   │ ← All installments paid
    └──────────────┘

    Alternative endings:
    ┌──────────────┐     ┌──────────────┐
    │  DEFAULTED   │     │  CANCELLED   │
    │ (missed      │     │ (written     │
    │  payments)   │     │  off)        │
    └──────────────┘     └──────────────┘
```

## 11. Data Flow: Payroll to Regulators

```
┌─────────────────────────────────────────────────────────────────────┐
│                        PAYROLL ENGINE                                │
│                                                                      │
│  Employee │ Gross    │ PAYE     │ NAPSA   │ NHIMA  │ Net Pay        │
│  ─────────┼──────────┼──────────┼─────────┼────────┼────────        │
│  EMP-001  │ 13,769   │ 2,743    │ 757     │ 80     │ 8,240         │
│  EMP-002  │  9,500   │ 1,143    │ 523     │ 70     │ 6,512         │
│  EMP-003  │  6,800   │   340    │ 374     │ 55     │ 5,231         │
│  ...      │ ...      │ ...      │ ...     │ ...    │ ...           │
│  ─────────┼──────────┼──────────┼─────────┼────────┼────────        │
│  TOTALS   │450,000   │ 67,500   │ 24,750  │ 4,500  │325,000        │
└──────┬──────────────────┬────────────────────┬──────────────────────┘
       │                  │                    │
       ▼                  ▼                    ▼
┌──────────────┐  ┌──────────────┐    ┌──────────────┐
│ ZRA PAYE     │  │ NAPSA Return │    │ NHIMA Return │
│ Return File  │  │ File         │    │ File         │
│              │  │              │    │              │
│ Per employee:│  │ Per employee:│    │ Per employee:│
│ • TPIN       │  │ • NAPSA #   │    │ • NHIMA #   │
│ • Gross      │  │ • Gross     │    │ • Basic     │
│ • Chargeable │  │ • Emp 5.5%  │    │ • Emp 1%    │
│ • Tax credit │  │ • Emr 5.5%  │    │ • Emr 1%    │
│ • PAYE       │  │ • Total     │    │ • Total     │
└──────────────┘  └──────────────┘    └──────────────┘
       │                  │                    │
       ▼                  ▼                    ▼
┌──────────────┐  ┌──────────────┐    ┌──────────────┐
│ ZRA Portal   │  │ NAPSA Portal │    │ NHIMA Portal │
│ (online/     │  │ (upload)     │    │ (upload)     │
│  manual)     │  │              │    │              │
│ Due: 10th    │  │ Due: Monthly │    │ Due: Monthly │
└──────────────┘  └──────────────┘    └──────────────┘
```

## 12. Setup Completion Checklist Flow

```
    ┌───────────────────────────────┐
    │     PAYROLL SETUP WIZARD      │
    │                               │
    │  ☐ 1. Configure Settings      │───► Work days, payment day, cutoff,
    │       (Required)              │     leave rules, loan limits, OT rates
    │                               │
    │  ☐ 2. Create Departments      │───► Company organizational structure
    │       (Required)              │
    │                               │
    │  ☐ 3. Create Designations     │───► Job titles / positions
    │       (Required)              │
    │                               │
    │  ☐ 4. Setup Allowances &      │───► Auto-creates 4 system allowances
    │       Deductions (Required)   │     + 1 system deduction on settings save
    │                               │
    │  ☐ 5. Add Employees           │───► Individual or bulk CSV import
    │       (Required)              │
    │                               │
    │  ☐ 6. Register NAPSA          │───► Employer number + portal credentials
    │       (Recommended)           │
    │                               │
    │  ☐ 7. Register NHIMA          │───► Employer number + portal credentials
    │       (Recommended)           │
    │                               │
    │  ☐ 8. Configure PAYE Bands    │───► Set up ZRA tax brackets
    │       (Recommended)           │
    │                               │
    │  All required steps done?     │
    │      YES → Dashboard unlocked │
    │      NO  → Setup wizard shown │
    └───────────────────────────────┘
```
