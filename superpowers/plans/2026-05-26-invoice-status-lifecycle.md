---
title: "Invoice Status Lifecycle Refactor — Implementation Plan"
description: "For agentic workers: REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this..."
---

**Goal:** Split the invoice status enum into document status (`draft`/`approved`/`fulfilled`/`cancelled`) and payment status (`unpaid`/`partial`/`paid`), move side effects to an explicit approval step, and guard immutability for ZRA-transmitted invoices.

**Architecture:** Add new DB enums + `paymentStatus` column to both invoice tables via migration. Create dedicated `approveInvoice`/`cancelInvoice` service functions that centralize inventory, ledger, and ZRA side effects. The payment service updates only `paymentStatus`. Frontend adds dual status badges, approval confirmation dialog, and updated filters.

**Tech Stack:** Drizzle ORM (PostgreSQL), Hono OpenAPI, Zod, React/TanStack Query, shadcn/ui

**Spec:** `docs/superpowers/specs/2026-05-26-invoice-status-lifecycle-design.md`

---

### Task 1: Add new database enums and columns

**Files:**
- Modify: `packages/database/src/schema/enums.ts`
- Modify: `packages/database/src/schema/invoicing/sales-invoices.ts`
- Modify: `packages/database/src/schema/invoicing/purchase-invoices.ts`

- [ ] **Step 1: Add new enums to `enums.ts`**

Add two new enums after the existing `invoiceStatusEnum` (which stays for now — migration handles the transition):

```typescript
// In packages/database/src/schema/enums.ts, after invoiceStatusEnum:

export const invoiceDocumentStatusEnum = pgEnum('invoice_document_status', [
  'draft',
  'approved',
  'fulfilled',
  'cancelled',
]);

export const invoicePaymentStatusEnum = pgEnum('invoice_payment_status', [
  'unpaid',
  'partial',
  'paid',
]);
```

- [ ] **Step 2: Update `sales-invoices.ts` schema**

Replace the `status` import and column, add `paymentStatus`:

```typescript
// In packages/database/src/schema/invoicing/sales-invoices.ts
// Change import:
import { invoiceDocumentStatusEnum, invoicePaymentStatusEnum, invoiceTypeEnum } from '../enums';

// Replace the status column:
status: invoiceDocumentStatusEnum('status').default('draft').notNull(),

// Add paymentStatus after the status column:
paymentStatus: invoicePaymentStatusEnum('payment_status').default('unpaid').notNull(),
```

Also add an index for the new payment_status column in the table's index array:

```typescript
index('idx_invoices_payment_status').on(table.paymentStatus),
```

- [ ] **Step 3: Update `purchase-invoices.ts` schema**

Same changes as sales-invoices — replace the `status` import/column, add `paymentStatus` column and index. The import line changes from:

```typescript
import { invoiceStatusEnum, invoiceTypeEnum } from '../enums';
```

to:

```typescript
import { invoiceDocumentStatusEnum, invoicePaymentStatusEnum, invoiceTypeEnum } from '../enums';
```

Replace the status column:

```typescript
status: invoiceDocumentStatusEnum('status').default('draft').notNull(),
paymentStatus: invoicePaymentStatusEnum('payment_status').default('unpaid').notNull(),
```

Add index:

```typescript
index('idx_purchase_invoice_payment_status').on(table.paymentStatus),
```

- [ ] **Step 4: Generate the migration**

Run:

```bash
cd packages/database && npx drizzle-kit generate
```

This will produce a new migration file. The auto-generated migration will create the new enum types and add the `payment_status` column. However, it may also try to change the `status` column type. If drizzle-kit generates a migration that drops and recreates the status column, you'll need to edit the migration SQL to use a safe approach instead.

- [ ] **Step 5: Edit the migration SQL for data safety**

Open the generated migration file in `packages/database/drizzle/`. Ensure the migration:

1. Creates the new enum types
2. Adds the `payment_status` column with default `'unpaid'`
3. Migrates the `status` column data safely

If drizzle-kit didn't handle the status column type change, add this SQL at the end of the migration:

```sql
-- Migrate existing status data to payment_status before changing status column type
UPDATE sales_invoices SET payment_status = 'paid' WHERE status = 'paid';
UPDATE sales_invoices SET payment_status = 'partial' WHERE status = 'partial';

UPDATE purchase_invoices SET payment_status = 'paid' WHERE status = 'paid';
UPDATE purchase_invoices SET payment_status = 'partial' WHERE status = 'partial';

-- Mark invoices with zra_invoice_id as fulfilled
UPDATE sales_invoices SET status = 'fulfilled' WHERE zra_invoice_id IS NOT NULL AND status NOT IN ('draft', 'cancelled');
UPDATE purchase_invoices SET status = 'fulfilled' WHERE zra_invoice_id IS NOT NULL AND status NOT IN ('draft', 'cancelled');
```

The exact migration will depend on what drizzle-kit generates. The key is: populate `payment_status` from old status values before changing the status column type.

- [ ] **Step 6: Update the drizzle meta journal**

Verify the generated migration appears in `packages/database/drizzle/meta/_journal.json`.

- [ ] **Step 7: Commit**

```bash
git add packages/database/src/schema/enums.ts packages/database/src/schema/invoicing/sales-invoices.ts packages/database/src/schema/invoicing/purchase-invoices.ts packages/database/drizzle/
git commit -m "feat: add invoice document status and payment status enums with migration"
```

---

### Task 2: Update Zod schemas and type exports

**Files:**
- Modify: `packages/api-services/src/domains/invoicing/invoicing.schema.ts`
- Modify: `apps/app/lib/queries/invoicing/types.ts`

- [ ] **Step 1: Add new status constants in `invoicing.schema.ts`**

At the top of the file where other enum constants are defined (around line 27), add:

