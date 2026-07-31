---
title: "Payroll Module — Service Inventory"
description: "Every payroll service-layer function by slice, with its parameters, return type, and behavior."
---

## 1. Service Layer Functions

The functions below live in `packages/payroll/src/<slice>/service.ts`, one slice per
grouping in this section (settings, allowances, deductions, statutory remittances,
employees, and so on). Each slice also carries its own `schema.ts`, `queries.ts`, and
colocated `*.test.ts`.

**Payroll Settings:**

| Function | Params | Returns | Description |
|----------|--------|---------|-------------|
| `getPayrollSetupService` | ctx, deps | SetupStatus | Check completion of settings, allowances, employees, departments, designations |
| `getPayrollSettings` | ctx, deps | PayrollSettings | Fetch active org settings |
| `createPayrollSettingsService` | ctx, deps, input | PayrollSettings | Create settings + auto-initialize system allowances/deductions |
| `updatePayrollSettingsService` | ctx, deps, input | PayrollSettings | Update settings with audit log |

**Allowances:**

| Function | Params | Returns | Description |
|----------|--------|---------|-------------|
| `getAllowances` | ctx, deps | Allowance[] | List org allowances |
| `createAllowanceService` | ctx, deps, input | Allowance | Create allowance type |
| `updateAllowanceService` | ctx, deps, id, input | Allowance | Update allowance type |
| `deleteAllowanceService` | ctx, deps, id | void | Delete (blocked for system-generated) |

**Deductions:**

| Function | Params | Returns | Description |
|----------|--------|---------|-------------|
| `getDeductions` | ctx, deps | Deduction[] | List org deductions |
| `createDeductionService` | ctx, deps, input | Deduction | Create deduction type |
| `updateDeductionService` | ctx, deps, id, input | Deduction | Update deduction type |
| `deleteDeductionService` | ctx, deps, id | void | Delete (blocked for system-generated) |

**Statutory Remittances:**

| Function | Params | Returns | Description |
|----------|--------|---------|-------------|
| `getStatutoryRemittanceTypes` | ctx, deps | RemittanceType[] | List PAYE/NAPSA/NHIMA types |
| `createStatutoryRemittanceTypeService` | ctx, deps, input | RemittanceType | Create with nested percentage or tax band data |
| `updateStatutoryRemittanceTypeService` | ctx, deps, id, input | RemittanceType | Update type + nested config |
| `deleteStatutoryRemittanceTypeService` | ctx, deps, id | void | Delete type + cascaded config |
| `getTaxBands` | ctx, deps, remittanceTypeId | TaxBand[] | Get progressive tax brackets |
| `createTaxBandService` | ctx, deps, input | TaxBand | Create tax bracket |
| `getPercentageRemittanceByTypeId` | ctx, deps, typeId | PercentageRemittance | Get contribution rates |
| `createPercentageRemittanceService` | ctx, deps, input | PercentageRemittance | Create rates |
| `updatePercentageRemittanceService` | ctx, deps, id, input | PercentageRemittance | Update rates |

**Employee Management:**

| Function | Params | Returns | Description |
|----------|--------|---------|-------------|
| `createEmployeeService` | ctx, deps, input | Employee | Create with auto-number (EMP-XXXX), encrypt PII |
| `getEmployeeService` | ctx, deps, id | Employee | Get with decrypted PII, filtered by role |
| `listEmployeesService` | ctx, deps, filters | &#123;data, meta&#125; | Paginated list with decryption |
| `updateEmployeeService` | ctx, deps, id, input | Employee | Re-encrypt changed fields |
| `activateEmployeeService` | ctx, deps, id | Employee | Set status to active |
| `deleteEmployeeService` | ctx, deps, id | void | Soft delete |
| `listEmployeesByDepartmentService` | ctx, deps, deptId | Employee[] | Filter by department |
| `listEmployeesByBranchService` | ctx, deps, branchId | Employee[] | Filter by branch |
| `getEmployeeStatsService` | ctx, deps | &#123;active, total&#125; | Employee counts |

**Employee Allowances & Deductions:**

