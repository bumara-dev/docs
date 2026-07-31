---
title: "Inventory Module - Overview"
description: "Problem statement, goals, scope, personas, and terminology for the Bumara Inventory Module."
---

## Problem Statement

Zambian SMEs face significant challenges in managing their stock:

1. **Lack of visibility**: Businesses don't know what they have, where it is, or when to reorder
2. **No audit trail**: When stock discrepancies occur, there's no way to trace what happened
3. **Manual processes**: Paper-based tracking leads to errors and double-counting
4. **Operational errors**: Negative stock, double-posting, and lost inventory are common
5. **Compliance gaps**: No evidence trail for auditors or regulators

The Inventory Module addresses these problems by providing:

- Real-time stock visibility across all locations
- An immutable ledger showing exactly what changed, when, and by whom
- Digital operations that prevent common errors (idempotency, negative stock blocking)
- Evidence attachment for compliance (GRNs, count sheets, supplier invoices)

---

## Goals

### Primary Goal

Enable SMEs to track stock accurately, across one or more locations, with an **audit-safe stock ledger**, simple operations (receive, issue, adjust, transfer, count), and **low-stock visibility + alerts**.

### Key Outcomes

| Outcome | How We Achieve It |
|---------|-------------------|
| Know On Hand stock per item per location | `stock_balances` table with real-time updates |
| Know what changed stock, when, by who | Immutable `stock_movements` ledger |
| Prevent operational errors | Idempotency keys, negative stock blocking, transaction isolation |
| Attach evidence | Documents module integration (GRNs, invoices, count sheets) |
| Basic reporting | Stock on hand, movement history, low stock list |

---

## Scope

### MVP (Phase 1) — Build Next

**Core inventory control:**
- Items (products), categories, units of measure (UoM)
- Locations (warehouse/store/van)
- Stock Ledger (`stock_movements`) + real-time balance (`stock_balances`)

**Stock operations:**
- Stock Adjustment (increase/decrease with reason)
- Stock Transfer (location A → B, with optional in-transit state)
- Stock Count (cycle count / full count) with variance posting

**Supporting features:**
- Low stock thresholds + notifications
- Attachments/evidence via Documents module (R2/S3)
- Permissions & audit logs
- Idempotent posting (prevent double-post on retry)

### Phase 2 (Soon After MVP)

| Feature | Description |
|---------|-------------|
| Suppliers + Purchases | PO → Receive → Supplier Invoice workflow |
| Sales integration | Invoice → issue stock (SALE_ISSUE movement) |
| Costing improvements | Moving average, FIFO optional |
| Barcode scanning | SKU enforcement + mobile scanning |
| Batches/expiry dates | For pharma/food industries |

### Out of Scope (Non-Goals)

| Feature | Reason |
|---------|--------|
| Manufacturing / assemblies / BOM | Complex domain, separate module |
| Advanced accounting journal posting | Expose events/hooks instead |
| Serial-number tracking | Deferred to future phase |
| Multi-currency stock valuation | Deferred to future phase |
| Warehouse bin/rack locations | MVP uses location-level only |

---

## Personas and Permissions

### Tenant-Side Roles

The Inventory Module uses Bumara's existing organization role system:

| Role | Description | Inventory Capabilities |
|------|-------------|------------------------|
| **Admin** | Organization owner/administrator | Full settings, approvals, all operations |
| **Manager** | Department/team lead | Operations, counts, transfers, item management |
| **Member** | Regular staff | Create drafts, view stock, limited posting |

### Permission Matrix (MVP)

| Action | Member | Manager | Admin |
|--------|--------|---------|-------|
| View items/stock/movements | Yes | Yes | Yes |
| Create/Update items | No | Yes | Yes |
| Post stock adjustments | No | Yes | Yes |
| Adjustment above threshold | No | No | Yes |
| Stock transfer | No | Yes | Yes |
| Stock count (create) | No | Yes | Yes |
| Stock count (finalize/post) | No | Yes | Yes |
| Settings (UoM, locations) | No | No | Yes |
| Override negative stock block | No | No | Yes |