```typescript
import { invoiceDocumentStatusEnum, invoicePaymentStatusEnum } from "@/schema";
```

Then add the constants:

```typescript
const INVOICE_DOCUMENT_STATUSES = invoiceDocumentStatusEnum.enumValues;
const INVOICE_PAYMENT_STATUSES = invoicePaymentStatusEnum.enumValues;
```

- [ ] **Step 2: Update `createSalesInvoiceInputSchema`**

Remove the `status` field from the schema (around line 294). The invoice is always created as draft — no user choice. Delete:

```typescript
  status: z.enum(INVOICE_STATUSES).default("draft"),
```

- [ ] **Step 3: Update `updateSalesInvoiceInputSchema`**

Since it's `createSalesInvoiceInputSchema.partial()`, removing `status` from create automatically removes it from update too. No extra change needed.

- [ ] **Step 4: Update `listInvoicesParamsSchema`**

Change the status filter to use new document statuses and add a paymentStatus filter. Replace:

```typescript
  status: z.enum(INVOICE_STATUSES).optional(),
```

With:

```typescript
  status: z.enum(INVOICE_DOCUMENT_STATUSES).optional(),
  paymentStatus: z.enum(INVOICE_PAYMENT_STATUSES).optional(),
```

- [ ] **Step 5: Update `createPurchaseInvoiceInputSchema`**

Remove the `status` field (around line 408). Delete:

```typescript
  status: z.enum(INVOICE_STATUSES).default("draft"),
```

- [ ] **Step 6: Update `listPurchaseInvoicesParamsSchema`**

Same as sales — replace `status` enum and add `paymentStatus`:

```typescript
  status: z.enum(INVOICE_DOCUMENT_STATUSES).optional(),
  paymentStatus: z.enum(INVOICE_PAYMENT_STATUSES).optional(),
```

- [ ] **Step 7: Add approve input schema**

Add near the other invoice schemas:

```typescript
export const approveInvoiceInputSchema = z.object({
  locationId: z.string().uuid().optional(),
});

export type ApproveInvoiceInput = z.infer<typeof approveInvoiceInputSchema>;
```

- [ ] **Step 8: Update frontend types in `types.ts`**

In `apps/app/lib/queries/invoicing/types.ts`, update the `SalesInvoice` and `PurchaseInvoice` types to include:

```typescript
export type InvoiceDocumentStatus = 'draft' | 'approved' | 'fulfilled' | 'cancelled';
export type InvoicePaymentStatus = 'unpaid' | 'partial' | 'paid';
```

And update the invoice interfaces to use `status: InvoiceDocumentStatus` and add `paymentStatus: InvoicePaymentStatus`.

- [ ] **Step 9: Commit**

```bash
git add packages/api-services/src/domains/invoicing/invoicing.schema.ts apps/app/lib/queries/invoicing/types.ts
git commit -m "feat: update Zod schemas and types for invoice status lifecycle"
```

---

### Task 3: Refactor sales invoice service — approve, cancel, guards

**Files:**
- Modify: `packages/api-services/src/domains/invoicing/invoices.service.ts`
- Modify: `packages/api-services/src/domains/invoicing/index.ts`

- [ ] **Step 1: Refactor `createInvoice` — remove side effects**

In `invoices.service.ts`, modify `createInvoice` (starting at line 106):

1. Remove the `status` from the spread — always set `status: "draft"` and `paymentStatus: "unpaid"` explicitly
2. Remove the entire ledger entry block (lines 147-162)
3. Remove the entire inventory deduction block (lines 164-174)
4. Remove the source estimate conversion block — keep this, it's not a side effect of approval

The function should now just: create the invoice record, create line items, record audit log, handle source estimate. No ledger. No inventory.

```typescript
export async function createInvoice(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  input: CreateSalesInvoiceInput
): Promise<Invoice> {
  const orgId = requireOrganizationContext(ctx);
  const { lineItems, ...invoiceData } = input;

  const invoiceNumber = await getNextDocumentNumber(deps.db, orgId, "sales_invoice");

  const invoice = await createInvoiceRepo(deps.db, {
    ...invoiceData,
    invoiceNumber,
    organizationId: orgId,
    status: "draft",
    paymentStatus: "unpaid",
  });

  if (lineItems && lineItems.length > 0) {
    await createInvoiceLineItemsBulk(
      deps.db,
      lineItems.map((item) => ({
        ...item,
        lineItemType: "sale",
        saleInvoiceId: invoice.id,
      }))
    );
  }

  recordAuditLog(ctx, deps, {
    action: "create",
    entityType: "invoicing_invoice",
    entityId: invoice.id,
    changes: { after: invoice },
  });

  if (input.sourceEstimateId) {
    try {
      const { updateEstimate: updateEstimateRepo } = await import("@repo/database/repositories");
      await updateEstimateRepo(deps.db, input.sourceEstimateId, {
        status: "converted",
        convertedInvoiceId: invoice.id,
      });
    } catch (error) {
      console.error("Failed to update source estimate:", error);
    }
  }

  return invoice;
}
```

- [ ] **Step 2: Add `approveInvoice` function**

Add after `createInvoice`:

```typescript
export async function approveInvoice(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  invoiceId: string,
  locationId?: string
): Promise<Invoice> {
  const orgId = requireOrganizationContext(ctx);
  const invoice = await findInvoiceById(deps.db, invoiceId, orgId);

  if (!invoice) {
    throw new ServiceError("NOT_FOUND", "Invoice not found");
  }

  if (invoice.status !== "draft") {
    throw new ServiceError("BAD_REQUEST", "Only draft invoices can be approved");
  }

  const updated = await updateInvoiceRepo(deps.db, invoiceId, orgId, {
    status: "approved",
  });

  if (!updated) {
    throw new ServiceError("NOT_FOUND", "Invoice not found");
  }

  recordAuditLog(ctx, deps, {
    action: "update",
    entityType: "invoicing_invoice",
    entityId: invoiceId,
    changes: { before: { status: "draft" }, after: { status: "approved" } },
  });

  // Side effect: Record debtor ledger entry
  await recordLedgerEntry({
    db: deps.db,
    organizationId: orgId,
    userId: ctx.userId ?? null,
    accountType: "debtor",
    customerId: invoice.customerId,
    transactionDate: invoice.invoiceDate,
    transactionType: "invoice",
    referenceId: invoiceId,
    referenceNumber: invoice.invoiceNumber,
    description: `Sales Invoice ${invoice.invoiceNumber}`,
    debitAmount: invoice.totalAmount,
    creditAmount: "0.00",
    currency: invoice.currency ?? undefined,
    exchangeRate: invoice.exchangeRate ?? undefined,
  });

  // Side effect: Deduct inventory
  if (locationId) {
    try {
      const { deductInventoryForInvoice } = await import("./inventory-link.service");
      await deductInventoryForInvoice(ctx, deps, invoiceId, locationId);
    } catch (error) {
      console.error("Failed to deduct inventory for invoice:", error);
    }
  }

  // Side effect: Initiate ZRA smart invoice transmission
  try {
    const { transmitInvoiceToZra } = await import("./zra-smart-invoice.service");
    await transmitInvoiceToZra(ctx, deps, invoiceId);
  } catch (error) {
    console.error("ZRA transmission failed (non-blocking):", error);
  }

  return updated;
}
```

- [ ] **Step 3: Add `cancelInvoice` function**

Add after `approveInvoice`:

```typescript
export async function cancelInvoice(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  invoiceId: string
): Promise<Invoice> {
  const orgId = requireOrganizationContext(ctx);
  const invoice = await findInvoiceById(deps.db, invoiceId, orgId);

  if (!invoice) {
    throw new ServiceError("NOT_FOUND", "Invoice not found");
  }

  if (invoice.status !== "draft" && invoice.status !== "approved") {
    throw new ServiceError("BAD_REQUEST", "Only draft or approved invoices can be cancelled");
  }

  if (invoice.status === "approved") {
    // Check no payments allocated
    if (invoice.paymentStatus !== "unpaid") {
      throw new ServiceError("BAD_REQUEST", "Cannot cancel invoice with payments. Reverse payments first.");
    }

    // Check no ZRA transmission
    const { findTransmissionByInvoice } = await import("@repo/database/repositories");
    const transmission = await findTransmissionByInvoice(deps.db, invoiceId);
    if (transmission) {
      throw new ServiceError("BAD_REQUEST", "Cannot cancel invoice that has been submitted to ZRA");
    }

    // Reverse ledger entry
    await reverseLedgerEntry({
      db: deps.db,
      organizationId: orgId,
      userId: ctx.userId ?? null,
      referenceId: invoiceId,
      referenceNumber: invoice.invoiceNumber,
      description: `Reversal: Sales Invoice ${invoice.invoiceNumber} (cancelled)`,
      transactionDate: new Date().toISOString().slice(0, 10),
    });
  }

  const updated = await updateInvoiceRepo(deps.db, invoiceId, orgId, {
    status: "cancelled",
  });

  if (!updated) {
    throw new ServiceError("NOT_FOUND", "Invoice not found");
  }

  recordAuditLog(ctx, deps, {
    action: "update",
    entityType: "invoicing_invoice",
    entityId: invoiceId,
    changes: { before: { status: invoice.status }, after: { status: "cancelled" } },
  });

  return updated;
}
```

- [ ] **Step 4: Guard `updateInvoice`**

Replace the current `updateInvoice` function with guards:

```typescript
export async function updateInvoice(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  invoiceId: string,
  input: UpdateSalesInvoiceInput
): Promise<Invoice> {
  const orgId = requireOrganizationContext(ctx);
  const { lineItems, ...invoiceData } = input;

  const existing = await findInvoiceById(deps.db, invoiceId, orgId);

  if (!existing) {
    throw new ServiceError("NOT_FOUND", "Invoice not found");
  }

  // Immutability guards
  if (existing.status === "fulfilled" || existing.status === "cancelled") {
    throw new ServiceError("BAD_REQUEST", `Cannot edit a ${existing.status} invoice`);
  }

  if (existing.status === "approved") {
    const { findTransmissionByInvoice } = await import("@repo/database/repositories");
    const transmission = await findTransmissionByInvoice(deps.db, invoiceId);
    if (transmission) {
      throw new ServiceError("BAD_REQUEST", "Cannot edit invoice that has been submitted to ZRA");
    }
    // Only allow limited fields for approved invoices
    const allowedFields = ["customerNotes", "termsAndConditions", "internalNotes", "dueDate"];
    const inputKeys = Object.keys(invoiceData);
    const disallowed = inputKeys.filter((k) => !allowedFields.includes(k));
    if (disallowed.length > 0) {
      throw new ServiceError("BAD_REQUEST", `Cannot modify ${disallowed.join(", ")} on an approved invoice`);
    }
  }

  const updated = await updateInvoiceRepo(deps.db, invoiceId, orgId, invoiceData);

  if (!updated) {
    throw new ServiceError("NOT_FOUND", "Invoice not found");
  }

  // Only allow line item changes for draft invoices
  if (lineItems && lineItems.length > 0 && existing.status === "draft") {
    await deleteInvoiceLineItems(deps.db, invoiceId);
    await createInvoiceLineItemsBulk(
      deps.db,
      lineItems.map((item) => ({
        ...item,
        saleInvoiceId: invoiceId,
      }))
    );
  }

  recordAuditLog(ctx, deps, {
    action: "update",
    entityType: "invoicing_invoice",
    entityId: updated.id,
    changes: { before: existing, after: updated },
  });

  return updated;
}
```