| Function | Params | Returns | Description |
|----------|--------|---------|-------------|
| `createEmployeeAllowanceService` | ctx, deps, input | EmployeeAllowance | Assign allowance to employee |
| `listEmployeeAllowancesService` | ctx, deps, employeeId | EmployeeAllowance[] | List with parent allowance details |
| `updateEmployeeAllowanceService` | ctx, deps, id, input | EmployeeAllowance | Update assignment |
| `createEmployeeDeductionService` | ctx, deps, input | EmployeeDeduction | Assign deduction to employee |
| `listEmployeeDeductionsService` | ctx, deps, employeeId | EmployeeDeduction[] | List with parent deduction details |
| `updateEmployeeDeductionService` | ctx, deps, id, input | EmployeeDeduction | Update assignment |

**Payroll Execution:**

| Function | Params | Returns | Description |
|----------|--------|---------|-------------|
| `runPayrunService` | ctx, deps, period, workDaysOverride? | PayrollRunResult | **Core**: atomic payroll calculation + payslip generation |
| `listPayrollRunsService` | ctx, deps, filters | PayrollRun[] | List runs with status filter |
| `getPayrollRunService` | ctx, deps, id | PayrollRun | Get single run detail |
| `listPendingPayrollsService` | ctx, deps | PendingOrg[] | Orgs with payroll due |

**Insights & Analytics:**

| Function | Params | Returns | Description |
|----------|--------|---------|-------------|
| `getPayrollStatsService` | ctx, deps | Stats | Aggregate stats across orgs |
| `getPayrollStatusDistributionService` | ctx, deps | Distribution | Status breakdown counts |
| `getPayrollRunsTimeseriesService` | ctx, deps, months? | Timeseries | Historical trend data |
| `getPayrollOrgLeaderboardService` | ctx, deps, limit? | Leaderboard | Top orgs by volume |
| `getPayrollCurrentPeriodSummaryService` | ctx, deps | Summary | Current month summary |

**Departments & Designations:**

| Function | Params | Returns | Description |
|----------|--------|---------|-------------|
| `listDepartmentsService` | ctx, deps | Department[] | List org departments |
| `listDesignationService` | ctx, deps | Designation[] | List job designations |

### `payroll-insights.service.ts`

| Function | Params | Returns | Description |
|----------|--------|---------|-------------|
| `getPayrollInsightsService` | ctx, deps | PayrollInsightsResponse | Last 6 runs: cost, count, trends |

### `payroll-encryption.ts`

| Function | Params | Returns | Description |
|----------|--------|---------|-------------|
| `encryptEmployeeFields` | input | EncryptedEmployeeData | AES-256-GCM encrypt 11 fields |
| `decryptEmployeeFields` | record | DecryptedEmployeeData | Decrypt 11 fields back to plaintext |
| `redactSensitiveFields` | employee, permissions | Partial&lt;Employee> | Strip unauthorized fields |
| `sanitizeForAuditLog` | employee | Partial&lt;Employee> | Strip all PII for logging |
| `getSensitiveFieldsPresent` | employee | string[] | List which sensitive fields exist |

### `payroll-access-control.ts`

| Function | Params | Returns | Description |
|----------|--------|---------|-------------|
| `hasPermission` | ctx, permission | boolean | Check single permission |
| `ensureHasPermission` | ctx, permission | void (throws) | Check or throw ServiceError |
| `filterEmployeeDataByPermissions` | employee, permissions | Partial&lt;Employee> | Remove unauthorized fields |
| `isAccessingOwnData` | ctx, employeeId | boolean | Check if user owns record |
| `logSensitiveDataAccess` | ctx, deps, action, fields | void | Audit trail entry |

---

## 2. Regulator Services

### `zra-paye.service.ts`

