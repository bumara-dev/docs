---
title: "ZRA Withholding Tax (WHT) Specification"
description: "Module: Compliance → ZRA → Withholding Tax Last Updated: 2026-01-24"
---

## Overview

ZRA Withholding Tax is a monthly obligation for businesses that deduct tax at source from payments made to suppliers and service providers. The rates vary based on payment type and recipient tax compliance status.

---

## Template Configuration

### Template Metadata

| Field | Value |
|-------|-------|
| Template Key | `ZRA_WHT_MONTHLY_V1` |
| Template Version | 1 |
| Regulator | ZRA |
| Frequency | Monthly |
| Due Date Rule | `PERIOD_END_PLUS_DAYS` (14 days) |
| Payment Required | Yes |
| Fee Key | `ZRA_WHT_MONTHLY` |
| Service Fee | K 50.00 |

### WHT Categories

| Category | Rate (Tax-Compliant) | Rate (Non-Compliant) |
|----------|---------------------|---------------------|
| Goods | 3% | 5% |
| Services | 15% | 20% |
| Rent | 10% | 20% |
| Professional Fees | 15% | 20% |
| Interest | 15% | 20% |
| Dividends | 15% | 20% |

### Tasks

| # | Key | Title | Type | Required |
|---|-----|-------|------|----------|
| 1 | `identify_wht_payments` | Identify WHT Payments | fill_form | Yes |
| 2 | `enter_wht_totals` | Enter WHT Totals | fill_form | Yes |
| 3 | `upload_wht_schedule` | Upload WHT Schedule | upload_document | Yes |
| 4 | `confirm_submission` | Review & Confirm | review_approve | Yes |

### Task Auto-Completion Triggers

- `form:ZRA_WHT_FIGURES:payment_identification` → Completes `identify_wht_payments`
- `form:ZRA_WHT_FIGURES:wht_totals` → Completes `enter_wht_totals`
- `doc:wht_schedule` → Completes `upload_wht_schedule`
- `confirm:submit` → Completes `confirm_submission`

### Document Requirements

| Key | Name | Required |
|-----|------|----------|
| `wht_schedule` | WHT Schedule | Yes |
| `supplier_invoices` | Supplier Invoices (Optional) | No |

---

## Core Concepts

### WHT Schedule

The WHT Schedule is a required document that lists:
- Supplier name and TPIN
- Payment date
- Payment amount
- WHT rate applied
- Tax withheld

### Nil Returns

Businesses with no qualifying payments in a period can file a nil return:
- Set `isNilReturn: true`
- Optionally provide `nilReason` (e.g., "No qualifying payments made")

### Tax Compliance Status

The WHT rate depends on whether the supplier is tax-compliant:
- **Tax-Compliant:** Supplier has valid Tax Clearance Certificate (TCC)
- **Non-Compliant:** Supplier has no TCC or expired TCC

---

## Filing Status Lifecycle

```
pending_data → in_progress → ready_for_submission → submission_in_progress → submitted → accepted
                     ↓                    ↓
                 cancelled            needs_correction → in_progress
```

### Status Transitions

| From | To | Trigger |
|------|-----|---------|
| `pending_data` | `in_progress` | WHT figures entered |
| `in_progress` | `ready_for_submission` | All required tasks done + schedule uploaded |
| `ready_for_submission` | `submission_in_progress` | Request submission clicked |
| `submission_in_progress` | `submitted` | Backoffice submits to ZRA |
| `submitted` | `accepted` | ZRA confirms acceptance |

---

## API Endpoints

### Tenant API

| Method | Path | Description |
|--------|------|-------------|
| GET | `/zra/wht/by-filing/:filingId` | Get WHT return by filing |
| POST | `/zra/wht/submissions` | Create WHT return |
| GET | `/zra/wht/returns` | List returns |
| GET | `/zra/wht/returns/:id` | Get return details |
| PATCH | `/zra/wht/returns/:id/status` | Update status |
| GET | `/zra/wht/transactions` | List WHT transactions |
| POST | `/zra/wht/transactions` | Add transaction |
| GET | `/zra/wht/certificates` | List WHT certificates |

---

## UI Components

### WhtFiguresSection

Location: `apps/app/features/zra/components/wht-figures-section.tsx`

- Renders for filings with `templateKey` containing "withholding" or "wht"
- Shows entry form or summary based on existing data
- Displays transaction count, total payments, and total WHT
- Supports nil returns

---

## Due Date Calculation

All ZRA monthly filings are due 14 days after the period end:

| Filing Period | Period End | Due Date |
|--------------|------------|----------|
| January 2026 | Jan 31 | Feb 14 |
| February 2026 | Feb 28 | Mar 14 |
| March 2026 | Mar 31 | Apr 14 |

---

## Data Model

### WHT Returns Table

```sql
wht_returns (
  id UUID PRIMARY KEY,
  organization_id TEXT NOT NULL,
  tax_profile_id UUID NOT NULL,
  tax_period TEXT NOT NULL,        -- "2026-01"
  period_start_date DATE NOT NULL,
  period_end_date DATE NOT NULL,
  total_transactions INTEGER,
  total_gross_amount DECIMAL(18,2),
  total_wht_amount DECIMAL(18,2),
  total_net_amount DECIMAL(18,2),
  total_remittance_due DECIMAL(18,2),
  status ENUM,
  due_date DATE NOT NULL,
  ...timestamps
)
```

### WHT Transactions Table

```sql
wht_transactions (
  id UUID PRIMARY KEY,
  return_id UUID REFERENCES wht_returns,
  supplier_tpin TEXT,
  supplier_name TEXT,
  payment_date DATE,
  gross_amount DECIMAL(18,2),
  wht_rate DECIMAL(5,4),
  wht_amount DECIMAL(18,2),
  payment_type TEXT,              -- goods, services, rent, etc.
  ...timestamps
)
```

---

## Related Files

| File | Purpose |
|------|---------|
| `packages/database/src/schema/zra/wht/wht-returns.ts` | Returns schema |
| `packages/database/src/schema/zra/wht/wht-transactions.ts` | Transactions schema |
| `packages/database/src/seeds/zra-templates.ts` | Template definition |
| `packages/api-services/src/domains/zra/zra-wht.service.ts` | Service logic |
| `packages/backend/src/modules/zra/wht/` | Routes & handlers |
| `apps/app/features/zra/components/wht-figures-section.tsx` | UI component |

---

## Testing Checklist

- [ ] Connect ZRA with Withholding Tax selected
- [ ] Verify obligation activated with correct template
- [ ] Verify filing created for current month
- [ ] Verify 4 tasks generated (all required)
- [ ] Enter WHT payment identification
- [ ] Enter WHT totals
- [ ] Upload WHT schedule document
- [ ] Confirm submission details
- [ ] Request submission enabled when ready
- [ ] Nil return flow works correctly

---

## WHT Certificate Issuance

After WHT is withheld and remitted:
1. Backoffice generates WHT certificate for supplier
2. Certificate shows tax withheld
3. Supplier uses certificate for tax credit claims