- [ ] **Step 5: Guard `deleteInvoice`**

Replace the function — only drafts can be deleted, no ledger reversal needed:

```typescript
export async function deleteInvoice(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  invoiceId: string
): Promise<void> {
  const orgId = requireOrganizationContext(ctx);
  const existing = await findInvoiceById(deps.db, invoiceId, orgId);

  if (!existing) {
    throw new ServiceError("NOT_FOUND", "Invoice not found");
  }

  if (existing.status !== "draft") {
    throw new ServiceError("BAD_REQUEST", "Only draft invoices can be deleted");
  }

  const deleted = await deleteInvoiceRepo(deps.db, invoiceId, orgId);

  if (!deleted) {
    throw new ServiceError("NOT_FOUND", "Invoice not found");
  }

  recordAuditLog(ctx, deps, {
    action: "delete",
    entityType: "invoicing_invoice",
    entityId: existing.id,
  });
}
```

- [ ] **Step 6: Export new functions from `index.ts`**

In `packages/api-services/src/domains/invoicing/index.ts`, add to the invoices.service exports:

```typescript
    approveInvoice,
    approvePurchaseInvoice,
    cancelInvoice,
    cancelPurchaseInvoice,
```

- [ ] **Step 7: Commit**

```bash
git add packages/api-services/src/domains/invoicing/invoices.service.ts packages/api-services/src/domains/invoicing/index.ts
git commit -m "feat: add approve/cancel for sales invoices, guard update/delete immutability"
```

---

### Task 4: Refactor purchase invoice service — approve, cancel, guards

**Files:**
- Modify: `packages/api-services/src/domains/invoicing/invoices.service.ts`

- [ ] **Step 1: Refactor `createPurchaseInvoice` — remove side effects**

Same pattern as sales: remove ledger entry and inventory receipt. Always set `status: "draft"` and `paymentStatus: "unpaid"`. Keep the vendor invoice number duplicate check.

Remove the ledger recording block (lines ~438-453) and the inventory receipt block (lines ~455-470).

Add explicit status fields to the create call:

```typescript
const purchaseInvoice = await createPurchaseInvoiceRepo(deps.db, {
  ...purchaseInvoiceData,
  referenceNumber,
  vendorInvoiceNumber,
  organizationId: orgId,
  status: "draft",
  paymentStatus: "unpaid",
});
```

- [ ] **Step 2: Add `approvePurchaseInvoice` function**

```typescript
export async function approvePurchaseInvoice(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  purchaseInvoiceId: string,
  locationId?: string
): Promise<PurchaseInvoice> {
  const orgId = requireOrganizationContext(ctx);
  const invoice = await findPurchaseInvoiceById(deps.db, purchaseInvoiceId, orgId);

  if (!invoice) {
    throw new ServiceError("NOT_FOUND", "Purchase invoice not found");
  }

  if (invoice.status !== "draft") {
    throw new ServiceError("BAD_REQUEST", "Only draft purchase invoices can be approved");
  }

  const updated = await updatePurchaseInvoiceRepo(deps.db, purchaseInvoiceId, orgId, {
    status: "approved",
  });

  if (!updated) {
    throw new ServiceError("NOT_FOUND", "Purchase invoice not found");
  }

  recordAuditLog(ctx, deps, {
    action: "update",
    entityType: "invoicing_invoice",
    entityId: purchaseInvoiceId,
    changes: { before: { status: "draft" }, after: { status: "approved" } },
  });

  // Side effect: Record creditor ledger entry
  await recordLedgerEntry({
    db: deps.db,
    organizationId: orgId,
    userId: ctx.userId ?? null,
    accountType: "creditor",
    vendorId: invoice.vendorId,
    transactionDate: invoice.invoiceDate,
    transactionType: "invoice",
    referenceId: purchaseInvoiceId,
    referenceNumber: invoice.referenceNumber,
    description: `Purchase Invoice ${invoice.referenceNumber}`,
    debitAmount: invoice.totalAmount,
    creditAmount: "0.00",
    currency: invoice.currency ?? undefined,
    exchangeRate: invoice.exchangeRate ?? undefined,
  });

  // Side effect: Receive inventory
  if (locationId) {
    try {
      const { receiveInventoryForPurchaseInvoice } = await import("./inventory-link.service");
      await receiveInventoryForPurchaseInvoice(ctx, deps, purchaseInvoiceId, locationId);
    } catch (error) {
      console.error("Failed to receive inventory for purchase invoice:", error);
    }
  }

  // Side effect: Initiate ZRA smart invoice transmission for purchase
  try {
    const { transmitInvoiceToZra } = await import("./zra-smart-invoice.service");
    await transmitInvoiceToZra(ctx, deps, purchaseInvoiceId);
  } catch (error) {
    console.error("ZRA transmission failed (non-blocking):", error);
  }

  return updated;
}
```

- [ ] **Step 3: Add `cancelPurchaseInvoice` function**

Same pattern as sales cancel but for purchase invoices:

