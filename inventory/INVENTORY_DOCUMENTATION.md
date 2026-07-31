---
title: "Bumara Inventory Module — Technical & Functional Documentation"
description: "The inventory module end to end: item tracking, multi-location stock, adjustments, transfers, and counts with a full audit trail."
---

**Version:** 1.0
**Date:** February 9, 2026
**Branch:** feature/inventory
**Platform:** Next.js 14+, TypeScript, Drizzle ORM, PostgreSQL

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Architecture Overview](#2-architecture-overview)
3. [Navigation Structure](#3-navigation-structure)
4. [Database Schema](#4-database-schema)
5. [Settings & Configuration](#5-settings--configuration)
6. [Items Management](#6-items-management)
7. [Stock Adjustments](#7-stock-adjustments)
8. [Stock Transfers](#8-stock-transfers)
9. [Stock Counts](#9-stock-counts)
10. [Stock Ledger & Movement Engine](#10-stock-ledger--movement-engine)
11. [Stock Levels & Movements Views](#11-stock-levels--movements-views)
12. [Dashboard & Analytics](#12-dashboard--analytics)
13. [Attachments System](#13-attachments-system)
14. [API Endpoints Reference](#14-api-endpoints-reference)
15. [Security & Data Integrity](#15-security--data-integrity)

---

## 1. Introduction

The Bumara Inventory Module is a full-featured stock management system built as part of the Bumara business platform. It enables organisations to track items, manage stock across multiple locations, and perform standard warehouse operations — adjustments, transfers, and physical counts — with a complete audit trail.

### Key Capabilities

- **Item Master Management** — Create and manage inventory items with SKU, barcode, category, unit of measure, and reorder settings.
- **Multi-Location Stock Tracking** — Track stock balances across warehouses, stores, vans, and custom locations.
- **Stock Adjustments** — Record stock increases or decreases for damage, loss, opening balances, corrections, and other reasons.
- **Stock Transfers** — Move stock between locations with a two-phase Ship/Receive workflow.
- **Physical Stock Counts** — Conduct full or cycle counts, compare system vs. actual quantities, and post variance corrections.
- **Immutable Movement Ledger** — Every stock change is recorded as an immutable movement entry for complete auditability.
- **Image Attachments** — Attach photos to items, adjustments, transfers, and counts (compressed to WebP, stored as base64).
- **Configurable Settings** — Categories, locations, units of measure, and unit conversions.
- **Dashboard** — At-a-glance KPIs including total items, low stock alerts, total on-hand, and recent movements.

---

## 2. Architecture Overview

The inventory module follows a layered architecture:

```
┌──────────────────────────────────────────────────────────────┐
│                     FRONTEND (Next.js App)                    │
│  Pages → Components → React Query Hooks → API Fetchers       │
├──────────────────────────────────────────────────────────────┤
│                     API LAYER (Hono Workers)                  │
│  Routes → Handlers → Middleware (Auth + Org)                 │
├──────────────────────────────────────────────────────────────┤
│                   SERVICE LAYER (api-services)                │
│  Business Logic → Validation (Zod) → Database Operations     │
├──────────────────────────────────────────────────────────────┤
│                  DATABASE LAYER (Drizzle ORM)                 │
│  Schema Definitions → Migrations → PostgreSQL                │
└──────────────────────────────────────────────────────────────┘
```

### Layer Responsibilities

| Layer | Location | Role |
|-------|----------|------|
| Frontend | `apps/app/` | UI components, pages, React Query state management |
| API | `apps/api-inventory/` | HTTP routes, request validation, auth middleware |
| Services | `packages/api-services/` | Business logic, schema validation, database queries |
| Database | `packages/database/` | Drizzle ORM schema, migrations, PostgreSQL |

### Data Flow

1. User interacts with a page component.
2. Component calls a React Query hook (e.g., `useCreateItem()`).
3. Hook calls a fetcher function that sends an HTTP request to the API worker.
4. API route handler passes the request to a service function.
5. Service function validates input with Zod, executes database operations, and returns the result.
6. Response flows back through the same chain to update the UI.

---

## 3. Navigation Structure

The inventory module is accessed through a dedicated sidebar with the following structure:

```
Inventory Workspace
│
├── Overview
│   └── Dashboard .......................... /inventory
│
├── Inventory
│   ├── Items ............................. /inventory/items
│   ├── Stock Levels ...................... /inventory/stock
│   └── Movements ......................... /inventory/movements
│
├── Operations
│   ├── Adjustments ....................... /inventory/adjustments
│   ├── Transfers ......................... /inventory/transfers
│   └── Stock Counts ...................... /inventory/counts
│
└── Configuration
    └── Settings .......................... /inventory/settings
```

Each section has list pages, detail pages, and form pages (new/edit) where applicable.

---

## 4. Database Schema

The inventory module uses 11 PostgreSQL tables, 10 enums, and multiple indexes for performance.

### 4.1 Inventory Items

Stores the master item catalogue.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK, auto-generated | Unique identifier |
| organization_id | TEXT | FK → organizations, NOT NULL | Owning organisation |
| name | TEXT | NOT NULL | Item name |
| sku | TEXT | UNIQUE per org, nullable | Stock keeping unit code |
| barcode | TEXT | UNIQUE per org, nullable | Barcode value |
| description | TEXT | nullable | Free-text description |
| category_id | UUID | FK → inventory_categories, nullable | Item category |
| default_uom_id | UUID | FK → inventory_units, NOT NULL | Default unit of measure |
| track_inventory | BOOLEAN | NOT NULL, default true | Whether to track stock |
| reorder_level | NUMERIC(18,4) | nullable | Low stock threshold |
| reorder_qty | NUMERIC(18,4) | nullable | Suggested reorder quantity |
| status | ENUM | NOT NULL, default 'active' | 'active' or 'archived' |
| created_at | TIMESTAMP | NOT NULL, auto | Creation timestamp |
| updated_at | TIMESTAMP | NOT NULL, auto | Last update timestamp |

### 4.2 Inventory Categories

Hierarchical item classification.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | Unique identifier |
| organization_id | TEXT | FK → organizations, NOT NULL | Owning organisation |
| name | TEXT | UNIQUE per org, NOT NULL | Category name |
| parent_id | UUID | nullable, self-referencing | Parent category for hierarchy |
| created_at | TIMESTAMP | NOT NULL | Creation timestamp |
| updated_at | TIMESTAMP | NOT NULL | Last update timestamp |

### 4.3 Inventory Locations

Physical or logical stock storage locations.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | Unique identifier |
| organization_id | TEXT | FK → organizations, NOT NULL | Owning organisation |
| name | TEXT | UNIQUE per org, NOT NULL | Location name |
| type | ENUM | NOT NULL, default 'warehouse' | warehouse, store, van, or other |
| is_default | BOOLEAN | NOT NULL, default false | Whether this is the default location |
| address | TEXT | nullable | Physical address |
| created_at | TIMESTAMP | NOT NULL | Creation timestamp |
| updated_at | TIMESTAMP | NOT NULL | Last update timestamp |

### 4.4 Inventory Units of Measure

Defines measurement units (e.g., Each, Kilogram, Litre).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | Unique identifier |
| organization_id | TEXT | nullable | NULL = global/seeded unit; set = custom org unit |
| code | TEXT | UNIQUE per org, NOT NULL | Short code (EA, KG, L, BOX) |
| name | TEXT | NOT NULL | Full name (Each, Kilogram) |
| precision | INTEGER | NOT NULL, default 0 | Decimal places for display |
| created_at | TIMESTAMP | NOT NULL | Creation timestamp |
| updated_at | TIMESTAMP | NOT NULL | Last update timestamp |

Units with a NULL `organization_id` are system-seeded global units visible to all organisations but not editable.

### 4.5 Unit Conversions

Defines conversion factors between units.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | Unique identifier |
| organization_id | TEXT | FK → organizations, nullable | Owning organisation |
| from_uom_id | UUID | FK → inventory_units, NOT NULL | Source unit |
| to_uom_id | UUID | FK → inventory_units, NOT NULL | Target unit |
| multiplier | NUMERIC(18,8) | NOT NULL | 1 fromUom = multiplier toUom |
| is_bidirectional | BOOLEAN | NOT NULL, default true | Whether reverse conversion applies |
| created_at | TIMESTAMP | NOT NULL | Creation timestamp |
| updated_at | TIMESTAMP | NOT NULL | Last update timestamp |

Example: 1 KG = 1000 G (from_uom = KG, to_uom = G, multiplier = 1000).

### 4.6 Inventory Adjustments

Header record for stock adjustment documents.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | Unique identifier |
| organization_id | TEXT | FK, NOT NULL | Owning organisation |
| location_id | UUID | FK → inventory_locations, NOT NULL | Adjustment location |
| reason | ENUM | NOT NULL | DAMAGE, LOSS, FOUND, OPENING_BALANCE, CORRECTION, OTHER |
| status | ENUM | NOT NULL, default 'draft' | draft, posted, or void |
| notes | TEXT | nullable | Free-text notes |
| posted_at | TIMESTAMP | nullable | When adjustment was posted |
| posted_by | TEXT | FK → users, nullable | Who posted it |
| voided_at | TIMESTAMP | nullable | When adjustment was voided |
| voided_by | TEXT | FK → users, nullable | Who voided it |
| void_reason | TEXT | nullable | Why it was voided |
| created_at | TIMESTAMP | NOT NULL | Creation timestamp |
| updated_at | TIMESTAMP | NOT NULL | Last update timestamp |
| created_by | TEXT | FK → users, NOT NULL | Who created it |

### 4.7 Adjustment Lines

Line items within an adjustment.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | Unique identifier |
| adjustment_id | UUID | FK → inventory_adjustments, NOT NULL | Parent adjustment |
| item_id | UUID | FK → inventory_items, NOT NULL | Item being adjusted |
| qty | NUMERIC(18,4) | NOT NULL | Signed quantity (positive = in, negative = out) |
| uom_id | UUID | FK → inventory_units, NOT NULL | Unit of measure |
| unit_cost | NUMERIC(18,4) | nullable | Cost per unit |
| created_at | TIMESTAMP | NOT NULL | Creation timestamp |

### 4.8 Inventory Transfers

Header record for stock transfer documents.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | Unique identifier |
| organization_id | TEXT | FK, NOT NULL | Owning organisation |
| from_location_id | UUID | FK → inventory_locations, NOT NULL | Source location |
| to_location_id | UUID | FK → inventory_locations, NOT NULL | Destination location |
| status | ENUM | NOT NULL, default 'draft' | draft, in_transit, received, or void |
| notes | TEXT | nullable | Free-text notes |
| shipped_at | TIMESTAMP | nullable | When transfer was shipped |
| shipped_by | TEXT | FK → users, nullable | Who shipped it |
| received_at | TIMESTAMP | nullable | When transfer was received |
| received_by | TEXT | FK → users, nullable | Who received it |
| voided_at | TIMESTAMP | nullable | When transfer was voided |
| voided_by | TEXT | FK → users, nullable | Who voided it |
| void_reason | TEXT | nullable | Why it was voided |
| created_at | TIMESTAMP | NOT NULL | Creation timestamp |
| updated_at | TIMESTAMP | NOT NULL | Last update timestamp |
| created_by | TEXT | FK → users, NOT NULL | Who created it |

### 4.9 Transfer Lines

Line items within a transfer.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | Unique identifier |
| transfer_id | UUID | FK → inventory_transfers, NOT NULL | Parent transfer |
| item_id | UUID | FK → inventory_items, NOT NULL | Item being transferred |
| qty | NUMERIC(18,4) | NOT NULL | Quantity (positive only) |
| uom_id | UUID | FK → inventory_units, NOT NULL | Unit of measure |
| created_at | TIMESTAMP | NOT NULL | Creation timestamp |

### 4.10 Inventory Counts

Header record for physical stock count documents.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | Unique identifier |
| organization_id | TEXT | FK, NOT NULL | Owning organisation |
| location_id | UUID | FK → inventory_locations, NOT NULL | Count location |
| count_type | ENUM | NOT NULL, default 'cycle' | 'cycle' (partial) or 'full' |
| status | ENUM | NOT NULL, default 'draft' | draft, in_progress, completed, posted, or void |
| notes | TEXT | nullable | Free-text notes |
| started_at | TIMESTAMP | nullable | When counting started |
| started_by | TEXT | FK → users, nullable | Who started it |
| completed_at | TIMESTAMP | nullable | When counting finished |
| completed_by | TEXT | FK → users, nullable | Who completed it |
| posted_at | TIMESTAMP | nullable | When count was posted |
| posted_by | TEXT | FK → users, nullable | Who posted it |
| voided_at | TIMESTAMP | nullable | When count was voided |
| voided_by | TEXT | FK → users, nullable | Who voided it |
| void_reason | TEXT | nullable | Why it was voided |
| created_at | TIMESTAMP | NOT NULL | Creation timestamp |
| updated_at | TIMESTAMP | NOT NULL | Last update timestamp |
| created_by | TEXT | FK → users, NOT NULL | Who created it |

### 4.11 Count Lines

Line items within a count, capturing system vs. actual quantities.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | Unique identifier |
| count_id | UUID | FK → inventory_counts, NOT NULL | Parent count |
| item_id | UUID | FK → inventory_items, NOT NULL | Item being counted |
| system_qty | NUMERIC(18,4) | NOT NULL | Snapshot of system balance at count start |
| counted_qty | NUMERIC(18,4) | nullable | Actual counted quantity |
| variance_qty | NUMERIC(18,4) | nullable | counted_qty minus system_qty |
| uom_id | UUID | FK → inventory_units, NOT NULL | Unit of measure |
| created_at | TIMESTAMP | NOT NULL | Creation timestamp |
| updated_at | TIMESTAMP | NOT NULL | Last update timestamp |

### 4.12 Stock Balances (Derived Cache)

Cached current stock levels per item per location. Updated automatically by the stock ledger engine.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | Unique identifier |
| organization_id | TEXT | FK, NOT NULL | Owning organisation |
| item_id | UUID | FK → inventory_items, NOT NULL | Item |
| location_id | UUID | FK → inventory_locations, NOT NULL | Location |
| on_hand_qty | NUMERIC(18,4) | NOT NULL, default 0 | Current on-hand quantity |
| reserved_qty | NUMERIC(18,4) | NOT NULL, default 0 | Reserved quantity |
| updated_at | TIMESTAMP | NOT NULL | Last update timestamp |

**Available quantity** is calculated at query time as: `on_hand_qty - reserved_qty`.

The stock balances table is a **performance cache** — the source of truth is always the stock movements ledger.

### 4.13 Stock Movements (Immutable Ledger)

The single source of truth for all stock changes. This table is **append-only** — records are never updated or deleted.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | Unique identifier |
| organization_id | TEXT | FK, NOT NULL | Owning organisation |
| item_id | UUID | FK → inventory_items, NOT NULL | Item affected |
| location_id | UUID | FK → inventory_locations, NOT NULL | Location affected |
| movement_type | ENUM | NOT NULL | Type of movement (see below) |
| qty | NUMERIC(18,4) | NOT NULL | Signed quantity (+ = in, - = out) |
| uom_id | UUID | FK → inventory_units, NOT NULL | Unit of measure |
| unit_cost | NUMERIC(18,4) | nullable | Cost per unit |
| total_cost | NUMERIC(18,4) | nullable | Total cost |
| ref_type | ENUM | NOT NULL | Source document type (ADJUSTMENT, TRANSFER, COUNT, OTHER) |
| ref_id | UUID | nullable | Source document ID |
| notes | TEXT | nullable | Movement notes |
| occurred_at | TIMESTAMP | NOT NULL | Business date of the movement |
| created_by | TEXT | FK → users, NOT NULL | User who created the movement |
| idempotency_key | TEXT | UNIQUE per org | Prevents duplicate postings |
| created_at | TIMESTAMP | NOT NULL | System creation timestamp |

**Movement Types:**

| Type | Direction | Created By |
|------|-----------|------------|
| ADJUSTMENT_IN | Stock increase (+) | Posting an adjustment with positive qty |
| ADJUSTMENT_OUT | Stock decrease (-) | Posting an adjustment with negative qty |
| TRANSFER_OUT | Stock decrease (-) | Shipping a transfer (source location) |
| TRANSFER_IN | Stock increase (+) | Receiving a transfer (destination location) |
| COUNT_VARIANCE_IN | Stock increase (+) | Posting a count where counted > system |
| COUNT_VARIANCE_OUT | Stock decrease (-) | Posting a count where counted &lt; system |

### 4.14 Inventory Attachments

Stores images attached to inventory entities.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | Unique identifier |
| organization_id | TEXT | FK, NOT NULL | Owning organisation |
| entity_type | ENUM | NOT NULL | 'item', 'adjustment', 'transfer', or 'count' |
| entity_id | UUID | NOT NULL | ID of the parent entity |
| file_name | TEXT | NOT NULL | Original filename (with .webp extension) |
| mime_type | TEXT | NOT NULL, default 'image/webp' | Always image/webp |
| size_bytes | INTEGER | NOT NULL | Binary size in bytes |
| data | TEXT | NOT NULL | Base64-encoded WebP image data |
| created_at | TIMESTAMP | NOT NULL | Creation timestamp |
| updated_at | TIMESTAMP | NOT NULL | Last update timestamp |

---

## 5. Settings & Configuration

The Settings page (`/inventory/settings`) provides four configuration tabs:

### 5.1 Item Categories

Categories allow hierarchical classification of items (e.g., Electronics > Phones > Accessories).

**Fields:**
- **Name** (required) — The category name.
- **Parent Category** (optional) — For creating nested hierarchies.

**Operations:** Create, Read, Update, Delete.

### 5.2 Locations

Locations represent physical or logical places where stock is stored.

**Fields:**
- **Name** (required) — The location name (e.g., "Main Warehouse").
- **Type** (required) — One of: Warehouse, Store, Van, Other.
- **Is Default** (toggle) — Only one location can be default at a time. Setting a new default automatically unsets the previous one.
- **Address** (optional) — Physical address.

**Operations:** Create, Read, Update, Delete.

### 5.3 Units of Measure

Units define how items are measured and counted.

**Fields:**
- **Code** (required) — Short uppercase code (e.g., EA, KG, LTR, BOX).
- **Name** (required) — Full name (e.g., Each, Kilogram, Litre, Box).
- **Precision** (required) — Number of decimal places (e.g., 0 for Each, 3 for Kilogram).

**Types of Units:**
- **System Units** — Pre-seeded global units (visible to all organisations, read-only). Shown with "System" in the Source column.
- **Custom Units** — Organisation-specific units. Shown with "Custom" in the Source column. These can be edited and deleted.

**Operations:** Create (custom only), Read, Update (custom only), Delete (custom only).

### 5.4 Unit Conversions

Conversions define the mathematical relationship between two units.

**Fields:**
- **From Unit** (required) — Source unit.
- **To Unit** (required) — Target unit.
- **Multiplier** (required) — How many of the target unit equal one of the source unit.
- **Bidirectional** (toggle, default: yes) — Whether the reverse conversion is also valid.

**Example:** 1 KG = 1000 G (From: KG, To: G, Multiplier: 1000, Bidirectional: Yes).

**Operations:** Create, Read, Delete.

---

## 6. Items Management

### 6.1 Creating an Item

Navigate to **Items > New Item** (`/inventory/items/new`).

**Required Fields:**
- **Name** — The item name (1–300 characters).
- **Default Unit of Measure** — Select from available units.

**Optional Fields:**
- **SKU** — Stock keeping unit code (unique per organisation, max 50 characters).
- **Barcode** — Barcode value (unique per organisation, max 50 characters).
- **Category** — Select from configured categories.
- **Track Inventory** — Toggle (default: on). When off, the item exists in the catalogue but stock is not tracked.
- **Reorder Level** — When stock falls to or below this quantity, the item appears in the Low Stock alert.
- **Reorder Quantity** — Suggested quantity to reorder.
- **Description** — Free-text description (max 1000 characters).

### 6.2 Item Detail Page

The item detail page (`/inventory/items/{id}`) has three tabs:

**Overview Tab:**
- Displays item settings: Default UoM, Track Inventory, Reorder Level, Reorder Quantity, Description.

**Stock Tab:**
- Shows a table of stock balances across all locations.
- Columns: Location, On Hand, Reserved, Available, Status (badge).
- The status badge shows: In Stock (green), Low Stock (amber), or Out of Stock (red).

**History Tab:**
- Shows the 20 most recent stock movements for this item.
- Columns: Date, Location, Type (badge), Qty (colour-coded: green for positive, red for negative), By (user name).

**Actions:**
- **Edit** — Opens the edit form (pre-populated with current values).
- **Archive** — Soft-deletes the item (sets status to 'archived'). The item is hidden from active lists but all history is preserved.
- **Restore** — Reverses an archive, setting status back to 'active'.
- **Delete** — Hard-deletes the item and all associated data (lines, movements, balances). Requires confirmation.

### 6.3 Items List

The items list page (`/inventory/items`) displays all items with:

**Filters:**
- Search (by name)
- Category filter
- Status filter (Active / Archived)

**Table Columns:**
- Name, SKU, Category, UoM, On Hand, Reorder Level, Status

**Features:**
- Load More pagination (20 items per page)
- Click any row to navigate to the detail page

---

## 7. Stock Adjustments

Adjustments are used to record stock changes that are not transfers or count variances — for example, opening balances, damage, loss, items found, or corrections.

### 7.1 Creating an Adjustment

Navigate to **Adjustments > New Adjustment** (`/inventory/adjustments/new`).

**Header Fields:**
- **Location** (required) — Where the adjustment applies.
- **Reason** (required) — One of:
  - **Opening Balance** — Initial stock entry for new items.
  - **Damage** — Items damaged and removed from stock.
  - **Loss** — Items lost (theft, misplacement).
  - **Found** — Items found and added back to stock.
  - **Correction** — General inventory correction.
  - **Other** — Any other reason.
- **Notes** (optional) — Free-text notes.

**Line Items:**
Each line represents one item being adjusted:
- **Item** (required) — Select from the item catalogue.
- **Quantity** (required) — Signed decimal. Positive values increase stock; negative values decrease stock.
- **Unit Cost** (optional) — Cost per unit for this adjustment.

Multiple lines can be added to a single adjustment.

### 7.2 Adjustment Workflow

```
                 ┌──────────┐
                 │  DRAFT   │
                 └────┬─────┘
                      │ Post
                      ▼
                 ┌──────────┐
                 │  POSTED  │
                 └────┬─────┘
                      │ Void (with reason)
                      ▼
                 ┌──────────┐
                 │   VOID   │
                 └──────────┘
```

**Draft:**
- The adjustment can be freely edited or deleted.
- No impact on stock levels.

**Posted:**
- The system creates stock movements for each line:
  - Positive qty → `ADJUSTMENT_IN` movement
  - Negative qty → `ADJUSTMENT_OUT` movement
- Stock balances are updated immediately.
- The adjustment is locked from further editing.
- Records `postedAt` and `postedBy`.

**Void:**
- The system creates reversal movements (equal and opposite to the original).
- Stock balances are reverted.
- Records `voidedAt`, `voidedBy`, and `voidReason`.
- The adjustment is permanently locked.

### 7.3 Adjustment Detail Page

Displays the full adjustment with:
- Status badge
- Location, reason, date, notes
- Line items table (Item, Qty, Unit Cost)
- Action buttons (Edit, Post, Void, Delete) based on current status
- Attachments panel

---

## 8. Stock Transfers

Transfers move stock between two locations using a two-phase Ship/Receive workflow.

### 8.1 Creating a Transfer

Navigate to **Transfers > New Transfer** (`/inventory/transfers/new`).

**Header Fields:**
- **From Location** (required) — Source location (stock decreases here).
- **To Location** (required) — Destination location (stock increases here). Cannot be the same as the source.
- **Notes** (optional) — Free-text notes.

**Line Items:**
Each line represents one item being transferred:
- **Item** (required) — Select from the item catalogue.
- **Quantity** (required) — Positive decimal only.

### 8.2 Transfer Workflow

```
          ┌──────────┐
          │  DRAFT   │
          └────┬─────┘
               │ Ship
               ▼
          ┌──────────┐
          │IN TRANSIT│
          └────┬─────┘
               │ Receive
               ▼
          ┌──────────┐
          │ RECEIVED │
          └────┬─────┘
               │ Void (with reason)
               ▼
          ┌──────────┐
          │   VOID   │
          └──────────┘
```

**Draft:**
- The transfer can be freely edited or deleted.
- No impact on stock levels.

**In Transit (Ship):**
- The system creates `TRANSFER_OUT` movements at the **source** location.
- Stock decreases at the source location immediately.
- Records `shippedAt` and `shippedBy`.
- The transfer is locked from editing.

**Received:**
- The system creates `TRANSFER_IN` movements at the **destination** location.
- Stock increases at the destination location.
- Records `receivedAt` and `receivedBy`.

**Void:**
- The system creates reversal movements for both the out and in movements.
- Stock at both locations is reverted.
- Records `voidedAt`, `voidedBy`, and `voidReason`.

### 8.3 Two-Phase Stock Impact

The two-phase design means stock is "in transit" between Ship and Receive:

| Phase | Source Location | Destination Location |
|-------|-----------------|---------------------|
| Before Ship | Has stock | No change |
| After Ship (In Transit) | Stock decreased | No change yet |
| After Receive | Stock decreased | Stock increased |

This accurately models real-world scenarios where goods are physically moving between locations.

### 8.4 Transfer Detail Page

Displays the full transfer with:
- Status badge
- From/To locations, date, notes
- Line items table (Item, Qty)
- Action buttons (Edit, Ship, Receive, Void, Delete) based on current status
- Attachments panel

---

## 9. Stock Counts

Stock counts (also called physical inventory or stocktakes) compare system quantities against actual counted quantities and post variance corrections.

### 9.1 Creating a Count

Navigate to **Stock Counts > New Count** (`/inventory/counts/new`).

**Header Fields:**
- **Location** (required) — The location being counted.
- **Count Type** (required) — One of:
  - **Cycle Count** — Partial count of selected items.
  - **Full Count** — Complete inventory count of all items at the location.
- **Notes** (optional) — Free-text notes.

### 9.2 Count Workflow

```
     ┌──────────┐
     │  DRAFT   │
     └────┬─────┘
          │ Start
          ▼
     ┌──────────┐
     │IN PROGRESS│
     └────┬─────┘
          │ Complete
          ▼
     ┌──────────┐
     │COMPLETED │
     └────┬─────┘
          │ Post
          ▼
     ┌──────────┐
     │  POSTED  │
     └────┬─────┘
          │ Void (with reason)
          ▼
     ┌──────────┐
     │   VOID   │
     └──────────┘
```

**Draft:**
- Only the header exists (location, type, notes).
- No count lines yet.
- Can be deleted.

**In Progress (Start):**
- The system takes a **snapshot** of current stock balances at the count location.
- Count lines are automatically created for each item, pre-populated with the `system_qty` (the snapshot value).
- The user enters the actual `counted_qty` for each item using a data entry grid.
- The grid shows: Item Name, SKU, System Qty, Counted Qty (input), and Variance (auto-calculated).
- Tab/Enter navigation between fields for fast data entry.
- Progress indicator shows "X of Y items counted".
- Can save partial progress at any time.

**Completed:**
- All items must have a counted quantity before completing.
- Records `completedAt` and `completedBy`.
- Ready for review before posting.

**Posted:**
- The system calculates variance for each line: `variance = counted_qty - system_qty`.
- For each non-zero variance:
  - Positive variance → `COUNT_VARIANCE_IN` movement (stock increase)
  - Negative variance → `COUNT_VARIANCE_OUT` movement (stock decrease)
- Stock balances are adjusted to match the counted quantities.
- Records `postedAt` and `postedBy`.

**Void:**
- Reverses all variance movements.
- Stock balances revert to pre-count values.
- Records `voidedAt`, `voidedBy`, and `voidReason`.

### 9.3 Count Entry Grid

The count entry interface is designed for speed:

| Item | SKU | System | Counted | Variance |
|------|-----|--------|---------|----------|
| Widget A | WDG-001 | 100 | [input] | — |
| Widget B | WDG-002 | 50 | [input] | — |

- **System** column shows the snapshotted balance (read-only).
- **Counted** column is an editable input field.
- **Variance** is automatically calculated and colour-coded:
  - Zero variance → muted text
  - Non-zero variance → amber text, bold

---

## 10. Stock Ledger & Movement Engine

The stock ledger is the **core transaction engine** of the inventory module. All stock changes flow through it.

### 10.1 Principles

1. **Single Entry Point:** All stock changes go through the `postMovements()` function.
2. **Immutable Ledger:** Stock movements are never updated or deleted. Corrections are made by posting offsetting entries (void operations).
3. **Balance Cache:** The `stock_balances` table is a derived cache, always updated in the same transaction as movement insertion.
4. **Pessimistic Locking:** Balance rows are locked with `SELECT FOR UPDATE` to prevent race conditions during concurrent operations.
5. **Idempotency:** Each movement has a unique `idempotency_key` to prevent duplicate postings if a request is retried.
6. **No Negative Stock:** The engine validates that no balance goes below zero. If a posting would result in negative stock, the entire transaction is rolled back.

### 10.2 Movement Processing

When `postMovements()` is called with an array of movement inputs, for each movement:

```
1. UPSERT balance row
   → Ensure a stock_balances row exists for this item + location.
   → If it doesn't exist, create one with on_hand_qty = 0.

2. LOCK balance row
   → SELECT ... FOR UPDATE (pessimistic lock).
   → Prevents other transactions from modifying this balance simultaneously.

3. INSERT movement record
   → Append to the immutable stock_movements ledger.

4. UPDATE balance
   → Add the signed qty to on_hand_qty.

5. VALIDATE
   → Check that the resulting on_hand_qty >= 0.
   → If negative, throw an error and rollback the entire transaction.
```

### 10.3 Idempotency Keys

Each movement is assigned a unique key based on its source:

| Source | Key Pattern | Example |
|--------|-------------|---------|
| Adjustment | `ADJ-{adjustmentId}-{lineId}` | `ADJ-abc123-def456` |
| Transfer Ship | `TFR-{transferId}-{lineId}-OUT` | `TFR-abc123-def456-OUT` |
| Transfer Receive | `TFR-{transferId}-{lineId}-IN` | `TFR-abc123-def456-IN` |
| Count Variance | `CNT-{countId}-{lineId}` | `CNT-abc123-def456` |
| Void Reversal | `VOID-{originalKey}` | `VOID-ADJ-abc123-def456` |

If a movement with the same key already exists, the operation is rejected (UNIQUE constraint violation), preventing double-posting.

### 10.4 Balance Recalculation

Because all movements are preserved in the ledger, balances can theoretically be recalculated at any time by replaying all movements for a given item + location in chronological order. This provides a safety net against balance corruption.

---

## 11. Stock Levels & Movements Views

### 11.1 Stock Levels Page

The Stock Levels page (`/inventory/stock`) displays a **pivot grid** of current stock levels:

- **Rows:** Items
- **Columns:** Locations
- **Cells:** On-hand quantity at that item + location intersection

This provides a quick overview of stock distribution across the organisation.

**Filters:**
- Item filter
- Location filter
- Low stock toggle

### 11.2 Movements Page

The Movements page (`/inventory/movements`) displays the full stock movement history.

**Filters:**
- Item filter
- Location filter
- Movement type filter (Adjustment In, Adjustment Out, Transfer In, Transfer Out, Count Variance In, Count Variance Out)

**Table Columns:**
- Date
- Item
- Location
- Type (colour-coded badge)
- Qty (green for positive, red for negative)
- By (user name)

**Features:**
- Load More pagination (20 movements per page)
- Responsive design with card layout on mobile

---

## 12. Dashboard & Analytics

The Inventory Dashboard (`/inventory`) provides at-a-glance metrics and alerts.

### 12.1 Stats Cards

Four key performance indicators displayed as cards:

| Metric | Description |
|--------|-------------|
| **Total Items** | Count of all active inventory items |
| **Low Stock** | Count of items at or below their reorder level. Shows "Needs attention" alert if greater than zero. |
| **Total On Hand** | Sum of all on-hand quantities across all locations |
| **Locations** | Count of configured stock locations |

### 12.2 Low Stock Alert Widget

Displays the top 5 items that are at or below their reorder level:
- Item name (clickable link to detail page)
- Stock status badge (colour-coded)
- Current on-hand vs. reorder level ratio

### 12.3 Recent Movements Widget

Displays the most recent stock movements across the organisation:
- Movement type badge
- Item name
- Quantity (colour-coded)
- Date

---

## 13. Attachments System

The attachments system allows users to attach images to inventory entities (items, adjustments, transfers, and counts).

### 13.1 How It Works

**Upload Flow:**

1. **User selects an image** — Any format (JPEG, PNG, etc.) via a file picker.
2. **Client-side compression** — The image is processed entirely in the browser:
   - Decoded into raw pixels using `createImageBitmap()`.
   - Resized to a maximum of **800 x 800 pixels** (maintaining aspect ratio).
   - Re-encoded as **WebP** format using `OffscreenCanvas.convertToBlob()`.
   - Quality is iteratively reduced (starting at 0.85, stepping down by 0.05) until the file is **60 KB or less**.
   - If still too large at minimum quality (0.1), the dimensions are further scaled down.
3. **Base64 encoding** — The compressed binary is converted to a base64 text string.
4. **API upload** — The base64 string is sent as JSON to the API along with the file name.
5. **Database storage** — The base64 string is stored directly in a PostgreSQL `TEXT` column.

**Retrieval Flow:**

1. **API fetch** — The attachment record (including the base64 `data` field) is fetched via GET request.
2. **Data URI rendering** — The image is displayed using: `<img src="data:image/webp;base64,{data}" />`.

### 13.2 Design Decisions

| Decision | Rationale |
|----------|-----------|
| Store in database (not S3/filesystem) | Simplicity — no external storage service needed. Suitable for small images. |
| 60 KB binary limit | Keeps database rows small and queries fast. |
| WebP format | Best compression-to-quality ratio for web images. |
| Client-side compression | Reduces upload size and server CPU load. No server-side image processing libraries needed. |
| Base64 encoding | Text-safe encoding that works with JSON APIs and PostgreSQL TEXT columns. |

### 13.3 Supported Entities

| Entity Type | Use Case |
|-------------|----------|
| Item | Product photos, packaging images |
| Adjustment | Damage evidence, receipts |
| Transfer | Shipment documentation, delivery photos |
| Count | Count verification evidence |

---

## 14. API Endpoints Reference

**Base Path:** `/api/v1/inventory`
**Authentication:** Bearer token required on all endpoints.
**Organisation Context:** Required via middleware (extracted from auth token).

### 14.1 Items

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/inventory/items` | List items (with search, category, status filters) |
| GET | `/inventory/items/{itemId}` | Get item by ID |
| GET | `/inventory/items/{itemId}/balances` | Get item with stock balances and default UoM |
| POST | `/inventory/items` | Create new item |
| PATCH | `/inventory/items/{itemId}` | Update item |
| POST | `/inventory/items/{itemId}/archive` | Archive item |
| POST | `/inventory/items/{itemId}/restore` | Restore archived item |
| DELETE | `/inventory/items/{itemId}` | Delete item |

### 14.2 Adjustments

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/inventory/adjustments` | List adjustments (with status, location filters) |
| GET | `/inventory/adjustments/{id}` | Get adjustment with lines |
| POST | `/inventory/adjustments` | Create draft adjustment |
| PATCH | `/inventory/adjustments/{id}` | Update draft adjustment |
| POST | `/inventory/adjustments/{id}/post` | Post adjustment (creates movements) |
| POST | `/inventory/adjustments/{id}/void` | Void posted adjustment |
| DELETE | `/inventory/adjustments/{id}` | Delete draft adjustment |

### 14.3 Transfers

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/inventory/transfers` | List transfers (with status, location filters) |
| GET | `/inventory/transfers/{id}` | Get transfer with lines |
| POST | `/inventory/transfers` | Create draft transfer |
| PATCH | `/inventory/transfers/{id}` | Update draft transfer |
| POST | `/inventory/transfers/{id}/ship` | Ship transfer |
| POST | `/inventory/transfers/{id}/receive` | Receive transfer |
| POST | `/inventory/transfers/{id}/void` | Void transfer |
| DELETE | `/inventory/transfers/{id}` | Delete draft transfer |

### 14.4 Counts

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/inventory/counts` | List counts (with status, location, type filters) |
| GET | `/inventory/counts/{id}` | Get count with lines |
| POST | `/inventory/counts` | Create draft count |
| POST | `/inventory/counts/{id}/start` | Start count (snapshot balances) |
| PATCH | `/inventory/counts/{id}/lines` | Update counted quantities |
| POST | `/inventory/counts/{id}/complete` | Complete count |
| POST | `/inventory/counts/{id}/post` | Post count (create variance movements) |
| POST | `/inventory/counts/{id}/void` | Void count |
| DELETE | `/inventory/counts/{id}` | Delete draft/in-progress count |

### 14.5 Stock

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/inventory/stock/balances` | Query stock balances (by item, location, low stock) |
| GET | `/inventory/stock/movements` | Query movement history (by item, location, type, date range) |

### 14.6 Settings

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/inventory/categories` | List categories |
| POST | `/inventory/categories` | Create category |
| PATCH | `/inventory/categories/{id}` | Update category |
| DELETE | `/inventory/categories/{id}` | Delete category |
| GET | `/inventory/locations` | List locations |
| POST | `/inventory/locations` | Create location |
| PATCH | `/inventory/locations/{id}` | Update location |
| DELETE | `/inventory/locations/{id}` | Delete location |
| GET | `/inventory/units` | List units (org + global) |
| POST | `/inventory/units` | Create custom unit |
| PATCH | `/inventory/units/{id}` | Update custom unit |
| DELETE | `/inventory/units/{id}` | Delete custom unit |
| GET | `/inventory/units/conversions` | List unit conversions |
| POST | `/inventory/units/conversions` | Create unit conversion |
| DELETE | `/inventory/units/conversions/{id}` | Delete unit conversion |

### 14.7 Attachments

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/inventory/attachments?entityType=X&entityId=Y` | List attachments for an entity |
| GET | `/inventory/attachments/{id}` | Get attachment with base64 data |
| POST | `/inventory/attachments` | Upload attachment |
| DELETE | `/inventory/attachments/{id}` | Delete attachment |

---

## 15. Security & Data Integrity

### 15.1 Organisation Isolation

Every database table includes an `organization_id` column. All queries are filtered by the authenticated user's organisation, ensuring complete data isolation between tenants.

### 15.2 Authentication & Authorisation

- All API endpoints require a valid Bearer token (JWT).
- Organisation context is extracted from the token and injected into the service layer.
- Middleware (`requireAuth`, `requireOrg`) enforces authentication before any handler executes.

### 15.3 Data Integrity Safeguards

| Safeguard | Description |
|-----------|-------------|
| **Pessimistic Locking** | `SELECT FOR UPDATE` on balance rows during movement posting prevents race conditions. |
| **Idempotency Keys** | UNIQUE constraint on `idempotency_key` prevents duplicate postings from retried requests. |
| **No Negative Stock** | The stock ledger validates that no balance goes below zero. The entire transaction rolls back on violation. |
| **Immutable Ledger** | Stock movements are append-only. No UPDATE or DELETE operations exist for this table. |
| **Foreign Key Constraints** | Items, locations, and units referenced by movements use `ON DELETE RESTRICT` — they cannot be deleted while movements reference them. |
| **Cascade Deletes** | Adjustment lines, transfer lines, and count lines cascade-delete when their parent document is deleted. |
| **Input Validation** | All API inputs are validated with Zod schemas before reaching the database layer. |

### 15.4 Audit Trail

The stock movements ledger provides a complete audit trail of every stock change:
- **Who** made the change (`created_by`)
- **What** changed (item, location, quantity, type)
- **When** it happened (`occurred_at`, `created_at`)
- **Why** it changed (`ref_type`, `ref_id` → links back to the source document)

Combined with the status timestamps on documents (`postedAt`, `postedBy`, `voidedAt`, `voidedBy`), this provides full traceability for compliance and investigation purposes.

---

*End of Documentation*
