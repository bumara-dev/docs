---
title: "POS & Scanners"
description: "This document covers the Point of Sale (POS) system and both scanner types — barcode and visual product recognition — across two inventory features..."
---

This document covers the Point of Sale (POS) system and both scanner types — **barcode** and **visual product recognition** — across two inventory features: **Cash Sales** and **Stock Counts**.

---

## Table of Contents

1. [Barcode Scanner](#barcode-scanner)
2. [Visual Product Recognition Scanner](#visual-product-recognition-scanner)
3. [POS — Cash Sales](#pos--cash-sales)
4. [Stock Count Scanner](#stock-count-scanner)
5. [API Reference](#api-reference)
6. [Key Files](#key-files)

---

## Barcode Scanner

**Component:** apps/app/zones/inventory/modules/pos/barcode-scanner-modal.tsx

The `BarcodeScannerModal` is a shared component reused by both Cash Sales and Stock Counts. It opens a camera feed, reads barcodes in real time, and calls `onFound` when a matching inventory item is located.

### How It Works

| Step | Detail |
|------|--------|
| Camera access | Requests the device's rear camera via `getUserMedia` |
| Frame capture | Reads frames from a `<video>` element onto a `<canvas>` at 20 FPS |
| Decoding | Uses **ZXing** (`@zxing/library`) to decode all common 1D/2D barcode formats |
| Upside-down support | Each frame is also decoded at 180° rotation to handle inverted barcodes |
| Deduplication | A 1.5-second cooldown prevents the same barcode firing twice in quick succession |
| Audio feedback | Success: two-tone beep (880 Hz + 1320 Hz). Error: square-wave buzz (300 Hz). Item name is spoken via the Web Speech API |

### Item Matching

The scanner looks up the scanned barcode value against the `barcode` field on inventory items. Each item's barcode is unique within an organisation (`uq_inventory_items_org_barcode`). If no match is found, the error beep fires and nothing is added.

### Props

```typescript
interface Props {
    open: boolean;
    onOpenChange: (open: boolean) => void;
    items: InventoryItem[];        // Pool of items to match against
    onFound: (item: InventoryItem) => void;  // Called on a successful scan
    description?: string;          // Helper text shown inside the modal
}
```

### Camera Error States

| Condition | Behaviour |
|-----------|-----------|
| No camera detected | Displays "No camera found" message |
| Permission denied | Displays "Camera permission denied" message |
| Unsupported browser | Falls back gracefully with an error message |

---

## Visual Product Recognition Scanner

**Component:** apps/app/zones/inventory/modules/pos/image-scanner-modal.tsx

The `ImageScannerModal` identifies inventory items from a camera photo using an AI vision model running entirely **in-browser** — no external API call is made. It is used in both Cash Sales and Stock Counts alongside the barcode scanner as an alternative for products that don't have barcodes.

### Technology Stack

| Layer | Detail |
|-------|--------|
| Library | `@huggingface/transformers` v4.2.0 |
| Model | `Xenova/clip-vit-base-patch32` (CLIP — Contrastive Language-Image Pre-training) |
| Runtime | WebAssembly (WASM) — runs fully client-side, offline-capable |
| Embedding cache | IndexedDB (`bumara-clip-cache`) — persists across sessions |

The CLIP model is loaded once globally as a singleton. A loading progress indicator is shown on first use; subsequent opens are instant.

### Matching Strategy

The scanner attempts two strategies in sequence:

**Strategy 1 — Visual Embedding Match**

1. Fetch reference product images from the API (`/inventory/attachments/items-images`).
2. Compute a CLIP vision embedding for each reference image (cached in IndexedDB by `attachmentId`).
3. Capture a photo from the live camera feed.
4. Compute the embedding for the captured photo.
5. Rank all items by **cosine similarity** between the captured embedding and their reference embeddings.
6. Accept the top match only if it meets both thresholds:

| Threshold | Value | Purpose |
|-----------|-------|---------|
| `MIN_COSINE_SIM` | `0.82` | Minimum similarity score for the winner |
| Margin | `0.04` | Winner must beat second-best by at least this much |

**Strategy 2 — Zero-Shot Text Classification (fallback)**

If Strategy 1 finds no confident visual match (e.g. the item has no reference photo uploaded), CLIP's zero-shot classification pipeline is used:

1. Build a text label for each item from its **name + SKU + description**.
2. Include rejection labels to prevent false positives (see below).
3. Score the captured photo against all labels.
4. Accept the winner only if:

| Threshold | Value | Purpose |
|-----------|-------|---------|
| `MIN_CONFIDENCE` | `0.22` | Minimum score for the winning label |
| `MIN_MARGIN` | `0.06` | Winner must beat second-best by at least this much |

**Rejection labels** used to suppress non-product matches:
- `"a human face or person"`
- `"not a retail product"`
- `"an empty background or surface"`
- `"an unrelated object"`

### Props

```typescript
interface Props {
    open: boolean;
    onOpenChange: (open: boolean) => void;
    items: InventoryItem[];              // Pool of items to identify
    onFound: (item: InventoryItem) => void;  // Called on a successful match
    description?: string;                // Helper text shown inside the modal
}
```

### How It Works (Step by Step)

```
Modal opens
    ↓
Camera feed starts (getUserMedia)
    ↓
CLIP model loads (singleton — instant after first load)
    ↓
Reference images fetched → embeddings computed → cached in IndexedDB
    ↓
User taps Capture button
    ↓
Canvas captures current video frame (JPEG, quality 0.85)
    ↓
Strategy 1: cosine similarity vs. reference embeddings
    ↓ (if no confident match)
Strategy 2: zero-shot text classification against item labels
    ↓
Match found? ──Yes──→ success beep + item name spoken
                       green card shows "X% confidence"
                       onFound(item) called
                       auto-resets to live view after 2 seconds
    ↓ No
Error beep → "Item not recognized" → user retakes photo
```

### Audio & Accessibility Feedback

| Event | Feedback |
|-------|----------|
| Match found | Two-tone beep (880 Hz + 1320 Hz) + item name spoken via Web Speech API |
| No match | Square-wave buzz (300 Hz) |
| Scan cooldown | 1.5-second lockout prevents duplicate processing |

### Embedding Cache (IndexedDB)

Reference image embeddings are stored as `Float32Array` entries keyed by `attachmentId` in the `bumara-clip-cache` IndexedDB store. This means:
- The first use per device incurs the cost of computing all embeddings.
- Subsequent uses load instantly from cache.
- Cache entries are reused across both Cash Sales and Stock Count sessions.

### Reference Images (Attachments API)

For the visual scanner to work on an item, at least one reference photo must be uploaded for that item. Images are stored via the Attachments API:

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/inventory/attachments/items-images` | Bulk-fetch first image per item (for embedding pre-load) |
| `POST` | `/inventory/attachments` | Upload a reference image (base64 WebP, max ~60 KB) |
| `GET` | `/inventory/attachments` | List all attachments for an entity |
| `GET` | `/inventory/attachments/{id}` | Get a single attachment with image data |
| `DELETE` | `/inventory/attachments/{id}` | Delete an attachment |

Images are stored as **WebP** format and retrieved as base64. The service layer is in inventory-attachments.service.ts.

### Camera Error States

| Condition | Behaviour |
|-----------|-----------|
| No camera detected | Displays "No camera found" message |
| Permission denied | Displays "Camera permission denied" message |
| Model load failure | Error shown with retry option |

### Comparison: Visual vs. Barcode Scanner

| | `ImageScannerModal` | `BarcodeScannerModal` |
|--|---------------------|----------------------|
| **Method** | AI vision (CLIP embeddings + zero-shot) | ZXing 1D/2D barcode decode |
| **Item requirement** | Reference photo uploaded | `barcode` field set on item |
| **Speed** | ~1–2 s (model inference) | Near-instant (20 FPS continuous) |
| **Works without labels** | Yes (zero-shot fallback) | No — barcode required |
| **Offline** | Yes (WASM, no external calls) | Yes |
| **Audio feedback** | Beep + speech | Beep + speech |
| **Used in POS** | Yes | Yes |
| **Used in Stock Counts** | Yes | Yes |

---

## POS — Cash Sales

Cash sales use the barcode scanner as part of a full shopping-cart workflow that records a sale and immediately decreases stock via an inventory adjustment.

### Key Components

| File | Role |
|------|------|
| pos/pos-item-search.tsx | Item picker with text search and barcode scanner trigger |
| cash-sales/cash-sale-form.tsx | Full cart, pricing, payment, and receipt |
| cash-sales/cash-sales-list.tsx | Transaction history with date/location filters and CSV/PDF export |

### Workflow

```
Select location
    ↓
Add items  ←── text/SKU search
           ←── BarcodeScannerModal  (scan product barcode)
           ←── ImageScannerModal    (photograph product)
    ↓
Adjust quantities & prices
    ↓
Apply discount (% or fixed)
    ↓
Select payment method
    ↓
Complete Sale
    ↓
POST /adjustments (reason = SALE, negative quantities)
    ↓
POST /adjustments/{id}/post  (creates stock movements)
    ↓
Print receipt (optional)
```

### Item Search (`PosItemSearch`)

- Searches by **item name** or **SKU**.
- Items with zero on-hand stock are **hidden** by default so staff cannot sell unavailable goods.
- Displays on-hand quantities grouped by location.
- Two scanner buttons are available:
  - **Barcode** — opens `BarcodeScannerModal`; continuous live decode adds the item instantly.
  - **Visual** — opens `ImageScannerModal`; user taps capture and the AI identifies the product from the photo.

### Cart Line Structure

```typescript
type CartLine = {
    itemId:    string;
    itemName:  string;
    qty:       string;
    uomId:     string;
    unitCost:  string;   // Used for profit calculation
    salePrice: string;   // Recorded on the adjustment line
    taxType:   string | null;
};
```

### Tax Calculation

Taxes are **inclusive** (already included in the sale price):

| Tax Code | Name | Rate |
|----------|------|------|
| `A` | Standard Rated VAT | 16% |
| `TOT` | Turnover Tax | 4% |
| `null` | Exempt / Zero-rated | 0% |

Tax amount per line = `salePrice × (rate / (1 + rate))`

### Discount

A single discount can be applied to the entire cart as either:
- **Percentage** — e.g. 10% off subtotal
- **Fixed amount** — e.g. ZMW 50 off

### Payment Methods

| Code | Label | Change calculated? |
|------|-------|-------------------|
| `cash` | Cash | Yes |
| `card` | Card | No |
| `mobile_money` | Mobile Money | No |

For cash payments, the cashier enters the **amount tendered** and the form shows the **change due**.

### Receipt

The receipt is formatted for an **80 mm thermal printer** and includes:
- Organisation name and location
- Sale date and time
- Line items: name, qty, unit price, line total
- Subtotal, discount, tax breakdown, grand total
- Payment method and change (if cash)

### Backend Integration

The cash sale is stored as an **inventory adjustment** with `reason = "SALE"`:

1. `useCreateAdjustment()` — creates a `draft` adjustment with negative quantities and `salePrice` on each line.
2. `usePostAdjustment()` — posts the adjustment, writing stock movement records and reducing on-hand balances.

---

## Stock Count Scanner

**Component:** apps/app/zones/inventory/modules/counts/count-entry-grid.tsx

During a stock count, staff scan item barcodes to increment counted quantities instead of typing them manually. The same `BarcodeScannerModal` component is used.

### Workflow

```
Create Count (select location + type)
    ↓
Start Count  →  system snapshots current on-hand quantities
    ↓
Entry Grid: scan items or type quantities manually
    ↓              ↑
  auto-save (400 ms debounce after last change)
    ↓
Complete Count  (all items must have a counted qty)
    ↓
Post Count  →  creates variance adjustments
                (positive if more than system, negative if less)
```

### Count Types

| Type | Description |
|------|-------------|
| `full` | Every item at the selected location |
| `partial` | A manually chosen subset of items |
| `cycle` | Rotating subset (same schema as partial) |

### Count Statuses

```
draft → in_progress → completed → posted → void (optional)
```

### Scanner Behaviour in Count Entry

- Only items that have a `barcode` value set can be scanned.
- Each successful scan **increments** the counted quantity by 1.
- Rapid successive scans of the same item are batched: the quantity is updated after a **400 ms debounce**, meaning two quick scans → quantity +2 in a single API call.
- The scanned row is highlighted in **green** for 2 seconds and the grid auto-scrolls to it.

### Manual Entry

- Keyboard navigation: **Tab** / **Enter** moves focus to the next row.
- Changes save automatically (same 400 ms debounce as scanner).
- A **Save Counts** button triggers an immediate save without waiting for the debounce.

### Count Line Structure

```typescript
type InventoryCountLine = {
    id:          string;
    countId:     string;
    itemId:      string;
    uomId:       string;
    systemQty:   string | null;   // Snapshot taken at count start
    countedQty:  string | null;   // What was physically counted
    varianceQty: string | null;   // countedQty − systemQty
    item?:       InventoryItem;
    uom?:        InventoryUnit;
};
```

### Variance & Posting

When a count is **posted**, the system creates one adjustment line per item where `varianceQty ≠ 0`:
- Positive variance → stock gain adjustment
- Negative variance → stock loss adjustment

These adjustments are then posted to the stock movements ledger.

---

## API Reference

### Adjustments (Cash Sales backend)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/inventory/adjustments` | List adjustments (filters: status, locationId, reason) |
| `GET` | `/inventory/adjustments/{id}` | Get adjustment with lines |
| `POST` | `/inventory/adjustments` | Create draft |
| `PATCH` | `/inventory/adjustments/{id}` | Update draft |
| `POST` | `/inventory/adjustments/{id}/post` | Finalise — writes stock movements |
| `POST` | `/inventory/adjustments/{id}/void` | Reverse a posted adjustment |
| `DELETE` | `/inventory/adjustments/{id}` | Delete a draft |

**Reason codes:** `DAMAGE`, `LOSS`, `FOUND`, `OPENING_BALANCE`, `CORRECTION`, `OTHER`, **`SALE`**

### Attachments (Visual Scanner — product reference images)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/inventory/attachments/items-images` | Bulk-fetch first image per item (used by scanner on open) |
| `POST` | `/inventory/attachments` | Upload a reference image (`entityType: "item"`, base64 WebP) |
| `GET` | `/inventory/attachments` | List attachments for an entity |
| `GET` | `/inventory/attachments/{id}` | Get single attachment with image data |
| `DELETE` | `/inventory/attachments/{id}` | Delete an attachment |

### Counts

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/inventory/counts` | List counts (filters: status, locationId) |
| `GET` | `/inventory/counts/{id}` | Get count with lines |
| `POST` | `/inventory/counts` | Create draft |
| `POST` | `/inventory/counts/{id}/start` | Snapshot system quantities |
| `PATCH` | `/inventory/counts/{id}/lines` | Save counted quantities |
| `POST` | `/inventory/counts/{id}/complete` | Mark as complete |
| `POST` | `/inventory/counts/{id}/post` | Create variance adjustments |
| `POST` | `/inventory/counts/{id}/void` | Reverse a posted count |
| `DELETE` | `/inventory/counts/{id}` | Delete a draft |

---

## Key Files

| Path | Purpose |
|------|---------|
| pos/barcode-scanner-modal.tsx | Core barcode scanning (ZXing + camera) |
| pos/image-scanner-modal.tsx | Visual product recognition (CLIP + WASM) |
| pos/pos-item-search.tsx | Item picker with barcode + visual scanner for POS |
| cash-sales/cash-sale-form.tsx | Cart, tax, payment, receipt |
| cash-sales/cash-sales-list.tsx | Sales history and exports |
| counts/count-entry-grid.tsx | Count entry with scanner integration |
| counts/count-form.tsx | Create count (location + type) |
| hooks/use-adjustments.ts | React Query hooks for adjustments |
| hooks/use-counts.ts | React Query hooks for counts |
| [schema/inventory/items.ts](https://github.com/bumara-dev/bumara/tree/main/packages/database/src/schema/inventory/items.ts) | Item schema — includes `barcode` field |
| [schema/inventory/adjustments.ts](https://github.com/bumara-dev/bumara/tree/main/packages/database/src/schema/inventory/adjustments.ts) | Adjustment schema — includes `SALE` reason |
| [schema/inventory/counts.ts](https://github.com/bumara-dev/bumara/tree/main/packages/database/src/schema/inventory/counts.ts) | Count + count lines schema |
| [modules/adjustments/routes.ts](https://github.com/bumara-dev/bumara/tree/main/apps/api-inventory/src/modules/adjustments/routes.ts) | Adjustments API routes |
| [modules/counts/routes.ts](https://github.com/bumara-dev/bumara/tree/main/apps/api-inventory/src/modules/counts/routes.ts) | Counts API routes |
| [modules/attachments/routes.ts](https://github.com/bumara-dev/bumara/tree/main/apps/api-inventory/src/modules/attachments/routes.ts) | Attachments API routes (product images) |
| [modules/attachments/handlers.ts](https://github.com/bumara-dev/bumara/tree/main/apps/api-inventory/src/modules/attachments/handlers.ts) | Attachment route handlers |
| inventory-attachments.service.ts | Business logic for storing/retrieving product images |
| fetchers/attachments.ts | Client-side API calls for attachment images |
