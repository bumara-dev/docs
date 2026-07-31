---
title: "Invoicing Module: Mining Export Invoices"
description: "Commercial, provisional, and final invoices for mining-sector exporters, plus RVAT principals and ZRA import declarations."
---

ZRA requires exporters of minerals (and incidental services such as transport and insurance) to issue a Commercial Invoice before any export. The Commercial Invoice acts as a customs declaration, and ZRA's export chain runs:

```
Commercial Invoice → Customs Declaration → Provisional Invoice → Final Tax Invoice / Final Credit Note
```

A Commercial Invoice is **acquitted** when a downstream Provisional or Final invoice is transmitted successfully against it.

This is a mining-only surface, deliberately segregated so its 40-plus fields do not bleed into the regular invoicing UI.

<Note>
**Implementation status on this branch.** The database schema, the `@repo/invoicing`
services, and the full `api-invoicing` HTTP surface are live here. The dedicated UI pages
(`/invoicing/commercial-invoices` and friends) are not yet part of the `apps/app` route
tree on this branch, so these flows are currently API-only. Customs Declaration is not
implemented at all.
</Note>

---

## Commercial invoices

Two tables carry the header and lines: `commercial_invoices` and `commercial_invoice_line_items`. The header mirrors the ZRA wire payload closely to keep the mapping obvious, grouped into identity, dates, shipment, currency, export, incoterms, sender, receiver, delivery, transporter, bank, and totals. Transmissions reuse the existing `zra_smart_invoice_transmissions` table via `document_type='commercial_invoice'`.

Commercial invoices do **not** move stock. The `saveStockItems` and `saveStockMaster` chain is sales-only and is deliberately skipped here.

### Lifecycle

`draft` → `issued` → `acquitted`, with `reversed` and `cancelled` as terminal branches. The relevant columns are `status`, `acquitted_at`, `acquitted_by_invoice_id`, `reversal_reason_code`, `reversed_at`, and `reversed_zra_receipt_no`.

### Reversal

A correction to an already-issued Commercial Invoice reuses ZRA's `saveCommercialInvoice` endpoint with `rcptTyCd='R'`, sending the original transmission's `rcptNo` as `orgInvcNo` along with `orgSdcId`. No new row is created: the same record gains its reversal columns. The reversal reason comes from ZRA's refund-reasons reference list. Reversing an already-reversed invoice is not supported.

### Endpoints

| Method | Path |
|--------|------|
| `GET` | `/invoicing/commercial-invoices` |
| `POST` | `/invoicing/commercial-invoices` |
| `GET` | `/invoicing/commercial-invoices/{id}` |
| `PATCH` | `/invoicing/commercial-invoices/{id}` |
| `POST` | `/invoicing/commercial-invoices/{id}/transmit` |
| `POST` | `/invoicing/commercial-invoices/{id}/cancel` |
| `POST` | `/invoicing/commercial-invoices/{id}/reverse` |

---

## Provisional and final invoices

Provisional and Final are sale-type variants of the ordinary `sales_invoices` flow rather than new tables. They are discriminated by `sales_invoices.sale_type`:

| `sale_type` | Meaning |
|-------------|---------|
| `normal` | Ordinary sales invoice (the default). |
| `rvat` | Reverse VAT invoice, linked to a `zra_principals` row. |
| `provisional` | Provisional invoice referencing one or more issued Commercial Invoices. |
| `final` | Final invoice closing a Provisional; the variant is carried on `final_variant`. |

A **Provisional Invoice** is issued when the final sale price is not yet known. It carries the same large sales payload as `saveSales` (full tax breakdown, items with `vatCatCd`, payment fields) plus a list of linked Commercial Invoice ids. On transmission each linked id is resolved to its issuing transmission's `rcptNo` and emitted as `orgCommInvcNoList`, hitting ZRA's `saveProvisionalInvoice`.

**Auto-acquittal** happens in the provisional service: after a successful transmission it iterates the linked Commercial Invoice ids and flips each from `issued` to `acquitted`, recording `acquitted_at` and `acquitted_by_invoice_id`. Acquitting a Commercial Invoice from a different organization is not supported, and there is no manual un-acquit.

A **Final Invoice** closes a Provisional. `final_variant` drives ZRA endpoint routing (tax invoice, credit note, or combined), and `linked_provisional_invoice_ids` records what it closes, with `finalized_by_invoice_id` recorded on the provisional side.

### Endpoints

Provisional and Final expose the same shape:

| Method | Path |
|--------|------|
| `GET` | `/invoicing/provisional-invoices`, `/invoicing/final-invoices` |
| `POST` | `/invoicing/provisional-invoices`, `/invoicing/final-invoices` |
| `GET` | `.../{id}` |
| `PATCH` | `.../{id}` |
| `POST` | `.../{id}/transmit` |
| `POST` | `.../{id}/cancel` |

---

## RVAT principals

Reverse VAT invoices are issued on behalf of a principal. `zra_principals` holds the local mirror of ZRA's principal list, and `sales_invoices` carries a local FK to it when `sale_type='rvat'`.

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/invoicing/principals` | List the mirrored principals. |
| `POST` | `/invoicing/principals/refresh` | Re-pull the list from ZRA. |

---

## Import declarations

ZRA publishes import declarations against an organization's TPIN. The imports slice mirrors them locally and lets a user approve or disregard each one.

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/invoicing/imports` | List mirrored imports. |
| `GET` | `/invoicing/imports/{importId}` | Import detail. |
| `GET` | `/invoicing/imports/declarations` | List raw ZRA declarations. |
| `GET` | `/invoicing/imports/declarations/{taskCd}` | A single declaration by task code. |
| `POST` | `/invoicing/imports/refresh` | Re-pull declarations from ZRA. |
| `POST` | `/invoicing/imports/{importId}/approve` | Accept the import. |
| `POST` | `/invoicing/imports/{importId}/disregard` | Reject the import. |

---

## Where the code lives

| Concern | Location |
|---------|----------|
| Commercial invoice schema | `packages/database/src/schema/invoicing/commercial-invoices.ts`, `commercial-invoice-line-items.ts` |
| Sales invoice discriminators | `packages/database/src/schema/invoicing/sales-invoices.ts` |
| Commercial invoice domain | `packages/invoicing/src/commercial-invoices/` (`service.ts`, `transmit.ts`, `provisional.ts`, `final.ts`, `queries.ts`, `schema.ts`) |
| ZRA client and payload builders | `packages/invoicing/src/zra/` |
| HTTP surface | `apps/api-invoicing/src/modules/{commercial-invoices,provisional-invoices,final-invoices,principals,imports}/` |

## Not implemented

- Customs Declaration endpoint.
- Manual un-acquit and re-acquit.
- Cross-organization acquittal.
- Reverse of an already-reversed Commercial Invoice.
- Dedicated PDF rendering for Commercial Invoices.
