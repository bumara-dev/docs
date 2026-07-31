---
title: "Inventory Module - API Specification"
description: "API endpoint contracts for the Inventory Module following Bumara's Hono + OpenAPI patterns."
---

## API Conventions

### Base URL

```
/inventory/*
```

### Authentication

All endpoints require:
- **Bearer token**: Clerk JWT in `Authorization` header
- **Organization context**: User must be member of an organization

### Middleware Stack

```typescript
// Standard middleware for inventory endpoints
const inventoryMiddleware = [requireAuth, requireOrg] as const;

// Admin-only endpoints (settings)
const adminMiddleware = [requireAuth, requireOrg, requireRole(['admin'])] as const;

// Manager+ endpoints (posting operations)
const managerMiddleware = [requireAuth, requireOrg, requireRole(['manager', 'admin'])] as const;
```

### Response Format

**Success:**

```typescript
{
  success: true,
  data: T,
  meta?: {
    page: number,
    pageSize: number,
    total: number,
    totalPages: number,
  }
}
```

**Error:**

```typescript
{
  success: false,
  error: {
    code: string,      // Machine-readable error code
    message: string,   // Human-readable message
    details?: object,  // Additional context
  }
}
```

### Domain Errors

| Code | HTTP | Description |
|------|------|-------------|
| `ITEM_NOT_FOUND` | 404 | Item does not exist or not accessible |
| `LOCATION_NOT_FOUND` | 404 | Location does not exist |
| `ADJUSTMENT_NOT_FOUND` | 404 | Adjustment does not exist |
| `TRANSFER_NOT_FOUND` | 404 | Transfer does not exist |
| `COUNT_NOT_FOUND` | 404 | Count does not exist |
| `INVALID_STATUS_TRANSITION` | 422 | Cannot transition from current status |
| `NEGATIVE_STOCK_BLOCKED` | 422 | Operation would result in negative stock |
| `IDEMPOTENCY_REPLAY` | 200 | Duplicate request detected, returning original result |
| `DUPLICATE_SKU` | 409 | SKU already exists for this organization |
| `DUPLICATE_BARCODE` | 409 | Barcode already exists for this organization |
| `ITEM_ARCHIVED` | 422 | Cannot perform operation on archived item |
| `UOM_CONVERSION_NOT_FOUND` | 422 | No conversion path between units |
| `OPERATION_ALREADY_POSTED` | 422 | Operation already posted, cannot modify |
| `OPERATION_ALREADY_VOID` | 422 | Operation already voided |

---

## Items API

### List Items