```typescript
export async function cancelPurchaseInvoice(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  purchaseInvoiceId: string
): Promise<PurchaseInvoice> {
  const orgId = requireOrganizationContext(ctx);
  const invoice = await findPurchaseInvoiceById(deps.db, purchaseInvoiceId, orgId);

  if (!invoice) {
    throw new ServiceError("NOT_FOUND", "Purchase invoice not found");
  }

  if (invoice.status !== "draft" && invoice.status !== "approved") {
    throw new ServiceError("BAD_REQUEST", "Only draft or approved purchase invoices can be cancelled");
  }

  if (invoice.status === "approved") {
    if (invoice.paymentStatus !== "unpaid") {
      throw new ServiceError("BAD_REQUEST", "Cannot cancel purchase invoice with payments. Reverse payments first.");
    }

    const { findTransmissionByInvoice } = await import("@repo/database/repositories");
    const transmission = await findTransmissionByInvoice(deps.db, purchaseInvoiceId);
    if (transmission) {
      throw new ServiceError("BAD_REQUEST", "Cannot cancel purchase invoice that has been submitted to ZRA");
    }

    await reverseLedgerEntry({
      db: deps.db,
      organizationId: orgId,
      userId: ctx.userId ?? null,
      referenceId: purchaseInvoiceId,
      referenceNumber: invoice.referenceNumber,
      description: `Reversal: Purchase Invoice ${invoice.referenceNumber} (cancelled)`,
      transactionDate: new Date().toISOString().slice(0, 10),
    });
  }

  const updated = await updatePurchaseInvoiceRepo(deps.db, purchaseInvoiceId, orgId, {
    status: "cancelled",
  });

  if (!updated) {
    throw new ServiceError("NOT_FOUND", "Purchase invoice not found");
  }

  recordAuditLog(ctx, deps, {
    action: "update",
    entityType: "invoicing_invoice",
    entityId: purchaseInvoiceId,
    changes: { before: { status: invoice.status }, after: { status: "cancelled" } },
  });

  return updated;
}
```

- [ ] **Step 4: Guard `updatePurchaseInvoice`**

Add the same immutability guards as sales invoices. Block if `fulfilled` or `cancelled`. If `approved` without ZRA transmission, allow only `notes`, `termsAndConditions`, `internalNotes`, `dueDate`. Remove the old inventory/ledger side effects.

- [ ] **Step 5: Guard `deletePurchaseInvoice`**

Only allow deletion of `draft` purchase invoices. Remove ledger reversal (no ledger entry exists for drafts).

- [ ] **Step 6: Commit**

```bash
git add packages/api-services/src/domains/invoicing/invoices.service.ts
git commit -m "feat: add approve/cancel for purchase invoices, guard update/delete immutability"
```

---

### Task 5: Update payment service to use `paymentStatus`

**Files:**
- Modify: `packages/api-services/src/domains/invoicing/payments.service.ts`

- [ ] **Step 1: Update `updateInvoiceBalances` function**

In `payments.service.ts`, change the `updateInvoiceBalances` function (around line 202) to update `paymentStatus` instead of `status`:

```typescript
async function updateInvoiceBalances(
  deps: ServiceDependencies,
  invoiceId: string,
  organizationId: string
): Promise<void> {
  const invoice = await findInvoiceById(deps.db, invoiceId, organizationId);
  if (!invoice) return;

  const allocations = await listAllocationsByInvoice(deps.db, invoiceId);
  const totalPaid = allocations.reduce(
    (sum, a) => sum + Number.parseFloat(a.amount),
    0
  );
  const totalAmount = Number.parseFloat(invoice.totalAmount);
  const amountDue = Math.max(0, totalAmount - totalPaid);

  let paymentStatus: "unpaid" | "partial" | "paid" = "unpaid";
  if (amountDue <= 0) {
    paymentStatus = "paid";
  } else if (totalPaid > 0) {
    paymentStatus = "partial";
  }

  await updateInvoice(deps.db, invoiceId, organizationId, {
    amountPaid: totalPaid.toFixed(2),
    amountDue: amountDue.toFixed(2),
    paymentStatus,
    paidDate: amountDue <= 0 ? new Date().toISOString().split("T")[0] : null,
  });
}
```

Key change: `status` is no longer modified — only `paymentStatus`, `amountPaid`, `amountDue`, and `paidDate`.

- [ ] **Step 2: Commit**

```bash
git add packages/api-services/src/domains/invoicing/payments.service.ts
git commit -m "feat: payment service updates paymentStatus instead of document status"
```

---

### Task 6: Update ZRA smart invoice service — set `fulfilled` on validation

**Files:**
- Modify: `packages/api-services/src/domains/invoicing/zra-smart-invoice.service.ts`

- [ ] **Step 1: Update `handleZraValidationWebhook`**

In `zra-smart-invoice.service.ts`, find the `handleZraValidationWebhook` function (around line 540). Currently when ZRA validates, it sets `zraInvoiceId`. Add setting `status: "fulfilled"`:

```typescript
  // If validated, update the invoice status to fulfilled
  if (payload.status === "validated") {
    await updateInvoice(
      deps.db,
      transmission.salesInvoiceId,
      transmission.organizationId,
      {
        zraInvoiceId: transmission.id,
        status: "fulfilled",
      }
    );
  }
```

- [ ] **Step 2: Commit**

```bash
git add packages/api-services/src/domains/invoicing/zra-smart-invoice.service.ts
git commit -m "feat: set invoice status to fulfilled on ZRA validation"
```

---

### Task 7: Add approve/cancel API routes and handlers

**Files:**
- Modify: `apps/api-invoicing/src/routes/invoicing/sales-invoices/routes.ts`
- Modify: `apps/api-invoicing/src/routes/invoicing/sales-invoices/handlers.ts`
- Modify: `apps/api-invoicing/src/routes/invoicing/sales-invoices/index.ts`
- Modify: `apps/api-invoicing/src/routes/invoicing/purchase-invoices/routes.ts`
- Modify: `apps/api-invoicing/src/routes/invoicing/purchase-invoices/handlers.ts`
- Modify: `apps/api-invoicing/src/routes/invoicing/purchase-invoices/index.ts`

- [ ] **Step 1: Add sales invoice approve/cancel routes**

In `sales-invoices/routes.ts`, add after the `sendSalesInvoiceRoute`:

```typescript
import { approveInvoiceInputSchema } from "@repo/api-services/domains/invoicing";

export const approveSalesInvoiceRoute = createRoute({
  tags,
  method: "post",
  path: "/invoicing/sales-invoices/{invoiceId}/approve",
  summary: "Approve sales invoice",
  request: {
    params: z.object({ invoiceId: z.string().uuid() }),
    body: jsonContentRequired(approveInvoiceInputSchema, "Approval options"),
  },
  middleware: [requireAuth, requireOrg],
  responses: {
    [HttpStatusCodes.OK]: jsonContent(successResponseSchema, "Sales invoice approved successfully"),
    [HttpStatusCodes.NOT_FOUND]: jsonContent(errorResponseSchema, "Sales invoice not found"),
    [HttpStatusCodes.BAD_REQUEST]: jsonContent(errorResponseSchema, "Cannot approve invoice"),
    [HttpStatusCodes.UNAUTHORIZED]: jsonContent(errorResponseSchema, "Unauthorized"),
    [HttpStatusCodes.INTERNAL_SERVER_ERROR]: jsonContent(errorResponseSchema, "Internal Server Error"),
  },
});

export const cancelSalesInvoiceRoute = createRoute({
  tags,
  method: "post",
  path: "/invoicing/sales-invoices/{invoiceId}/cancel",
  summary: "Cancel sales invoice",
  request: { params: z.object({ invoiceId: z.string().uuid() }) },
  middleware: [requireAuth, requireOrg],
  responses: {
    [HttpStatusCodes.OK]: jsonContent(successResponseSchema, "Sales invoice cancelled successfully"),
    [HttpStatusCodes.NOT_FOUND]: jsonContent(errorResponseSchema, "Sales invoice not found"),
    [HttpStatusCodes.BAD_REQUEST]: jsonContent(errorResponseSchema, "Cannot cancel invoice"),
    [HttpStatusCodes.UNAUTHORIZED]: jsonContent(errorResponseSchema, "Unauthorized"),
    [HttpStatusCodes.INTERNAL_SERVER_ERROR]: jsonContent(errorResponseSchema, "Internal Server Error"),
  },
});

export type ApproveSalesInvoiceRoute = typeof approveSalesInvoiceRoute;
export type CancelSalesInvoiceRoute = typeof cancelSalesInvoiceRoute;
```

- [ ] **Step 2: Add sales invoice approve/cancel handlers**

In `sales-invoices/handlers.ts`, add imports for `approveInvoice`, `cancelInvoice` from `@repo/api-services/domains/invoicing` and the route types. Then add:

```typescript
export const approveSalesInvoiceHandler: AppRouteHandler<ApproveSalesInvoiceRoute> = async (c) => {
  try {
    const ctx = buildServiceContext(c);
    const deps = buildServiceDependencies(c);
    const { invoiceId } = c.req.valid("param");
    const body = await c.req.json<{ locationId?: string }>();
    const invoice = await approveInvoice(ctx, deps, invoiceId, body.locationId);
    return c.json(
      { success: true, data: invoice, message: "Sales invoice approved successfully" },
      HttpStatusCodes.OK
    );
  } catch (error) {
    return handleServiceError(c, error, "Failed to approve sales invoice");
  }
};

export const cancelSalesInvoiceHandler: AppRouteHandler<CancelSalesInvoiceRoute> = async (c) => {
  try {
    const ctx = buildServiceContext(c);
    const deps = buildServiceDependencies(c);
    const { invoiceId } = c.req.valid("param");
    const invoice = await cancelInvoice(ctx, deps, invoiceId);
    return c.json(
      { success: true, data: invoice, message: "Sales invoice cancelled successfully" },
      HttpStatusCodes.OK
    );
  } catch (error) {
    return handleServiceError(c, error, "Failed to cancel sales invoice");
  }
};
```

- [ ] **Step 3: Register sales routes in `index.ts`**

In `sales-invoices/index.ts`, import the new routes and add to the router chain:

```typescript
import { approveSalesInvoiceRoute, cancelSalesInvoiceRoute } from "./routes";

// Add to router chain:
  .openapi(approveSalesInvoiceRoute, lazy("approveSalesInvoiceHandler"))
  .openapi(cancelSalesInvoiceRoute, lazy("cancelSalesInvoiceHandler"))
```

- [ ] **Step 4: Repeat for purchase invoices**

Add the same `approvePurchaseInvoiceRoute`, `cancelPurchaseInvoiceRoute` to `purchase-invoices/routes.ts` at paths:
- `POST /invoicing/purchase-invoices/{invoiceId}/approve`
- `POST /invoicing/purchase-invoices/{invoiceId}/cancel`

Add corresponding handlers in `purchase-invoices/handlers.ts` calling `approvePurchaseInvoice` and `cancelPurchaseInvoice`.

Register in `purchase-invoices/index.ts`.

- [ ] **Step 5: Remove or repurpose `sendSalesInvoiceRoute`**

The old `/send` route set `status: "sent"`. This status no longer exists. Remove `sendSalesInvoiceRoute`, its handler `sendSalesInvoiceHandler`, and the route registration from `index.ts`. The "send" action is now "approve". Also remove the type export.

- [ ] **Step 6: Commit**

```bash
git add apps/api-invoicing/src/routes/invoicing/sales-invoices/ apps/api-invoicing/src/routes/invoicing/purchase-invoices/
git commit -m "feat: add approve/cancel API endpoints, remove send route"
```

---

### Task 8: Add frontend fetchers and hooks for approve/cancel

**Files:**
- Modify: `apps/app/lib/queries/invoicing/fetchers/sales-invoices.ts`
- Modify: `apps/app/lib/queries/invoicing/fetchers/purchase-invoices.ts`
- Modify: `apps/app/lib/queries/invoicing/hooks/use-sales-invoices.ts`
- Modify: `apps/app/lib/queries/invoicing/hooks/use-purchase-invoices.ts`

