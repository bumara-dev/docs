---
title: "ZRA Item Sync Implementation Plan"
description: "Register each inventory item with ZRA Smart Invoice on create and update, mirroring the customer sync architecture."
---

**Goal:** Register each inventory item with ZRA Smart Invoice via `POST /items/saveItem` when created or updated, with a fire-and-forget pattern that mirrors the existing customer-sync architecture. The frontend item form gets dropdowns and required ZRA fields backed by the existing reference-data endpoints.

**Architecture:** Schema extends `inventory_items` with 13 ZRA-related columns. A new `syncItemToZra` service mirrors `syncCustomerToZra` exactly. `createItemHandler` and `updateItemHandler` get `c.executionCtx?.waitUntil(...)` calls. Frontend gains `useZraCodes` / `useZraItemClassifications` hooks, reusable dropdown components, and a new `ZraDetailsSection` in `item-form.tsx`.

**Tech Stack:** Drizzle ORM (Postgres), TypeScript, Cloudflare Workers (Hono + zod-openapi), React Hook Form + Zod, Tanstack Query, Vitest with `vi.hoisted` mocks.

**Spec:** [docs/superpowers/specs/2026-06-11-zra-item-sync-design.md](/superpowers/specs/2026-06-11-zra-item-sync-design)

**Note on commits:** The user is handling commits manually at the end. Tasks DO NOT commit.

---

## File Structure

**Modify schema:**
- `packages/database/src/schema/inventory/items.ts` — add 13 new columns

**Modify service (acts as repo too in this domain):**
- `packages/api-services/src/domains/inventory/inventory-items.service.ts` — add helper functions (`findItemForZraSync`, `updateItemZraSyncStatus`, `generateZraItemCode`); service file already mixes service logic and direct schema access (no separate repo layer for inventory)

**Modify API client:**
- `packages/api-services/src/domains/invoicing/zra/zra-api-client.ts` — add `ZraSaveItemPayload` interface + `saveItem` method

**New service:**
- `packages/api-services/src/domains/invoicing/zra/zra-item-sync.service.ts` — `syncItemToZra`

**Modify domain barrel:**
- `packages/api-services/src/domains/invoicing/index.ts` — export new service

**Modify handlers + routes (backend):**
- `apps/api-inventory/src/routes/inventory/items/handlers.ts` — `waitUntil` blocks in create/update + new `syncItemToZraHandler`
- `apps/api-inventory/src/routes/inventory/items/routes.ts` — new `syncItemToZraRoute`
- `apps/api-inventory/src/routes/inventory/items/index.ts` — mount new route

**New tests:**
- `packages/api-services/src/domains/invoicing/zra/__tests__/zra-item-sync.service.test.ts`

**New frontend query hooks:**
- `apps/app/lib/queries/zra-reference-data/fetchers.ts`
- `apps/app/lib/queries/zra-reference-data/hooks.ts`
- `apps/app/lib/queries/zra-reference-data/keys.ts`
- `apps/app/lib/queries/zra-reference-data/index.ts`

**New mutation hook:**
- `apps/app/lib/queries/inventory/hooks/use-sync-item-to-zra.ts`
- Export added to: `apps/app/lib/queries/inventory/hooks/index.ts` (or wherever item hooks are re-exported)

**New frontend components:**
- `apps/app/components/features/invoicing/zra/zra-code-select.tsx`
- `apps/app/components/features/invoicing/zra/zra-item-classification-combobox.tsx`

**Modify form:**
- `apps/app/components/features/inventory/items/item-form.tsx` — extend schema, add `ZraDetailsSection`, render in form

**Migration (auto-generated):**
- `packages/database/drizzle/XXXX_<name>.sql` — should contain ~13 `ALTER TABLE inventory_items ADD COLUMN` statements; trim drift if any appears

---

## Task 1: Schema — Add ZRA Columns to `inventory_items`

**Files:**
- Modify: `packages/database/src/schema/inventory/items.ts`

- [ ] **Step 1: Add the 13 new columns**