| Function | Params | Returns | Description |
|----------|--------|---------|-------------|
| `createPayeSubmissionService` | ctx, deps, input | PayeSubmission | Create PAYE filing for period |
| `getPayeSubmissionService` | ctx, deps, id | PayeSubmission | Get submission detail |
| `listPayeSubmissionsService` | ctx, deps, filters | PayeSubmission[] | List submissions |
| `savePayePayrollService` | ctx, deps, submissionId, data | PayeSubmission | Save payroll totals (step 2) |
| `savePayeEmployeesService` | ctx, deps, submissionId, employees | PayeSubmission | Save employee details (step 3) |
| `prefillPayeDataService` | ctx, deps, period | PrefillData | Pre-fill from payroll run |
| `updatePayeSubmissionStatusService` | ctx, deps, id, status | PayeSubmission | Update to submitted/accepted/rejected |
| `downloadPayeTemplateService` | ctx, deps | Buffer | Excel template download |

### `napsa.service.ts`

| Function | Params | Returns | Description |
|----------|--------|---------|-------------|
| `getNapsaProfile` | ctx, deps | NapsaProfile | Fetch org NAPSA profile |
| `createNapsaProfileService` | ctx, deps, input | NapsaProfile | Register employer (encrypted credentials) |
| `updateNapsaProfileService` | ctx, deps, input | NapsaProfile | Update profile |
| `createNapsaEmployeeService` | ctx, deps, input | NapsaEmployee | Enroll employee |
| `listNapsaEmployeesService` | ctx, deps | NapsaEmployee[] | List enrolled employees |
| `createNapsaReturnService` | ctx, deps, input | NapsaReturn | File contribution return |
| `calculateMonthlyContributionService` | ctx, deps, employeeId | Contribution | Compute NAPSA obligation |

### `nhima.service.ts`

| Function | Params | Returns | Description |
|----------|--------|---------|-------------|
| `getNhimaProfile` | ctx, deps | NhimaProfile | Fetch org NHIMA profile |
| `createNhimaProfileService` | ctx, deps, input | NhimaProfile | Register employer (encrypted credentials) |
| `createNhimaEmployeeService` | ctx, deps, input | NhimaEmployee | Enroll employee |
| `listNhimaEmployeesService` | ctx, deps | NhimaEmployee[] | List enrolled employees |
| `createNhimaReturnService` | ctx, deps, input | NhimaReturn | File contribution return |
| `calculateMonthlyContributionService` | ctx, deps, employeeId | Contribution | Compute NHIMA obligation |

---

## 3. API Handlers

### `apps/api-payroll/src/modules/handlers.ts` (~130KB)

Maps route definitions to service calls. Each handler:
1. Extracts `ServiceContext` and `ServiceDependencies` from Hono context
2. Parses validated input from body/params/query
3. Calls the corresponding service function
4. Returns JSON response with `{ success: true, data: ... }`
5. Catches `ServiceError` and maps to HTTP status codes

### `apps/api-regulators/src/modules/returns/handlers.ts`

Handles all PAYE submission endpoints with the same pattern.

---

## 4. API Routes

### `apps/api-payroll/src/modules/routes.ts`

83 route definitions using `@hono/zod-openapi` `createRoute()`:
- Tags: Payroll Setup, Payroll Dashboard, Payroll Allowances, Payroll Deductions, Statutory Remittances, Employees, Payroll Runs, Payslips, Loans, Loan Repayments, Departments, Designations, Monthly Inputs
- All routes: `requireAuth` + `requireOrg` middleware
- Request validation via Zod schemas
- OpenAPI documentation auto-generated

### `apps/api-regulators/src/modules/returns/routes.ts`

12 PAYE-specific route definitions.

---

## 5. Frontend Pages

### `apps/app/app/(authenticated)/(modules)/payroll/`

| Page | Path | Description |
|------|------|-------------|
| Dashboard | `/payroll` | KPIs, recent runs, setup wizard, quick actions |
| Settings | `/payroll/settings` | Payroll configuration form |
| Employees | `/payroll/employees` | Employee list with search, filters, pagination |
| Add Employee | `/payroll/employees/add` | Multi-section employee creation form |
| Employee Detail | `/payroll/employees/[id]` | Profile, allowances, deductions, payslips, loans |
| Edit Employee | `/payroll/employees/[id]/edit` | Edit employee form |
| Departments | `/payroll/departments` | Department CRUD table |
| Positions | `/payroll/positions` | Designation/position CRUD table |
| Allowances & Deductions | `/payroll/allowances-deductions` | Company-wide compensation config |
| Payroll Runs | `/payroll/payruns` | List of payroll runs with status filters |
| Payrun Detail | `/payroll/payruns/[id]` | Run summary + payslip list |
| Monthly Inputs | `/payroll/monthly-inputs` | Monthly attendance/overtime entry |
| Loans | `/payroll/loans` | Employee loan management |
| Reports | `/payroll/reports` | Reports & data exports |

