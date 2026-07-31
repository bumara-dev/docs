---
title: "Invoice Status Lifecycle Refactor"
description: "Design: split the conflated invoice status enum into separate document and payment status columns with explicit transition rules."
---

**Date:** 2026-05-26
**Status:** Approved

## Problem

The current `invoiceStatusEnum` (`draft`, `pending`, `sent`, `viewed`, `partial`, `paid`, `overdue`, `cancelled`, `void`) conflates document lifecycle with payment tracking. The payment service overwrites `status` to `paid`/`partial` when allocations happen, destroying the document's lifecycle state. There is no explicit approval gate — inventory and ledger side effects fire inconsistently. Nothing prevents editing an invoice after ZRA submission.

## Design

### Two Separate Status Columns

**Document Status** (`status` column — replaces current enum):

```
draft → approved → fulfilled
  │        │
  └──→ cancelled ←── approved (only if no payments + no ZRA transmission)
```

- `draft`: Editable. No side effects. Can be deleted.
- `approved`: Triggers inventory deduction/receipt, ledger entry, and ZRA smart invoice posting. Limited editability based on ZRA state.
- `fulfilled`: ZRA transmission validated. Completely immutable.
- `cancelled`: Terminal state. Reversals applied if transitioning from `approved`.

**Payment Status** (`paymentStatus` column — new):

```
unpaid → partial → paid
```

- Managed exclusively by the payment allocation service.
- Independent of document lifecycle.
- Default: `unpaid`.

Both columns apply to `sales_invoices` and `purchase_invoices`.

### Status Transition Rules

| From | To | Conditions |
|---|---|---|
| `draft` | `approved` | Always allowed |
| `draft` | `cancelled` | Always allowed |
| `approved` | `fulfilled` | Only via ZRA validation callback |
| `approved` | `cancelled` | Only if `paymentStatus = unpaid` AND no ZRA transmission exists |

Invalid transitions are rejected with `BAD_REQUEST`.

### Side Effects by Transition

| Transition | Inventory | Ledger | ZRA Posting |
|---|---|---|---|
| Create as `draft` | None | None | None |
| `draft` → `approved` (sales) | Deduct stock | Record debtor entry (debit customer) | Initiate transmission |
| `draft` → `approved` (purchase) | Receive stock | Record creditor entry (debit vendor) | Initiate transmission |
| `approved` → `fulfilled` | None (already done) | None (already done) | None (this IS the callback) |
| `draft` → `cancelled` | None | None | None |
| `approved` → `cancelled` (sales) | Reverse stock deduction | Reverse ledger entry | Blocked if ZRA transmitted |
| `approved` → `cancelled` (purchase) | Reverse stock receipt | Reverse ledger entry | Blocked if ZRA transmitted |

### Immutability Rules

| State | Can Edit | Can Delete | Constraints |
|---|---|---|---|
| `draft` | Full | Yes | — |
| `approved` (no ZRA transmission) | Limited: `customerNotes`, `termsAndConditions`, `internalNotes`, `dueDate` only | No | No amount/line item/customer/vendor changes |
| `approved` (ZRA transmitted) | No | No | — |
| `fulfilled` | No | No | — |
| `cancelled` | No | No | — |

### Cancellation Guards

Cancelling an `approved` invoice requires ALL of:
1. `paymentStatus` must be `unpaid` (no payments allocated)
2. No ZRA smart invoice transmission exists for this invoice
3. If both pass: reverse inventory and ledger, then set status to `cancelled`

If payments exist, the user must reverse payments first. If ZRA transmission exists, cancellation is blocked entirely.

## Database Changes

### 1. Replace `invoiceStatusEnum`

```sql
-- Old: draft, pending, sent, viewed, partial, paid, overdue, cancelled, void
-- New: draft, approved, fulfilled, cancelled
```

Since PostgreSQL enums cannot have values removed, create a new enum:

```sql
CREATE TYPE invoice_document_status AS ENUM ('draft', 'approved', 'fulfilled', 'cancelled');
```

### 2. New `invoicePaymentStatusEnum`

```sql
CREATE TYPE invoice_payment_status AS ENUM ('unpaid', 'partial', 'paid');
```

### 3. Alter `sales_invoices`

```sql
ALTER TABLE sales_invoices
  ADD COLUMN document_status invoice_document_status NOT NULL DEFAULT 'draft',
  ADD COLUMN payment_status invoice_payment_status NOT NULL DEFAULT 'unpaid';

-- Migrate data:
-- draft → draft/unpaid
-- sent, pending, viewed → approved/unpaid
-- partial → approved/partial
-- paid → approved/paid
-- overdue → approved/unpaid (overdue is a computed state, not stored)
-- cancelled → cancelled/unpaid
-- void → cancelled/unpaid

-- Then drop old status column and rename document_status → status
```