- [ ] **Step 1: Add sales invoice fetchers**

In `fetchers/sales-invoices.ts`, add:

```typescript
export async function approveSalesInvoice(
  getToken: () => Promise<string | null>,
  id: string,
  locationId?: string
): Promise<SalesInvoice> {
  const headers = await authHeaders(getToken);
  const res = await fetch(`${BASE}/invoicing/sales-invoices/${id}/approve`, {
    method: "POST",
    headers,
    body: JSON.stringify({ locationId }),
  });
  await assertOk(res);
  const data = await res.json();
  return extractSuccessData<SalesInvoice>(data);
}

export async function cancelSalesInvoice(
  getToken: () => Promise<string | null>,
  id: string
): Promise<SalesInvoice> {
  const headers = await authHeaders(getToken);
  const res = await fetch(`${BASE}/invoicing/sales-invoices/${id}/cancel`, {
    method: "POST",
    headers,
  });
  await assertOk(res);
  const data = await res.json();
  return extractSuccessData<SalesInvoice>(data);
}
```

Remove the old `sendSalesInvoice` fetcher function.

- [ ] **Step 2: Add sales invoice hooks**

In `hooks/use-sales-invoices.ts`, replace `useSendSalesInvoice` with:

```typescript
export function useApproveSalesInvoice() {
  const { getToken } = useAuth();
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, locationId }: { id: string; locationId?: string }) =>
      approveSalesInvoice(getToken, id, locationId),
    onSuccess: (_data, variables) => {
      toast.success("Sales invoice approved successfully");
      queryClient.invalidateQueries({ queryKey: invoicingQueryKeys.salesInvoices.all });
      queryClient.invalidateQueries({ queryKey: invoicingQueryKeys.salesInvoices.detail(variables.id) });
      queryClient.invalidateQueries({ queryKey: invoicingQueryKeys.reports.dashboard });
    },
    onError: (error) => {
      toast.error(error.message || "Failed to approve sales invoice");
    },
  });
}

export function useCancelSalesInvoice() {
  const { getToken } = useAuth();
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (id: string) => cancelSalesInvoice(getToken, id),
    onSuccess: (_data, id) => {
      toast.success("Sales invoice cancelled");
      queryClient.invalidateQueries({ queryKey: invoicingQueryKeys.salesInvoices.all });
      queryClient.invalidateQueries({ queryKey: invoicingQueryKeys.salesInvoices.detail(id) });
    },
    onError: (error) => {
      toast.error(error.message || "Failed to cancel sales invoice");
    },
  });
}
```

Update the imports to use `approveSalesInvoice` and `cancelSalesInvoice` from the fetchers. Remove the import and export of `sendSalesInvoice`/`useSendSalesInvoice`.

- [ ] **Step 3: Add purchase invoice fetchers and hooks**

Same pattern for `fetchers/purchase-invoices.ts` and `hooks/use-purchase-invoices.ts` — add `approvePurchaseInvoice`, `cancelPurchaseInvoice` fetcher functions and `useApprovePurchaseInvoice`, `useCancelPurchaseInvoice` hooks.

- [ ] **Step 4: Update hooks index exports**

In `hooks/index.ts`, export the new hooks and remove old `useSendSalesInvoice`.

- [ ] **Step 5: Commit**

```bash
git add apps/app/lib/queries/invoicing/fetchers/ apps/app/lib/queries/invoicing/hooks/
git commit -m "feat: add approve/cancel fetchers and hooks for sales and purchase invoices"
```

---

### Task 9: Update sales invoice frontend components

**Files:**
- Modify: `apps/app/components/features/invoicing/sales-invoices/sales-invoice-detail.tsx`
- Modify: `apps/app/components/features/invoicing/sales-invoices/sales-invoices-list.tsx`
- Modify: `apps/app/components/features/invoicing/sales-invoices/sales-invoice-form.tsx`

- [ ] **Step 1: Update `sales-invoice-detail.tsx` — status badges and actions**

Replace the `getStatusColor` function to handle both document and payment status:

```typescript
function getDocumentStatusColor(status: string) {
  switch (status) {
    case "approved": return "bg-blue-100 text-blue-800";
    case "fulfilled": return "bg-green-100 text-green-800";
    case "cancelled": return "bg-red-100 text-red-800";
    default: return "bg-gray-100 text-gray-800"; // draft
  }
}

function getPaymentStatusColor(status: string) {
  switch (status) {
    case "paid": return "bg-green-100 text-green-800";
    case "partial": return "bg-yellow-100 text-yellow-800";
    default: return "bg-orange-100 text-orange-800"; // unpaid
  }
}
```

In the JSX, show both badges side by side:

```tsx
<Badge className={getDocumentStatusColor(invoice.status)}>
  {invoice.status}
</Badge>
{invoice.status !== "draft" && invoice.status !== "cancelled" && (
  <Badge className={getPaymentStatusColor(invoice.paymentStatus)}>
    {invoice.paymentStatus}
  </Badge>
)}
```

Replace action buttons based on new statuses:

```tsx
{invoice.status === "draft" && (
  <>
    <Button onClick={() => router.push(`/invoicing/sales-invoices/${invoice.id}/edit`)}>Edit</Button>
    <Button variant="destructive" onClick={handleDelete}>Delete</Button>
    <Button onClick={handleApprove}>Approve</Button>
  </>
)}
{invoice.status === "approved" && (
  <>
    <Button onClick={handleCancel}>Cancel</Button>
    {/* Email, PDF, Record Payment buttons */}
  </>
)}
{invoice.status === "fulfilled" && (
  <>
    {/* Email, PDF, Record Payment buttons only */}
  </>
)}
```