---

## 6. Frontend Components

### `apps/app/zones/payroll/modules/`

**Dashboard:**

| Component | File | Description |
|-----------|------|-------------|
| PayrollOverviewBlock | `payroll-overview-block.tsx` | KPI cards (employees, gross, net, next payrun) |
| PayrollStatsBlock | `payroll-stats-block.tsx` | Statistics charts |
| PayrollChartsBlock | `payroll-charts-block.tsx` | Trend visualizations |
| PayrollPayrunsBlock | `payroll-payruns-block.tsx` | Recent payrun cards |
| PayrollQuickActionsBlock | `payroll-quick-actions-block.tsx` | Action buttons |
| PayrollSetupBlock | `payroll-setup-block.tsx` | Setup completion checklist |
| PayrollSetupRegistration | `payroll-setup-registration.tsx` | NAPSA/NHIMA/ZRA registration steps |
| PayrollSetupTasks | `payroll-setup-tasks.tsx` | Setup task tracking |
| PayrollHelperPanel | `payroll-helper-panel.tsx` | Guidance tooltips |
| PayrollNextPayrunAlert | `payroll-next-payrun-alert.tsx` | Next payroll due date alert |
| PayrollCountdownTimer | `payroll-countdown-timer.tsx` | Countdown to cutoff |

**Data Entry:**

| Component | File | Description |
|-----------|------|-------------|
| PayrollMonthlyInputsList | `payroll-monthly-inputs-list.tsx` | Monthly input entry interface |
| PayrollSettingsModal | `payroll-settings-modal.tsx` | Settings configuration modal |
| PayrollCompanyAllowances | `payroll-company-allowances.tsx` | Company allowance CRUD |
| PayrollCompanyDeductions | `payroll-company-deductions.tsx` | Company deduction CRUD |
| PayrollDepartmentsTable | `payroll-departments-table.tsx` | Department list + CRUD |
| PayrollPositionsTable | `payroll-positions-table.tsx` | Position list + CRUD |
| EmployeePayrollInfo | `employee-payroll-info.tsx` | Employee allowances/deductions summary |

---

## 7. Frontend Query Hooks

### `apps/app/lib/queries/payroll/payroll.ts`

| Hook | Key | Description |
|------|-----|-------------|
| `usePayrollDashboard` | `['payroll', 'dashboard']` | Dashboard KPIs and data |
| `usePayrollRuns` | `['payroll', 'runs', filters]` | Paginated payroll runs |
| `usePayrollPayslips` | `['payroll', 'runs', id, 'payslips']` | Payslips for a run |
| `useEmployees` | `['payroll', 'employees', filters]` | Paginated employee list |
| `useEmployee` | `['payroll', 'employees', id]` | Single employee detail |
| `useAllowances` | `['payroll', 'allowances']` | Company allowance types |
| `useDeductions` | `['payroll', 'deductions']` | Company deduction types |
| `useLoans` | `['payroll', 'loans', filters]` | Employee loans |
| `useMonthlyInputs` | `['payroll', 'monthly-inputs']` | Monthly input data |

### `apps/app/lib/queries/payroll/payroll-mutations.ts`