```
GET /inventory/items
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `search` | string | No | Search by name, SKU, or barcode |
| `categoryId` | uuid | No | Filter by category |
| `status` | enum | No | Filter by status (active, archived) |
| `trackInventory` | boolean | No | Filter by tracking flag |
| `lowStockOnly` | boolean | No | Only items at/below reorder level |
| `page` | number | No | Page number (default: 1) |
| `pageSize` | number | No | Items per page (default: 20, max: 100) |

**Response Schema:**

```typescript
const listItemsResponseSchema = z.object({
  success: z.literal(true),
  data: z.array(z.object({
    id: z.string().uuid(),
    name: z.string(),
    sku: z.string().nullable(),
    barcode: z.string().nullable(),
    categoryId: z.string().uuid().nullable(),
    categoryName: z.string().nullable(),
    defaultUomId: z.string().uuid(),
    defaultUomCode: z.string(),
    trackInventory: z.boolean(),
    reorderLevel: z.string().nullable(), // Numeric as string
    reorderQty: z.string().nullable(),
    status: z.enum(['active', 'archived']),
    totalOnHand: z.string(), // Sum across all locations
    createdAt: z.string().datetime(),
    updatedAt: z.string().datetime(),
  })),
  meta: paginationMetaSchema,
});
```

**Example Response:**

```json
{
  "success": true,
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "Office Chair",
      "sku": "CHAIR-001",
      "barcode": "1234567890123",
      "categoryId": "660e8400-e29b-41d4-a716-446655440001",
      "categoryName": "Furniture",
      "defaultUomId": "770e8400-e29b-41d4-a716-446655440002",
      "defaultUomCode": "EA",
      "trackInventory": true,
      "reorderLevel": "10.0000",
      "reorderQty": "50.0000",
      "status": "active",
      "totalOnHand": "45.0000",
      "createdAt": "2026-01-15T10:30:00Z",
      "updatedAt": "2026-01-20T14:45:00Z"
    }
  ],
  "meta": {
    "page": 1,
    "pageSize": 20,
    "total": 1,
    "totalPages": 1
  }
}
```

### Create Item

```
POST /inventory/items
```

**Auth:** Manager+

**Request Schema:**

```typescript
const createItemInputSchema = z.object({
  name: z.string().min(1).max(200),
  sku: z.string().max(50).optional(),
  barcode: z.string().max(50).optional(),
  description: z.string().max(2000).optional(),
  categoryId: z.string().uuid().optional(),
  defaultUomId: z.string().uuid(),
  trackInventory: z.boolean().default(true),
  reorderLevel: z.string().regex(/^\d+(\.\d{1,4})?$/).optional(),
  reorderQty: z.string().regex(/^\d+(\.\d{1,4})?$/).optional(),
});
```

**Example Request:**

```json
{
  "name": "Office Chair",
  "sku": "CHAIR-001",
  "barcode": "1234567890123",
  "description": "Ergonomic office chair with lumbar support",
  "categoryId": "660e8400-e29b-41d4-a716-446655440001",
  "defaultUomId": "770e8400-e29b-41d4-a716-446655440002",
  "trackInventory": true,
  "reorderLevel": "10",
  "reorderQty": "50"
}
```

### Get Item

```
GET /inventory/items/:id
```

**Response includes:**
- Item details
- Stock by location (array)
- Recent movements (last 10)

### Update Item

```
PATCH /inventory/items/:id
```

**Auth:** Manager+

**Note:** Cannot update if item is archived.

### Archive Item

```
POST /inventory/items/:id/archive
```

**Auth:** Admin

**Business Rules:**
- Archived items cannot have new movements
- Existing stock remains but is frozen
- Can be unarchived later

### Unarchive Item

```
POST /inventory/items/:id/unarchive
```

**Auth:** Admin

---

## Locations API

### List Locations

```
GET /inventory/locations
```

**Response Schema:**

```typescript
const listLocationsResponseSchema = z.object({
  success: z.literal(true),
  data: z.array(z.object({
    id: z.string().uuid(),
    name: z.string(),
    type: z.enum(['warehouse', 'store', 'van', 'other']),
    isDefault: z.boolean(),
    address: z.string().nullable(),
    itemCount: z.number(), // Items with stock at this location
    createdAt: z.string().datetime(),
  })),
});
```

### Create Location

```
POST /inventory/locations
```

**Auth:** Admin

**Request Schema:**

```typescript
const createLocationInputSchema = z.object({
  name: z.string().min(1).max(100),
  type: z.enum(['warehouse', 'store', 'van', 'other']),
  isDefault: z.boolean().default(false),
  address: z.string().max(500).optional(),
});
```

### Update Location

```
PATCH /inventory/locations/:id
```

**Auth:** Admin

### Delete Location

```
DELETE /inventory/locations/:id
```

**Auth:** Admin

**Business Rules:**
- Cannot delete if location has stock
- Cannot delete default location

---

## Units of Measure API

### List Units

```
GET /inventory/units
```

**Response includes:**
- Global units (organizationId = null)
- Organization-specific units

### Create Unit

```
POST /inventory/units
```

**Auth:** Admin

**Request Schema:**

```typescript
const createUnitInputSchema = z.object({
  code: z.string().min(1).max(10).toUpperCase(),
  name: z.string().min(1).max(50),
  precision: z.number().int().min(0).max(8).default(0),
});
```

### List Conversions

```
GET /inventory/conversions
```

### Create Conversion

```
POST /inventory/conversions
```

**Auth:** Admin

**Request Schema:**

```typescript
const createConversionInputSchema = z.object({
  fromUomId: z.string().uuid(),
  toUomId: z.string().uuid(),
  multiplier: z.string().regex(/^\d+(\.\d{1,8})?$/),
  isBidirectional: z.boolean().default(true),
});
```

**Example:** 1 BOX = 12 EA

```json
{
  "fromUomId": "box-uuid",
  "toUomId": "ea-uuid",
  "multiplier": "12",
  "isBidirectional": true
}
```

### Delete Conversion

```
DELETE /inventory/conversions/:id
```

**Auth:** Admin

---

## Stock API

### Get Stock Balances

```
GET /inventory/stock/balances
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `itemId` | uuid | No | Filter by item |
| `locationId` | uuid | No | Filter by location |
| `lowStockOnly` | boolean | No | Only balances at/below reorder level |
| `page` | number | No | Page number |
| `pageSize` | number | No | Items per page |

