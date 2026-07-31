---
title: "ZRA Item Sync (Inventory → ZRA Smart Invoice) — Design"
description: "Design: best-effort registration of each inventory item with ZRA Smart Invoice, mirroring the customer sync architecture."
---

**Date:** 2026-06-11
**Status:** Approved
**Owner:** Invoicing domain

## Summary

Register each inventory item with ZRA Smart Invoice via `POST /items/saveItem` when the item is created or updated. The integration is best-effort, mirroring the existing customer-sync architecture: inventory writes never block on ZRA, failures land on the item row, and a manual retry endpoint backs a UI "Retry sync" action. The inventory item form gains the dropdowns and required ZRA fields (item classification, packaging unit, quantity unit) backed by the existing reference-data endpoints.

## Motivation

The existing ZRA Smart Invoice integration sends per-sale transmissions and per-customer registrations, but inventory items themselves are not registered with ZRA. Sales transmissions reference items by `itemCd` / `itemClsCd`; without prior item registration, those references aren't recognised on ZRA's side. The frontend item form also doesn't expose the ZRA-required fields (item classification, packaging unit, quantity unit, country of origin) so users have no way to enter the data ZRA expects.

## Scope

**In scope:**
- Backend: schema additions to `inventory_items`, `ZraSaveItemPayload` interface + `saveItem` method on `ZraApiClient`, new `syncItemToZra` service, repo helper, manual sync endpoint, fire-and-forget `waitUntil` integration into create/update handlers.
- Frontend: new `useZraCodes` / `useZraItemClassifications` query hooks against existing reference-data endpoints, reusable `ZraCodeSelect` + `ZraItemClassificationCombobox` components, new `ZraDetailsSection` in `item-form.tsx`, form schema additions, `useSyncItemToZra` mutation hook.

**Out of scope:**
- Bulk backfill script for items created before this feature ships.
- UI badge + "Retry sync" button on the item detail page (one-line follow-up using the new mutation hook).
- Exposing the "Advanced" ZRA fields (manufacturerTpin, manufacturerItemCd, iplCatCd, tlCatCd, exciseTxCatCd, rentalYn, isrcAplcbYn) in the UI — these default to sensible values when sent to ZRA.
- Stock-IO sync (`saveStockMaster`/`saveStockItems`) — separate concern.

## Approach

**Trigger:** Fire-and-forget from `createInventoryItemHandler` and `updateInventoryItemHandler` via `c.executionCtx?.waitUntil(...)`. The DB write happens first; the ZRA call runs after the response is sent. Matches the customer-sync pattern exactly.

**Failure model:** Service never throws. All outcomes (synced, failed, skipped_no_device) are stored on the item row.

**No-device behavior:** Soft skip with status `skipped_no_device`. Items can be created without a registered VSDC device.

**itemCd generation:** Items get a ZRA `itemCd` generated on first successful sync: `<branchId>-<7-digit org-scoped counter>` (e.g. `"000-0000001"`). Persisted to `zra_item_code`. Reused on subsequent syncs.

**itemTyCd mapping:** From existing `productType`: `raw_material → "1"`, `finished_good → "2"`, `service → "3"`, any other value → `"2"`. No new UI field.

## Database schema

Add 13 columns to `inventory_items`:

| Column | Type | Notes |
|---|---|---|
| `zra_item_classification_code` | text NULL | 8-digit ZRA item class (e.g. `"43322555"`). Required for sync — null = sync will skip until set. |
| `zra_origin_country_code` | text NULL | 2-letter ZRA country code (e.g. `"ZM"`). Defaults to `"ZM"` at sync time when null. |
| `zra_packaging_unit_code` | text NULL | ZRA packaging code (e.g. `"BOX"`). Required for sync. |
| `zra_quantity_unit_code` | text NULL | ZRA quantity code (e.g. `"U"`). Required for sync. |
| `zra_item_code` | text NULL | Our generated `itemCd`. Generated on first sync; persisted; reused thereafter. |
| `zra_default_price` | numeric(18,4) NULL | ZRA `dftPrc` / `rrp` source. Defaults to 0 when null. |
| `zra_sync_status` | text NULL | `'synced'` \| `'failed'` \| `'skipped_no_device'`. |
| `zra_synced_at` | timestamp NULL | Last successful sync. |
| `zra_sync_attempted_at` | timestamp NULL | Last attempt. |
| `zra_sync_error` | text NULL | Last error message; null on success. |
| `zra_result_cd` | text NULL | Raw ZRA resultCd from last call. |
| `zra_result_msg` | text NULL | Raw ZRA resultMsg from last call. |
| `zra_result_dt` | text NULL | Raw ZRA resultDt (YYYYMMDDHHmmss). |

No new indexes. Existing migrations cover the rest of the table.

## API client addition

In `packages/api-services/src/domains/invoicing/zra/zra-api-client.ts`:

```typescript
export interface ZraSaveItemPayload {
  tpin: string;
  bhfId: string;
  itemCd: string;
  itemClsCd: string;
  itemTyCd: string;          // "1" | "2" | "3"
  itemNm: string;
  itemStdNm: string;
  orgnNatCd: string;         // ISO 2-letter (e.g. "ZM")
  pkgUnitCd: string;
  qtyUnitCd: string;
  vatCatCd: string;          // "A" | "B" | "C1" ... — reuses tax type
  iplCatCd: string | null;
  tlCatCd: string | null;
  exciseTxCatCd: string | null;
  btchNo: string | null;
  bcd: string | null;        // barcode
  dftPrc: number;
  manufacturerTpin: string | null;
  manufacturerItemCd: string | null;
  rrp: string;               // ZRA wants string
  svcChargeYn: "Y" | "N";
  rentalYn: "Y" | "N";
  addInfo: string | null;
  sftyQty: number;
  isrcAplcbYn: "Y" | "N";
  useYn: "Y" | "N";
  regrNm: string;
  regrId: string;
  modrNm: string;
  modrId: string;
}

saveItem(payload: ZraSaveItemPayload): Promise<ZraApiResponse>
```

Reuses the existing `ZRA_API_ENDPOINTS.ITEMS = "/items/saveItem"` constant. Non-async method, bare return (matches `saveBranchCustomer` and the `fetch*` convention).

## Field mapping

| ZRA field | Source | Default |
|---|---|---|
| `tpin` | `device.tpin` | required |
| `bhfId` | `device.branchId` | `"000"` |
| `itemCd` | `item.zraItemCode` or freshly generated | required |
| `itemClsCd` | `item.zraItemClassificationCode` | required — if null, skip sync (see "Pre-sync validation" below) |
| `itemTyCd` | mapped from `item.productType` | `raw_material→"1"`, `finished_good→"2"`, `service→"3"`, other→`"2"` |
| `itemNm` | `item.name` | required (notNull) |
| `itemStdNm` | `item.name` | same as itemNm |
| `orgnNatCd` | `item.zraOriginCountryCode` | `"ZM"` |
| `pkgUnitCd` | `item.zraPackagingUnitCode` | required — if null, skip sync |
| `qtyUnitCd` | `item.zraQuantityUnitCode` | required — if null, skip sync |
| `vatCatCd` | `item.taxType` | `"A"` |
| `iplCatCd` | — | `null` |
| `tlCatCd` | — | `null` |
| `exciseTxCatCd` | — | `null` |
| `btchNo` | — | `null` |
| `bcd` | `item.barcode` | `null` |
| `dftPrc` | `item.zraDefaultPrice` | `0` |
| `manufacturerTpin` | — | `null` |
| `manufacturerItemCd` | — | `null` |
| `rrp` | `String(item.zraDefaultPrice ?? 0)` | `"0"` |
| `svcChargeYn` | derived: `productType === "service" ? "Y" : "N"` | `"N"` |
| `rentalYn` | — | `"N"` |
| `addInfo` | — | `null` |
| `sftyQty` | `Number(item.reorderLevel ?? 0)` | `0` |
| `isrcAplcbYn` | — | `"N"` |
| `useYn` | `item.isActive ? "Y" : "N"` | `"Y"` |
| `regrNm`/`regrId`/`modrNm`/`modrId` | `ctx.userId ?? "SYSTEM"` | `"SYSTEM"` |