| Hook | Invalidates | Description |
|------|------------|-------------|
| `useCreatePayrollSettings` | `['payroll', 'settings']` | Create org settings |
| `useUpdatePayrollSettings` | `['payroll', 'settings']` | Update org settings |
| `useCreateEmployee` | `['payroll', 'employees']` | Create employee |
| `useUpdateEmployee` | `['payroll', 'employees']` | Update employee |
| `useDeleteEmployee` | `['payroll', 'employees']` | Delete employee |
| `useCreateAllowance` | `['payroll', 'allowances']` | Create allowance type |
| `useUpdateAllowance` | `['payroll', 'allowances']` | Update allowance type |
| `useDeleteAllowance` | `['payroll', 'allowances']` | Delete allowance type |
| `useCreateDeduction` | `['payroll', 'deductions']` | Create deduction type |
| `useUpdateDeduction` | `['payroll', 'deductions']` | Update deduction type |
| `useDeleteDeduction` | `['payroll', 'deductions']` | Delete deduction type |
| `useRunPayroll` | `['payroll', 'runs']` | Execute payroll run |
| `useCreateLoan` | `['payroll', 'loans']` | Create employee loan |
| `useLoanRepayment` | `['payroll', 'loans']` | Record loan repayment |

### `apps/app/lib/queries/payroll/payroll-setup.ts`

Setup wizard state management using React context.

### Regulator Hooks

**`apps/app/lib/queries/zra/zra-paye.ts`:**

| Hook | Description |
|------|-------------|
| `usePayeSubmissions` | List PAYE submissions |
| `usePayeByFiling` | Get submission by filing ID |
| `useCreatePayeSubmission` | Create PAYE submission |
| `usePaePrefill` | Pre-fill from payroll data |

**`apps/app/lib/queries/napsa/napsa-profile.ts`:**

| Hook | Description |
|------|-------------|
| `useNapsaProfile` | Get NAPSA profile |
| `useCreateNapsaProfile` | Register NAPSA |

**`apps/app/lib/queries/nhima/nhima-connect.ts`:**

| Hook | Description |
|------|-------------|
| `useNhimaProfile` | Get NHIMA profile |
| `useCreateNhimaProfile` | Register NHIMA |

---

## 8. Database Repositories

### `packages/database/src/repositories/payroll.ts`

**Settings & Setup:**
- `getPayrollSetup(db, orgId)` — existence checks for all setup steps
- `findActivePayrollSettings(db, orgId)` — fetch active settings
- `createPayrollSettings(db, data)` — insert settings
- `updatePayrollSettings(db, orgId, data)` — update settings

**Allowances & Deductions:**
- `listAllowances(db, orgId)` / `createAllowance(db, data)` / `updateAllowance(db, id, orgId, data)` / `deleteAllowance(db, id, orgId)`
- `listDeductions(db, orgId)` / `createDeduction(db, data)` / `updateDeduction(db, id, orgId, data)` / `deleteDeduction(db, id, orgId)`

**Statutory:**
- `listStatutoryRemittanceTypes(db, orgId)` / `createStatutoryRemittanceType(db, data)` / `updateStatutoryRemittanceType(db, id, orgId, data)` / `deleteStatutoryRemittanceType(db, id, orgId)`
- `listTaxBands(db, typeId)` / `createTaxBand(db, data)`
- `findPercentageRemittance(db, typeId)` / `createPercentageRemittance(db, data)` / `updatePercentageRemittance(db, id, data)`

**Employees:**
- `listEmployees(db, orgId, filters)` — paginated with search, status, department, branch filters
- `findEmployeeById(db, id, orgId)` / `createEmployee(db, data)` / `updateEmployee(db, id, orgId, data)` / `deleteEmployee(db, id, orgId)`
- `listEmployeeAllowances(db, empId)` / `createEmployeeAllowance(db, data)` / `updateEmployeeAllowance(db, id, data)`
- `listEmployeeDeductions(db, empId)` / `createEmployeeDeduction(db, data)` / `updateEmployeeDeduction(db, id, data)`

**Payroll Runs:**
- `listPayrollRuns(db, orgId, filters)` / `findPayrollRunById(db, id, orgId)` / `createPayrollRun(db, data)` / `updatePayrollRun(db, id, orgId, data)`
- `createPayslip(db, data)` / `createPayslipsBulk(db, data[])` / `listPayslipsByRun(db, runId)` / `listPayslipsByEmployee(db, empId)`
- `createPayslipAllowancesBulk(db, data[])` / `createPayslipDeductionsBulk(db, data[])` / `createPayslipStatutoryRemittancesBulk(db, data[])`