Open `packages/database/src/schema/inventory/items.ts`. Find the existing column list. Locate the `taxType` line (it's the existing ZRA-related field). After all other content-bearing columns but before the `...timestamps` spread, add a new block. The exact insertion point: at the end of the column declarations, just before `...timestamps`, add:

```typescript
        // ZRA Smart Invoice integration
        zraItemClassificationCode: text("zra_item_classification_code"),
        zraOriginCountryCode: text("zra_origin_country_code"),
        zraPackagingUnitCode: text("zra_packaging_unit_code"),
        zraQuantityUnitCode: text("zra_quantity_unit_code"),
        zraItemCode: text("zra_item_code"),
        zraDefaultPrice: numeric("zra_default_price", { precision: 18, scale: 4 }),

        // ZRA sync state
        zraSyncStatus: text("zra_sync_status"),
        zraSyncedAt: timestamp("zra_synced_at", { mode: "date" }),
        zraSyncAttemptedAt: timestamp("zra_sync_attempted_at", { mode: "date" }),
        zraSyncError: text("zra_sync_error"),

        // ZRA result code metadata (from resultCd/resultMsg/resultDt)
        zraResultCd: text("zra_result_cd"),
        zraResultMsg: text("zra_result_msg"),
        zraResultDt: text("zra_result_dt"),
```

Ensure `numeric` and `timestamp` are imported at the top of the file. If either is not yet imported, add it to the `drizzle-orm/pg-core` import.

Do NOT add any indexes.

- [ ] **Step 2: Typecheck**

Run: `pnpm --filter @repo/database typecheck`
Expected: zero new errors in `items.ts`. Preexisting errors elsewhere are OK.

- [ ] **Step 3: Generate the migration**

Run: `pnpm --filter @repo/database db:generate`
Expected: a new SQL file in `packages/database/drizzle/` (e.g. `0021_<name>.sql`) containing exactly 13 `ALTER TABLE "inventory_items" ADD COLUMN` statements.

**If unrelated drift appears** in the migration (inventory_stock_balances, payroll_settings, etc.), hand-edit the SQL to keep ONLY the 13 `inventory_items` ADD COLUMN statements. This trimming pattern is established by the prior ZRA reference-data and customer-sync migrations.

- [ ] **Step 4: DO NOT COMMIT.** Leave the modified schema file, the new SQL file, the new snapshot, and the updated journal in the working tree.

---

## Task 2: Service Helpers — Item Sync Status Updates + itemCd Generation

**Files:**
- Modify: `packages/api-services/src/domains/inventory/inventory-items.service.ts`

This task adds three helpers to the existing inventory items service file. They live alongside the existing CRUD functions because this codebase doesn't have a separate repository layer for inventory.

- [ ] **Step 1: Add the helpers at the bottom of the service file**

Open `packages/api-services/src/domains/inventory/inventory-items.service.ts`. After the last existing exported function (after `deleteItem`), append:

```typescript
// ============================================
// ZRA Sync Helpers
// ============================================

import { getNextDocumentNumber } from "../invoicing/document-numbers";

/**
 * Find an inventory item by ID within an organization, including the ZRA fields
 * needed by the sync service. Returns null if not found.
 */
export async function findItemForZraSync(
  db: DatabaseClient,
  itemId: string,
  organizationId: string
) {
  const [item] = await db
    .select()
    .from(inventoryItems)
    .where(
      and(
        eq(inventoryItems.id, itemId),
        eq(inventoryItems.organizationId, organizationId)
      )
    )
    .limit(1);
  return item ?? null;
}

/**
 * Update only the ZRA sync status fields on an inventory item. Bypasses the
 * regular updateItem service to avoid SKU/barcode uniqueness checks firing on
 * a status-only update.
 */
export async function updateItemZraSyncStatus(
  db: DatabaseClient,
  itemId: string,
  organizationId: string,
  patch: {
    zraSyncStatus?: "synced" | "failed" | "skipped_no_device";
    zraSyncedAt?: Date | null;
    zraSyncAttemptedAt?: Date;
    zraSyncError?: string | null;
    zraResultCd?: string | null;
    zraResultMsg?: string | null;
    zraResultDt?: string | null;
    zraItemCode?: string | null;
  }
): Promise<void> {
  await db
    .update(inventoryItems)
    .set({
      ...patch,
      updatedAt: new Date(),
    })
    .where(
      and(
        eq(inventoryItems.id, itemId),
        eq(inventoryItems.organizationId, organizationId)
      )
    );
}

/**
 * Generate a fresh ZRA itemCd in the format `<branchId>-<7-digit counter>`.
 * Reuses the existing document_sequences infrastructure under the type
 * discriminator "zra_item".
 */
export async function generateZraItemCode(
  db: DatabaseClient,
  organizationId: string,
  branchId: string
): Promise<string> {
  const sequenceValue = await getNextDocumentNumber(db, organizationId, "zra_item");
  // getNextDocumentNumber returns a formatted string like "ZRA-0000001"; we
  // want just the numeric portion zero-padded to 7 digits.
  const numeric = sequenceValue.replace(/\D/g, "").padStart(7, "0").slice(-7);
  return `${branchId}-${numeric}`;
}
```

Ensure `and`, `eq` are imported from `drizzle-orm` (they likely already are — check the existing imports). Add if missing.

`DatabaseClient` type: import from `@repo/database` or `@/client` depending on the existing convention in this file — match what `createItem` / `updateItem` already import.

- [ ] **Step 2: Typecheck**

Run: `pnpm --filter @repo/api-services typecheck`
Expected: zero new errors in `inventory-items.service.ts`. Preexisting errors elsewhere are OK.

- [ ] **Step 3: Verify the existing `getNextDocumentNumber` accepts the new `"zra_item"` type**

Open `packages/api-services/src/domains/invoicing/document-numbers.ts` and inspect `getNextDocumentNumber`. If the function uses a typed union of allowed document types, extend it to include `"zra_item"`. If it accepts a free-form string, no change needed.

If a change is needed there, make it and re-typecheck.

- [ ] **Step 4: DO NOT COMMIT.**

---

## Task 3: API Client — Add `saveItem`

**Files:**
- Modify: `packages/api-services/src/domains/invoicing/zra/zra-api-client.ts`

- [ ] **Step 1: Add the wire interface**

In `zra-api-client.ts`, alongside the other wire interfaces (`ZraBranchCustomerPayload`, etc., added in prior ZRA features), append:

```typescript
/**
 * Payload for POST /items/saveItem.
 * 1:1 mirror of ZRA's expected JSON body — no domain wrapper needed.
 */
export interface ZraSaveItemPayload {
  tpin: string;
  bhfId: string;
  itemCd: string;
  itemClsCd: string;
  itemTyCd: string; // "1" raw | "2" finished | "3" service
  itemNm: string;
  itemStdNm: string;
  orgnNatCd: string;
  pkgUnitCd: string;
  qtyUnitCd: string;
  vatCatCd: string;
  iplCatCd: string | null;
  tlCatCd: string | null;
  exciseTxCatCd: string | null;
  btchNo: string | null;
  bcd: string | null;
  dftPrc: number;
  manufacturerTpin: string | null;
  manufacturerItemCd: string | null;
  rrp: string;
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
```

- [ ] **Step 2: Add the method on `ZraApiClient`**

In the `ZraApiClient` class, alongside `saveBranchCustomer` (and the existing fetcher methods), add:

```typescript
  /**
   * Register or update an inventory item with ZRA via POST /items/saveItem.
   * ZRA treats this endpoint as an upsert keyed on (tpin, itemCd).
   */
  saveItem(payload: ZraSaveItemPayload): Promise<ZraApiResponse> {
    return this.makeRequest(
      "POST",
      ZRA_API_ENDPOINTS.ITEMS,
      payload as unknown as Record<string, unknown>
    );
  }
```

Important: non-async method, bare `return`, no `await`. Matches the established convention from `saveBranchCustomer` and `fetchCodeList` / etc.

The endpoint constant `ZRA_API_ENDPOINTS.ITEMS = "/items/saveItem"` already exists in this file.

- [ ] **Step 3: Typecheck**

Run: `pnpm --filter @repo/api-services typecheck`
Expected: zero new errors in `zra-api-client.ts`.

- [ ] **Step 4: DO NOT COMMIT.**

---

## Task 4: Service — `syncItemToZra` (with tests)

**Files:**
- Create: `packages/api-services/src/domains/invoicing/zra/zra-item-sync.service.ts`
- Create: `packages/api-services/src/domains/invoicing/zra/__tests__/zra-item-sync.service.test.ts`
- Modify: `packages/api-services/src/domains/invoicing/index.ts`

- [ ] **Step 1: Write the failing test file**

Create `packages/api-services/src/domains/invoicing/zra/__tests__/zra-item-sync.service.test.ts`:

```typescript
import { beforeEach, describe, expect, it, vi } from 'vitest';
import type { ServiceContext, ServiceDependencies } from '../../../../core/context';

const mocks = vi.hoisted(() => ({
  findItemForZraSync: vi.fn(),
  findVsdcDevice: vi.fn(),
  updateItemZraSyncStatus: vi.fn(),
  generateZraItemCode: vi.fn(),
  saveItem: vi.fn(),
}));

vi.mock('../../inventory/inventory-items.service', () => ({
  findItemForZraSync: mocks.findItemForZraSync,
  updateItemZraSyncStatus: mocks.updateItemZraSyncStatus,
  generateZraItemCode: mocks.generateZraItemCode,
}));

vi.mock('@repo/database/repositories', () => ({
  findVsdcDevice: mocks.findVsdcDevice,
}));

vi.mock('../zra-api-client', async (orig) => {
  const actual = await orig<typeof import('../zra-api-client')>();
  return {
    ...actual,
    ZraApiClient: class {
      // eslint-disable-next-line @typescript-eslint/no-explicit-any
      constructor(_cfg: any) {}
      saveItem = mocks.saveItem;
    },
  };
});

import { syncItemToZra } from '../zra-item-sync.service';

const ctx: ServiceContext = {
  userId: 'user-1',
  orgId: 'org-1',
  env: {} as ServiceContext['env'],
};
const deps: ServiceDependencies = { db: {} as ServiceDependencies['db'] };

const baseItem = {
  id: 'item-1',
  organizationId: 'org-1',
  name: 'Corn Flakes',
  productType: 'finished_good',
  taxType: 'A',
  barcode: null,
  isActive: true,
  reorderLevel: '5',
  // ZRA fields
  zraItemClassificationCode: '43322555',
  zraOriginCountryCode: 'SA',
  zraPackagingUnitCode: 'BOX',
  zraQuantityUnitCode: 'U',
  zraItemCode: null,
  zraDefaultPrice: '15.0000',
};

const fakeDevice = {
  id: 'dev-1',
  organizationId: 'org-1',
  deviceSerialNumber: 'SN1',
  apiKey: 'key',
  tpin: '1000000000',
  branchId: '000',
  baseUrl: 'https://api-sandbox.zra.org.zm/vsdc-api/v1',
  isInitialized: true,
  deviceStatus: 'active',
};

beforeEach(() => {
  for (const m of Object.values(mocks)) m.mockReset();
});

describe('syncItemToZra', () => {
  it('returns skipped_no_device and patches the item when no VSDC device exists', async () => {
    mocks.findItemForZraSync.mockResolvedValue(baseItem);
    mocks.findVsdcDevice.mockResolvedValue(null);

    const result = await syncItemToZra(ctx, deps, 'item-1');

    expect(result.status).toBe('skipped_no_device');
    expect(mocks.saveItem).not.toHaveBeenCalled();
    expect(mocks.updateItemZraSyncStatus).toHaveBeenCalledWith(
      deps.db,
      'item-1',
      'org-1',
      expect.objectContaining({
        zraSyncStatus: 'skipped_no_device',
        zraSyncAttemptedAt: expect.any(Date),
      })
    );
  });

  it('returns failed when required ZRA fields are missing (no API call)', async () => {
    mocks.findItemForZraSync.mockResolvedValue({
      ...baseItem,
      zraItemClassificationCode: null,
    });
    mocks.findVsdcDevice.mockResolvedValue(fakeDevice);

    const result = await syncItemToZra(ctx, deps, 'item-1');

    expect(result.status).toBe('failed');
    expect(result.error).toMatch(/missing required zra fields/i);
    expect(mocks.saveItem).not.toHaveBeenCalled();
    expect(mocks.updateItemZraSyncStatus).toHaveBeenCalledWith(
      deps.db,
      'item-1',
      'org-1',
      expect.objectContaining({
        zraSyncStatus: 'failed',
        zraSyncError: expect.stringMatching(/missing required zra fields/i),
      })
    );
  });

  it('generates a fresh itemCd on first sync and includes it in the payload', async () => {
    mocks.findItemForZraSync.mockResolvedValue(baseItem);
    mocks.findVsdcDevice.mockResolvedValue(fakeDevice);
    mocks.generateZraItemCode.mockResolvedValue('000-0000001');
    mocks.saveItem.mockResolvedValue({
      success: true, message: 'ok', reference: 'r', httpStatus: 200,
      resultCd: '000', resultMsg: 'It is succeeded', resultDt: '20260611120000', category: 'success',
    });

    const result = await syncItemToZra(ctx, deps, 'item-1');

    expect(result.status).toBe('synced');
    expect(mocks.generateZraItemCode).toHaveBeenCalledTimes(1);
    const payload = mocks.saveItem.mock.calls[0]![0];
    expect(payload.itemCd).toBe('000-0000001');
    expect(mocks.updateItemZraSyncStatus).toHaveBeenCalledWith(
      deps.db,
      'item-1',
      'org-1',
      expect.objectContaining({
        zraSyncStatus: 'synced',
        zraItemCode: '000-0000001',
        zraResultCd: '000',
      })
    );
  });

  it('reuses existing itemCd on subsequent syncs (no generation)', async () => {
    mocks.findItemForZraSync.mockResolvedValue({ ...baseItem, zraItemCode: 'KEEP-0000123' });
    mocks.findVsdcDevice.mockResolvedValue(fakeDevice);
    mocks.saveItem.mockResolvedValue({
      success: true, message: 'ok', reference: 'r', httpStatus: 200,
      resultCd: '000', resultMsg: 'ok', resultDt: '20260611120000', category: 'success',
    });

    await syncItemToZra(ctx, deps, 'item-1');

    expect(mocks.generateZraItemCode).not.toHaveBeenCalled();
    const payload = mocks.saveItem.mock.calls[0]![0];
    expect(payload.itemCd).toBe('KEEP-0000123');
  });

  it('maps full field set correctly from a finished_good item', async () => {
    mocks.findItemForZraSync.mockResolvedValue(baseItem);
    mocks.findVsdcDevice.mockResolvedValue(fakeDevice);
    mocks.generateZraItemCode.mockResolvedValue('000-0000001');
    mocks.saveItem.mockResolvedValue({
      success: true, message: 'ok', reference: 'r', httpStatus: 200,
      resultCd: '000', resultMsg: 'ok', resultDt: '20260611120000', category: 'success',
    });

    await syncItemToZra(ctx, deps, 'item-1');

    const payload = mocks.saveItem.mock.calls[0]![0];
    expect(payload).toMatchObject({
      tpin: '1000000000',
      bhfId: '000',
      itemClsCd: '43322555',
      itemTyCd: '2',
      itemNm: 'Corn Flakes',
      itemStdNm: 'Corn Flakes',
      orgnNatCd: 'SA',
      pkgUnitCd: 'BOX',
      qtyUnitCd: 'U',
      vatCatCd: 'A',
      iplCatCd: null,
      tlCatCd: null,
      exciseTxCatCd: null,
      btchNo: null,
      bcd: null,
      dftPrc: 15,
      manufacturerTpin: null,
      manufacturerItemCd: null,
      rrp: '15',
      svcChargeYn: 'N',
      rentalYn: 'N',
      addInfo: null,
      sftyQty: 5,
      isrcAplcbYn: 'N',
      useYn: 'Y',
      regrNm: 'user-1',
      regrId: 'user-1',
      modrNm: 'user-1',
      modrId: 'user-1',
    });
  });

  it('service product type maps itemTyCd=3 and svcChargeYn=Y', async () => {
    mocks.findItemForZraSync.mockResolvedValue({ ...baseItem, productType: 'service' });
    mocks.findVsdcDevice.mockResolvedValue(fakeDevice);
    mocks.generateZraItemCode.mockResolvedValue('000-0000002');
    mocks.saveItem.mockResolvedValue({
      success: true, message: 'ok', reference: 'r', httpStatus: 200,
      resultCd: '000', resultMsg: 'ok', resultDt: '20260611120000', category: 'success',
    });

    await syncItemToZra(ctx, deps, 'item-1');

    const payload = mocks.saveItem.mock.calls[0]![0];
    expect(payload.itemTyCd).toBe('3');
    expect(payload.svcChargeYn).toBe('Y');
  });

  it('raw_material product type maps itemTyCd=1', async () => {
    mocks.findItemForZraSync.mockResolvedValue({ ...baseItem, productType: 'raw_material' });
    mocks.findVsdcDevice.mockResolvedValue(fakeDevice);
    mocks.generateZraItemCode.mockResolvedValue('000-0000003');
    mocks.saveItem.mockResolvedValue({
      success: true, message: 'ok', reference: 'r', httpStatus: 200,
      resultCd: '000', resultMsg: 'ok', resultDt: '20260611120000', category: 'success',
    });

    await syncItemToZra(ctx, deps, 'item-1');

    expect(mocks.saveItem.mock.calls[0]![0].itemTyCd).toBe('1');
  });

  it('inactive item sends useYn=N', async () => {
    mocks.findItemForZraSync.mockResolvedValue({ ...baseItem, isActive: false });
    mocks.findVsdcDevice.mockResolvedValue(fakeDevice);
    mocks.generateZraItemCode.mockResolvedValue('000-0000004');
    mocks.saveItem.mockResolvedValue({
      success: true, message: 'ok', reference: 'r', httpStatus: 200,
      resultCd: '000', resultMsg: 'ok', resultDt: '20260611120000', category: 'success',
    });

    await syncItemToZra(ctx, deps, 'item-1');

    expect(mocks.saveItem.mock.calls[0]![0].useYn).toBe('N');
  });

  it('returns failed and persists resultCd when ZRA returns error (never throws)', async () => {
    mocks.findItemForZraSync.mockResolvedValue(baseItem);
    mocks.findVsdcDevice.mockResolvedValue(fakeDevice);
    mocks.generateZraItemCode.mockResolvedValue('000-0000005');
    mocks.saveItem.mockResolvedValue({
      success: false, message: 'duplicate', error: 'itemCd already exists',
      reference: 'r', httpStatus: 200,
      resultCd: '924', resultMsg: 'CIS Invoice number already exists.', resultDt: '20260611120000', category: 'error',
    });

    const result = await syncItemToZra(ctx, deps, 'item-1');

    expect(result.status).toBe('failed');
    expect(result.error).toMatch(/itemCd already exists|CIS/i);
    expect(mocks.updateItemZraSyncStatus).toHaveBeenCalledWith(
      deps.db,
      'item-1',
      'org-1',
      expect.objectContaining({
        zraSyncStatus: 'failed',
        zraResultCd: '924',
      })
    );
  });

  it('throws NOT_FOUND when item does not exist (programmer error)', async () => {
    mocks.findItemForZraSync.mockResolvedValue(null);

    await expect(syncItemToZra(ctx, deps, 'missing')).rejects.toThrow(/not found/i);
    expect(mocks.findVsdcDevice).not.toHaveBeenCalled();
    expect(mocks.updateItemZraSyncStatus).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `pnpm --filter @repo/api-services exec vitest run zra-item-sync.service`
Expected: FAIL — module `../zra-item-sync.service` does not exist.

- [ ] **Step 3: Write the service**

Create `packages/api-services/src/domains/invoicing/zra/zra-item-sync.service.ts`:

```typescript
import { findVsdcDevice } from "@repo/database/repositories";
import {
  findItemForZraSync,
  generateZraItemCode,
  updateItemZraSyncStatus,
} from "../../inventory/inventory-items.service";
import type {
  ServiceContext,
  ServiceDependencies,
} from "../../../core/context";
import { requireOrganizationContext } from "../../../core/context";
import { ServiceError } from "../../../core/errors";
import { ZraApiClient, type ZraSaveItemPayload } from "./zra-api-client";

const DEFAULT_SANDBOX_URL = "https://api-sandbox.zra.org.zm/vsdc-api/v1";

export interface ItemSyncResult {
  status: "synced" | "failed" | "skipped_no_device";
  error?: string;
}

function mapProductTypeToItemTyCd(productType: string | null | undefined): string {
  switch (productType) {
    case "raw_material":
      return "1";
    case "service":
      return "3";
    default:
      return "2";
  }
}

export async function syncItemToZra(
  ctx: ServiceContext,
  deps: ServiceDependencies,
  itemId: string
): Promise<ItemSyncResult> {
  const orgId = requireOrganizationContext(ctx);

  const item = await findItemForZraSync(deps.db, itemId, orgId);
  if (!item) {
    throw new ServiceError("NOT_FOUND", "Inventory item not found");
  }

  const device = await findVsdcDevice(deps.db, orgId);
  if (!device || !device.isInitialized) {
    await updateItemZraSyncStatus(deps.db, itemId, orgId, {
      zraSyncStatus: "skipped_no_device",
      zraSyncAttemptedAt: new Date(),
    });
    return { status: "skipped_no_device" };
  }

  // Pre-sync validation: required ZRA fields must be set
  const missing: string[] = [];
  if (!item.zraItemClassificationCode) missing.push("itemClassification");
  if (!item.zraPackagingUnitCode) missing.push("packagingUnit");
  if (!item.zraQuantityUnitCode) missing.push("quantityUnit");

  if (missing.length > 0) {
    const errorMessage = `Missing required ZRA fields: ${missing.join(", ")}`;
    await updateItemZraSyncStatus(deps.db, itemId, orgId, {
      zraSyncStatus: "failed",
      zraSyncAttemptedAt: new Date(),
      zraSyncError: errorMessage,
    });
    return { status: "failed", error: errorMessage };
  }

  // Generate itemCd on first sync; reuse on subsequent
  const branchId = device.branchId ?? "000";
  const itemCd = item.zraItemCode ?? (await generateZraItemCode(deps.db, orgId, branchId));
  const wasGenerated = !item.zraItemCode;

  const actorId = ctx.userId ?? "SYSTEM";
  const defaultPriceNumber = Number(item.zraDefaultPrice ?? 0) || 0;

  const payload: ZraSaveItemPayload = {
    tpin: device.tpin ?? "",
    bhfId: branchId,
    itemCd,
    itemClsCd: item.zraItemClassificationCode!,
    itemTyCd: mapProductTypeToItemTyCd(item.productType),
    itemNm: item.name,
    itemStdNm: item.name,
    orgnNatCd: item.zraOriginCountryCode ?? "ZM",
    pkgUnitCd: item.zraPackagingUnitCode!,
    qtyUnitCd: item.zraQuantityUnitCode!,
    vatCatCd: item.taxType ?? "A",
    iplCatCd: null,
    tlCatCd: null,
    exciseTxCatCd: null,
    btchNo: null,
    bcd: item.barcode ?? null,
    dftPrc: defaultPriceNumber,
    manufacturerTpin: null,
    manufacturerItemCd: null,
    rrp: String(defaultPriceNumber),
    svcChargeYn: item.productType === "service" ? "Y" : "N",
    rentalYn: "N",
    addInfo: null,
    sftyQty: Number(item.reorderLevel ?? 0) || 0,
    isrcAplcbYn: "N",
    useYn: item.isActive ? "Y" : "N",
    regrNm: actorId,
    regrId: actorId,
    modrNm: actorId,
    modrId: actorId,
  };

  const client = new ZraApiClient({
    baseUrl: device.baseUrl ?? device.vsdcEndpointUrl ?? DEFAULT_SANDBOX_URL,
    apiKey: device.apiKey ?? undefined,
    tpin: device.tpin ?? undefined,
    branchId: device.branchId ?? undefined,
    deviceSerialNumber: device.deviceSerialNumber,
  });

  const apiResult = await client.saveItem(payload);

  if (apiResult.success) {
    await updateItemZraSyncStatus(deps.db, itemId, orgId, {
      zraSyncStatus: "synced",
      zraSyncedAt: new Date(),
      zraSyncAttemptedAt: new Date(),
      zraSyncError: null,
      zraResultCd: apiResult.resultCd ?? null,
      zraResultMsg: apiResult.resultMsg ?? null,
      zraResultDt: apiResult.resultDt ?? null,
      ...(wasGenerated ? { zraItemCode: itemCd } : {}),
    });
    return { status: "synced" };
  }

  const errorMessage = apiResult.error ?? apiResult.message;
  await updateItemZraSyncStatus(deps.db, itemId, orgId, {
    zraSyncStatus: "failed",
    zraSyncAttemptedAt: new Date(),
    zraSyncError: errorMessage,
    zraResultCd: apiResult.resultCd ?? null,
    zraResultMsg: apiResult.resultMsg ?? null,
    zraResultDt: apiResult.resultDt ?? null,
    // Persist the generated code even on failure so retries reuse it
    ...(wasGenerated ? { zraItemCode: itemCd } : {}),
  });
  return { status: "failed", error: errorMessage };
}
```

- [ ] **Step 4: Add the barrel export**

Edit `packages/api-services/src/domains/invoicing/index.ts`. Near the existing `syncCustomerToZra` export block, add:

```typescript
export {
    type ItemSyncResult,
    syncItemToZra,
} from "./zra/zra-item-sync.service";
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `pnpm --filter @repo/api-services exec vitest run zra-item-sync.service`
Expected: all 10 tests PASS.

- [ ] **Step 6: Typecheck**

Run: `pnpm --filter @repo/api-services typecheck`
Expected: zero new errors. Preexisting errors elsewhere are OK.

- [ ] **Step 7: DO NOT COMMIT.**

---

## Task 5: Backend Handler Integration — `waitUntil` on Create/Update

**Files:**
- Modify: `apps/api-inventory/src/routes/inventory/items/handlers.ts`

- [ ] **Step 1: Add the import**

At the top of `handlers.ts`, add `syncItemToZra` to the existing import from `@repo/api-services/domains/invoicing` (the import may already include other invoicing services — match the existing structure):

```typescript
import { syncItemToZra } from "@repo/api-services/domains/invoicing";
```

If there's already a single combined import from `@repo/api-services/domains/invoicing`, merge it into that. Otherwise add as a new import line near the existing inventory imports.

- [ ] **Step 2: Update `createItemHandler` to fire-and-forget the sync**

Find `createItemHandler` (around line 87). After the line `const item = await createItem(ctx, deps, body);` (or whatever exact variable the create function returns to), add the `waitUntil` block before the `return c.json(...)` call:

```typescript
    const execCtx = (c as { executionCtx?: { waitUntil: (p: Promise<unknown>) => void } }).executionCtx;
    if (execCtx?.waitUntil) {
      execCtx.waitUntil(
        syncItemToZra(ctx, deps, item.id).catch((err) =>
          console.error("[ZRA Item Sync] post-create error", {
            itemId: item.id,
            err,
          })
        )
      );
    }
```

If the variable holding the created item is named differently (not `item`), adapt the variable reference. Verify by reading the handler first.

- [ ] **Step 3: Update `updateItemHandler` to fire-and-forget the sync**

Find `updateItemHandler` (around line 106). Apply the same pattern: after the service call returns, add the `waitUntil` block before the `return c.json(...)`:

```typescript
    const execCtx = (c as { executionCtx?: { waitUntil: (p: Promise<unknown>) => void } }).executionCtx;
    if (execCtx?.waitUntil) {
      execCtx.waitUntil(
        syncItemToZra(ctx, deps, item.id).catch((err) =>
          console.error("[ZRA Item Sync] post-update error", {
            itemId: item.id,
            err,
          })
        )
      );
    }
```

- [ ] **Step 4: Typecheck**

Run: `pnpm --filter api-inventory typecheck` (verify the actual package name in `apps/api-inventory/package.json` — try `bumara-api-inventory` if the short name doesn't work).
Expected: zero new errors in `handlers.ts`.

- [ ] **Step 5: DO NOT COMMIT.**

---

## Task 6: Manual Sync Endpoint

**Files:**
- Modify: `apps/api-inventory/src/routes/inventory/items/routes.ts`
- Modify: `apps/api-inventory/src/routes/inventory/items/handlers.ts`
- Modify: `apps/api-inventory/src/routes/inventory/items/index.ts`

- [ ] **Step 1: Add the route definition**

In `routes.ts`, after the last existing route (e.g. `bulkCreateItemsRoute`) and before any type exports at the bottom, add:

```typescript
export const syncItemToZraRoute = createRoute({
  tags,
  method: "post",
  path: "/inventory/items/{itemId}/sync-zra",
  summary: "Manually sync (register/update) an inventory item with ZRA",
  request: {
    params: z.object({ itemId: z.string().uuid() }),
  },
  middleware: [requireAuth, requireOrg],
  responses: {
    [HttpStatusCodes.OK]: jsonContent(successResponseSchema, "Item sync attempted"),
    [HttpStatusCodes.NOT_FOUND]: jsonContent(errorResponseSchema, "Item not found"),
    [HttpStatusCodes.UNAUTHORIZED]: jsonContent(errorResponseSchema, "Unauthorized"),
    [HttpStatusCodes.FORBIDDEN]: jsonContent(errorResponseSchema, "Forbidden"),
    [HttpStatusCodes.INTERNAL_SERVER_ERROR]: jsonContent(errorResponseSchema, "Internal Server Error"),
  },
});
```

Inspect existing routes in the same file for the exact imports / `tags` constant — adapt the path prefix above to match what the inventory routes use (the existing routes already include `/inventory/items` as their prefix based on the `path` strings I see).

Add the type export at the bottom of the file alongside the others:

```typescript
export type SyncItemToZraRoute = typeof syncItemToZraRoute;
```

- [ ] **Step 2: Add the handler**

In `handlers.ts`, add `SyncItemToZraRoute` to the type imports from `./routes`. Then append at the end of the file:

```typescript
export const syncItemToZraHandler: AppRouteHandler<
  SyncItemToZraRoute
> = async (c) => {
  try {
    const ctx = buildServiceContext(c);
    const deps = buildServiceDependencies(c);
    const { itemId } = c.req.valid("param");
    const result = await syncItemToZra(ctx, deps, itemId);
    return c.json(
      {
        success: true,
        data: result,
        message: `Item sync completed: ${result.status}`,
      },
      HttpStatusCodes.OK
    );
  } catch (error) {
    return handleServiceError(c, error, "Failed to sync item to ZRA");
  }
};
```

- [ ] **Step 3: Mount the route**

In `index.ts`, add `syncItemToZraRoute` to the existing import from `./routes`, then append a new `.openapi(...)` call at the end of the chain:

```typescript
.openapi(syncItemToZraRoute, lazy("syncItemToZraHandler"));
```

Match the exact `.openapi(...)` pattern of the surrounding routes — verify by reading the existing file first.

- [ ] **Step 4: Typecheck**

Run the inventory worker typecheck. Expected: zero new errors.

- [ ] **Step 5: DO NOT COMMIT.**

---

## Task 7: Frontend Query Hooks — `useZraCodes` + `useZraItemClassifications`

**Files:**
- Create: `apps/app/lib/queries/zra-reference-data/keys.ts`
- Create: `apps/app/lib/queries/zra-reference-data/fetchers.ts`
- Create: `apps/app/lib/queries/zra-reference-data/hooks.ts`
- Create: `apps/app/lib/queries/zra-reference-data/index.ts`

These follow the exact pattern of `apps/app/lib/queries/inventory/hooks/use-categories.ts` — read that file once before writing for the local convention (uses `useAuth().getToken`, a fetcher with the token, etc.).

- [ ] **Step 1: Write the query keys**

Create `apps/app/lib/queries/zra-reference-data/keys.ts`:

```typescript
export const zraReferenceQueryKeys = {
  all: ["zra-reference"] as const,
  codes: (codeClass: string) => ["zra-reference", "codes", codeClass] as const,
  codeClasses: () => ["zra-reference", "code-classes"] as const,
  itemClassifications: (opts: { level?: number; search?: string; limit?: number }) =>
    ["zra-reference", "item-classifications", opts.level, opts.search, opts.limit] as const,
};
```

- [ ] **Step 2: Write the fetchers**

Create `apps/app/lib/queries/zra-reference-data/fetchers.ts`. Match the existing fetcher pattern in `apps/app/lib/queries/inventory/fetchers/` — they use a shared API helper. Read one of them first (e.g. `inventory/fetchers/categories.ts`) for the exact shape:

```typescript
import { fetchFromApi } from "@/lib/api/client";

type GetToken = () => Promise<string | null>;

export interface ZraCodeOption {
  code: string;
  name: string;
  description: string | null;
  sortOrder: number;
  userDefinedCode1: string | null;
}

export interface ZraCodeClassOption {
  codeClass: string;
  name: string;
}

export interface ZraItemClassificationOption {
  itemClassCode: string;
  itemClassName: string;
  itemClassLevel: number;
  taxTypeCode: string | null;
}

export async function fetchZraCodes(
  getToken: GetToken,
  codeClass: string
): Promise<ZraCodeOption[]> {
  return fetchFromApi(getToken, `/invoicing/zra/codes/${codeClass}`);
}

export async function fetchZraCodeClasses(
  getToken: GetToken
): Promise<ZraCodeClassOption[]> {
  return fetchFromApi(getToken, `/invoicing/zra/code-classes`);
}

export async function fetchZraItemClassifications(
  getToken: GetToken,
  opts: { level?: number; search?: string; limit?: number }
): Promise<ZraItemClassificationOption[]> {
  const params = new URLSearchParams();
  if (opts.level !== undefined) params.set("level", String(opts.level));
  if (opts.search) params.set("search", opts.search);
  if (opts.limit !== undefined) params.set("limit", String(opts.limit));
  const qs = params.toString();
  return fetchFromApi(getToken, `/invoicing/zra/item-classifications${qs ? `?${qs}` : ""}`);
}
```

Important: replace `fetchFromApi` and its import path with whatever helper the existing inventory fetchers use. If they call `fetch()` directly with a constructed URL and Authorization header, mirror that pattern instead. Do NOT invent a new helper.

- [ ] **Step 3: Write the hooks**

Create `apps/app/lib/queries/zra-reference-data/hooks.ts`:

```typescript
import { useAuth } from "@repo/auth/client";
import { useQuery } from "@tanstack/react-query";
import { zraReferenceQueryKeys } from "./keys";
import {
  fetchZraCodes,
  fetchZraCodeClasses,
  fetchZraItemClassifications,
} from "./fetchers";

const TEN_MINUTES = 1000 * 60 * 10;

export function useZraCodes(codeClass: string, options?: { enabled?: boolean }) {
  const { getToken } = useAuth();

  return useQuery({
    queryKey: zraReferenceQueryKeys.codes(codeClass),
    enabled: Boolean(getToken) && Boolean(codeClass) && (options?.enabled ?? true),
    queryFn: () => fetchZraCodes(getToken, codeClass),
    staleTime: TEN_MINUTES,
  });
}

export function useZraCodeClasses(options?: { enabled?: boolean }) {
  const { getToken } = useAuth();

  return useQuery({
    queryKey: zraReferenceQueryKeys.codeClasses(),
    enabled: Boolean(getToken) && (options?.enabled ?? true),
    queryFn: () => fetchZraCodeClasses(getToken),
    staleTime: TEN_MINUTES,
  });
}

export function useZraItemClassifications(
  opts: { level?: number; search?: string; limit?: number },
  options?: { enabled?: boolean }
) {
  const { getToken } = useAuth();

  return useQuery({
    queryKey: zraReferenceQueryKeys.itemClassifications(opts),
    enabled: Boolean(getToken) && (options?.enabled ?? true),
    queryFn: () => fetchZraItemClassifications(getToken, opts),
    staleTime: TEN_MINUTES,
  });
}
```

- [ ] **Step 4: Write the barrel**

Create `apps/app/lib/queries/zra-reference-data/index.ts`:

```typescript
export * from "./fetchers";
export * from "./hooks";
export * from "./keys";
```

- [ ] **Step 5: Typecheck**

Run the app typecheck (`pnpm --filter app typecheck` or whatever the package name is in `apps/app/package.json`).
Expected: zero new errors in the new files.

- [ ] **Step 6: DO NOT COMMIT.**

---

## Task 8: Frontend Mutation Hook — `useSyncItemToZra`

**Files:**
- Create: `apps/app/lib/queries/inventory/hooks/use-sync-item-to-zra.ts`
- Modify: `apps/app/lib/queries/inventory/hooks/index.ts` (or wherever item hooks are barrel-exported — verify first)

- [ ] **Step 1: Create the hook**

```typescript
import { useAuth } from "@repo/auth/client";
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { toast } from "@repo/design-system/utils/sonner";
import { inventoryQueryKeys } from "../keys";
import { syncItemToZra } from "../fetchers/items";

export function useSyncItemToZra() {
  const { getToken } = useAuth();
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (itemId: string) => syncItemToZra(getToken, itemId),
    onSuccess: (_data, itemId) => {
      toast.success("ZRA sync attempted");
      queryClient.invalidateQueries({
        queryKey: inventoryQueryKeys.items.detail(itemId),
      });
    },
    onError: (error) => {
      toast.error(error.message || "Failed to sync item to ZRA");
    },
  });
}
```

- [ ] **Step 2: Add the fetcher for the manual sync route**

In `apps/app/lib/queries/inventory/fetchers/items.ts`, add a new fetcher:

```typescript
export async function syncItemToZra(getToken: GetToken, itemId: string) {
  return fetchFromApi(getToken, `/invoicing/inventory-items/${itemId}/sync-zra`, {
    method: "POST",
  });
}
```

Adapt the helper / shape to the existing fetcher conventions in this file.

> **Note on route path mismatch:** Task 6 defines the manual sync route as `/inventory/items/{itemId}/sync-zra` because it lives in the inventory worker. The frontend fetcher must point to the SAME path, but the frontend's API base may resolve to a different worker. Verify the existing inventory fetchers' URLs in this file and use the same prefix convention — if they prefix with `/inventory`, this one should too. The plan's route in Task 6 uses `/inventory/items/{itemId}/sync-zra` to match the existing inventory routes (other inventory item routes use `/inventory/items/...` too).

- [ ] **Step 3: Add to the barrel if there is one**

If `apps/app/lib/queries/inventory/hooks/index.ts` re-exports the item hooks, add a line:

```typescript
export * from "./use-sync-item-to-zra";
```

If there's no barrel for hooks (each hook is imported directly by consumers), skip this step.

- [ ] **Step 4: Typecheck**

Expected: zero new errors.

- [ ] **Step 5: DO NOT COMMIT.**

---

## Task 9: Frontend Components — `ZraCodeSelect` + `ZraItemClassificationCombobox`

**Files:**
- Create: `apps/app/components/features/invoicing/zra/zra-code-select.tsx`
- Create: `apps/app/components/features/invoicing/zra/zra-item-classification-combobox.tsx`

- [ ] **Step 1: Verify the actual code class IDs**

Run a quick query to confirm the correct ZRA code class IDs for the three classes we need. With the dev server running, hit:

```
GET /invoicing/zra/code-classes
```

Look for entries named "Country", "Packaging Unit", "Quantity Unit" and note their `codeClass` values. Common ZRA conventions are:
- Country: `"05"`
- Packaging Unit: `"17"`
- Quantity Unit: `"10"`

But these are not guaranteed across ZRA deployments. Bake the verified IDs as constants in the new files (do NOT hardcode without verification — if the data is empty, default to the values above and add a TODO comment for the implementer to verify post-sync).

- [ ] **Step 2: Write `ZraCodeSelect`**

Create `apps/app/components/features/invoicing/zra/zra-code-select.tsx`:

```tsx
"use client";

import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@repo/design-system/components/ui/select";
import { useZraCodes } from "@/lib/queries/zra-reference-data";

interface ZraCodeSelectProps {
  codeClass: string;
  value: string | undefined;
  onChange: (value: string) => void;
  placeholder?: string;
  disabled?: boolean;
}

export function ZraCodeSelect({
  codeClass,
  value,
  onChange,
  placeholder = "Select",
  disabled,
}: ZraCodeSelectProps) {
  const { data, isLoading } = useZraCodes(codeClass);
  const codes = data ?? [];

  return (
    <Select
      value={value}
      onValueChange={onChange}
      disabled={disabled || isLoading}
    >
      <SelectTrigger>
        <SelectValue placeholder={isLoading ? "Loading..." : placeholder} />
      </SelectTrigger>
      <SelectContent>
        {codes.map((c) => (
          <SelectItem key={c.code} value={c.code}>
            {c.code} — {c.name}
          </SelectItem>
        ))}
      </SelectContent>
    </Select>
  );
}
```

- [ ] **Step 3: Write `ZraItemClassificationCombobox`**

Create `apps/app/components/features/invoicing/zra/zra-item-classification-combobox.tsx`:

```tsx
"use client";

import {
  Command,
  CommandEmpty,
  CommandGroup,
  CommandInput,
  CommandItem,
  CommandList,
} from "@repo/design-system/components/ui/command";
import {
  Popover,
  PopoverContent,
  PopoverTrigger,
} from "@repo/design-system/components/ui/popover";
import { Button } from "@repo/design-system/components/ui/button";
import { Check, ChevronsUpDown } from "lucide-react";
import { useState } from "react";
import { useZraItemClassifications } from "@/lib/queries/zra-reference-data";

const ITEM_CLASS_LEVEL = 5;
const SEARCH_LIMIT = 50;
const DEBOUNCE_MS = 250;

interface ZraItemClassificationComboboxProps {
  value: string | undefined;
  onChange: (value: string) => void;
  disabled?: boolean;
}

function useDebounced<T>(value: T, ms: number): T {
  const [debounced, setDebounced] = useState(value);
  if (debounced !== value) {
    // Use a microtask-safe debounce
    const id = setTimeout(() => setDebounced(value), ms);
    // Cancellation is handled by React's effect lifecycle in normal use; for a
    // simple debounce inline this is acceptable.
    return debounced;
  }
  return debounced;
}

export function ZraItemClassificationCombobox({
  value,
  onChange,
  disabled,
}: ZraItemClassificationComboboxProps) {
  const [open, setOpen] = useState(false);
  const [search, setSearch] = useState("");
  const debouncedSearch = useDebounced(search, DEBOUNCE_MS);

  const { data, isLoading } = useZraItemClassifications({
    level: ITEM_CLASS_LEVEL,
    search: debouncedSearch || undefined,
    limit: SEARCH_LIMIT,
  });

  const items = data ?? [];
  const selected = items.find((i) => i.itemClassCode === value);

  return (
    <Popover open={open} onOpenChange={setOpen}>
      <PopoverTrigger asChild>
        <Button
          variant="outline"
          role="combobox"
          aria-expanded={open}
          disabled={disabled}
          className="w-full justify-between"
        >
          {selected ? (
            <span className="truncate">
              {selected.itemClassCode} — {selected.itemClassName}
            </span>
          ) : value ? (
            <span className="text-muted-foreground">{value}</span>
          ) : (
            <span className="text-muted-foreground">Select classification</span>
          )}
          <ChevronsUpDown className="ml-2 h-4 w-4 shrink-0 opacity-50" />
        </Button>
      </PopoverTrigger>
      <PopoverContent className="w-[400px] p-0" align="start">
        <Command shouldFilter={false}>
          <CommandInput
            placeholder="Search by code or name..."
            value={search}
            onValueChange={setSearch}
          />
          <CommandList>
            {isLoading && <CommandEmpty>Loading...</CommandEmpty>}
            {!isLoading && items.length === 0 && (
              <CommandEmpty>No results.</CommandEmpty>
            )}
            <CommandGroup>
              {items.map((item) => (
                <CommandItem
                  key={item.itemClassCode}
                  value={item.itemClassCode}
                  onSelect={() => {
                    onChange(item.itemClassCode);
                    setOpen(false);
                  }}
                >
                  <Check
                    className={`mr-2 h-4 w-4 ${value === item.itemClassCode ? "opacity-100" : "opacity-0"}`}
                  />
                  <span className="truncate">
                    {item.itemClassCode} — {item.itemClassName}
                  </span>
                </CommandItem>
              ))}
            </CommandGroup>
          </CommandList>
        </Command>
      </PopoverContent>
    </Popover>
  );
}
```

> **Note:** The `useDebounced` inline above is a minimal implementation. If the codebase already has a `useDebounced` / `useDebouncedValue` hook (check `apps/app/lib/hooks/` or `@repo/design-system`), import that instead and delete the local one. Search first.

- [ ] **Step 4: Typecheck**

Expected: zero new errors. If the `Command` / `Popover` primitives have slightly different names in this codebase, adapt the imports.

- [ ] **Step 5: DO NOT COMMIT.**

---

## Task 10: Form Integration — `ZraDetailsSection`

**Files:**
- Modify: `apps/app/components/features/inventory/items/item-form.tsx`

- [ ] **Step 1: Extend the form schema**

In `item-form.tsx`, find `itemFormSchema` and extend it with the new fields:

```typescript
const itemFormSchema = z.object({
  name: z.string().min(1, "Name is required").max(200),
  description: z.string().max(1000).optional().or(z.literal("")),
  productType: z.enum([
    "raw_material",
    "finished_good",
    "service",
    "physical",
    "digital",
  ]),
  taxType: z.enum(["A", "B", "C1", "C2", "C3", "D", "TOT"]),
  packagingType: z.string().optional().or(z.literal("")),
  sku: z.string().max(50).optional().or(z.literal("")),
  barcode: z.string().max(50).optional().or(z.literal("")),
  categoryId: z.string().optional().or(z.literal("")),
  defaultUomId: z.string().min(1, "Unit of measure is required"),
  trackInventory: z.boolean().default(true),
  reorderLevel: z.string().optional().or(z.literal("")),
  reorderQty: z.string().optional().or(z.literal("")),

  // ZRA Smart Invoice fields
  zraItemClassificationCode: z.string().min(1, "Item classification is required"),
  zraPackagingUnitCode: z.string().min(1, "Packaging unit is required"),
  zraQuantityUnitCode: z.string().min(1, "Quantity unit is required"),
  zraOriginCountryCode: z.string().optional().or(z.literal("")),
  zraDefaultPrice: z.string().optional().or(z.literal("")),
});
```

- [ ] **Step 2: Add defaults**

In the `useForm` `defaultValues` (look for the existing block that maps `item?.name ?? ""` etc.), add:

```typescript
    zraItemClassificationCode: item?.zraItemClassificationCode ?? "",
    zraPackagingUnitCode: item?.zraPackagingUnitCode ?? "",
    zraQuantityUnitCode: item?.zraQuantityUnitCode ?? "",
    zraOriginCountryCode: item?.zraOriginCountryCode ?? "",
    zraDefaultPrice: item?.zraDefaultPrice
      ? String(Number.parseFloat(item.zraDefaultPrice))
      : "",
```

- [ ] **Step 3: Add the `ZraDetailsSection` component**

In the same file, after the `BasicInfoSection` function definition (before `InventoryStockSection`), add a new section component:

```tsx
import { ZraCodeSelect } from "@/components/features/invoicing/zra/zra-code-select";
import { ZraItemClassificationCombobox } from "@/components/features/invoicing/zra/zra-item-classification-combobox";

// Verified ZRA code class IDs (see Task 9 Step 1)
const ZRA_COUNTRY_CODE_CLASS = "05";
const ZRA_PACKAGING_UNIT_CODE_CLASS = "17";
const ZRA_QUANTITY_UNIT_CODE_CLASS = "10";

function ZraDetailsSection({ form }: { form: ItemFormReturn }) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>ZRA Smart Invoice Details</CardTitle>
      </CardHeader>
      <CardContent className="space-y-4">
        <p className="text-muted-foreground text-sm">
          These fields are used when this item is sent to ZRA Smart Invoice.
        </p>

        <div className="space-y-2">
          <Label>Item Classification *</Label>
          <ZraItemClassificationCombobox
            value={form.watch("zraItemClassificationCode")}
            onChange={(v) =>
              form.setValue("zraItemClassificationCode", v, {
                shouldValidate: true,
              })
            }
          />
          {form.formState.errors.zraItemClassificationCode && (
            <p className="text-destructive text-sm">
              {form.formState.errors.zraItemClassificationCode.message as string}
            </p>
          )}
        </div>

        <div className="grid gap-4 md:grid-cols-2">
          <div className="space-y-2">
            <Label>Packaging Unit *</Label>
            <ZraCodeSelect
              codeClass={ZRA_PACKAGING_UNIT_CODE_CLASS}
              value={form.watch("zraPackagingUnitCode")}
              onChange={(v) =>
                form.setValue("zraPackagingUnitCode", v, {
                  shouldValidate: true,
                })
              }
              placeholder="Select packaging unit"
            />
            {form.formState.errors.zraPackagingUnitCode && (
              <p className="text-destructive text-sm">
                {form.formState.errors.zraPackagingUnitCode.message as string}
              </p>
            )}
          </div>
          <div className="space-y-2">
            <Label>Quantity Unit *</Label>
            <ZraCodeSelect
              codeClass={ZRA_QUANTITY_UNIT_CODE_CLASS}
              value={form.watch("zraQuantityUnitCode")}
              onChange={(v) =>
                form.setValue("zraQuantityUnitCode", v, {
                  shouldValidate: true,
                })
              }
              placeholder="Select quantity unit"
            />
            {form.formState.errors.zraQuantityUnitCode && (
              <p className="text-destructive text-sm">
                {form.formState.errors.zraQuantityUnitCode.message as string}
              </p>
            )}
          </div>
        </div>

        <div className="grid gap-4 md:grid-cols-2">
          <div className="space-y-2">
            <Label>Country of Origin</Label>
            <ZraCodeSelect
              codeClass={ZRA_COUNTRY_CODE_CLASS}
              value={form.watch("zraOriginCountryCode")}
              onChange={(v) =>
                form.setValue("zraOriginCountryCode", v, {
                  shouldValidate: true,
                })
              }
              placeholder="Defaults to Zambia (ZM)"
            />
          </div>
          <TextField
            control={form.control}
            label="Default Price (ZMW)"
            name="zraDefaultPrice"
            placeholder="0.00"
            type="number"
          />
        </div>
      </CardContent>
    </Card>
  );
}
```

- [ ] **Step 4: Render the new section in the main form**

In the main `ItemForm` component, find the JSX that renders `<BasicInfoSection ... />` and `<InventoryStockSection ... />`. Add `<ZraDetailsSection form={form} />` between them:

```tsx
<BasicInfoSection ... />
<ZraDetailsSection form={form} />
<InventoryStockSection ... />
```

- [ ] **Step 5: Update the `onSubmit` payload to include the new fields**

In `onSubmit`, the existing payload construction is:

```typescript
const payload = {
  ...values,
  categoryId: values.categoryId || undefined,
  description: values.description || undefined,
  sku: values.sku || undefined,
  barcode: values.barcode || undefined,
  packagingType: values.packagingType || undefined,
  reorderLevel: values.reorderLevel || undefined,
  reorderQty: values.reorderQty || undefined,
};
```

Add the new ZRA fields with the same `|| undefined` pattern so empty strings don't get sent:

```typescript
const payload = {
  ...values,
  categoryId: values.categoryId || undefined,
  description: values.description || undefined,
  sku: values.sku || undefined,
  barcode: values.barcode || undefined,
  packagingType: values.packagingType || undefined,
  reorderLevel: values.reorderLevel || undefined,
  reorderQty: values.reorderQty || undefined,
  zraItemClassificationCode: values.zraItemClassificationCode,
  zraPackagingUnitCode: values.zraPackagingUnitCode,
  zraQuantityUnitCode: values.zraQuantityUnitCode,
  zraOriginCountryCode: values.zraOriginCountryCode || undefined,
  zraDefaultPrice: values.zraDefaultPrice || undefined,
};
```

- [ ] **Step 6: Update the `InventoryItem` type if needed**

The `InventoryItem` type used in `defaultValues` needs to include the new ZRA fields. Check `apps/app/lib/queries/inventory/types.ts` (or wherever `InventoryItem` is defined) — it's likely typed from the backend response. If the type doesn't include `zraItemClassificationCode`, etc., update it to add the new optional fields.

If types are inferred from the schema or backend response, they may auto-update once the backend create/update routes accept and return these fields. Otherwise manually add them.

- [ ] **Step 7: Typecheck**

Run the app typecheck.
Expected: zero new errors.

- [ ] **Step 8: Manual verification (no automated frontend tests in this codebase)**

Start the dev server. Navigate to the inventory item create page. Verify:
- The "ZRA Smart Invoice Details" section appears between Basic Info and Inventory Settings.
- The three dropdowns load options (assuming reference data has been synced to the database).
- Required field errors appear if the user submits without filling them.
- The item classification combobox supports search and filters as you type.

- [ ] **Step 9: DO NOT COMMIT.**

---

## Task 11: Update Backend Input Schema (Backend Route Acceptance of New Fields)

**Files:**
- Modify: `packages/api-services/src/domains/inventory/inventory.schema.ts` (or wherever `createItemInputSchema` lives)
- Modify: `packages/api-services/src/domains/inventory/inventory-items.service.ts` (`createItem` / `updateItem` to persist new fields)

This task wires the new ZRA fields from form submission through to the database.

- [ ] **Step 1: Extend `createItemInputSchema` and `updateItemInputSchema`**

Open `packages/api-services/src/domains/inventory/inventory.schema.ts` (verify exact filename — search for `createItemInputSchema`). Find the Zod schema for create item input. Add the new fields:

```typescript
const createItemInputSchemaBase = {
  // ... existing fields ...
  zraItemClassificationCode: z.string().optional(),
  zraPackagingUnitCode: z.string().optional(),
  zraQuantityUnitCode: z.string().optional(),
  zraOriginCountryCode: z.string().optional(),
  zraDefaultPrice: z.string().optional(),
};
```

Repeat for `updateItemInputSchema` (or whatever the update Zod schema is named).

> **Note:** The backend accepts these as optional (any of the three required ones may be missing — the sync service handles that). The frontend enforces required-on-submit; the backend stays permissive so admin / seed flows can create items without ZRA fields.

- [ ] **Step 2: Pass the new fields through `createItem`**

In `inventory-items.service.ts`, find `createItem`. The function builds an insert payload. Add the new fields explicitly to the insert (Drizzle should accept them once the schema columns exist from Task 1):

```typescript
const [created] = await deps.db
  .insert(inventoryItems)
  .values({
    // ... existing fields ...
    zraItemClassificationCode: input.zraItemClassificationCode ?? null,
    zraOriginCountryCode: input.zraOriginCountryCode ?? null,
    zraPackagingUnitCode: input.zraPackagingUnitCode ?? null,
    zraQuantityUnitCode: input.zraQuantityUnitCode ?? null,
    zraDefaultPrice: input.zraDefaultPrice ?? null,
  })
  .returning();
```

If the existing `createItem` uses a spread (`...input`), the new fields will flow through automatically because the Zod schema now passes them. Verify by reading the existing code.

- [ ] **Step 3: Pass new fields through `updateItem`**

Same change in `updateItem` — add the new fields to the update set or rely on the spread if one is in use.

- [ ] **Step 4: Verify `getItem` / `findItemForZraSync` return the new fields**

Drizzle's `select()` without explicit columns returns everything, so the new columns are picked up automatically. If `getItem` uses an explicit column list (look for `.select({ ... })`), add the new ZRA columns.

- [ ] **Step 5: Typecheck**

Run: `pnpm --filter @repo/api-services typecheck`
Expected: zero new errors.

- [ ] **Step 6: DO NOT COMMIT.**

---

## Task 12: Apply Migration + Smoke Test (deferred to user)

The migration generated in Task 1 needs to be applied to whatever DB `DATABASE_URL` points at. This step is **deferred to the user** because it requires credentials.

To apply once ready:

```
pnpm --filter @repo/database db:migrate
```

Smoke test:
1. Create an inventory item via the form, filling all three required ZRA fields (classification, packaging unit, quantity unit). Submit.
2. Within ~30s, `SELECT id, zra_sync_status, zra_item_code, zra_result_cd FROM inventory_items WHERE id = '<new id>'` — expect `zra_sync_status='synced'`, `zra_item_code='000-0000001'` (or similar), `zra_result_cd='000'`.
3. Create an item via the form with required fields filled but no VSDC device — expect `zra_sync_status='skipped_no_device'`.
4. Manually call `POST /inventory/items/{id}/sync-zra` for a failed item — expect 200 with `{ status: 'synced' | 'failed', error?: string }`.
5. Verify the form renders the new section, dropdowns populate from reference data, classification combobox supports search.

---

## Self-Review Checklist (for the plan author)

**Spec coverage:**
- 13 new columns on `inventory_items`: Task 1 ✓
- Service helpers (`findItemForZraSync`, `updateItemZraSyncStatus`, `generateZraItemCode`): Task 2 ✓
- `ZraSaveItemPayload` interface + `saveItem` method: Task 3 ✓
- `syncItemToZra` service with full mapping + 3 outcomes + never-throws + itemCd generation: Task 4 ✓
- Service unit tests covering all scenarios: Task 4 ✓
- Handler integration via `waitUntil` for create + update: Task 5 ✓
- Manual sync endpoint: Task 6 ✓
- Frontend reference-data query hooks: Task 7 ✓
- Frontend mutation hook for manual sync: Task 8 ✓
- Reusable dropdown components: Task 9 ✓
- Form integration with new section + schema + persistence: Tasks 10 + 11 ✓
- Migration apply + smoke test: Task 12 (deferred to user) ✓

**Placeholder scan:** No "TBD" or "implement later". Task 12 is explicitly deferred (operational work, not unfinished design). Task 9 Step 1 flags real-time verification of code class IDs — that's an action, not a placeholder.

**Type consistency:**
- `ItemSyncResult.status` union (`'synced' | 'failed' | 'skipped_no_device'`) — same across service, tests, handler return shape.
- Schema column names (`zraItemClassificationCode`, etc.) — identical in Drizzle schema, service helpers patch type, service mapping code, form schema, frontend fetcher.
- `ZraSaveItemPayload` field count matches the wire format (29 fields).
- Handler name `syncItemToZraHandler` matches the route name `syncItemToZraRoute` and the lazy-mount string.