**Response Schema:**

```typescript
const stockBalanceSchema = z.object({
  id: z.string().uuid(),
  itemId: z.string().uuid(),
  itemName: z.string(),
  itemSku: z.string().nullable(),
  locationId: z.string().uuid(),
  locationName: z.string(),
  onHandQty: z.string(),
  reservedQty: z.string(),
  availableQty: z.string(), // Computed: onHand - reserved
  uomCode: z.string(),
  reorderLevel: z.string().nullable(),
  isLowStock: z.boolean(),
  updatedAt: z.string().datetime(),
});
```

### Get Stock Movements (Ledger)

```
GET /inventory/stock/movements
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `itemId` | uuid | No | Filter by item |
| `locationId` | uuid | No | Filter by location |
| `movementType` | enum | No | Filter by type |
| `refType` | enum | No | Filter by reference type |
| `refId` | uuid | No | Filter by reference ID |
| `startDate` | date | No | Movements on or after date |
| `endDate` | date | No | Movements on or before date |
| `page` | number | No | Page number |
| `pageSize` | number | No | Items per page |

**Response Schema:**

```typescript
const stockMovementSchema = z.object({
  id: z.string().uuid(),
  itemId: z.string().uuid(),
  itemName: z.string(),
  itemSku: z.string().nullable(),
  locationId: z.string().uuid(),
  locationName: z.string(),
  movementType: inventoryMovementTypeEnum,
  qty: z.string(),
  uomCode: z.string(),
  unitCost: z.string().nullable(),
  totalCost: z.string().nullable(),
  refType: inventoryRefTypeEnum,
  refId: z.string().uuid().nullable(),
  notes: z.string().nullable(),
  occurredAt: z.string().datetime(),
  createdBy: z.string(),
  createdByName: z.string(),
  createdAt: z.string().datetime(),
});
```

---

## Adjustments API

### List Adjustments

```
GET /inventory/adjustments
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `locationId` | uuid | No | Filter by location |
| `status` | enum | No | Filter by status |
| `reason` | enum | No | Filter by reason |
| `startDate` | date | No | Created on or after |
| `endDate` | date | No | Created on or before |
| `page` | number | No | Page number |
| `pageSize` | number | No | Items per page |

### Create Adjustment (Draft)

```
POST /inventory/adjustments
```

**Auth:** Manager+

**Request Schema:**

```typescript
const createAdjustmentInputSchema = z.object({
  locationId: z.string().uuid(),
  reason: z.enum(['DAMAGE', 'LOSS', 'FOUND', 'OPENING_BALANCE', 'CORRECTION', 'OTHER']),
  notes: z.string().max(2000).optional(),
  lines: z.array(z.object({
    itemId: z.string().uuid(),
    qty: z.string().regex(/^-?\d+(\.\d{1,4})?$/), // Positive = increase, negative = decrease
    uomId: z.string().uuid(),
    unitCost: z.string().regex(/^\d+(\.\d{1,4})?$/).optional(),
  })).min(1),
});
```

**Example Request:**

```json
{
  "locationId": "loc-uuid",
  "reason": "DAMAGE",
  "notes": "Water damage from roof leak",
  "lines": [
    {
      "itemId": "item-uuid-1",
      "qty": "-5",
      "uomId": "ea-uuid"
    },
    {
      "itemId": "item-uuid-2",
      "qty": "-3",
      "uomId": "ea-uuid"
    }
  ]
}
```

