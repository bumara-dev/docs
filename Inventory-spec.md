---
title: "Inventory Spec"
description: "Bumara Inventory Module — Detailed Specification & Requirements 1) Goal & outcomes Primary goal"
---

Bumara Inventory Module — Detailed Specification & Requirements
1) Goal & outcomes
Primary goal

Enable SMEs to track stock accurately, across one or more locations, with an audit-safe stock ledger, simple operations (receive, issue, adjust, transfer, count), and low-stock visibility + alerts.

Key outcomes

Know On Hand stock per item per location at any time

Know what changed stock, when, by who (immutable ledger)

Prevent common operational errors (negative stock, double-posting)

Attach evidence (GRNs, supplier invoices, count sheets) via Documents module

Produce basic reports (stock valuation, movement history, low stock list)

2) Scope
MVP (Phase 1 — build next)

Core inventory control:

Items (products), categories, units of measure (UoM)

Locations (warehouse/store)

Stock Ledger (stock movements) + real-time stock balance

Stock operations:

Stock Adjustment (increase/decrease)

Stock Transfer (location A → B)

Stock Count (cycle count / full count) with variance posting

Low stock thresholds + notifications

Attachments/evidence stored in Documents module (R2)

Permissions & audit logs

Phase 2 (soon after MVP)

Suppliers + Purchases (PO → Receive → Supplier Invoice)

Sales integration (Invoice → issue stock)

Costing/valuation improvements (moving average, FIFO optional)

Barcode scanning + SKU enforcement

Batches/expiry dates (pharma/food)

Out of scope for MVP

Manufacturing / assemblies / BOM

Advanced accounting journal posting (we’ll expose events/hooks instead)

Serial-number tracking

3) Users, roles, and permissions
Roles (tenant-side)

Use your existing org roles:

Admin: full inventory settings + approvals + edits

Manager: operations + counts + transfers

Member: create drafts, view stock, cannot finalize sensitive actions (configurable)

Backoffice roles (internal)

Support analyst can view inventory to help clients troubleshoot, but cannot post stock unless explicitly allowed by an “impersonation/support mode” policy.

Permission matrix (MVP)

Actions:

View items/stock/movements: Member+

Create/Update Items: Manager+

Post stock adjustments: Manager+ (optionally require Admin approval if value > threshold)

Stock transfer: Manager+

Stock count create: Manager+, finalize: Manager+ (optional Admin approval)

Settings (UoM, locations): Admin only

4) Data model (Drizzle/Postgres)

Design principle: treat stock as an immutable ledger (movements), and derive balances from it (with a cached balance table for speed). This gives you auditability + easy debugging.

4.1 Core tables
inventory_items

id (uuid)

org_id (uuid, indexed)

name (text)

sku (text, nullable, unique per org)

barcode (text, nullable, unique per org)

category_id (uuid, nullable)

default_uom_id (uuid)

track_inventory (bool, default true)

reorder_level (numeric, nullable) — per item default

reorder_qty (numeric, nullable)

status enum: ACTIVE | ARCHIVED

timestamps

inventory_categories

id, org_id, name, optional parent_id for nested

inventory_locations

id, org_id

name (e.g., “Main Store”, “Warehouse”)

type enum: WAREHOUSE | STORE | VAN | OTHER

is_default (bool)

timestamps

inventory_units

id, org_id (or global seed)

code (EA, KG, L, M, BOX…)

name

precision (int: 0 for EA, 3 for KG etc.)

inventory_unit_conversions

id, org_id

from_uom_id, to_uom_id

multiplier (numeric) e.g., 1 BOX = 12 EA

is_bidirectional (bool)

validation: no loops that break conversions; allow explicit pairs

inventory_stock_balances (cache/denormalized)

id

org_id

item_id

location_id

on_hand_qty numeric

reserved_qty numeric (keep at 0 in MVP unless you already have sales reservations)

available_qty computed or stored

updated_at

unique constraint: (org_id, item_id, location_id)

inventory_stock_movements (immutable ledger)

id

org_id

item_id

location_id

movement_type enum:

ADJUSTMENT_IN, ADJUSTMENT_OUT

TRANSFER_OUT, TRANSFER_IN

COUNT_VARIANCE_IN, COUNT_VARIANCE_OUT

(Phase 2) PURCHASE_RECEIPT, SALE_ISSUE

qty numeric (signed or store qty_delta)

uom_id (store the posted uom)