Add the `handleApprove` function using `useApproveSalesInvoice`:

```typescript
const approveMutation = useApproveSalesInvoice();

const handleApprove = () => {
  if (confirm("Approving will record ledger entries, deduct inventory, and submit to ZRA. Continue?")) {
    approveMutation.mutate({ id: invoice.id });
  }
};
```

Add the `handleCancel` function using `useCancelSalesInvoice`:

```typescript
const cancelMutation = useCancelSalesInvoice();

const handleCancel = () => {
  if (confirm("Are you sure you want to cancel this invoice?")) {
    cancelMutation.mutate(invoice.id);
  }
};
```

- [ ] **Step 2: Update `sales-invoices-list.tsx` — dual filters and badges**

Replace the status filter options:

```typescript
const statusOptions = [
  { value: "all", label: "All Statuses" },
  { value: "draft", label: "Draft" },
  { value: "approved", label: "Approved" },
  { value: "fulfilled", label: "Fulfilled" },
  { value: "cancelled", label: "Cancelled" },
];

const paymentStatusOptions = [
  { value: "all", label: "All Payments" },
  { value: "unpaid", label: "Unpaid" },
  { value: "partial", label: "Partial" },
  { value: "paid", label: "Paid" },
];
```

Add a second filter dropdown for `paymentStatus` and pass both to the query. In the table rows, show both badges.

Update row action menus: only show "Edit" and "Delete" for draft invoices.

- [ ] **Step 3: Update `sales-invoice-form.tsx` — remove status dropdown**

Remove the status select field from the form. The form always saves as draft. Replace the "Save & Send" button text with "Save" (draft save) or offer a "Save & Approve" secondary action.

For "Save & Approve": the form saves first, then on success calls the approve mutation:

```typescript
const handleSaveAndApprove = async (data: FormValues) => {
  const result = await createMutation.mutateAsync(data);
  if (result?.id) {
    approveMutation.mutate({ id: result.id });
  }
};
```

- [ ] **Step 4: Commit**

```bash
git add apps/app/components/features/invoicing/sales-invoices/
git commit -m "feat: update sales invoice UI with dual status badges and approve/cancel actions"
```

---

### Task 10: Update purchase invoice frontend components

**Files:**
- Modify: `apps/app/components/features/invoicing/purchase-invoices/purchase-invoice-detail.tsx`
- Modify: `apps/app/components/features/invoicing/purchase-invoices/purchase-invoices-list.tsx`
- Modify: `apps/app/components/features/invoicing/purchase-invoices/purchase-invoice-form.tsx`

- [ ] **Step 1: Update `purchase-invoice-detail.tsx`**

Same pattern as sales invoice detail:
- Replace `getStatusColor` with `getDocumentStatusColor` and `getPaymentStatusColor`
- Show dual badges
- Replace "Mark as Paid" button with "Approve" (for draft) and "Cancel" (for approved)
- Remove the old `canMarkPaid` logic that set `status: "paid"`
- Add `handleApprove` and `handleCancel` using `useApprovePurchaseInvoice` and `useCancelPurchaseInvoice`

- [ ] **Step 2: Update `purchase-invoices-list.tsx`**

Same as sales list:
- Update status filter options to `draft`/`approved`/`fulfilled`/`cancelled`
- Add payment status filter dropdown
- Show dual badges per row
- Row actions: Edit/Delete only for draft

- [ ] **Step 3: Update `purchase-invoice-form.tsx`**

Remove the status dropdown from the form. The Zod schema in the form (around line 45) with `z.enum(["draft", "sent", ...])` should be removed — the form no longer submits a status. Remove the status `<Select>` field (around lines 377-381).

- [ ] **Step 4: Commit**

```bash
git add apps/app/components/features/invoicing/purchase-invoices/
git commit -m "feat: update purchase invoice UI with dual status badges and approve/cancel actions"
```

---

### Task 11: Update recurring invoice generation

**Files:**
- Modify: `packages/api-services/src/domains/invoicing/recurring.service.ts`

- [ ] **Step 1: Update `generateDueInvoices`**

The recurring service at line 220 creates invoices with `status: "draft"`. This is already correct. Just add the `paymentStatus: "unpaid"` field:

```typescript
const invoice = await createInvoiceRepo(deps.db, {
  organizationId,
  customerId: template.customerId,
  invoiceNumber,
  invoiceDate: today,
  dueDate,
  status: "draft",
  paymentStatus: "unpaid",
  // ... rest of fields
});
```

- [ ] **Step 2: Commit**

```bash
git add packages/api-services/src/domains/invoicing/recurring.service.ts
git commit -m "fix: add paymentStatus to recurring invoice generation"
```

---

### Task 12: Final verification

- [ ] **Step 1: Type check**

Run:

```bash
cd packages/database && npx tsc --noEmit
cd packages/api-services && npx tsc --noEmit
cd apps/api-invoicing && npx tsc --noEmit
cd apps/app && npx tsc --noEmit
```

Fix any type errors that arise from the enum changes.

- [ ] **Step 2: Verify the migration**

Review the generated migration SQL to ensure:
- New enum types are created
- `payment_status` column added to both tables with default `'unpaid'`
- Data migration maps old statuses correctly

- [ ] **Step 3: Search for stale status references**

Search across the codebase for any remaining references to the old status values:

```bash
grep -rn '"sent"\|"pending"\|"viewed"\|"overdue"\|"void"\|"partial"' packages/api-services/src/domains/invoicing/ apps/app/components/features/invoicing/ --include='*.ts' --include='*.tsx'
```

Fix any remaining references. Note: `"partial"` may appear in payment contexts — that's fine, just ensure it's not used as a document status.

- [ ] **Step 4: Commit any fixes**

```bash
git add -A
git commit -m "fix: resolve type errors and stale status references"
```