### Backoffice Roles

| Role | Capabilities |
|------|--------------|
| Support Analyst | View inventory to help clients troubleshoot |
| Support (with impersonation) | Cannot post stock unless explicitly allowed |

> **Note**: Backoffice access is read-only by default. Posting requires explicit "impersonation/support mode" policy.

---

## Glossary

| Term | Definition |
|------|------------|
| **Item** | A product or material tracked in inventory. Has SKU, barcode, default UoM, and reorder settings. |
| **Location** | A physical place where stock is held (warehouse, store, van). Items have separate balances per location. |
| **UoM (Unit of Measure)** | The unit used to count/measure items (EA=each, KG, L, BOX). Items have a base UoM. |
| **UoM Conversion** | A multiplier to convert between units (1 BOX = 12 EA). |
| **Stock Balance** | The current quantity of an item at a location. Cached/derived from movements. |
| **Stock Movement** | An immutable record of stock change. Source of truth for the ledger. |
| **Ledger** | The complete history of stock movements. Immutable and audit-safe. |
| **Adjustment** | An operation that increases or decreases stock with a reason (damage, loss, found, correction). |
| **Transfer** | An operation that moves stock from one location to another. |
| **Count** | A stock-taking operation that compares system quantity to physical count and posts variances. |
| **On Hand Qty** | The total quantity available at a location. |
| **Reserved Qty** | Quantity set aside for pending orders (Phase 2). |
| **Available Qty** | On Hand minus Reserved. |
| **Reorder Level** | Threshold below which low-stock alerts trigger. |
| **Reorder Qty** | Suggested quantity to order when reordering. |
| **Idempotency Key** | A unique key per operation to prevent duplicate posts on retry. |
| **Variance** | The difference between system quantity and counted quantity. |

---

## Integration with Bumara Platform

### Documents Module

Attachments for operations are stored via the Documents module:

- `entity_type` = `inventory_adjustment` | `inventory_transfer` | `inventory_count`
- Evidence is immutable once operation is posted (recommended)
- Storage: R2/S3 with presigned URLs

**Example use cases:**
- GRN (Goods Received Note) attached to adjustment
- Delivery note attached to transfer
- Count sheet photo attached to stock count

### Tasks Module

Auto-created tasks for inventory events:

| Event | Task Created |
|-------|--------------|
| Low stock detected | "Reorder Item: &#123;item name&#125;" |
| Scheduled count due | "Cycle count Location X" (future) |

Tasks use existing template pattern from `packages/tasks/src/tasks/service.ts`.

### Notifications Module

Events emitted for real-time alerts:

| Event | Trigger | Recipients |
|-------|---------|------------|
| `inventory.low_stock` | Balance drops below reorder level | Managers, Admins |
| `inventory.negative_stock_attempted` | Blocked negative stock operation | Admins |
| `inventory.count_posted` | Stock count finalized | Managers, Admins |
| `inventory.transfer_received` | Transfer completed | Originator, Managers |

Uses existing Knock Labs integration in [`packages/notifications/`](https://github.com/bumara-dev/bumara/tree/main/packages/notifications).

---

## Related Documentation

- [Architecture](/inventory/02-architecture) — System design and code locations
- [Data Model](/inventory/03-data-model) — Database schema details
- [API Spec](/inventory/04-api-spec) — Endpoint contracts
- [Workflows](/inventory/05-workflows) — Operation state machines

---

## Open Questions

1. **Approval workflow**: Should high-value adjustments require Admin approval (two-step post)?
2. **Count frequency**: Should we support scheduled/recurring counts in MVP?
3. **Multi-UoM display**: Show balances in base UoM only, or allow display in any convertible UoM?
4. **Item import**: CSV import for initial item setup — MVP or Phase 2?
5. **Stock valuation**: Display stock value on dashboard if `unit_cost` is captured?