unit_cost numeric nullable (MVP: optional; phase 2: required for valuation)

total_cost numeric (derived if unit_cost provided)

ref_type enum: ADJUSTMENT | TRANSFER | COUNT | PURCHASE | SALE | OTHER

ref_id uuid nullable (points to the source doc)

notes text

occurred_at timestamp (business date)

created_by user_id

idempotency_key text nullable, unique per org (prevents double-post)

timestamps

4.2 Operational tables (MVP)
inventory_adjustments

id, org_id, location_id

reason enum: DAMAGE | LOSS | FOUND | OPENING_BALANCE | CORRECTION | OTHER

status enum: DRAFT | POSTED | VOID

posted_at, posted_by

notes

attachments via Documents linking

timestamps

inventory_adjustment_lines

id, adjustment_id

item_id

qty numeric (signed or separate in/out)

uom_id

unit_cost nullable

validation: qty != 0

inventory_transfers

id, org_id

from_location_id, to_location_id

status enum: DRAFT | IN_TRANSIT | RECEIVED | VOID

shipped_at, received_at

notes, attachments

timestamps

inventory_transfer_lines

id, transfer_id, item_id, qty, uom_id

inventory_counts

id, org_id, location_id

count_type enum: CYCLE | FULL

status enum: DRAFT | IN_PROGRESS | COMPLETED | POSTED | VOID

started_at, completed_at, posted_at

notes, attachments

timestamps

inventory_count_lines

id, count_id, item_id

system_qty numeric (snapshot at start or at finalize)

counted_qty numeric

variance_qty numeric (counted - system)

uom_id

5) Business rules & validations (important)
Stock integrity

Stock balances must be updated only through posting movements (no direct edits).

Posting must be transactional:

create movement rows

update balances

commit

Prevent negative stock (config):

MVP default: block posting that would go negative, except Admin override (flagged + audited)

Idempotency:

Any “post” operation supports an idempotency_key to prevent duplicate movements if client retries.

Units of measure (UoM)

Store stock balances in the item’s base UoM (recommended).

When posting in a different UoM, convert using inventory_unit_conversions.

MVP default UoM set (seed):

EA (each), BOX, PACK, KG, G, L, ML, M

Precision rules:

EA precision 0, KG precision 3, L precision 3, etc.

Auditability

Editing a posted transaction is not allowed.

Corrections happen via:

VOID (creates reverse movements) OR

new adjustment/count variance

Every post records created_by, timestamps, and source references.

6) Key workflows (MVP)
6.1 Setup flow (first-run)

Create default location (“Main Store”)

Seed UoM + optionally allow user to add conversions (BOX→EA)

Add first items (or import CSV later)

Set opening balances via Adjustment: OPENING_BALANCE

6.2 Stock adjustment

User creates adjustment draft → adds lines → Post

System creates movement rows:

qty > 0 → ADJUSTMENT_IN

qty &lt; 0 → ADJUSTMENT_OUT

Update balances accordingly

6.3 Transfer stock

Create transfer draft → add lines → mark IN_TRANSIT (optional) → RECEIVED

On ship:

create TRANSFER_OUT movements at from_location

On receive:

create TRANSFER_IN movements at to_location

(MVP simplification): allow direct “Transfer & Receive” in one step if you want

6.4 Stock count (cycle count)

Create count → system snapshots system_qty

User enters counted quantities

Finalize → system calculates variance

Post → create COUNT_VARIANCE_IN/OUT movements

6.5 Low stock alerts

When on_hand_qty &lt;= reorder_level, raise:

in-app alert (notification)

optional task “Reorder item X”

Frequency control: avoid spamming (e.g., once per day per item/location)

7) UI/UX requirements (Next.js + shadcn)
Navigation

Add app section: Inventory

Dashboard

Items

Stock

Movements

Transfers

Counts

Reports

Settings

Screens
Inventory Dashboard

Cards:

Total SKUs

Low stock items

Stock value (if cost enabled)

Recent movements

Quick actions: New Adjustment, New Transfer, New Count

Items list

Search + filters (category, status, track_inventory)

Columns: Name, SKU, On Hand (default location), Reorder Level, Status

Row actions: View, Edit, Archive

Item details

Stock by location table

Recent movements for item

Settings: reorder level, base uom, barcode

Attachments (product spec sheets) via Documents module

Stock view

Pivot-like: item × location

Export CSV button (MVP)