### 4. Alter `purchase_invoices`

Same as sales_invoices.

### 5. Existing columns preserved

- `amountPaid`, `amountDue`, `paidDate` — still tracked on the invoice, updated by payment service alongside `paymentStatus`.

## Service Layer Changes

### New Functions

**`approveInvoice(ctx, deps, invoiceId, locationId?)`** — Sales invoice approval:
1. Validate invoice exists and `status === 'draft'`
2. Set `status = 'approved'`
3. Deduct inventory (if locationId provided)
4. Record debtor ledger entry
5. Initiate ZRA smart invoice transmission
6. Audit log

**`approvePurchaseInvoice(ctx, deps, invoiceId, locationId?)`** — Purchase invoice approval:
1. Validate invoice exists and `status === 'draft'`
2. Set `status = 'approved'`
3. Receive inventory (if locationId provided)
4. Record creditor ledger entry
5. Initiate ZRA smart invoice transmission
6. Audit log

**`cancelInvoice(ctx, deps, invoiceId)`** — Sales invoice cancellation:
1. Validate `status` is `draft` or `approved`
2. If `approved`: check `paymentStatus === 'unpaid'`, check no ZRA transmission
3. If `approved`: reverse inventory + ledger
4. Set `status = 'cancelled'`
5. Audit log

**`cancelPurchaseInvoice(ctx, deps, invoiceId)`** — Same pattern for purchases.

### Modified Functions

**`createInvoice`** / **`createPurchaseInvoice`**:
- Always creates with `status = 'draft'`, `paymentStatus = 'unpaid'`
- Remove all side effects (inventory, ledger) — these move to approve functions
- Remove status parameter from input (always draft on create)

**`updateInvoice`** / **`updatePurchaseInvoice`**:
- Block if `status !== 'draft'` (full block for non-draft)
- Exception: if `status === 'approved'` and no ZRA transmission, allow only: `customerNotes`, `termsAndConditions`, `internalNotes`, `dueDate`
- Remove status from updatable fields (transitions via dedicated functions only)
- Remove inventory/ledger side effects (already handled by approve)

**`deleteInvoice`** / **`deletePurchaseInvoice`**:
- Block if `status !== 'draft'`
- Remove ledger reversal (draft invoices have no ledger entries)

### Payment Service Changes

**`updateInvoiceBalances()`**:
- Update `paymentStatus` instead of `status`:
  - `amountDue <= 0` → `paymentStatus = 'paid'`
  - `totalPaid > 0` → `paymentStatus = 'partial'`
  - else → `paymentStatus = 'unpaid'`
- Do NOT touch `status` column

### ZRA Smart Invoice Service Changes

**`updateTransmissionStatus()`**:
- When ZRA validates (status = "validated"), set invoice `status = 'fulfilled'`
- Currently only sets `zraInvoiceId` — add status transition

### Recurring Invoice Service

**`generateDueInvoices()`**:
- Currently creates invoices with `status: 'draft'` — this is correct, no changes needed
- Auto-generated invoices remain draft until explicitly approved

## Schema (Zod) Changes

### New Enums

```typescript
const INVOICE_DOCUMENT_STATUSES = ['draft', 'approved', 'fulfilled', 'cancelled'] as const;
const INVOICE_PAYMENT_STATUSES = ['unpaid', 'partial', 'paid'] as const;
```

### Input Schemas

- `createSalesInvoiceInputSchema`: Remove `status` field (always draft)
- `updateSalesInvoiceInputSchema`: Remove `status` field (transitions via dedicated endpoints)
- `createPurchaseInvoiceInputSchema`: Remove `status` field
- `updatePurchaseInvoiceInputSchema`: Remove `status` field
- `listInvoicesParamsSchema`: Add `paymentStatus` filter, update `status` to new enum values
- `listPurchaseInvoicesParamsSchema`: Same

## API Route Changes

### New Endpoints

| Method | Path | Description |
|---|---|---|
| POST | `/invoices/sales/{id}/approve` | Approve sales invoice |
| POST | `/invoices/sales/{id}/cancel` | Cancel sales invoice |
| POST | `/invoices/purchases/{id}/approve` | Approve purchase invoice |
| POST | `/invoices/purchases/{id}/cancel` | Cancel purchase invoice |

### Modified Endpoints