### Pre-sync validation: missing required ZRA fields

When `zraItemClassificationCode`, `zraPackagingUnitCode`, or `zraQuantityUnitCode` is null on an item, the sync service does NOT call ZRA. Instead it patches `zra_sync_status='failed'`, `zra_sync_error='Missing required ZRA fields: <list>'`. This means items can be created via the API without those fields (admin/seed flows), and the sync status visibly reflects what's blocking them. The frontend form requires the fields on submit so normal user-created items always have them.

## Service

New file `packages/api-services/src/domains/invoicing/zra/zra-item-sync.service.ts`:

```typescript
export interface ItemSyncResult {
  status: "synced" | "failed" | "skipped_no_device";
  error?: string;
}

export async function syncItemToZra(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  itemId: string
): Promise<ItemSyncResult>
```

Flow:
1. `requireOrganizationContext(ctx)` → `orgId`.
2. `findInventoryItemById(deps.db, itemId, orgId)`. If null → throw `NOT_FOUND` (programmer error).
3. `findVsdcDevice(deps.db, orgId)`. If null or not initialized → patch status `skipped_no_device`, return.
4. Validate required ZRA fields on the item. If any missing → patch status `failed` with descriptive error, return.
5. If `item.zraItemCode` is null → `generateZraItemCode(db, orgId, device.branchId)` (see Repository).
6. Build `ZraSaveItemPayload` from item + device per the mapping above.
7. Build `ZraApiClient` from device. Call `client.saveItem(payload)`.
8. On `apiResult.success === true` → patch `zra_sync_status='synced'`, `zra_synced_at=now`, `zra_sync_attempted_at=now`, `zra_sync_error=null`, plus `zra_result_cd/msg/dt` from `apiResult`, plus `zra_item_code` if it was generated this run. Return `{ status: 'synced' }`.
9. On `apiResult.success === false` → patch `zra_sync_status='failed'`, `zra_sync_attempted_at=now`, `zra_sync_error=apiResult.error ?? apiResult.message`, plus `zra_result_cd/msg/dt`. Return `{ status: 'failed', error: ... }`.

**Invariant:** Function never throws except for `NOT_FOUND` in step 2. Every other failure path returns a result.

### itemCd generation idempotency

Step 5 only runs when `zraItemCode` is null. Once generated and persisted on a successful sync, every subsequent sync reuses the same value. Even if the first sync fails, the generated code is persisted on the row so a retry uses the same `itemCd` (matters because ZRA may have already partially registered it).

## Repository helpers

In `packages/database/src/repositories/inventory-items.ts` (or wherever `inventoryItems` queries live — verified during implementation):

```typescript
updateInventoryItemZraSyncStatus(
  db: DatabaseClient,
  itemId: string,
  organizationId: string,
  patch: {
    zraSyncStatus?: 'synced' | 'failed' | 'skipped_no_device';
    zraSyncedAt?: Date | null;
    zraSyncAttemptedAt?: Date;
    zraSyncError?: string | null;
    zraResultCd?: string | null;
    zraResultMsg?: string | null;
    zraResultDt?: string | null;
    zraItemCode?: string | null;
  }
): Promise<void>
```

Bypasses the regular `updateInventoryItem` to avoid triggering SKU/barcode uniqueness validation on a status-only update.

Plus the sequence helper:

```typescript
getNextZraItemCounter(
  db: DatabaseClient,
  organizationId: string
): Promise<number>
```

Reuses the existing `document_sequences` infrastructure (used by `getNextDocumentNumber`) under a new type discriminator `"zra_item"`. The service formats the result as `<branchId>-<padStart(counter, 7, "0")>`.

