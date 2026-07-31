---
title: "Inventory Module - Testing Strategy"
description: "Test strategy, fixtures, factories, and scenarios for the Inventory Module."
---

## Testing Philosophy

The Inventory Module requires rigorous testing due to:

1. **Financial implications** — Incorrect stock affects business operations
2. **Data integrity** — Ledger must be accurate and auditable
3. **Concurrency** — Multiple users posting simultaneously
4. **Idempotency** — Retries must not create duplicates

Testing priorities:
1. **Ledger correctness** — Balance updates match movements
2. **Business rules** — Negative stock blocking, status transitions
3. **Idempotency** — Duplicate requests handled correctly
4. **Integration** — Documents, Tasks, Notifications work together

---

## Test Structure

### Directory Layout

Presets come from [`packages/testing/`](https://github.com/bumara-dev/bumara/tree/main/packages/testing):
`@repo/testing/node` for domain logic, `@repo/testing/react` for components.

Tests are colocated with the code they exercise: `foo.ts` is tested by `foo.test.ts` in the
same directory. There are no `__tests__/` folders and no central fixtures directory. A slice
that needs shared setup carries its own `test-harness.ts`.

```
packages/inventory/src/
├── items/service.test.ts             # Items CRUD
├── locations/service.test.ts         # Locations CRUD
├── units/service.test.ts             # Units + conversions
├── stock/service.test.ts             # Core ledger logic
├── adjustments/service.test.ts       # Adjustment operations
├── transfers/service.test.ts         # Transfer operations
├── counts/service.test.ts            # Count operations
├── batches/service.test.ts           # Batch tracking
├── categories/service.test.ts        # Categories
├── suppliers/service.test.ts         # Suppliers
├── members/service.test.ts           # Location membership
├── location-assignments/service.test.ts
├── attachments/service.test.ts       # Product images
├── applications/service.test.ts
├── reset/service.test.ts
├── cash-outs/service.test.ts         # Till cash-outs
├── expenses/service.test.ts          # Operating expenses
├── layby-sales/service.test.ts       # Lay-by lifecycle
├── insights/
│   ├── service.test.ts               # Dashboard aggregates
│   ├── sales-analytics.test.ts
│   └── test-harness.ts               # Shared setup for this slice
└── purchase-orders/
    ├── service.test.ts
    ├── drafts.test.ts
    ├── lifecycle.test.ts
    ├── receive.test.ts
    └── test-harness.ts
```

### Worker route tests

Route-level tests belong to the worker that owns the routes, colocated beside the slice:

```
apps/api-inventory/src/modules/<slice>/routes.test.ts
```

### Frontend Component Tests

```
apps/app/zones/inventory/modules/
├── items-list.test.tsx
├── item-form.test.tsx
├── adjustment-form.test.tsx
├── count-entry-grid.test.tsx
└── stock-grid.test.tsx
```

---

## Test Fixtures & Factories

### Base Fixtures

```typescript
// packages/inventory/src/items/test-fixtures.ts

import { faker } from '@faker-js/faker';

export function createTestItem(overrides?: Partial<NewInventoryItem>): NewInventoryItem {
  return {
    organizationId: overrides?.organizationId ?? 'test-org-id',
    name: overrides?.name ?? faker.commerce.productName(),
    sku: overrides?.sku ?? faker.string.alphanumeric(10).toUpperCase(),
    barcode: overrides?.barcode ?? faker.string.numeric(13),
    categoryId: overrides?.categoryId ?? null,
    defaultUomId: overrides?.defaultUomId ?? 'test-ea-uom-id',
    trackInventory: overrides?.trackInventory ?? true,
    reorderLevel: overrides?.reorderLevel ?? '10',
    reorderQty: overrides?.reorderQty ?? '50',
    status: overrides?.status ?? 'active',
  };
}

export function createTestItems(count: number, overrides?: Partial<NewInventoryItem>): NewInventoryItem[] {
  return Array.from({ length: count }, () => createTestItem(overrides));
}
```

```typescript
// packages/inventory/src/locations/test-fixtures.ts

export function createTestLocation(overrides?: Partial<NewInventoryLocation>): NewInventoryLocation {
  return {
    organizationId: overrides?.organizationId ?? 'test-org-id',
    name: overrides?.name ?? faker.company.name() + ' Warehouse',
    type: overrides?.type ?? 'warehouse',
    isDefault: overrides?.isDefault ?? false,
    address: overrides?.address ?? faker.location.streetAddress(),
  };
}
```

```typescript
// packages/inventory/src/units/test-fixtures.ts

export const DEFAULT_TEST_UOMS = [
  { id: 'test-ea-uom-id', code: 'EA', name: 'Each', precision: 0 },
  { id: 'test-kg-uom-id', code: 'KG', name: 'Kilogram', precision: 3 },
  { id: 'test-box-uom-id', code: 'BOX', name: 'Box', precision: 0 },
];

export const DEFAULT_TEST_CONVERSIONS = [
  { fromUomId: 'test-box-uom-id', toUomId: 'test-ea-uom-id', multiplier: '12', isBidirectional: true },
];
```

### Operation Fixtures

```typescript
// packages/inventory/src/shared/test-fixtures.ts

export function createTestAdjustment(overrides?: Partial<CreateAdjustmentInput>): CreateAdjustmentInput {
  return {
    locationId: overrides?.locationId ?? 'test-location-id',
    reason: overrides?.reason ?? 'CORRECTION',
    notes: overrides?.notes ?? 'Test adjustment',
    lines: overrides?.lines ?? [
      { itemId: 'test-item-id', qty: '10', uomId: 'test-ea-uom-id' },
    ],
  };
}

export function createTestTransfer(overrides?: Partial<CreateTransferInput>): CreateTransferInput {
  return {
    fromLocationId: overrides?.fromLocationId ?? 'test-location-1-id',
    toLocationId: overrides?.toLocationId ?? 'test-location-2-id',
    notes: overrides?.notes ?? 'Test transfer',
    lines: overrides?.lines ?? [
      { itemId: 'test-item-id', qty: '5', uomId: 'test-ea-uom-id' },
    ],
  };
}

export function createTestCount(overrides?: Partial<CreateCountInput>): CreateCountInput {
  return {
    locationId: overrides?.locationId ?? 'test-location-id',
    countType: overrides?.countType ?? 'cycle',
    notes: overrides?.notes ?? 'Test count',
    itemIds: overrides?.itemIds,
  };
}
```

### Test Context Factory

```typescript
// packages/inventory/src/shared/test-context.ts

export function createTestContext(overrides?: Partial<ServiceContext>): ServiceContext {
  return {
    userId: overrides?.userId ?? 'test-user-id',
    actor: overrides?.actor ?? { kind: 'tenant', userId: 'test-user-id' },
    orgId: overrides?.orgId ?? 'test-org-id',
    roles: overrides?.roles ?? ['admin'],
    requestId: overrides?.requestId ?? faker.string.uuid(),
    ipAddress: overrides?.ipAddress ?? '127.0.0.1',
    userAgent: overrides?.userAgent ?? 'test-agent',
    env: overrides?.env ?? {},
  };
}

export function createTestDependencies(db: DrizzleClient): ServiceDeps {
  return {
    db,
    // Add other dependencies as needed
  };
}
```

---

## Unit Tests

### UoM Conversion Math

```typescript
// packages/inventory/src/units/service.test.ts

import { describe, it, expect } from 'vitest';
import { convertQuantity, findConversionPath } from '../uom.service';

describe('UoM Conversion', () => {
  const conversions = [
    { fromUomId: 'box', toUomId: 'ea', multiplier: '12', isBidirectional: true },
    { fromUomId: 'kg', toUomId: 'g', multiplier: '1000', isBidirectional: true },
  ];

  describe('convertQuantity', () => {
    it('converts BOX to EA correctly', () => {
      const result = convertQuantity('2', 'box', 'ea', conversions);
      expect(result).toBe('24'); // 2 * 12
    });

    it('converts EA to BOX correctly (reverse)', () => {
      const result = convertQuantity('24', 'ea', 'box', conversions);
      expect(result).toBe('2'); // 24 / 12
    });

    it('handles decimal quantities', () => {
      const result = convertQuantity('1.5', 'kg', 'g', conversions);
      expect(result).toBe('1500'); // 1.5 * 1000
    });

    it('returns same quantity for same UoM', () => {
      const result = convertQuantity('10', 'ea', 'ea', conversions);
      expect(result).toBe('10');
    });

    it('throws error for unknown conversion', () => {
      expect(() => convertQuantity('10', 'ea', 'kg', conversions))
        .toThrow('UOM_CONVERSION_NOT_FOUND');
    });

    it('handles precision correctly', () => {
      const result = convertQuantity('1', 'ea', 'box', conversions);
      expect(result).toBe('0.0833'); // 1 / 12, rounded to 4 decimals
    });
  });

  describe('findConversionPath', () => {
    it('finds direct conversion', () => {
      const path = findConversionPath('box', 'ea', conversions);
      expect(path).toHaveLength(1);
      expect(path[0].multiplier).toBe('12');
    });

    it('finds reverse conversion', () => {
      const path = findConversionPath('ea', 'box', conversions);
      expect(path).toHaveLength(1);
      expect(path[0].isReverse).toBe(true);
    });

    it('returns empty for no path', () => {
      const path = findConversionPath('ea', 'kg', conversions);
      expect(path).toHaveLength(0);
    });
  });
});
```

### Ledger Posting Rules

```typescript
// packages/inventory/src/stock/service.test.ts

import { describe, it, expect, beforeEach } from 'vitest';
import { createMovement, updateBalance, validateNegativeStock } from '../stock-ledger.service';

describe('Stock Ledger', () => {
  describe('validateNegativeStock', () => {
    it('allows positive balance', () => {
      expect(() => validateNegativeStock('10', '-5', false)).not.toThrow();
    });

    it('blocks negative result when policy is strict', () => {
      expect(() => validateNegativeStock('5', '-10', false))
        .toThrow('NEGATIVE_STOCK_BLOCKED');
    });

    it('allows negative when override is true', () => {
      expect(() => validateNegativeStock('5', '-10', true)).not.toThrow();
    });

    it('handles zero balance', () => {
      expect(() => validateNegativeStock('5', '-5', false)).not.toThrow();
    });
  });

  describe('updateBalance', () => {
    it('creates balance row if not exists', async () => {
      // Test upsert behavior
    });

    it('updates existing balance', async () => {
      // Test increment/decrement
    });

    it('handles concurrent updates with locking', async () => {
      // Test SELECT FOR UPDATE behavior
    });
  });
});
```

### Status Transition Validation

```typescript
// packages/inventory/src/adjustments/service.test.ts

import { describe, it, expect } from 'vitest';
import { validateStatusTransition } from '../adjustments.service';

describe('Adjustment Status Transitions', () => {
  it('allows draft → posted', () => {
    expect(validateStatusTransition('draft', 'posted')).toBe(true);
  });

  it('allows posted → void for admin', () => {
    expect(validateStatusTransition('posted', 'void', ['admin'])).toBe(true);
  });

  it('blocks posted → void for non-admin', () => {
    expect(validateStatusTransition('posted', 'void', ['member'])).toBe(false);
  });

  it('blocks posted → draft', () => {
    expect(validateStatusTransition('posted', 'draft')).toBe(false);
  });

  it('blocks void → any', () => {
    expect(validateStatusTransition('void', 'draft')).toBe(false);
    expect(validateStatusTransition('void', 'posted')).toBe(false);
  });
});
```

---

## Integration Tests

### Adjustment Posting Flow

```typescript
// packages/inventory/src/adjustments/posting.test.ts

import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { setupTestDatabase, cleanupTestDatabase } from '@repo/testing';

describe('Adjustment Posting Integration', () => {
  let db: DrizzleClient;
  let ctx: ServiceContext;
  let deps: ServiceDeps;

  beforeEach(async () => {
    db = await setupTestDatabase();
    ctx = createTestContext();
    deps = createTestDependencies(db);

    // Seed test data
    await seedTestUoms(db);
    await seedTestLocation(db, ctx.orgId);
    await seedTestItem(db, ctx.orgId);
    await seedInitialBalance(db, ctx.orgId, 'test-item-id', 'test-location-id', '100');
  });

  afterEach(async () => {
    await cleanupTestDatabase(db);
  });

  it('creates movements and updates balance on post', async () => {
    // 1. Create adjustment
    const adjustment = await createAdjustment(ctx, deps, {
      locationId: 'test-location-id',
      reason: 'DAMAGE',
      lines: [{ itemId: 'test-item-id', qty: '-10', uomId: 'test-ea-uom-id' }],
    });

    expect(adjustment.status).toBe('draft');

    // 2. Post adjustment
    const idempotencyKey = faker.string.uuid();
    const posted = await postAdjustment(ctx, deps, adjustment.id, idempotencyKey);

    expect(posted.status).toBe('posted');
    expect(posted.postedAt).toBeTruthy();

    // 3. Verify movement created
    const movements = await getMovements(ctx, deps, { refId: adjustment.id });
    expect(movements).toHaveLength(1);
    expect(movements[0].movementType).toBe('ADJUSTMENT_OUT');
    expect(movements[0].qty).toBe('-10');

    // 4. Verify balance updated
    const balance = await getBalance(ctx, deps, 'test-item-id', 'test-location-id');
    expect(balance.onHandQty).toBe('90'); // 100 - 10
  });

  it('blocks negative stock when posting', async () => {
    const adjustment = await createAdjustment(ctx, deps, {
      locationId: 'test-location-id',
      reason: 'LOSS',
      lines: [{ itemId: 'test-item-id', qty: '-150', uomId: 'test-ea-uom-id' }],
    });

    await expect(postAdjustment(ctx, deps, adjustment.id, faker.string.uuid()))
      .rejects.toThrow('NEGATIVE_STOCK_BLOCKED');

    // Verify nothing changed
    const balance = await getBalance(ctx, deps, 'test-item-id', 'test-location-id');
    expect(balance.onHandQty).toBe('100'); // Unchanged
  });

  it('handles idempotency correctly', async () => {
    const adjustment = await createAdjustment(ctx, deps, {
      locationId: 'test-location-id',
      reason: 'FOUND',
      lines: [{ itemId: 'test-item-id', qty: '10', uomId: 'test-ea-uom-id' }],
    });

    const idempotencyKey = faker.string.uuid();

    // First post
    const result1 = await postAdjustment(ctx, deps, adjustment.id, idempotencyKey);
    expect(result1.status).toBe('posted');

    // Second post with same key (retry)
    const result2 = await postAdjustment(ctx, deps, adjustment.id, idempotencyKey);
    expect(result2.id).toBe(result1.id);

    // Verify only one movement
    const movements = await getMovements(ctx, deps, { refId: adjustment.id });
    expect(movements).toHaveLength(1);

    // Verify balance updated only once
    const balance = await getBalance(ctx, deps, 'test-item-id', 'test-location-id');
    expect(balance.onHandQty).toBe('110'); // 100 + 10, not 120
  });
});
```

### Transfer Flow

```typescript
// packages/inventory/src/transfers/service.test.ts

describe('Transfer Flow Integration', () => {
  it('moves stock between locations on complete', async () => {
    // Setup: 100 at location 1, 0 at location 2
    await seedInitialBalance(db, ctx.orgId, 'test-item-id', 'loc-1', '100');
    await seedInitialBalance(db, ctx.orgId, 'test-item-id', 'loc-2', '0');

    // Create and complete transfer
    const transfer = await createTransfer(ctx, deps, {
      fromLocationId: 'loc-1',
      toLocationId: 'loc-2',
      lines: [{ itemId: 'test-item-id', qty: '30', uomId: 'test-ea-uom-id' }],
    });

    await completeTransfer(ctx, deps, transfer.id, faker.string.uuid());

    // Verify balances
    const balance1 = await getBalance(ctx, deps, 'test-item-id', 'loc-1');
    const balance2 = await getBalance(ctx, deps, 'test-item-id', 'loc-2');

    expect(balance1.onHandQty).toBe('70'); // 100 - 30
    expect(balance2.onHandQty).toBe('30'); // 0 + 30
  });

  it('creates TRANSFER_OUT and TRANSFER_IN movements', async () => {
    // ...similar setup...
    
    await completeTransfer(ctx, deps, transfer.id, faker.string.uuid());

    const movements = await getMovements(ctx, deps, { refId: transfer.id });
    expect(movements).toHaveLength(2);
    
    const outMovement = movements.find(m => m.movementType === 'TRANSFER_OUT');
    const inMovement = movements.find(m => m.movementType === 'TRANSFER_IN');
    
    expect(outMovement?.locationId).toBe('loc-1');
    expect(outMovement?.qty).toBe('-30');
    
    expect(inMovement?.locationId).toBe('loc-2');
    expect(inMovement?.qty).toBe('30');
  });
});
```

### Concurrent Posting

```typescript
// packages/inventory/src/stock/concurrent-posting.test.ts

describe('Concurrent Posting', () => {
  it('handles concurrent adjustments without corruption', async () => {
    // Setup: 100 initial
    await seedInitialBalance(db, ctx.orgId, 'test-item-id', 'test-location-id', '100');

    // Create two adjustments that each subtract 30
    const adj1 = await createAdjustment(ctx, deps, {
      locationId: 'test-location-id',
      reason: 'DAMAGE',
      lines: [{ itemId: 'test-item-id', qty: '-30', uomId: 'test-ea-uom-id' }],
    });
    
    const adj2 = await createAdjustment(ctx, deps, {
      locationId: 'test-location-id',
      reason: 'LOSS',
      lines: [{ itemId: 'test-item-id', qty: '-30', uomId: 'test-ea-uom-id' }],
    });

    // Post both concurrently
    const results = await Promise.allSettled([
      postAdjustment(ctx, deps, adj1.id, faker.string.uuid()),
      postAdjustment(ctx, deps, adj2.id, faker.string.uuid()),
    ]);

    // Both should succeed (100 - 30 - 30 = 40, still positive)
    expect(results[0].status).toBe('fulfilled');
    expect(results[1].status).toBe('fulfilled');

    // Verify final balance is correct
    const balance = await getBalance(ctx, deps, 'test-item-id', 'test-location-id');
    expect(balance.onHandQty).toBe('40'); // 100 - 30 - 30
  });

  it('blocks second adjustment if it would cause negative', async () => {
    // Setup: 50 initial
    await seedInitialBalance(db, ctx.orgId, 'test-item-id', 'test-location-id', '50');

    // Create two adjustments that each subtract 40
    const adj1 = await createAdjustment(ctx, deps, {
      locationId: 'test-location-id',
      reason: 'DAMAGE',
      lines: [{ itemId: 'test-item-id', qty: '-40', uomId: 'test-ea-uom-id' }],
    });
    
    const adj2 = await createAdjustment(ctx, deps, {
      locationId: 'test-location-id',
      reason: 'LOSS',
      lines: [{ itemId: 'test-item-id', qty: '-40', uomId: 'test-ea-uom-id' }],
    });

    // Post sequentially (one should fail)
    await postAdjustment(ctx, deps, adj1.id, faker.string.uuid());
    
    await expect(postAdjustment(ctx, deps, adj2.id, faker.string.uuid()))
      .rejects.toThrow('NEGATIVE_STOCK_BLOCKED');

    // Verify balance
    const balance = await getBalance(ctx, deps, 'test-item-id', 'test-location-id');
    expect(balance.onHandQty).toBe('10'); // 50 - 40, second blocked
  });
});
```

---

## E2E Smoke Tests

### Golden Path Scenarios

```typescript
// e2e/inventory/golden-path.spec.ts

import { test, expect } from '@playwright/test';

test.describe('Inventory Golden Path', () => {
  test.beforeEach(async ({ page }) => {
    // Login as manager
    await loginAsManager(page);
  });

  test('complete inventory workflow: item → adjustment → transfer → count', async ({ page }) => {
    // 1. Create item
    await page.goto('/inventory/items/new');
    await page.fill('[name="name"]', 'Test Widget');
    await page.fill('[name="sku"]', 'WIDGET-001');
    await page.selectOption('[name="defaultUomId"]', 'EA');
    await page.fill('[name="reorderLevel"]', '10');
    await page.click('button:has-text("Create Item")');
    
    await expect(page).toHaveURL(/\/inventory\/items\/[a-f0-9-]+/);
    await expect(page.locator('h1')).toContainText('Test Widget');

    // 2. Create opening balance adjustment
    await page.goto('/inventory/adjustments/new');
    await page.selectOption('[name="reason"]', 'OPENING_BALANCE');
    await page.click('button:has-text("Add Item")');
    await page.fill('[name="lines.0.itemId"]', 'Test Widget');
    await page.fill('[name="lines.0.qty"]', '100');
    await page.click('button:has-text("Post")');
    
    await expect(page.locator('.toast')).toContainText('Adjustment posted');

    // 3. Verify stock
    await page.goto('/inventory/stock');
    await expect(page.locator('tr:has-text("Test Widget") td:nth-child(2)')).toContainText('100');

    // 4. Create transfer
    await page.goto('/inventory/transfers/new');
    await page.selectOption('[name="fromLocationId"]', 'Main Warehouse');
    await page.selectOption('[name="toLocationId"]', 'Retail Store');
    await page.click('button:has-text("Add Item")');
    await page.fill('[name="lines.0.qty"]', '20');
    await page.click('button:has-text("Ship & Receive")');
    
    await expect(page.locator('.toast')).toContainText('Transfer completed');

    // 5. Verify stock at both locations
    await page.goto('/inventory/stock');
    // Main Warehouse: 80, Retail Store: 20

    // 6. Create stock count
    await page.goto('/inventory/counts/new');
    await page.selectOption('[name="locationId"]', 'Main Warehouse');
    await page.click('button:has-text("Start Count")');
    
    // Enter count (find 78 instead of 80)
    await page.fill('input[data-item="Test Widget"]', '78');
    await page.click('button:has-text("Complete")');
    await page.click('button:has-text("Post")');
    
    await expect(page.locator('.toast')).toContainText('Count posted');

    // 7. Verify variance in movements
    await page.goto('/inventory/movements');
    await expect(page.locator('tr:has-text("COUNT_VARIANCE_OUT")')).toBeVisible();
  });
});
```

### Failure Scenarios

```typescript
test.describe('Inventory Error Handling', () => {
  test('shows error when posting negative stock', async ({ page }) => {
    // Setup: Item with 10 qty
    // Try to adjust -15
    
    await page.goto('/inventory/adjustments/new');
    // ... fill form with qty = -15 ...
    await page.click('button:has-text("Post")');
    
    await expect(page.locator('.alert-destructive'))
      .toContainText('would result in negative stock');
  });

  test('shows warning for low stock', async ({ page }) => {
    // Adjust item to below reorder level
    // ...
    
    await expect(page.locator('.toast')).toContainText('Low stock alert');
  });

  test('handles network retry gracefully', async ({ page }) => {
    // Intercept and fail first request
    let requestCount = 0;
    await page.route('**/inventory/adjustments/*/post', async (route) => {
      requestCount++;
      if (requestCount === 1) {
        await route.abort('connectionfailed');
      } else {
        await route.continue();
      }
    });

    // Try to post, should auto-retry
    await page.click('button:has-text("Post")');
    
    // Should eventually succeed with idempotency
    await expect(page.locator('.toast')).toContainText('Adjustment posted');
  });
});
```

---

## Test Data Seeding

### Database Seed Script

```typescript
// packages/database/src/seeds/seed-inventory-test-data.ts

export async function seedInventoryTestData(db: DrizzleClient, orgId: string) {
  // 1. Seed locations
  const [mainWarehouse, retailStore] = await db.insert(inventoryLocations).values([
    { organizationId: orgId, name: 'Main Warehouse', type: 'warehouse', isDefault: true },
    { organizationId: orgId, name: 'Retail Store', type: 'store' },
  ]).returning();

  // 2. Seed items
  const items = await db.insert(inventoryItems).values([
    { organizationId: orgId, name: 'Office Chair', sku: 'CHAIR-001', defaultUomId: 'ea-id', reorderLevel: '10' },
    { organizationId: orgId, name: 'Printer Paper', sku: 'PAPER-001', defaultUomId: 'reams-id', reorderLevel: '50' },
    { organizationId: orgId, name: 'Stapler', sku: 'STPL-001', defaultUomId: 'ea-id', reorderLevel: '5' },
  ]).returning();

  // 3. Seed initial balances
  await db.insert(inventoryStockBalances).values([
    { organizationId: orgId, itemId: items[0].id, locationId: mainWarehouse.id, onHandQty: '45' },
    { organizationId: orgId, itemId: items[0].id, locationId: retailStore.id, onHandQty: '15' },
    { organizationId: orgId, itemId: items[1].id, locationId: mainWarehouse.id, onHandQty: '100' },
    { organizationId: orgId, itemId: items[1].id, locationId: retailStore.id, onHandQty: '20' },
    { organizationId: orgId, itemId: items[2].id, locationId: mainWarehouse.id, onHandQty: '10' },
  ]);

  return { locations: [mainWarehouse, retailStore], items };
}
```

---

## Running Tests

### Commands

```bash
# Run all inventory tests
pnpm --filter @repo/inventory test

# Run a specific test file
pnpm --filter @repo/inventory test src/adjustments/service.test.ts

# Run every suite, per-package and cached (what CI runs)
turbo run test

# Run every suite in one process (local DX)
pnpm test:local

# Run e2e tests
pnpm test:e2e -- e2e/inventory/

# Run with coverage
pnpm --filter @repo/inventory test -- --coverage
```

### CI Configuration

```yaml
# .github/workflows/test.yml
jobs:
  test-inventory:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: bumara_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - run: pnpm install
      - run: pnpm db:migrate --env=test
      - run: pnpm test --filter=@repo/api-services -- --grep="inventory"
```

---

## Related Documentation

- [Data Model](/inventory/03-data-model) — Schema for test fixtures
- [API Spec](/inventory/04-api-spec) — Endpoints to test
- [Workflows](/inventory/05-workflows) — Business rules to verify

---

## Open Questions

1. **Test database**: Shared vs isolated database per test suite?
2. **Performance tests**: Load testing for concurrent posting?
3. **Visual regression**: Screenshot tests for UI components?
4. **Contract tests**: API contract tests with backend?