- POST/PUT create/update: no longer accept `status` in body
- GET list: accept `paymentStatus` query param alongside `status`

## Frontend Changes

### Status Badge Mapping

| Document Status | Badge Color | Label |
|---|---|---|
| `draft` | Gray | Draft |
| `approved` | Blue | Approved |
| `fulfilled` | Green | Fulfilled |
| `cancelled` | Red | Cancelled |

| Payment Status | Badge Color | Label |
|---|---|---|
| `unpaid` | Orange | Unpaid |
| `partial` | Yellow | Partially Paid |
| `paid` | Green | Paid |

Both badges shown on invoice detail and list rows.

### Detail Page Actions

| Document Status | Available Actions |
|---|---|
| `draft` | Edit, Delete, Approve |
| `approved` (no ZRA) | Cancel, Record Payment, Email, Download PDF |
| `approved` (ZRA transmitted) | Record Payment, Email, Download PDF |
| `fulfilled` | Record Payment, Email, Download PDF |
| `cancelled` | (none) |

### List Page Filters

Two filter dropdowns:
- **Status**: All, Draft, Approved, Fulfilled, Cancelled
- **Payment**: All, Unpaid, Partial, Paid

### Form Changes

- Sales invoice form: Remove status dropdown. Save always creates/updates as draft.
- Purchase invoice form: Remove status dropdown. Save always creates/updates as draft.
- "Save & Approve" button option on forms (creates as draft then immediately approves).

### Sales Invoice Form Submit

Current "Save & Send" becomes "Save & Approve":
- First saves the invoice as draft
- Then calls the approve endpoint
- Shows confirmation dialog: "Approving will deduct inventory, record ledger entries, and submit to ZRA. Continue?"

## Data Migration Strategy

Map existing statuses to new model:

| Old Status | New `status` | New `paymentStatus` |
|---|---|---|
| `draft` | `draft` | `unpaid` |
| `pending` | `approved` | `unpaid` |
| `sent` | `approved` | `unpaid` |
| `viewed` | `approved` | `unpaid` |
| `partial` | `approved` | `partial` |
| `paid` | `approved` | `paid` |
| `overdue` | `approved` | `unpaid` |
| `cancelled` | `cancelled` | `unpaid` |
| `void` | `cancelled` | `unpaid` |

Invoices with existing `zraInvoiceId` set → `status = 'fulfilled'`.

## Files to Modify

### Database
1. `packages/database/src/schema/enums.ts` — new enums
2. `packages/database/src/schema/invoicing/sales-invoices.ts` — replace status column, add paymentStatus
3. `packages/database/src/schema/invoicing/purchase-invoices.ts` — same
4. New migration file via `drizzle-kit generate`

### Service Layer
5. `packages/api-services/src/domains/invoicing/invoices.service.ts` — refactor create/update/delete, add approve/cancel functions
6. `packages/api-services/src/domains/invoicing/payments.service.ts` — update paymentStatus instead of status
7. `packages/api-services/src/domains/invoicing/zra-smart-invoice.service.ts` — set fulfilled on validation
8. `packages/api-services/src/domains/invoicing/invoicing.schema.ts` — new enums, remove status from create/update inputs

### API Routes
9. `apps/api-invoicing/src/routes/invoicing/sales-invoice-routes.ts` — add approve/cancel routes
10. `apps/api-invoicing/src/routes/invoicing/sales-invoice-handlers.ts` — add approve/cancel handlers
11. `apps/api-invoicing/src/routes/invoicing/purchase-invoice-routes.ts` — add approve/cancel routes
12. `apps/api-invoicing/src/routes/invoicing/purchase-invoice-handlers.ts` — add approve/cancel handlers

### Frontend
13. `apps/app/components/features/invoicing/sales-invoices/sales-invoice-detail.tsx` — dual badges, approve/cancel actions
14. `apps/app/components/features/invoicing/sales-invoices/sales-invoices-list.tsx` — dual filters, dual badges
15. `apps/app/components/features/invoicing/sales-invoices/sales-invoice-form.tsx` — remove status dropdown, add Save & Approve
16. `apps/app/components/features/invoicing/purchase-invoices/purchase-invoice-detail.tsx` — same
17. `apps/app/components/features/invoicing/purchase-invoices/purchase-invoices-list.tsx` — same
18. `apps/app/components/features/invoicing/purchase-invoices/purchase-invoice-form.tsx` — same
19. `apps/app/lib/queries/invoicing/types.ts` — update status types
20. `apps/app/lib/queries/invoicing/hooks/` — add useApproveInvoice, useCancelInvoice mutations