## Backend HTTP routes

Modify `apps/api-invoicing/src/routes/invoicing/inventory/items/handlers.ts` (path verified during implementation):

- `createInventoryItemHandler`: after `await createInventoryItem(...)`, add the fire-and-forget `waitUntil(syncItemToZra(...))` block. Same pattern as `createCustomerHandler`.
- `updateInventoryItemHandler`: same after `await updateInventoryItem(...)`.

New manual sync route:

| Method | Path | Auth | Handler |
|---|---|---|---|
| POST | `/invoicing/inventory-items/{itemId}/sync-zra` | `requireAuth`, `requireOrg` | `syncInventoryItemToZraHandler` |

Awaits `syncItemToZra` synchronously and returns 200 with `{ status, error? }`.

## Frontend query hooks

New files in `apps/app/lib/queries/zra-reference-data/`:

```typescript
useZraCodes(codeClass: string)
// → GET /invoicing/zra/codes/:codeClass
// queryKey: ['zra', 'codes', codeClass]
// staleTime: 1000 * 60 * 10
// returns: Array<{ code: string; name: string; description: string | null; sortOrder: number; userDefinedCode1: string | null }>

useZraItemClassifications(opts: { level?: number; search?: string; limit?: number })
// → GET /invoicing/zra/item-classifications?level=5&search=...&limit=50
// queryKey: ['zra', 'item-classifications', opts.level, opts.search, opts.limit]
// staleTime: 1000 * 60 * 10
// returns: Array<{ itemClassCode: string; itemClassName: string; itemClassLevel: number; taxTypeCode: string | null }>
```

Both follow the existing `useCategories` shape (uses `useAuth().getToken`, calls a fetcher with the token, returns the raw `useQuery` result).

New mutation hook in `apps/app/lib/queries/inventory/hooks/`:

```typescript
useSyncItemToZra(itemId: string)
// → POST /invoicing/inventory-items/:itemId/sync-zra
// On success: invalidates the item detail query so the new zra_sync_status renders.
```

## Frontend reusable components

In `apps/app/components/features/invoicing/zra/`:

**`<ZraCodeSelect codeClass="05" value={...} onChange={...} placeholder={...} disabled={...} />`**
- Backed by `useZraCodes(codeClass)`.
- Renders existing `<Select>` from `@repo/design-system`.
- Shows loading state while fetching.
- Used three places: country (`zraOriginCountryCode`), packaging unit (`zraPackagingUnitCode`), quantity unit (`zraQuantityUnitCode`).

**`<ZraItemClassificationCombobox value={...} onChange={...} disabled={...} />`**
- Backed by `useZraItemClassifications({ level: 5, search: debouncedQuery, limit: 50 })`.
- Renders existing `<Command>` + `<Popover>` (searchable). 250 ms debounce.
- Displays `<itemClassCode> — <itemClassName>` in dropdown rows. Stores `itemClassCode` as the value.
- Used once: item classification on the form.

### Code class IDs

Exact ZRA code class IDs for country, packaging unit, quantity unit are vendor-defined. During implementation, run `GET /invoicing/zra/code-classes` once against the synced reference data to confirm the real IDs and bake them in as named constants in the component file. The plan flags this as a discrete verification step.

## Form schema additions

Extend `itemFormSchema` in `apps/app/components/features/inventory/items/item-form.tsx`:

```typescript
const itemFormSchema = z.object({
  // all existing fields unchanged
  // ...

  // REQUIRED for ZRA
  zraItemClassificationCode: z.string().min(1, "Item classification is required"),
  zraPackagingUnitCode: z.string().min(1, "Packaging unit is required"),
  zraQuantityUnitCode: z.string().min(1, "Quantity unit is required"),

  // OPTIONAL — defaults applied server-side
  zraOriginCountryCode: z.string().optional().or(z.literal("")),
  zraDefaultPrice: z.string().optional().or(z.literal("")),
});
```

