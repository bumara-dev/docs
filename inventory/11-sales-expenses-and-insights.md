---
title: "Inventory Module: Sales, Expenses and Insights"
description: "Cash sales, lay-by sales, expenses, cash-outs, and the inventory insights dashboard, including how each one touches the stock ledger."
---

Beyond the core stock operations covered in [Workflows](/inventory/05-workflows), the inventory zone carries the day to day money movements of a shop floor: selling stock over the counter, holding goods against instalments, and recording what leaves the till.

## Cash sales

Cash sales are the counter-sale path described in [POS and scanners](/inventory/10-pos-and-scanners). They are not a separate table. The till form builds a cart, then creates an adjustment with `reason: "SALE"` and immediately posts it, so the sale lands on the stock movements ledger through the same audited path as every other stock change.

Because the adjustment is created and posted in two calls, a sale that fails to post leaves a draft adjustment behind rather than silently losing the sale.

**UI:** `/inventory/sales/cash-sales` and `/inventory/sales/cash-sales/new`.

## Lay-by sales

A lay-by holds goods for a customer who pays in instalments. Stock is only deducted when the lay-by completes, so an active lay-by does not reduce on-hand quantities.

**Statuses** (`layby_sale_status`):

| Status | Meaning |
|--------|---------|
| `active` | Payments ongoing. No stock movement yet. |
| `completed` | Fully paid. Stock is deducted at this point. |
| `cancelled` | Cancelled. No stock change. |

A lay-by is created against a single location with at least one line (item, quantity, unit of measure, sale price, optional unit cost), an optional due date, and an opening deposit that defaults to zero.

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/inventory/layby-sales` | List, filterable by `status` and `locationId`. |
| `GET` | `/inventory/layby-sales/{laybySaleId}` | Detail. |
| `POST` | `/inventory/layby-sales` | Create with opening deposit and lines. |
| `POST` | `/inventory/layby-sales/{laybySaleId}/payments` | Record an instalment. |
| `POST` | `/inventory/layby-sales/{laybySaleId}/complete` | Settle and deduct stock. |
| `POST` | `/inventory/layby-sales/{laybySaleId}/cancel` | Cancel without stock change. |

**UI:** `/inventory/sales/layby-sales`, with detail and create pages.

## Expenses

Operating costs recorded against a location. Expenses never touch the stock ledger; they exist so the profit view can net them against sales.

**Categories** (`expense_category`): `supplies`, `utilities`, `transport`, `maintenance`, `marketing`, `salaries`, `rent`, `other`.

**Payment methods:** `cash`, `mobile_money`, `bank_transfer`, `other` (plain text, not a Postgres enum), defaulting to `cash`.

Full CRUD at `/inventory/expenses` and `/inventory/expenses/{expenseId}`. Listing accepts `locationId`, `category`, a `from`/`to` date window, and `limit`/`offset` paging (default 50, max 200).

**UI:** `/inventory/expenses` and `/inventory/expenses/new`.

## Cash-outs

Money taken out of the till, recorded separately from expenses because the reasons differ and some require an authorizing user.

**Reasons** (`cash_out_reason`): `float`, `petty_cash`, `withdrawal`, `refund`, `other`.

A cash-out carries a description, an amount, a date, an optional `authorizedBy` user, and optional notes. Full CRUD at `/inventory/cash-outs` and `/inventory/cash-outs/{cashOutId}`, with the same filtering and paging shape as expenses.

## Insights

`GET /inventory/insights` backs the inventory dashboard. It is location-scoped: the caller's location assignments are resolved first, and if the resolved scope reaches no location the endpoint returns a zeroed payload rather than leaking org-wide totals. See [Location-based access](/inventory/02-architecture) for how that scope is derived.

The response carries:

| Field | Meaning |
|-------|---------|
| `totalStockValue` | Summed value of on-hand stock in scope. |
| `lowStockItems` | Count of items at or below their reorder level. |
| `totalItems` | Count of items in scope. |
| `activeLocations` | Count of locations in scope. |
| `recentMovements` | Latest stock ledger entries. |
| `stockValueTrend` | Stock value over time, for the trend chart. |
| `stockDistribution` | Value split across locations, for the distribution chart. |

**UI:** the inventory landing page at `/inventory`, plus `/inventory/profits` and `/inventory/movements`.

## Where the code lives

| Concern | Location |
|---------|----------|
| Domain logic | `packages/inventory/src/{cash-outs,expenses,layby-sales,insights}/` |
| Shared enums | `packages/database/src/schema/sales-expenses/enums.ts` |
| HTTP surface | `apps/api-inventory/src/modules/{cash-outs,expenses,layby-sales,insights}/` |
| UI | `apps/app/zones/inventory/modules/{cash-sales,layby-sales,expenses}/` |

<Note>
`packages/inventory/src/purchase-orders/` and `packages/inventory/src/location-assignments/`
carry domain logic that `apps/api-inventory` does not expose over HTTP on this branch.
Purchase orders reach users through the invoicing zone at
`/invoicing/purchases/purchase-orders`; location assignments are applied internally by the
members slice and by the insights location scope.
</Note>
