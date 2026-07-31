---
title: "ZRA Turnover Tax (TOT) Specification"
description: "Specification for ZRA Turnover Tax: template configuration, core concepts, the filing status lifecycle, and API endpoints."
---

## Overview

ZRA Turnover Tax is a simplified monthly tax regime for small businesses with annual turnover up to K5,000,000. The tax rate is 5% of gross monthly turnover.

---

## Template Configuration

### Template Metadata

| Field | Value |
|-------|-------|
| Template Key | `ZRA_TURNOVER_TAX_MONTHLY_V1` |
| Template Version | 1 |
| Regulator | ZRA |
| Frequency | Monthly |
| Due Date Rule | `PERIOD_END_PLUS_DAYS` (14 days) |
| Payment Required | Yes |
| Fee Key | `ZRA_TURNOVER_TAX_MONTHLY` |
| Service Fee | K 120.00 |

### Tax Calculation

- **Tax Rate:** 5% of gross monthly turnover
- **Eligibility:** Annual turnover between K12,000 and K5,000,000
- **Threshold Monitoring:** YTD turnover tracked to warn when approaching K5M limit

### Tasks

| # | Key | Title | Type | Required |
|---|-----|-------|------|----------|
| 1 | `enter_turnover` | Enter Monthly Turnover | fill_form | Yes |
| 2 | `upload_sales_summary` | Upload Sales Summary | upload_document | No |
| 3 | `confirm_turnover` | Review & Confirm | review_approve | Yes |

### Task Auto-Completion Triggers

- `form:ZRA_TOT_FIGURES:turnover_entry` → Completes `enter_turnover`
- `doc:sales_summary` → Completes `upload_sales_summary`
- `confirm:submit` → Completes `confirm_turnover`

### Document Requirements

| Key | Name | Required |
|-----|------|----------|
| `sales_summary` | Sales Summary / Z Report | No |

---

## Core Concepts

### Nil Returns

Businesses with no turnover for a period can file a nil return:
- Set `isNilReturn: true`
- Optionally provide `nilReason`

### Threshold Monitoring

The system tracks year-to-date (YTD) turnover:
- **Warning:** When YTD reaches 80% of K5,000,000
- **Critical:** When YTD exceeds K5,000,000 (must switch to standard tax)

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
| `pending_data` | `in_progress` | TOT figures entered |
| `in_progress` | `ready_for_submission` | All required tasks done |
| `ready_for_submission` | `submission_in_progress` | Request submission clicked |
| `submission_in_progress` | `submitted` | Backoffice submits to ZRA |
| `submitted` | `accepted` | ZRA confirms acceptance |

---

## API Endpoints

### Tenant API

| Method | Path | Description |
|--------|------|-------------|
| GET | `/zra/tot/by-filing/:filingId` | Get TOT return by filing |
| POST | `/zra/tot/submissions` | Create TOT return |
| GET | `/zra/tot/submissions` | List returns |
| GET | `/zra/tot/submissions/:id` | Get return details |
| PATCH | `/zra/tot/submissions/:id/status` | Update status |
| GET | `/zra/tot/filing-info` | Get filing info with YTD |

---

## UI Components

### TotFiguresSection

Location: `apps/app/features/zra/components/tot-figures-section.tsx`

- Renders for filings with `templateKey` containing "turnover" or "tot"
- Shows entry form or summary based on existing data
- Displays calculated tax (5% of turnover)
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

## Related Files

| File | Purpose |
|------|---------|
| `packages/database/src/schema/zra/tot/turnover-tax.ts` | Schema |
| `packages/database/src/seeds/zra-templates.ts` | Template definition |
| `packages/api-services/src/domains/zra/zra-tot.service.ts` | Service logic |
| `packages/backend/src/modules/zra/tot/` | Routes & handlers |
| `apps/app/features/zra/components/tot-figures-section.tsx` | UI component |

---

## Testing Checklist

- [ ] Connect ZRA with Turnover Tax selected
- [ ] Verify obligation activated with correct template
- [ ] Verify filing created for current month
- [ ] Verify 3 tasks generated
- [ ] Enter turnover figures
- [ ] Verify tax calculated at 5%
- [ ] Complete all required tasks
- [ ] Request submission enabled when ready
- [ ] Nil return flow works correctly