The three required fields are checked at form submit. The optional pair are passed through; the sync service applies defaults if absent.

## Form UI: new section

A new section `ZraDetailsSection` rendered between `BasicInfoSection` and `InventoryStockSection`:

```
┌─ ZRA Smart Invoice Details ─────────────────────────────────┐
│ These fields are used when this item is sent to ZRA Smart    │
│ Invoice.                                                     │
│                                                              │
│ Item Classification *                                        │
│ [ZraItemClassificationCombobox]                              │
│                                                              │
│ Packaging Unit *           Quantity Unit *                   │
│ [ZraCodeSelect "<pkg ID>"] [ZraCodeSelect "<qty ID>"]        │
│                                                              │
│ Country of Origin          Default Price (ZMW)               │
│ [ZraCodeSelect "<ctry ID>"] [number input]                   │
└──────────────────────────────────────────────────────────────┘
```

Section renders regardless of whether the org has a VSDC device — fields are still persisted, just not synced.

### Edit behavior

On edit, fields pre-populate from `item.zraItemClassificationCode` etc. If the user changes any of these on an existing item, the post-update `waitUntil` re-syncs to ZRA automatically.

## Error handling

| Scenario | Behavior |
|---|---|
| Org has no active VSDC device | `zra_sync_status='skipped_no_device'`. No ZRA call. Item create/update succeeds normally. |
| Required ZRA fields missing on item | `zra_sync_status='failed'`, `zra_sync_error='Missing required ZRA fields: <list>'`. No ZRA call. |
| ZRA returns `resultCd !== "000"` | `zra_sync_status='failed'`, error persisted. Item create/update already succeeded. |
| ZRA returns `924 — itemCd already exists` (or similar) | Treated as failure; admin can inspect via the item row's `zra_result_msg`. (Could become a special-case future enhancement; not in this plan.) |
| Network/timeout (after `makeRequest` retries) | Same as above; `ZraApiClient` returns `success: false`. |
| DB write inside sync throws | Caught by the `.catch()` on `waitUntil`; logged via `console.error`. No way to surface to the client (response already sent). Manual retry resets. |
| Item not found in org | `NOT_FOUND` ServiceError. Only possible if `itemId` is wrong — never in the post-create path. |

**Invariant:** HTTP create/update never fails because of ZRA.

## Testing

Service unit tests at `packages/api-services/src/domains/invoicing/zra/__tests__/zra-item-sync.service.test.ts`, mocked repo + `ZraApiClient`:
- No device → `skipped_no_device`, no API call, status patched.
- Missing required field (e.g. `zraItemClassificationCode` null) → `failed` with descriptive error, no API call.
- ZRA success → `synced`, full field mapping verified by inspecting the payload passed to `saveBranchCustomer`-style assertion.
- ZRA success on item without `zraItemCode` → generates one, persists, payload uses generated code.
- ZRA success on item with existing `zraItemCode` → reuses, no generation.
- ZRA failure → `failed`, error persisted, `zra_result_cd` populated, never throws.
- `itemTyCd` mapping: `productType='service'` → `"3"` AND `svcChargeYn='Y'`.
- Inactive item → `useYn='N'`.

No frontend component tests (codebase convention).

## Observability

- `inventory_items.zra_sync_status` + `zra_sync_attempted_at` + `zra_sync_error` + `zra_result_cd` are the debugging surface.
- `console.error` with `[ZRA Item Sync]` prefix for CF Logpush filtering.
- The result-code persistence comes free from the prior ZRA Result Codes feature pattern.

## Open follow-ups (not in this plan)

- Item detail page: sync status badge + "Retry sync" button (uses `useSyncItemToZra`).
- Bulk backfill script for items created before this feature.
- Stock movement sync via `/stockIO/saveStockItems` and `/stockMaster/saveStockMaster`.
- Surface ZRA `924` (duplicate itemCd) as an auto-resolution flow.
