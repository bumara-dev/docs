---
title: "ERP Tools — Detailed Feature Documentation"
description: "Ten standalone financial calculators and compliance tools for the Zambian business environment, covering tax, employment law, and planning."
---

## Table of Contents

1. [Overview](#1-overview)
2. [Tool Categories and Navigation](#2-tool-categories-and-navigation)
3. [PAYE Calculator](#3-paye-calculator)
4. [VAT Calculator](#4-vat-calculator)
5. [NAPSA iCare Guide Hub](#5-napsa-icare-guide-hub)
6. [Gratuity Calculator](#6-gratuity-calculator)
7. [Terminal Benefits Calculator](#7-terminal-benefits-calculator)
8. [Overtime Calculator](#8-overtime-calculator)
9. [Loan Calculator](#9-loan-calculator)
10. [Currency Converter](#10-currency-converter)
11. [GRZ Bond Calculator](#11-grz-bond-calculator)
12. [Employer Return Validator](#12-employer-return-validator)
13. [PDF Export System](#13-pdf-export-system)
14. [Technical Architecture](#14-technical-architecture)

---

## 1. Overview

The ERP Tools module is a collection of 10 standalone financial calculators and compliance tools designed specifically for the Zambian business environment. These tools help employers, HR professionals, and finance teams perform complex calculations related to taxation, employment law, and financial planning — all in compliance with Zambian legislation.

**Key Characteristics:**
- All tools are client-side only — no backend API calls (except Currency Converter which fetches live exchange rates)
- Real-time calculations that update as the user types
- PDF export with Bumara branding on every tool
- Fully responsive layouts (desktop and mobile)
- Based on current Zambian law: Employment Code Act 2019, Income Tax Act, NAPSA Act

**Access Path:** `/employment-tools` (main hub page listing all tools)

---

## 2. Tool Categories and Navigation

### 2.1 Category Structure

Tools are organized into three categories:

| Category | Tools |
|----------|-------|
| **Tax & Compliance** | PAYE Calculator, VAT Calculator, NAPSA iCare Guide Hub |
| **Employment** | Gratuity Calculator, Terminal Benefits Calculator, Overtime Calculator |
| **Financial** | Loan Calculator, Currency Converter, GRZ Bond Calculator |

### 2.2 Tools Hub Page

**Route:** `/employment-tools`

The hub page displays all tools as cards organized by category. Each card shows:
- Tool title and subtitle
- Brief description
- Category-based color theming (blue for Tax, green for Employment, purple for Financial)
- Lucide icon
- Click navigates to the tool's dedicated page

### 2.3 Tool Card Component

Each tool card uses a consistent design with category-aware color schemes:
- **Tax & Compliance:** Blue accent colors
- **Employment:** Green accent colors
- **Financial:** Purple accent colors

The card component renders the tool's icon, title, subtitle, description, and a category badge.

---

## 3. PAYE Calculator

**Route:** `/tax-compliance-tools/payee`
**Component:** `payeeCalculator.tsx`
**Legal Basis:** Zambia Revenue Authority (ZRA) 2025 Tax Bands

### 3.1 Purpose

Calculates Pay As You Earn (PAYE) tax, NAPSA contributions (5%), and NHIMA contributions (1%) for employees in Zambia. Supports both forward calculation (gross to net) and reverse calculation (net to gross).

### 3.2 Tax Bands (2025)

| Income Range (ZMW) | Tax Rate |
|---------------------|----------|
| K0 — K5,100 | 0% (Tax-free) |
| K5,100 — K7,100 | 20% |
| K7,100 — K9,200 | 30% |
| Above K9,200 | 37.5% |

### 3.3 Deduction Rates

| Deduction | Rate | Applied To |
|-----------|------|------------|
| NAPSA | 5% | Gross salary (employee contribution) |
| NHIMA | 1% | Gross salary |
| PAYE | Progressive | Taxable income (Gross - NAPSA) |

### 3.4 Calculation Modes

**Forward Mode (Gross to Net):**
1. User enters gross salary
2. System calculates NAPSA (5% of gross)
3. Taxable income = Gross - NAPSA
4. PAYE calculated using progressive tax bands
5. NHIMA = 1% of gross
6. Net salary = Gross - NAPSA - PAYE - NHIMA

**Reverse Mode (Net to Gross):**
1. User enters desired net salary
2. System uses binary search algorithm to find the gross salary that produces the target net
3. Binary search iterates with precision to 2 decimal places
4. All deductions are then displayed

### 3.5 Features

- Real-time calculation as user types
- Toggle between forward and reverse modes
- Detailed breakdown showing each deduction
- Percentage breakdown of salary composition
- PDF export with full calculation details
- Employer cost summary (Gross + Employer NAPSA 5%)

### 3.6 PDF Export

Generates a branded PDF document containing:
- Bumara logo header
- Salary type (Forward/Reverse)
- Complete deduction breakdown
- Net salary highlighted
- Generated date
- Legal disclaimer referencing ZRA 2025 tax bands

---

## 4. VAT Calculator

**Route:** `/tax-compliance-tools/vat`
**Component:** `vatCalculator.tsx`
**Legal Basis:** Zambia Revenue Authority — Value Added Tax Act

### 4.1 Purpose

Calculates Value Added Tax (VAT) for business transactions. Supports standard VAT calculation, excise duty, and multiple product categories.

### 4.2 Calculation Features

- **Standard VAT Rate:** 16% (configurable from 1% to 50% via dropdown)
- **Excise Duty:** Optional, with configurable rate (1% to 50%)
- **Calculation Direction:** Add VAT to amount OR extract VAT from VAT-inclusive amount
- **Product Categories:** Dropdown with common product types for reference

### 4.3 Calculation Logic

**Adding VAT:**
- Base Amount = User input
- Excise = Base Amount x Excise Rate (if enabled)
- VAT = (Base Amount + Excise) x VAT Rate
- Total = Base Amount + Excise + VAT

**Extracting VAT (from inclusive price):**
- Total = User input
- Base Amount = Total / (1 + VAT Rate + Excise Rate)
- VAT and Excise calculated from base

### 4.4 Features

- Real-time calculation
- Configurable VAT and excise duty rates
- Product category selection
- Detailed breakdown table
- PDF export with Bumara branding

---

## 5. NAPSA iCare Guide Hub

**Route:** `/employment-tools/napsa-guide`
**Component:** `napsaGuide.tsx`
**Legal Basis:** NAPSA Act, Zambia

### 5.1 Purpose

A comprehensive reference hub containing 11 detailed guides for navigating the NAPSA (National Pension Scheme Authority) iCare system. This is NOT a calculator — it is a searchable knowledge base.

### 5.2 Guide Categories

| Category | # Guides |
|----------|----------|
| Getting Started | 3 (Profile Creation, KYC Update, Employer Registration) |
| Monthly Operations | 3 (Returns Submission, Return Types, Return Tracking) |
| Payments & Finance | 2 (Online Payments, Payment Methods & NPIN) |
| Account Management | 1 (Link Existing Employer Account) |
| Compliance | 2 (Compliance Certificate, Penalty Waiver) |

### 5.3 Individual Guides

**Profile Creation Guide** — Step-by-step guide for creating a NAPSA iCare profile. Covers required documents (NRC, passport photo, phone, email), the 8-step registration process, and post-approval actions.

**KYC Update Guide** — How to update Know Your Customer information including bank details, beneficiary management (with percentage allocation), and employment history updates.

**Employer Registration Guide** — The 7-step process for registering a company with NAPSA. Covers pre-registration requirements (Certificate of Incorporation, TPIN, NRC), and post-approval benefits.

**Returns Submission Guide** — Three methods for submitting monthly employee contribution returns: CSV Upload (recommended), Manual Entry, and Contributions Without Returns. Includes common error troubleshooting.

**Return Types Guide** — Explains Monthly, NIL, Top-up, and Correction returns. Key note: NIL returns are NOT optional and attract K500 penalty if missed.

**Return Tracking Guide** — Status codes explained (Submitted, Under Review, Approved, Query Raised, Rejected, Completed) and common error codes (E001-E008).

**Online Payments Guide** — Payment methods: Mobile Money (MTN, Airtel, Zamtel), Bank Transfer (7 supported banks), and NPIN System. Includes security features.

**Payment Methods & NPIN Guide** — Detailed comparison of payment methods with processing times, fees, daily limits, and setup instructions.

**Link Existing Employer Account** — Two methods: Account Number Linking (for non-eNAPSA users) and Email Address Linking (for previous eNAPSA users).

**Compliance Certificate Guide** — Standard (3-month validity, free) vs Enhanced (6-month validity, K250) certificates. Includes application process and maintenance requirements.

**Penalty Waiver Guide** — General Waiver (automated, 85-95% success rate) vs Special Waiver (manual review, 40-60% success rate). Includes eligibility criteria and required documentation.

### 5.4 UI Features

- Searchable guide list with text filtering
- Category filter buttons
- Difficulty level badges (Beginner, Intermediate, Advanced)
- Estimated reading time per guide
- Modal overlay for reading guide content
- Organized sections with numbered headings
- Tips (green) and Warnings (red) callout boxes

---

## 6. Gratuity Calculator

**Route:** `/employment-tools/gratuity`
**Component:** `gratuityCalculator.tsx`
**Legal Basis:** Employment Code Act 2019, Section 73

### 6.1 Purpose

Calculates contract gratuity payments for employees on fixed-term contracts. The legal minimum gratuity rate is 25% of total basic pay earned during the contract period.

### 6.2 Input Fields

| Field | Description |
|-------|-------------|
| Monthly Basic Salary | Base salary in ZMW |
| Contract Period (Years) | Whole years served |
| Contract Period (Months) | Additional months served |
| Gratuity Rate | Minimum 25% (configurable) |
| Apply Tax Deductions | Toggle for PAYE, NAPSA, NHIMA |

### 6.3 Calculation Logic

1. Total Months = (Years x 12) + Months
2. Total Earnings = Monthly Salary x Total Months
3. Gross Gratuity = Total Earnings x Gratuity Rate
4. If tax deductions enabled:
   - NAPSA = 5% of Gross Gratuity
   - Taxable = Gross Gratuity - NAPSA
   - PAYE = Progressive tax bands applied to Taxable amount
   - NHIMA = 1% of Gross Gratuity
5. Net Gratuity = Gross - NAPSA - PAYE - NHIMA

### 6.4 Features

- Real-time calculation
- Flexible contract period input (years + months)
- Configurable gratuity rate (minimum 25%)
- Optional tax deduction simulation
- Detailed breakdown with all deductions
- PDF export

---

## 7. Terminal Benefits Calculator

**Route:** `/employment-tools/terminal-benefits`
**Component:** `terminalBenefitsCalculator.tsx`
**Legal Basis:** Employment Code Act 2019, Sections 36-37, 53-55, 73

### 7.1 Purpose

Calculates five types of end-of-employment benefits: Notice Pay, Severance Pay, Redundancy Pay, Leave Pay, and Gratuity.

### 7.2 Benefit Types

**Notice Pay (Section 53):**
- Formula: (Basic Salary / 30) x Notice Days
- Notice periods: 1 week, 2 weeks, 1 month, 2 months, 3 months

**Severance Pay (Section 54):**
- Fixed-term method: 25% of basic pay earned during contract
- Medical/Death method: 2 months salary per year of service

**Redundancy Pay (Section 55):**
- Minimum 2 months basic salary per completed year of service
- First K2,000,000 is tax exempt

**Leave Pay (Sections 36-37):**
- Formula: (Monthly Pay / 26 working days) x Accrued Leave Days

**Gratuity (Section 73):**
- 25% of basic pay earned during fixed-term contract

### 7.3 Tax Deductions

When enabled, all benefit types (except tax-exempt portions) are subject to:
- NAPSA: 5% of taxable amount
- PAYE: Progressive tax bands on (taxable - NAPSA)
- NHIMA: 1% of taxable amount

Redundancy pay has a special K2,000,000 tax exemption threshold.

### 7.4 Features

- Dropdown selector for benefit type
- Conditional input fields that change based on selected type
- Tax deduction toggle with automatic calculation
- Detailed calculation breakdown
- Legal reference sidebar
- PDF export with benefit-type-specific title

---

## 8. Overtime Calculator

**Route:** `/employment-tools/overtime`
**Component:** `overtimeCalculator.tsx`
**Legal Basis:** Employment Code Act 2019, Section 75

### 8.1 Purpose

Calculates overtime pay based on Zambian labor law. Regular overtime is paid at 1.5x and public holidays/rest days at 2x the hourly rate.

### 8.2 Overtime Types

| Type | Multiplier | Description |
|------|-----------|-------------|
| Regular Overtime | 1.5x (150%) | Weekday overtime beyond normal hours |
| Holiday/Rest Day | 2.0x (200%) | Public holidays and weekly rest days |

### 8.3 Input Modes

**Hourly Rate Mode:**
- User directly enters their hourly rate
- Tip: Hourly rate = Monthly Salary / 208 hours

**Monthly Salary Mode:**
- User enters monthly basic salary
- System calculates hourly rate automatically
- Two working-hour standards:
  - Standard: 208 hours/month (52 weeks x 40 hours / 12 months)
  - Banking: 240 hours/month

### 8.4 Calculation Logic

1. Hourly Rate = (from input or Monthly Salary / Monthly Hours)
2. Regular Pay = Hourly Rate x Hours Worked (at 1x)
3. Overtime Pay = Hourly Rate x Hours Worked x Multiplier
4. Extra Earnings = Overtime Pay - Regular Pay

### 8.5 Features

- Toggle between hourly rate and monthly salary input
- Two monthly working hour standards (208 Standard, 240 Banking)
- Real-time hourly rate calculation display
- Results panel with overtime pay, regular pay, and extra earnings
- Quick reference card with legal thresholds (48 hrs/week, max 8 hrs/day OT, max 48 hrs/month OT)
- Regulation information section
- PDF export

---

## 9. Loan Calculator

**Route:** `/financial-tools/loan`
**Component:** `loanCalculator.tsx`

### 9.1 Purpose

Calculates monthly payments, total interest, and generates a complete amortization schedule for personal loans. Contextualised for the Zambian market with typical interest rate ranges.

### 9.2 Input Fields

| Field | Description |
|-------|-------------|
| Loan Amount | Principal in ZMW |
| Loan Term | In years or months (toggle) |
| Annual Interest Rate | 1% to 50% (dropdown, 10-28% typical Zambian range) |

### 9.3 Calculation Formula

Standard amortization formula:

**M = P x [r(1+r)^n] / [(1+r)^n - 1]**

Where:
- M = Monthly Payment
- P = Principal (Loan Amount)
- r = Monthly Interest Rate (Annual Rate / 12 / 100)
- n = Total Number of Payments (months)

### 9.4 Amortization Schedule

The calculator generates a complete month-by-month table showing:
- Month number
- Monthly payment amount
- Principal portion of payment
- Interest portion of payment
- Remaining balance

The schedule is collapsible (Show/Hide toggle) and scrollable for long loan terms.

### 9.5 Visual Breakdown

A horizontal bar chart shows the Principal vs Interest ratio:
- Blue bar = Principal percentage
- Amber bar = Interest percentage

### 9.6 Features

- Toggle between years and months for loan term
- Annotated interest rate dropdown (Typical, Below Average, Above Average)
- Visual payment breakdown bar
- Collapsible amortization schedule table
- Monthly rate display
- PDF export with loan summary
- Quick tips sidebar

---

## 10. Currency Converter

**Route:** `/financial-tools/currency`
**Component:** `currencyCalculator.tsx`
**Data Source:** Open Exchange Rates API (open.er-api.com)

### 10.1 Purpose

Real-time currency converter supporting 45+ currencies with a focus on African currencies. This is the only tool that makes external API calls.

### 10.2 Supported Currencies

**African Currencies (21):**
ZMW (Zambian Kwacha), ZAR (South African Rand), KES (Kenyan Shilling), TZS (Tanzanian Shilling), UGX (Ugandan Shilling), NGN (Nigerian Naira), GHS (Ghanaian Cedi), BWP (Botswana Pula), NAD (Namibian Dollar), MWK (Malawian Kwacha), MZN (Mozambican Metical), ZWL (Zimbabwean Dollar), RWF (Rwandan Franc), MUR (Mauritian Rupee), EGP (Egyptian Pound), MAD (Moroccan Dirham), ETB (Ethiopian Birr), AOA (Angolan Kwanza), CDF (Congolese Franc), XAF (Central African CFA), XOF (West African CFA)

**Major World Currencies (24):**
USD, EUR, GBP, JPY, CNY, INR, AUD, CAD, CHF, NZD, SGD, HKD, SEK, NOK, DKK, AED, SAR, BRL, MXN, RUB, KRW, THB, MYR, IDR, PHP, PLN, TRY, ILS, CZK, HUF

### 10.3 API Integration

- **Source:** `https://open.er-api.com/v6/latest/USD`
- **Base Currency:** USD (all rates relative to USD)
- **Auto-refresh:** Every 5 minutes
- **Fallback:** Hardcoded rates used if API fails
- **Live status indicator:** Green pulsing dot when rates are current

### 10.4 Conversion Logic

1. Get source currency rate relative to USD (fromRate)
2. Get target currency rate relative to USD (toRate)
3. Convert: Amount in USD = Input / fromRate
4. Result = Amount in USD x toRate
5. Direct rate = toRate / fromRate

### 10.5 Features

- Currency swap button (reverses From/To)
- Country flag icons via flagcdn.com
- Common conversions table (1, 10, 50, 100, 500, 1000, 5000, 10000)
- ZMW exchange rates panel (8 popular currencies)
- Rate details panel with exchange rate and inverse rate
- Real-time status indicator (loading spinner / live indicator)
- Manual refresh button
- PDF export with conversion details

---

## 11. GRZ Bond Calculator

**Route:** `/financial-tools/bond`
**Component:** `grzBondCalculator.tsx`
**Data Source:** Bank of Zambia (BOZ) auction results (December 19, 2025)

### 11.1 Purpose

Calculates returns on Government of the Republic of Zambia (GRZ) bonds. Shows semi-annual interest payments, total interest earned, and maturity value.

### 11.2 Bond Options

| Tenure | Coupon Rate |
|--------|-------------|
| 2 Years | 18.5% |
| 3 Years | 19.0% |
| 5 Years | 20.5% |
| 7 Years | 22.0% |
| 10 Years | 24.0% |
| 15 Years | 25.5% |

**Minimum Investment:** K5,000

### 11.3 Input Fields

| Field | Description |
|-------|-------------|
| Investment Amount | Principal in ZMW (min K5,000) |
| Bond Tenure | 2, 3, 5, 7, 10, or 15 years |
| Coupon Rate | Auto-filled based on tenure, editable |
| Face Value | Optional field |
| Withholding Tax | 15% default (optional) |
| Handling Fee | 0.5% default (optional) |

### 11.4 Calculation Logic

1. Annual Interest = Principal x (Coupon Rate / 100)
2. Semi-Annual Payment = Annual Interest / 2
3. Monthly Equivalent = Annual Interest / 12
4. Total Interest Earned = Annual Interest x Years
5. Total Maturity Value = Principal + Total Interest
6. Total Return % = (Total Interest / Principal) x 100

**With Tax & Fees:**
7. Withholding Tax = Total Interest x Tax Rate (15%)
8. Handling Fee = Principal x Fee Rate (0.5%)
9. Net Interest = Total Interest - Withholding Tax
10. Net Maturity Value = Principal + Net Interest - Handling Fee

### 11.5 Features

- Minimum investment validation (K5,000) with error message
- Auto-adjusting coupon rate when tenure changes
- Optional tax and fee deduction simulation
- Clickable tenure selector in sidebar
- Semi-annual payment highlight (primary result)
- Monthly equivalent display
- Detailed investment summary table
- Tax-adjusted results section
- Link to BOZ auction results website
- Bond information sidebar (issuer, payment schedule, auction schedule)
- PDF export

---

## 12. Employer Return Validator

**Route:** `/tax-compliance-tools/napsa-validator`
**Component:** `employeeReturnValidator.tsx`
**Status:** Currently commented out in the tools hub (not visible to users)

### 12.1 Purpose

Validates employer contribution return CSV files before submission to NAPSA. Checks for data completeness, correct contribution calculations, and compliance with NAPSA rules.

### 12.2 File Format

Expected CSV columns (12 fields):

| Column | Description | Required |
|--------|-------------|----------|
| Employer ID | Unique employer identifier | Yes |
| Year | Contribution year (2000-2025) | Yes |
| Month | Contribution month (1-12) | Yes |
| SSN | Social Security Number | Yes |
| National ID | NRC or Passport | Yes |
| Surname | Employee surname | Yes |
| First Name | Employee first name | Yes |
| Other Name | Other names | No |
| Date of Birth | DOB (YYYY-MM-DD) | Yes |
| Gross Pay | Monthly gross pay | Yes |
| Employee Share | 5% of gross pay | Yes |
| Employer Share | 5% of gross pay | Yes |

### 12.3 Validation Rules

1. All required fields must be present and non-empty
2. Year must be between 2000 and 2025
3. Month must be between 1 and 12
4. Gross Pay must be greater than zero
5. Employee Share must equal 5% of Gross Pay (tolerance: K0.01)
6. Employer Share must equal 5% of Gross Pay (tolerance: K0.01)
7. Neither share can exceed the monthly contribution ceiling

### 12.4 Contribution Ceilings (Monthly)

| Year | Monthly Ceiling |
|------|----------------|
| 2020-2021 | K1,221.80 |
| 2022 | K1,278.90 |
| 2023 | K1,338.45 |
| 2024 | K1,399.60 |
| 2025 | K1,465.35 |

### 12.5 Features

- Drag-and-drop file upload with visual feedback
- Upload progress indicator
- Sample CSV download
- Expected file format reference table
- Results view with summary cards (Total, Valid, Invalid records)
- Tabbed record viewer (All, Valid, Invalid)
- Invalid records highlighted in red with error descriptions
- Contribution period auto-detection
- PDF export of validation report
- Reset to validate another file

---

## 13. PDF Export System

All calculators (except NAPSA Guide Hub) include PDF export via the jsPDF library.

### 13.1 Consistent PDF Structure

Every exported PDF follows this layout:

1. **Header Bar** — Light gray background with Bumara logo centered
2. **Tool Title** — Tool name and subtitle below logo
3. **Document Title** — Bold heading (e.g., "Salary Slip", "Overtime Calculation")
4. **Generated Date** — Right-aligned date stamp
5. **Sections** — Gray section headers (e.g., "WORK DETAILS", "CALCULATION")
6. **Data Rows** — Label-value pairs with alternating separator lines
7. **Highlighted Result** — Green background for the final/net amount
8. **Footer** — Legal disclaimer and "computer-generated document" notice

### 13.2 Bumara Logo

The logo is loaded via `getBumaraLogoPng()` from `@/components/erp-tools/utils/pdf-logo`. This function returns a base64-encoded PNG that is embedded in every PDF.

### 13.3 Color Scheme

| Element | Color |
|---------|-------|
| Header background | RGB(245, 245, 245) |
| Section header background | RGB(249, 250, 251) |
| Label text | RGB(107, 114, 128) — Gray |
| Value text | RGB(55, 65, 81) — Dark gray |
| Highlighted result background | RGB(240, 253, 244) — Light green |
| Highlighted result text | RGB(21, 128, 61) — Green |
| Deduction text | RGB(220, 38, 38) — Red |
| Footer text | RGB(107, 114, 128) — Gray |

---

## 14. Technical Architecture

### 14.1 Technology Stack

| Technology | Usage |
|------------|-------|
| Next.js (App Router) | Page routing and layout |
| React 18+ | Component framework |
| TypeScript | Type safety |
| Tailwind CSS | Styling |
| jsPDF | PDF generation |
| Lucide React | Tool card icons |
| Intl.NumberFormat / toLocaleString | Number formatting |

### 14.2 Route Structure

```
(authenticated)/(erp_tools)/
  employment-tools/
    page.tsx              → Tools hub (all tools listed)
    gratuity/page.tsx     → Gratuity Calculator
    overtime/page.tsx     → Overtime Calculator
    terminal-benefits/page.tsx → Terminal Benefits Calculator
    napsa-guide/page.tsx  → NAPSA Guide Hub
  tax-compliance-tools/
    page.tsx              → Tax tools sub-hub
    payee/page.tsx        → PAYE Calculator
    vat/page.tsx          → VAT Calculator
    napsa-validator/page.tsx → Return Validator (hidden)
  financial-tools/
    page.tsx              → Financial tools sub-hub
    loan/page.tsx         → Loan Calculator
    currency/page.tsx     → Currency Converter
    bond/page.tsx         → GRZ Bond Calculator
  erp-tools/page.tsx      → Alternative hub page
```

### 14.3 Component Structure

```
components/erp-tools/
  tools-links/
    tools-data.ts         → Tool definitions (id, title, category, icon, href)
    tool-card.tsx          → Reusable tool card component
    tools-carousel.tsx     → Carousel display for dashboard
    tools-block.tsx        → Block display for dashboard
    tools-skeleton.tsx     → Loading skeleton
  tax-compliance-tools/
    payeeCalculator.tsx    → PAYE calculator component
    vatCalculator.tsx      → VAT calculator component
    employeeReturnValidator.tsx → Return validator component
  employment-tools/
    gratuityCalculator.tsx → Gratuity calculator component
    terminalBenefitsCalculator.tsx → Terminal benefits component
    overtimeCalculator.tsx → Overtime calculator component
    napsaGuide.tsx         → NAPSA guide hub component
  financial-tools/
    loanCalculator.tsx     → Loan calculator component
    currencyCalculator.tsx → Currency converter component
    grzBondCalculator.tsx  → GRZ bond calculator component
  utils/
    pdf-logo.ts            → Bumara logo for PDF exports
```

### 14.4 Common Patterns

All calculator components follow a consistent pattern:

1. **State Management:** React useState for all inputs, useCallback for calculation functions, useEffect to trigger recalculation on input changes
2. **Input Parsing:** `parseValue()` function strips commas and converts to number
3. **Number Formatting:** `formatNumber()` using `toLocaleString('en-US')` with 2 decimal places
4. **Calculation:** Pure function wrapped in `useCallback` with all dependencies listed
5. **Auto-recalculate:** `useEffect` watches the memoized calculation function
6. **PDF Export:** `handleExport()` async function using jsPDF with Bumara logo

### 14.5 Zambian Legal Constants

Constants embedded across the tools:

| Constant | Value | Source |
|----------|-------|--------|
| NAPSA Rate | 5% (employee + employer) | NAPSA Act |
| NHIMA Rate | 1% | NHIMA Act |
| PAYE Tax-Free Threshold | K5,100 | ZRA 2025 |
| Standard Working Hours/Month | 208 | Employment Code |
| Banking Working Hours/Month | 240 | Banking convention |
| Overtime Multiplier (Regular) | 1.5x | Section 75 |
| Overtime Multiplier (Holiday) | 2.0x | Section 75 |
| Gratuity Minimum Rate | 25% | Section 73 |
| Redundancy Tax Exemption | K2,000,000 | Income Tax Act |
| Working Days/Month | 26 | Standard |
| Overtime Weekly Threshold | 48 hours | Section 75 |
| Max Daily Overtime | 8 hours | Section 75 |
| Max Monthly Overtime | 48 hours | Section 75 |
| NAPSA Contribution Ceiling (2025) | K1,465.35/month | NAPSA |
| GRZ Bond Minimum Investment | K5,000 | BOZ |

---

*This documentation covers the ERP Tools feature as implemented in the `apps/app/app/(authenticated)/(erp_tools)/` directory and the `apps/app/components/erp-tools/` components directory. All tools are client-side calculators (except Currency Converter) with no database dependencies.*