### Get Adjustment

```
GET /inventory/adjustments/:id
```

### Update Adjustment (Draft Only)

```
PATCH /inventory/adjustments/:id
```

**Auth:** Manager+

**Business Rules:**
- Only drafts can be updated
- Can add/remove/modify lines

### Post Adjustment

```
POST /inventory/adjustments/:id/post
```

**Auth:** Manager+

**Request Schema:**

```typescript
const postAdjustmentInputSchema = z.object({
  idempotencyKey: z.string().uuid(), // Client-generated
});
```

**Business Rules:**
- Creates `ADJUSTMENT_IN` or `ADJUSTMENT_OUT` movements per line
- Updates stock balances in transaction
- Checks negative stock constraint
- Records audit trail
- Triggers low-stock check

**Response:**
- HTTP 200: Successfully posted
- HTTP 200 with `IDEMPOTENCY_REPLAY`: Duplicate request, returns original result
- HTTP 422 with `NEGATIVE_STOCK_BLOCKED`: Would cause negative stock

### Void Adjustment

```
POST /inventory/adjustments/:id/void
```

**Auth:** Admin

**Request Schema:**

```typescript
const voidAdjustmentInputSchema = z.object({
  reason: z.string().min(1).max(500),
  idempotencyKey: z.string().uuid(),
});
```

**Business Rules:**
- Creates reverse movements
- Only posted adjustments can be voided
- Records void reason and user

---

## Transfers API

### List Transfers

```
GET /inventory/transfers
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `fromLocationId` | uuid | No | Filter by source |
| `toLocationId` | uuid | No | Filter by destination |
| `status` | enum | No | Filter by status |
| `page` | number | No | Page number |
| `pageSize` | number | No | Items per page |

### Create Transfer (Draft)

```
POST /inventory/transfers
```

**Auth:** Manager+

**Request Schema:**

```typescript
const createTransferInputSchema = z.object({
  fromLocationId: z.string().uuid(),
  toLocationId: z.string().uuid(),
  notes: z.string().max(2000).optional(),
  lines: z.array(z.object({
    itemId: z.string().uuid(),
    qty: z.string().regex(/^\d+(\.\d{1,4})?$/), // Always positive
    uomId: z.string().uuid(),
  })).min(1),
});
```

### Get Transfer

```
GET /inventory/transfers/:id
```

### Update Transfer (Draft Only)

```
PATCH /inventory/transfers/:id
```

**Auth:** Manager+

### Ship Transfer

```
POST /inventory/transfers/:id/ship
```

**Auth:** Manager+

**Request Schema:**

```typescript
const shipTransferInputSchema = z.object({
  idempotencyKey: z.string().uuid(),
});
```

**Business Rules:**
- Creates `TRANSFER_OUT` movements at source location
- Updates status to `in_transit`
- Deducts from source balance
- Checks negative stock at source

### Receive Transfer

```
POST /inventory/transfers/:id/receive
```

**Auth:** Manager+

**Request Schema:**

```typescript
const receiveTransferInputSchema = z.object({
  idempotencyKey: z.string().uuid(),
});
```

**Business Rules:**
- Creates `TRANSFER_IN` movements at destination
- Updates status to `received`
- Adds to destination balance
- Triggers notifications

### Ship and Receive (Simplified MVP)

```
POST /inventory/transfers/:id/complete
```

**Auth:** Manager+

Combines ship + receive in single transaction for MVP simplicity.

### Void Transfer

```
POST /inventory/transfers/:id/void
```

**Auth:** Admin

**Business Rules:**
- If `in_transit`: Creates `TRANSFER_IN` at source (return stock)
- If `received`: Creates reverse movements at both locations
- Cannot void `draft` (just delete instead)

---

## Counts API

### List Counts

```
GET /inventory/counts
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `locationId` | uuid | No | Filter by location |
| `status` | enum | No | Filter by status |
| `countType` | enum | No | Filter by type (cycle, full) |
| `page` | number | No | Page number |
| `pageSize` | number | No | Items per page |

### Create Count (Draft)

```
POST /inventory/counts
```