**Loans:**
- `listLoans(db, orgId, filters)` / `findLoanById(db, id)` / `createLoan(db, data)` / `updateLoan(db, id, data)` / `deleteLoan(db, id)`
- `listLoanRepayments(db, loanId)` / `createLoanRepayment(db, data)` / `updateLoanRepayment(db, id, data)` / `deleteLoanRepayment(db, id)`

**Statistics:**
- `getPayrollStats(db)` — aggregate totals
- `getPayrollStatusDistribution(db)` — status counts
- `getPayrollRunsTimeseries(db, months)` — monthly trend
- `getPayrollOrgLeaderboard(db, limit)` — top orgs
- `getPayrollCurrentPeriodSummary(db)` — current month

**Monthly Inputs:**
- `listMonthlyInputs(db, orgId, month, year)` / `createMonthlyInput(db, data)` / `updateMonthlyInput(db, id, data)` / `bulkCreateMonthlyInputs(db, data[])`

**Departments & Designations:**
- `listDepartments(db, orgId)` / `createDepartment(db, data)` / `updateDepartment(db, id, orgId, data)` / `deleteDepartment(db, id, orgId)`
- `listDesignations(db, orgId)` / `createDesignation(db, data)` / `updateDesignation(db, id, orgId, data)` / `deleteDesignation(db, id, orgId)`

---

## 9. Background Jobs

### `apps/api-jobs/src/jobs/`

| Job | File | Schedule | Description |
|-----|------|----------|-------------|
| Payroll Run | `payroll-run.ts` | Configurable | Auto-run for orgs with `nextPayrollDue <= today` |
| Payroll Reminders | `payroll-reminders.ts` | Daily | Send reminders before cutoff dates |

---

## 10. Zod Validation Schemas

### `packages/payroll/src/<slice>/schema.ts`

**Input Schemas:**
- `createPayrollSettingsInputSchema` — work days, payment day, cutoff, leave, loan, overtime config
- `updatePayrollSettingsInputSchema` — partial of above
- `createAllowanceInputSchema` — name, code, calculationType, payType, isTaxable, isProRata
- `updateAllowanceInputSchema` — partial of above
- `createDeductionInputSchema` — extends allowance with calculationBase, contribution rates
- `updateDeductionInputSchema` — partial of above
- `createStatutoryRemittanceTypeInputSchema` — code, type, category, frequency, authority, nested config
- `createTaxBandInputSchema` — bandNumber, min/max income, rate, fixedAmount, effective dates
- `createEmployeeInputSchema` — 50+ fields, all personal/employment/banking/regulatory
- `updateEmployeeInputSchema` — partial of above
- `createEmployeeAllowanceInputSchema` — employeeId, allowanceId, amount, effective dates
- `createEmployeeDeductionInputSchema` — employeeId, deductionId, amount, frequency
- `createLoanInputSchema` — principal, interest, installments, dates, status
- `updateLoanInputSchema` — partial of above
- `createPayslipInputSchema` — full payslip breakdown
- `monthlyInputResponseSchema` — attendance, overtime, advances per employee/month
- `loanRepaymentSchema` — installment number, amount, date

**Response Schemas:**
- Corresponding `*ResponseSchema` for each input with id, timestamps, computed fields
- `payrollRunResponseSchema` — full run with all totals and status tracking
- `payslipResponseSchema` — full payslip with all earnings/deductions breakdown
- `payrollDashboardResponseSchema` — dashboard KPIs
- `payrollSetupStepsSchema` — setup completion state
- `paginationMetadataSchema` — total, page, limit, totalPages

**Validation Patterns:**
- NRC: `/^\d{6}\/\d{2}\/\d{1}$/`
- Phone: `/^\+260\d{9}$/`
- TPIN: `/^\d{10}$/`
- NAPSA: `/^[A-Z]{2}\d+$/`
- NHIMA: `/^NH\d+$/`