Movements ledger

Filters: date range, item, location, type, ref_type

Click movement → open source doc (adjustment/transfer/count)

Adjustments

List + create/edit draft + post

Line editor supports:

item search

qty

uom

(optional) unit cost

Attach evidence (photo, supplier note)

Transfers

From/To location, lines, post/receive

Attach delivery note

Counts

Start count (choose items scope: all, category, selected)

Count entry grid optimized for speed (keyboard-friendly)

Post variances with summary

Settings

Locations management

Units + conversions

Negative stock policy toggle (Admin-only)

8) API design (Hono RPC + Zod)
Resource groups

/inventory/items

GET / list (pagination, search)

POST / create

GET /:id details

PATCH /:id update

POST /:id/archive

/inventory/locations

CRUD (Admin-only except read)

/inventory/stock

GET /balances (filters by location/item)

GET /movements (ledger query)

/inventory/adjustments

POST / create draft

PATCH /:id edit draft

POST /:id/post (transactional) + idempotency

POST /:id/void

/inventory/transfers

POST /

PATCH /:id

POST /:id/ship

POST /:id/receive

POST /:id/void

/inventory/counts

POST /

POST /:id/start

PATCH /:id/lines (bulk update)

POST /:id/complete

POST /:id/post

POST /:id/void

API requirements

Must enforce org_id isolation on every query

Must validate role permissions per action

Must support idempotency_key on posting endpoints

Posting endpoints must run inside DB transactions

Return “domain errors” (e.g., NEGATIVE_STOCK_BLOCKED) with user-friendly messages

9) Integration points with other Bumara modules
Documents module

Attachments for adjustments/transfers/counts stored as documents with:

entity_type = inventory_adjustment | inventory_transfer | inventory_count

entity_id

Evidence is immutable once posted (recommended)

Tasks module

Auto-create tasks:

Low stock → “Reorder Item”

Stock count schedule → monthly/weekly “Cycle count Location X”

Notifications module

Emit events:

inventory.low_stock

inventory.negative_stock_attempted

inventory.count_posted

inventory.transfer_received

Compliance / Smart Invoice (future-facing)

If you want a clean bridge:

Invoice posted → SALE_ISSUE movements

Supplier invoice / goods receipt → PURCHASE_RECEIPT movements

This keeps inventory consistent with ZRA Smart Invoice flows later.

10) Reporting requirements (MVP)

Reports (view + export):

Low Stock Report (by location)

Stock On Hand Report

Stock Movement Report (date range)

Stock Count Variance Report

(Optional if cost captured) Stock Valuation (sum(qty_on_hand × avg_cost))

11) Non-functional requirements
Performance

Use cached inventory_stock_balances for UI stock queries

Ledger queries paginated + indexed (org_id, occurred_at, item_id, location_id)

Concurrency & correctness

Posting must lock balance rows or use optimistic locking:

approach A: SELECT ... FOR UPDATE on relevant balance rows

approach B: version column for optimistic concurrency

Must be resilient to double-click/retry via idempotency

Security

No cross-org access

Sensitive actions audited

No PII in logs

12) Acceptance criteria (MVP)

 Admin can create locations, UoM conversions, and items

 Manager can post an opening balance adjustment and see correct on-hand stock

 Transfer stock between locations updates both balances correctly

 Stock count posts variance and ledger shows count reference

 Negative stock is blocked (unless Admin override enabled)

 Movement ledger shows who/when/what reference for every change

 Low stock items appear on dashboard and trigger a notification event

 Attachments can be uploaded and linked to adjustments/transfers/counts

 All posting operations are idempotent (same idempotency_key = no duplicates)

13) Implementation plan (recommended milestones)
Milestone 1 — Foundations

Schema: items, categories, units, conversions, locations

Balances + movements tables

Read-only screens: Items, Stock, Movements

Milestone 2 — Operations

Adjustments (draft → post)

Transfers (draft → receive)

Counts (draft → post variance)

Milestone 3 — Alerts + docs + hardening

Low stock detection + notifications

Attachments integration

Audit log coverage

Tests + edge cases (idempotency, concurrency)

14) Engineering checklist

DB migrations + constraints (unique SKU/org, unique balance row)

Seed data for UoMs

Transaction wrappers for “post” endpoints

Unit conversion helper (pure function + tests)

Shared “money/qty” numeric precision strategy (avoid float)

End-to-end test scripts for posting flows