**Auth:** Manager+

**Request Schema:**

```typescript
const createCountInputSchema = z.object({
  locationId: z.string().uuid(),
  countType: z.enum(['cycle', 'full']),
  notes: z.string().max(2000).optional(),
  // Optional: specify items to count (cycle count)
  // If empty for 'full', includes all items with balance at location
  itemIds: z.array(z.string().uuid()).optional(),
});
```

### Get Count

```
GET /inventory/counts/:id
```

**Response includes:**
- Count header
- Lines with system qty, counted qty, variance
- Summary stats (total lines, counted, variance count)

### Start Count

```
POST /inventory/counts/:id/start
```

**Auth:** Manager+

**Business Rules:**
- Snapshots current `system_qty` for all lines
- Creates lines for items if not already created
- Updates status to `in_progress`

### Update Count Lines

```
PATCH /inventory/counts/:id/lines
```

**Auth:** Manager+

**Request Schema:**

```typescript
const updateCountLinesInputSchema = z.object({
  lines: z.array(z.object({
    itemId: z.string().uuid(),
    countedQty: z.string().regex(/^\d+(\.\d{1,4})?$/),
  })),
});
```

### Complete Count

```
POST /inventory/counts/:id/complete
```

**Auth:** Manager+

**Business Rules:**
- Calculates variance for all lines
- Updates status to `completed`
- Ready for review before posting

### Post Count

```
POST /inventory/counts/:id/post
```

**Auth:** Manager+

**Request Schema:**

```typescript
const postCountInputSchema = z.object({
  idempotencyKey: z.string().uuid(),
});
```

**Business Rules:**
- Creates `COUNT_VARIANCE_IN` or `COUNT_VARIANCE_OUT` per line with variance
- Updates balances to match counted quantities
- Updates status to `posted`

### Void Count

```
POST /inventory/counts/:id/void
```

**Auth:** Admin

**Business Rules:**
- Only `posted` counts can be voided
- Creates reverse variance movements
- Records void reason

---

## Reports API (Read-Only)

### Low Stock Report

```
GET /inventory/reports/low-stock
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `locationId` | uuid | No | Filter by location |
| `categoryId` | uuid | No | Filter by category |

### Stock Valuation Report

```
GET /inventory/reports/valuation
```

**Note:** Requires `unit_cost` on movements. Optional in MVP.

### Movement Summary Report

```
GET /inventory/reports/movement-summary
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `startDate` | date | Yes | Period start |
| `endDate` | date | Yes | Period end |
| `groupBy` | enum | No | item, location, type |

---

## Idempotency

### Purpose

Prevent duplicate operations when clients retry failed requests.

### Implementation

1. Client generates a UUID `idempotencyKey` before calling
2. Server checks if key exists in `stock_movements`
3. If exists, return original result (HTTP 200)
4. If not, proceed with operation

### Endpoints Requiring Idempotency

| Endpoint | Required |
|----------|----------|
| `POST /adjustments/:id/post` | Yes |
| `POST /adjustments/:id/void` | Yes |
| `POST /transfers/:id/ship` | Yes |
| `POST /transfers/:id/receive` | Yes |
| `POST /transfers/:id/complete` | Yes |
| `POST /transfers/:id/void` | Yes |
| `POST /counts/:id/post` | Yes |
| `POST /counts/:id/void` | Yes |

### Client Usage

```typescript
// Frontend: Generate idempotency key before mutation
const idempotencyKey = crypto.randomUUID();

await postAdjustment(adjustmentId, { idempotencyKey });

// On retry (e.g., network failure), use same key
await postAdjustment(adjustmentId, { idempotencyKey }); // Returns same result
```

---

## Related Documentation

- [Data Model](/inventory/03-data-model) — Database schema
- [Workflows](/inventory/05-workflows) — State machines
- [Architecture](/inventory/02-architecture) — System design

---

## Open Questions

1. **Bulk operations**: Should we support bulk item creation/update endpoints?
2. **Export endpoints**: CSV export for items, movements, balances?
3. **Webhook events**: Expose inventory events via external webhooks?
4. **Rate limiting**: Different limits for read vs write operations?
