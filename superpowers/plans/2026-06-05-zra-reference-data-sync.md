---
title: "ZRA Reference Data Sync Implementation Plan"
description: "For agentic workers: REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this..."
---

**Goal:** Pull and cache ZRA's standard codes, item classifications, and notices into the database so ERP dropdowns match ZRA exactly, refreshed daily by the existing cron worker with a backoffice manual trigger.

**Architecture:** Four new global tables (`zra_code_classes`, `zra_codes`, `zra_item_classifications`, `zra_notices`, plus audit log `zra_reference_sync_log`) populated by three new `ZraApiClient` methods, exposed via a new service (`zra-reference-data.service.ts`), called by a new daily job in `apps/api-jobs` and by new HTTP routes in `apps/api-invoicing`. Never deletes — uses `useYn='N'` for soft deactivation.

**Tech Stack:** Drizzle ORM (Postgres), TypeScript, Cloudflare Workers (Hono + zod-openapi), Vitest with `vi.hoisted` mocks.

**Spec:** [docs/superpowers/specs/2026-06-05-zra-reference-data-sync-design.md](/superpowers/specs/2026-06-05-zra-reference-data-sync-design)

---

## File Structure

**New schema:**
- `packages/database/src/schema/invoicing/zra-reference-data.ts` — four reference tables + sync log

**Modify schema indexes:**
- `packages/database/src/schema/invoicing/index.ts` — export new types/tables

**New repository:**
- `packages/database/src/repositories/zra-reference-data.ts` — all CRUD for the new tables + `findAnyActiveVsdcDevice`

**Modify repository index:**
- `packages/database/src/repositories/index.ts` — re-export new repo

**API client additions:**
- `packages/api-services/src/domains/invoicing/zra/zra-api-client.ts` — add 3 endpoint constants, 4 wire types, 3 fetcher methods

**New service:**
- `packages/api-services/src/domains/invoicing/zra/zra-reference-data.service.ts` — sync + read functions
- `packages/api-services/src/domains/invoicing/zra/zra-reference-data.types.ts` — DTOs for handlers

**Modify domain barrel:**
- `packages/api-services/src/domains/invoicing/index.ts` — export new service

**New routes:**
- `apps/api-invoicing/src/routes/invoicing/zra-reference-data/routes.ts`
- `apps/api-invoicing/src/routes/invoicing/zra-reference-data/handlers.ts`
- `apps/api-invoicing/src/routes/invoicing/zra-reference-data/index.ts`

**Modify route registration:**
- `apps/api-invoicing/src/routes/invoicing/index.ts` (or wherever invoicing routers compose) — mount new router

**New cron job:**
- `apps/api-jobs/src/jobs/zra-reference-sync.ts` — `runZraReferenceSyncJob(db, env)`

**Modify scheduler:**
- `apps/api-jobs/src/jobs/scheduler.ts` — wire the new job

**Tests:**
- `packages/database/src/repositories/__tests__/zra-reference-data.repo.test.ts`
- `packages/api-services/src/domains/invoicing/zra/__tests__/zra-reference-data.service.test.ts`

**Migration (auto-generated):**
- `packages/database/drizzle/XXXX_<auto_name>.sql`

---

## Task 1: Schema — ZRA Reference Data Tables

**Files:**
- Create: `packages/database/src/schema/invoicing/zra-reference-data.ts`

- [ ] **Step 1: Create the schema file**

Write the file:

```typescript
import type { InferInsertModel, InferSelectModel } from 'drizzle-orm';
import {
  index,
  integer,
  jsonb,
  pgTable,
  text,
  timestamp,
  uniqueIndex,
  uuid,
} from 'drizzle-orm/pg-core';
import { timestamps } from '../../helpers/timestamps';
import { zraVsdcDevices } from './zra-smart-invoice';

/**
 * ZRA Code Classes — metadata for code groups (e.g. "04" = Tax Type)
 */
export const zraCodeClasses = pgTable(
  'zra_code_classes',
  {
    codeClass: text('code_class').primaryKey(),
    codeClassName: text('code_class_name').notNull(),
    codeClassDescription: text('code_class_description'),
    userDefinedName1: text('user_defined_name_1'),
    userDefinedName2: text('user_defined_name_2'),
    userDefinedName3: text('user_defined_name_3'),
    useYn: text('use_yn').notNull().default('Y'),
    syncedAt: timestamp('synced_at', { mode: 'date' }).notNull(),
    ...timestamps,
  },
  (table) => [index('idx_zra_code_classes_use_yn').on(table.useYn)]
);

/**
 * ZRA Codes — flat table of all code detail rows.
 * For tax-type class ("04"), userDefinedCode1 holds the tax rate as a string.
 */
export const zraCodes = pgTable(
  'zra_codes',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    codeClass: text('code_class')
      .notNull()
      .references(() => zraCodeClasses.codeClass, { onDelete: 'cascade' }),
    code: text('code').notNull(),
    codeName: text('code_name').notNull(),
    codeDescription: text('code_description'),
    sortOrder: integer('sort_order').notNull().default(0),
    userDefinedCode1: text('user_defined_code_1'),
    userDefinedCode2: text('user_defined_code_2'),
    userDefinedCode3: text('user_defined_code_3'),
    useYn: text('use_yn').notNull().default('Y'),
    syncedAt: timestamp('synced_at', { mode: 'date' }).notNull(),
    ...timestamps,
  },
  (table) => [
    uniqueIndex('uq_zra_codes_class_code').on(table.codeClass, table.code),
    index('idx_zra_codes_class_use_sort').on(
      table.codeClass,
      table.useYn,
      table.sortOrder
    ),
  ]
);

/**
 * ZRA Item Classifications — hierarchical HS-like item classes (1–5 deep).
 */
export const zraItemClassifications = pgTable(
  'zra_item_classifications',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    itemClassCode: text('item_class_code').notNull(),
    itemClassName: text('item_class_name').notNull(),
    itemClassLevel: integer('item_class_level').notNull(),
    taxTypeCode: text('tax_type_code'),
    majorTargetYn: text('major_target_yn'),
    useYn: text('use_yn').notNull().default('Y'),
    syncedAt: timestamp('synced_at', { mode: 'date' }).notNull(),
    ...timestamps,
  },
  (table) => [
    uniqueIndex('uq_zra_item_class_code').on(table.itemClassCode),
    index('idx_zra_item_class_level_use').on(
      table.itemClassLevel,
      table.useYn
    ),
  ]
);

/**
 * ZRA Notices — official ZRA announcements.
 */
export const zraNotices = pgTable(
  'zra_notices',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    noticeNumber: integer('notice_number').notNull(),
    title: text('title').notNull(),
    contents: text('contents').notNull(),
    registeredAt: timestamp('registered_at', { mode: 'date' }).notNull(),
    detailUrl: text('detail_url'),
    syncedAt: timestamp('synced_at', { mode: 'date' }).notNull(),
    ...timestamps,
  },
  (table) => [
    uniqueIndex('uq_zra_notices_number').on(table.noticeNumber),
    index('idx_zra_notices_registered_at').on(table.registeredAt),
  ]
);

/**
 * ZRA Reference Sync Log — observability for sync runs.
 */
export const zraReferenceSyncLog = pgTable(
  'zra_reference_sync_log',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    datasetType: text('dataset_type').notNull(), // 'codes' | 'item_classifications' | 'notices'
    triggeredBy: text('triggered_by').notNull(), // 'cron' | 'manual'
    triggeredByUserId: text('triggered_by_user_id'),
    vsdcDeviceId: uuid('vsdc_device_id').references(() => zraVsdcDevices.id, {
      onDelete: 'set null',
    }),
    status: text('status').notNull(), // 'success' | 'failed' | 'partial'
    recordsFetched: integer('records_fetched').notNull().default(0),
    recordsUpserted: integer('records_upserted').notNull().default(0),
    recordsDeactivated: integer('records_deactivated').notNull().default(0),
    perClassStats: jsonb('per_class_stats'),
    errorMessage: text('error_message'),
    durationMs: integer('duration_ms'),
    startedAt: timestamp('started_at', { mode: 'date' }).notNull(),
    completedAt: timestamp('completed_at', { mode: 'date' }),
    ...timestamps,
  },
  (table) => [
    index('idx_zra_sync_log_dataset_started').on(
      table.datasetType,
      table.startedAt
    ),
    index('idx_zra_sync_log_status_started').on(
      table.status,
      table.startedAt
    ),
  ]
);

// ============================================
// Type Exports
// ============================================

export type ZraCodeClass = InferSelectModel<typeof zraCodeClasses>;
export type ZraCodeClassInsert = InferInsertModel<typeof zraCodeClasses>;

export type ZraCode = InferSelectModel<typeof zraCodes>;
export type ZraCodeInsert = InferInsertModel<typeof zraCodes>;

export type ZraItemClassification = InferSelectModel<typeof zraItemClassifications>;
export type ZraItemClassificationInsert = InferInsertModel<typeof zraItemClassifications>;

export type ZraNotice = InferSelectModel<typeof zraNotices>;
export type ZraNoticeInsert = InferInsertModel<typeof zraNotices>;

export type ZraReferenceSyncLog = InferSelectModel<typeof zraReferenceSyncLog>;
export type ZraReferenceSyncLogInsert = InferInsertModel<typeof zraReferenceSyncLog>;
```

- [ ] **Step 2: Export from invoicing schema index**

Edit `packages/database/src/schema/invoicing/index.ts`. Find the existing `// ZRA Smart Invoice` export block and add right after it:

```typescript
// ZRA Reference Data
export {
    type ZraCodeClass,
    type ZraCodeClassInsert,
    zraCodeClasses,
    type ZraCode,
    type ZraCodeInsert,
    zraCodes,
    type ZraItemClassification,
    type ZraItemClassificationInsert,
    zraItemClassifications,
    type ZraNotice,
    type ZraNoticeInsert,
    zraNotices,
    type ZraReferenceSyncLog,
    type ZraReferenceSyncLogInsert,
    zraReferenceSyncLog,
} from "./zra-reference-data";
```

- [ ] **Step 3: Typecheck the schema package**

Run: `pnpm --filter @repo/database typecheck`
Expected: PASS (no type errors)

- [ ] **Step 4: Generate the migration**

Run: `pnpm --filter @repo/database db:generate`
Expected: A new file appears in `packages/database/drizzle/` (e.g. `0XXX_<name>.sql`) containing `CREATE TABLE zra_code_classes`, `CREATE TABLE zra_codes`, `CREATE TABLE zra_item_classifications`, `CREATE TABLE zra_notices`, `CREATE TABLE zra_reference_sync_log`, and the indexes.

Inspect the file briefly — confirm all five tables and indexes are present, and no unrelated changes were generated. If unrelated changes appear, the generator picked up drift from other schemas; investigate before continuing.

- [ ] **Step 5: Commit**

```bash
git add packages/database/src/schema/invoicing/zra-reference-data.ts \
        packages/database/src/schema/invoicing/index.ts \
        packages/database/drizzle/
git commit -m "feat(db): add ZRA reference data schema (codes, item classes, notices, sync log)"
```

---

## Task 2: Repository — Reference Data Queries

**Files:**
- Create: `packages/database/src/repositories/zra-reference-data.ts`
- Modify: `packages/database/src/repositories/index.ts`
- Test: `packages/database/src/repositories/__tests__/zra-reference-data.repo.test.ts`

> **IMPORTANT — plan revision during execution:** The original plan called for behavior-level repo tests (idempotency, deactivation, restoration) using an in-memory database. Inspection of `packages/database/src/repositories/__tests__/mock-db.ts` and existing tests (e.g. `regulator-payouts.test.ts`) revealed this codebase uses **fully-mocked DB clients** for repo tests — they verify call shape, not SQL behavior. Behavior-level verification happens at the service layer (Task 4) and via the smoke test (Task 7).
>
> **Adjustment:** Write minimal call-shape tests for the repo following the local mock-db pattern. Skip behavior assertions (idempotency, conflict semantics, deactivation correctness) — those are covered by service tests in Task 4 and by the smoke test in Task 7.

- [ ] **Step 1: Write the failing test file**

Create `packages/database/src/repositories/__tests__/zra-reference-data.repo.test.ts` following the **local mock-db pattern**. See `regulator-payouts.test.ts` and `mock-db.ts` in the same directory for the exact convention. Tests should only verify that each repo function calls the expected drizzle chain (e.g. `db.insert(...).values(...).onConflictDoUpdate(...)`), not actual SQL behavior. Cover one test per public function — keep it minimal.

Use this pattern (adapt to actual function names):

```typescript
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { mockDb } from './mock-db';

vi.mock('../../schema', () => {
  const col = (n: string) => ({ name: n, getSQL: () => n });
  const tbl = (name: string, cols: string[]) => {
    const t: Record<string, unknown> = { _: { name } };
    for (const c of cols) t[c] = col(`${name}.${c}`);
    return t;
  };
  return {
    zraCodeClasses: tbl('zra_code_classes', ['codeClass','codeClassName','useYn','syncedAt']),
    zraCodes: tbl('zra_codes', ['id','codeClass','code','codeName','useYn','syncedAt','sortOrder']),
    zraItemClassifications: tbl('zra_item_classifications', ['id','itemClassCode','itemClassName','itemClassLevel','useYn','syncedAt']),
    zraNotices: tbl('zra_notices', ['id','noticeNumber','title','contents','registeredAt','syncedAt']),
    zraReferenceSyncLog: tbl('zra_reference_sync_log', ['id','datasetType','status','startedAt']),
    zraVsdcDevices: tbl('zra_vsdc_devices', ['id','deviceStatus','isInitialized','deletedAt','createdAt']),
  };
});

vi.mock('drizzle-orm', () => ({
  and: vi.fn((...a) => ({ _t: 'and', a })),
  eq: vi.fn((a, b) => ({ _t: 'eq', a, b })),
  asc: vi.fn((a) => ({ _t: 'asc', a })),
  desc: vi.fn((a) => ({ _t: 'desc', a })),
  ilike: vi.fn((a, b) => ({ _t: 'ilike', a, b })),
  inArray: vi.fn((a, b) => ({ _t: 'inArray', a, b })),
  isNull: vi.fn((a) => ({ _t: 'isNull', a })),
  not: vi.fn((a) => ({ _t: 'not', a })),
  sql: Object.assign(vi.fn(), {
    raw: vi.fn(),
  }),
}));

import * as repo from '../zra-reference-data';

describe('zra-reference-data repository (call shape)', () => {
  beforeEach(() => vi.clearAllMocks());

  it('upsertCodeClasses calls insert→values→onConflictDoUpdate', async () => {
    const db = mockDb();
    await repo.upsertCodeClasses(db, [{ codeClass: '04', codeClassName: 'Tax', useYn: 'Y', syncedAt: new Date() }]);
    expect(db.insert).toHaveBeenCalledTimes(1);
  });

  // One similar minimal test per function — see existing tests for shape.
});
```

The point is to verify the module imports and chain shape; behavior coverage is in Task 4 service tests.

- [ ] **Step 2: Run the test to verify it fails**

Run: `pnpm --filter @repo/database test -- zra-reference-data.repo`
Expected: FAIL — module `../zra-reference-data` does not exist.

- [ ] **Step 3: Create the repository file**

Create `packages/database/src/repositories/zra-reference-data.ts`:

```typescript
/**
 * Database queries for ZRA reference data (codes, item classifications,
 * notices, and sync log). All functions are pure data access — no business logic.
 *
 * @module repositories/zra-reference-data
 */

import { and, asc, desc, eq, ilike, inArray, isNull, not, sql } from 'drizzle-orm';
import type { DatabaseClient } from '@/client';
import {
  zraCodeClasses,
  type ZraCodeClass,
  type ZraCodeClassInsert,
  zraCodes,
  type ZraCode,
  type ZraCodeInsert,
  zraItemClassifications,
  type ZraItemClassification,
  type ZraItemClassificationInsert,
  zraNotices,
  type ZraNotice,
  type ZraNoticeInsert,
  zraReferenceSyncLog,
  type ZraReferenceSyncLog,
  type ZraReferenceSyncLogInsert,
  zraVsdcDevices,
  type ZraVsdcDevice,
} from '@/schema/invoicing';

// ============================================
// Code classes + codes
// ============================================

export async function upsertCodeClasses(
  db: DatabaseClient,
  classes: ZraCodeClassInsert[]
): Promise<number> {
  if (classes.length === 0) return 0;
  await db
    .insert(zraCodeClasses)
    .values(classes)
    .onConflictDoUpdate({
      target: zraCodeClasses.codeClass,
      set: {
        codeClassName: sql`excluded.code_class_name`,
        codeClassDescription: sql`excluded.code_class_description`,
        userDefinedName1: sql`excluded.user_defined_name_1`,
        userDefinedName2: sql`excluded.user_defined_name_2`,
        userDefinedName3: sql`excluded.user_defined_name_3`,
        useYn: sql`excluded.use_yn`,
        syncedAt: sql`excluded.synced_at`,
      },
    });
  return classes.length;
}

export async function upsertCodes(
  db: DatabaseClient,
  codes: ZraCodeInsert[]
): Promise<number> {
  if (codes.length === 0) return 0;
  await db
    .insert(zraCodes)
    .values(codes)
    .onConflictDoUpdate({
      target: [zraCodes.codeClass, zraCodes.code],
      set: {
        codeName: sql`excluded.code_name`,
        codeDescription: sql`excluded.code_description`,
        sortOrder: sql`excluded.sort_order`,
        userDefinedCode1: sql`excluded.user_defined_code_1`,
        userDefinedCode2: sql`excluded.user_defined_code_2`,
        userDefinedCode3: sql`excluded.user_defined_code_3`,
        useYn: sql`'Y'`, // always restore on re-appear
        syncedAt: sql`excluded.synced_at`,
      },
    });
  return codes.length;
}

export async function deactivateCodesNotIn(
  db: DatabaseClient,
  codeClass: string,
  activeCodes: string[]
): Promise<number> {
  const condition =
    activeCodes.length === 0
      ? and(eq(zraCodes.codeClass, codeClass), eq(zraCodes.useYn, 'Y'))
      : and(
          eq(zraCodes.codeClass, codeClass),
          eq(zraCodes.useYn, 'Y'),
          not(inArray(zraCodes.code, activeCodes))
        );
  const result = await db
    .update(zraCodes)
    .set({ useYn: 'N' })
    .where(condition)
    .returning({ id: zraCodes.id });
  return result.length;
}

export async function listCodesByClass(
  db: DatabaseClient,
  codeClass: string
): Promise<ZraCode[]> {
  return db
    .select()
    .from(zraCodes)
    .where(and(eq(zraCodes.codeClass, codeClass), eq(zraCodes.useYn, 'Y')))
    .orderBy(asc(zraCodes.sortOrder), asc(zraCodes.code));
}

export async function listAllCodeClasses(
  db: DatabaseClient
): Promise<ZraCodeClass[]> {
  return db
    .select()
    .from(zraCodeClasses)
    .where(eq(zraCodeClasses.useYn, 'Y'))
    .orderBy(asc(zraCodeClasses.codeClass));
}

// ============================================
// Item classifications
// ============================================

export async function upsertItemClassifications(
  db: DatabaseClient,
  items: ZraItemClassificationInsert[]
): Promise<number> {
  if (items.length === 0) return 0;
  await db
    .insert(zraItemClassifications)
    .values(items)
    .onConflictDoUpdate({
      target: zraItemClassifications.itemClassCode,
      set: {
        itemClassName: sql`excluded.item_class_name`,
        itemClassLevel: sql`excluded.item_class_level`,
        taxTypeCode: sql`excluded.tax_type_code`,
        majorTargetYn: sql`excluded.major_target_yn`,
        useYn: sql`'Y'`,
        syncedAt: sql`excluded.synced_at`,
      },
    });
  return items.length;
}

export async function deactivateItemClassificationsNotIn(
  db: DatabaseClient,
  activeCodes: string[]
): Promise<number> {
  const condition =
    activeCodes.length === 0
      ? eq(zraItemClassifications.useYn, 'Y')
      : and(
          eq(zraItemClassifications.useYn, 'Y'),
          not(inArray(zraItemClassifications.itemClassCode, activeCodes))
        );
  const result = await db
    .update(zraItemClassifications)
    .set({ useYn: 'N' })
    .where(condition)
    .returning({ id: zraItemClassifications.id });
  return result.length;
}

export async function listItemClassifications(
  db: DatabaseClient,
  opts?: { level?: number; search?: string; limit?: number }
): Promise<ZraItemClassification[]> {
  const conditions = [eq(zraItemClassifications.useYn, 'Y')];
  if (opts?.level !== undefined) {
    conditions.push(eq(zraItemClassifications.itemClassLevel, opts.level));
  }
  if (opts?.search) {
    conditions.push(ilike(zraItemClassifications.itemClassName, `%${opts.search}%`));
  }
  let query = db
    .select()
    .from(zraItemClassifications)
    .where(and(...conditions))
    .orderBy(asc(zraItemClassifications.itemClassCode));
  if (opts?.limit) {
    query = query.limit(opts.limit) as typeof query;
  }
  return query;
}

// ============================================
// Notices
// ============================================

export async function upsertNotices(
  db: DatabaseClient,
  notices: ZraNoticeInsert[]
): Promise<number> {
  if (notices.length === 0) return 0;
  await db
    .insert(zraNotices)
    .values(notices)
    .onConflictDoUpdate({
      target: zraNotices.noticeNumber,
      set: {
        title: sql`excluded.title`,
        contents: sql`excluded.contents`,
        registeredAt: sql`excluded.registered_at`,
        detailUrl: sql`excluded.detail_url`,
        syncedAt: sql`excluded.synced_at`,
      },
    });
  return notices.length;
}

export async function listRecentNotices(
  db: DatabaseClient,
  limit: number
): Promise<ZraNotice[]> {
  return db
    .select()
    .from(zraNotices)
    .orderBy(desc(zraNotices.registeredAt))
    .limit(limit);
}

// ============================================
// Sync log
// ============================================

export async function createSyncLog(
  db: DatabaseClient,
  log: ZraReferenceSyncLogInsert
): Promise<ZraReferenceSyncLog> {
  const [row] = await db.insert(zraReferenceSyncLog).values(log).returning();
  return row!;
}

export async function updateSyncLog(
  db: DatabaseClient,
  id: string,
  patch: Partial<ZraReferenceSyncLogInsert>
): Promise<ZraReferenceSyncLog> {
  const [row] = await db
    .update(zraReferenceSyncLog)
    .set(patch)
    .where(eq(zraReferenceSyncLog.id, id))
    .returning();
  return row!;
}

export async function listRecentSyncLogs(
  db: DatabaseClient,
  limit: number
): Promise<ZraReferenceSyncLog[]> {
  return db
    .select()
    .from(zraReferenceSyncLog)
    .orderBy(desc(zraReferenceSyncLog.startedAt))
    .limit(limit);
}

export async function getLastSuccessfulSync(
  db: DatabaseClient,
  datasetType: string
): Promise<ZraReferenceSyncLog | null> {
  const [row] = await db
    .select()
    .from(zraReferenceSyncLog)
    .where(
      and(
        eq(zraReferenceSyncLog.datasetType, datasetType),
        eq(zraReferenceSyncLog.status, 'success')
      )
    )
    .orderBy(desc(zraReferenceSyncLog.startedAt))
    .limit(1);
  return row ?? null;
}

// ============================================
// VSDC device helper (used by sync service)
// ============================================

export async function findAnyActiveVsdcDevice(
  db: DatabaseClient
): Promise<ZraVsdcDevice | null> {
  const [device] = await db
    .select()
    .from(zraVsdcDevices)
    .where(
      and(
        eq(zraVsdcDevices.deviceStatus, 'active'),
        eq(zraVsdcDevices.isInitialized, true),
        isNull(zraVsdcDevices.deletedAt)
      )
    )
    .orderBy(asc(zraVsdcDevices.createdAt))
    .limit(1);
  return device ?? null;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm --filter @repo/database test -- zra-reference-data.repo`
Expected: PASS (all `describe` blocks green).

If `findAnyActiveVsdcDevice` test fails with a foreign-key error related to `organizations`, the test fixture doesn't include the `organizations` table dependency. In that case adjust the test fixture or skip that single assertion — the function is also exercised in Task 5 service tests.

- [ ] **Step 5: Export the repo from the index**

Edit `packages/database/src/repositories/index.ts`. Add this line in alphabetical order with the other ZRA exports:

```typescript
export * from './zra-reference-data';
```

- [ ] **Step 6: Typecheck**

Run: `pnpm --filter @repo/database typecheck`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add packages/database/src/repositories/zra-reference-data.ts \
        packages/database/src/repositories/index.ts \
        packages/database/src/repositories/__tests__/zra-reference-data.repo.test.ts
git commit -m "feat(db): add ZRA reference data repository with upsert/deactivate semantics"
```

---

## Task 3: API Client — Add Reference Data Fetchers

**Files:**
- Modify: `packages/api-services/src/domains/invoicing/zra/zra-api-client.ts`

- [ ] **Step 1: Add endpoint constants**

In `zra-api-client.ts`, find the `ZRA_API_ENDPOINTS` const (around line 230). Update it:

```typescript
export const ZRA_API_ENDPOINTS = {
  INITIALIZE: "/initializer/selectInitInfo",
  SALES: "/trnsSales/saveSales",
  PURCHASES: "/trnsPurchase/savePurchase",
  STOCK: "/stockMaster/saveStockMaster",
  STOCK_ITEMS: "/stockIO/saveStockItems",
  ITEMS: "/items/saveItem",
  CODE_LIST: "/code/selectCodes",
  ITEM_CLASS_LIST: "/itemClass/selectItemsClass",
  NOTICES: "/notices/selectNotices",
} as const;
```

- [ ] **Step 2: Add wire types**

In the same file, in the "Wire format types" section (after `ZraWirePurchasePayload`, around line 367), add:

```typescript
// ============================================
// Wire types — Reference Data responses
// ============================================

export interface ZraCodeDetailWire {
  cd: string;
  cdNm: string;
  cdDesc: string | null;
  useYn: "Y" | "N";
  srtOrd: number;
  userDfnCd1: string | null;
  userDfnCd2: string | null;
  userDfnCd3: string | null;
}

export interface ZraCodeClassWire {
  cdCls: string;
  cdClsNm: string;
  cdClsDesc: string | null;
  useYn: "Y" | "N";
  userDfnNm1: string | null;
  userDfnNm2: string | null;
  userDfnNm3: string | null;
  dtlList: ZraCodeDetailWire[];
}

export interface ZraItemClassWire {
  itemClsCd: string;
  itemClsNm: string;
  itemClsLvl: number;
  taxTyCd: string | null;
  mjrTgYn: "Y" | "N" | null;
  useYn: "Y" | "N";
}

export interface ZraNoticeWire {
  noticeNo: number;
  title: string;
  cont: string;
  regDt: string; // YYYYMMDDHHmmss
  dtlUrl: string | null;
}
```

- [ ] **Step 3: Add the three fetcher methods on ZraApiClient**

In the `ZraApiClient` class, after `sendStockData` and before `healthCheck` (around line 460), add:

```typescript
  /**
   * Fetch ZRA standard code list. Returns all code classes and their detail entries.
   * Pass lastReqDt to fetch only changes since that timestamp; default fetches all.
   */
  async fetchCodeList(
    lastReqDt = "20180101000000"
  ): Promise<ZraApiResponse<{ clsList: ZraCodeClassWire[] }>> {
    return this.makeRequest("POST", ZRA_API_ENDPOINTS.CODE_LIST, {
      tpin: this.config.tpin ?? "",
      bhfId: this.config.branchId ?? "000",
      lastReqDt,
    });
  }

  /**
   * Fetch ZRA item classification list (HS-like hierarchical codes).
   */
  async fetchItemClassList(
    lastReqDt = "20180101000000"
  ): Promise<ZraApiResponse<{ itemClsList: ZraItemClassWire[] }>> {
    return this.makeRequest("POST", ZRA_API_ENDPOINTS.ITEM_CLASS_LIST, {
      tpin: this.config.tpin ?? "",
      bhfId: this.config.branchId ?? "000",
      lastReqDt,
    });
  }

  /**
   * Fetch ZRA notices (announcements).
   */
  async fetchNotices(
    lastReqDt = "20180101000000"
  ): Promise<ZraApiResponse<{ noticeList: ZraNoticeWire[] }>> {
    return this.makeRequest("POST", ZRA_API_ENDPOINTS.NOTICES, {
      tpin: this.config.tpin ?? "",
      bhfId: this.config.branchId ?? "000",
      lastReqDt,
    });
  }
```

- [ ] **Step 4: Typecheck**

Run: `pnpm --filter @repo/api-services typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/api-services/src/domains/invoicing/zra/zra-api-client.ts
git commit -m "feat(api-services): add ZRA reference data fetchers to ZraApiClient"
```

---

## Task 4: Service — Sync Functions

**Files:**
- Create: `packages/api-services/src/domains/invoicing/zra/zra-reference-data.service.ts`
- Create: `packages/api-services/src/domains/invoicing/zra/zra-reference-data.types.ts`
- Test: `packages/api-services/src/domains/invoicing/zra/__tests__/zra-reference-data.service.test.ts`

- [ ] **Step 1: Write the types file**

Create `packages/api-services/src/domains/invoicing/zra/zra-reference-data.types.ts`:

```typescript
import type {
  ZraCode,
  ZraCodeClass,
  ZraItemClassification,
  ZraNotice,
  ZraReferenceSyncLog,
} from "@repo/database/schema/invoicing";

export type ZraReferenceDatasetType =
  | "codes"
  | "item_classifications"
  | "notices";

export type ZraSyncTrigger = "cron" | "manual";

export interface ZraSyncResult {
  datasetType: ZraReferenceDatasetType;
  status: "success" | "failed" | "partial";
  fetched: number;
  upserted: number;
  deactivated: number;
  durationMs: number;
  errorMessage?: string;
  syncLogId: string | null;
}

// DTOs returned by read functions (trimmed for dropdown use)
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

export type { ZraCode, ZraCodeClass, ZraItemClassification, ZraNotice, ZraReferenceSyncLog };
```

- [ ] **Step 2: Write the failing service test**

Create `packages/api-services/src/domains/invoicing/zra/__tests__/zra-reference-data.service.test.ts`:

```typescript
import { beforeEach, describe, expect, it, vi } from 'vitest';
import type { ServiceContext, ServiceDependencies } from '../../../../core/context';

const mocks = vi.hoisted(() => ({
  // repo
  findAnyActiveVsdcDevice: vi.fn(),
  upsertCodeClasses: vi.fn(),
  upsertCodes: vi.fn(),
  deactivateCodesNotIn: vi.fn(),
  listCodesByClass: vi.fn(),
  listAllCodeClasses: vi.fn(),
  upsertItemClassifications: vi.fn(),
  deactivateItemClassificationsNotIn: vi.fn(),
  listItemClassifications: vi.fn(),
  upsertNotices: vi.fn(),
  listRecentNotices: vi.fn(),
  createSyncLog: vi.fn(),
  updateSyncLog: vi.fn(),
  listRecentSyncLogs: vi.fn(),
  getLastSuccessfulSync: vi.fn(),
  // api client
  fetchCodeList: vi.fn(),
  fetchItemClassList: vi.fn(),
  fetchNotices: vi.fn(),
}));

vi.mock('@repo/database/repositories', () => ({
  findAnyActiveVsdcDevice: mocks.findAnyActiveVsdcDevice,
  upsertCodeClasses: mocks.upsertCodeClasses,
  upsertCodes: mocks.upsertCodes,
  deactivateCodesNotIn: mocks.deactivateCodesNotIn,
  listCodesByClass: mocks.listCodesByClass,
  listAllCodeClasses: mocks.listAllCodeClasses,
  upsertItemClassifications: mocks.upsertItemClassifications,
  deactivateItemClassificationsNotIn: mocks.deactivateItemClassificationsNotIn,
  listItemClassifications: mocks.listItemClassifications,
  upsertNotices: mocks.upsertNotices,
  listRecentNotices: mocks.listRecentNotices,
  createSyncLog: mocks.createSyncLog,
  updateSyncLog: mocks.updateSyncLog,
  listRecentSyncLogs: mocks.listRecentSyncLogs,
  getLastSuccessfulSync: mocks.getLastSuccessfulSync,
}));

vi.mock('../zra-api-client', async (orig) => {
  const actual = await orig<typeof import('../zra-api-client')>();
  return {
    ...actual,
    ZraApiClient: class {
      // eslint-disable-next-line @typescript-eslint/no-explicit-any
      constructor(_cfg: any) {}
      fetchCodeList = mocks.fetchCodeList;
      fetchItemClassList = mocks.fetchItemClassList;
      fetchNotices = mocks.fetchNotices;
    },
  };
});

import {
  syncZraCodes,
  syncZraItemClassifications,
  syncZraNotices,
  syncAllZraReferenceData,
  getZraCodesByClass,
  getZraSyncStatus,
} from '../zra-reference-data.service';

const ctx: ServiceContext = { userId: null, env: {} as ServiceContext['env'] };
const deps: ServiceDependencies = { db: {} as ServiceDependencies['db'] };

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
  mocks.createSyncLog.mockImplementation(async (_db, row) => ({ ...row, id: 'log-1' }));
  mocks.updateSyncLog.mockImplementation(async (_db, _id, patch) => ({ ...patch, id: 'log-1' }));
});

describe('syncZraCodes', () => {
  it('returns failed result when no active VSDC device (cron trigger)', async () => {
    mocks.findAnyActiveVsdcDevice.mockResolvedValue(null);
    const result = await syncZraCodes(deps, 'cron');
    expect(result.status).toBe('failed');
    expect(result.errorMessage).toMatch(/no active vsdc device/i);
    expect(mocks.fetchCodeList).not.toHaveBeenCalled();
  });

  it('throws when no active VSDC device (manual trigger)', async () => {
    mocks.findAnyActiveVsdcDevice.mockResolvedValue(null);
    await expect(syncZraCodes(deps, 'manual')).rejects.toThrow(/no active vsdc device/i);
  });

  it('transforms the sample tax-type response correctly', async () => {
    mocks.findAnyActiveVsdcDevice.mockResolvedValue(fakeDevice);
    mocks.fetchCodeList.mockResolvedValue({
      success: true,
      message: 'ok',
      reference: 'r',
      httpStatus: 200,
      data: {
        clsList: [
          {
            cdCls: '04',
            cdClsNm: 'Tax Type',
            cdClsDesc: null,
            useYn: 'Y',
            userDfnNm1: 'Tax Rate',
            userDfnNm2: null,
            userDfnNm3: null,
            dtlList: [
              { cd: 'A', cdNm: 'A-EX', cdDesc: '...', useYn: 'Y', srtOrd: 1, userDfnCd1: '0', userDfnCd2: null, userDfnCd3: null },
              { cd: 'B', cdNm: 'B 18%', cdDesc: 'B 18%', useYn: 'Y', srtOrd: 2, userDfnCd1: '18', userDfnCd2: null, userDfnCd3: null },
            ],
          },
        ],
      },
    });
    mocks.upsertCodeClasses.mockResolvedValue(1);
    mocks.upsertCodes.mockResolvedValue(2);
    mocks.deactivateCodesNotIn.mockResolvedValue(0);

    const result = await syncZraCodes(deps, 'cron');

    expect(result.status).toBe('success');
    expect(result.fetched).toBe(2);
    expect(result.upserted).toBe(2);

    const codesCall = mocks.upsertCodes.mock.calls[0]![1] as Array<Record<string, unknown>>;
    expect(codesCall[0]).toMatchObject({
      codeClass: '04', code: 'A', codeName: 'A-EX', userDefinedCode1: '0', sortOrder: 1, useYn: 'Y',
    });
    expect(codesCall[1]).toMatchObject({ code: 'B', userDefinedCode1: '18' });
  });

  it('marks sync failed when ZRA returns non-success', async () => {
    mocks.findAnyActiveVsdcDevice.mockResolvedValue(fakeDevice);
    mocks.fetchCodeList.mockResolvedValue({
      success: false,
      message: 'bad credentials',
      error: 'bad credentials',
      reference: 'r',
      httpStatus: 401,
    });
    const result = await syncZraCodes(deps, 'cron');
    expect(result.status).toBe('failed');
    expect(result.errorMessage).toMatch(/bad credentials/);
    expect(mocks.updateSyncLog).toHaveBeenCalledWith(
      deps.db, 'log-1',
      expect.objectContaining({ status: 'failed', errorMessage: expect.stringMatching(/bad credentials/) })
    );
  });

  it('handles empty list (deactivates all)', async () => {
    mocks.findAnyActiveVsdcDevice.mockResolvedValue(fakeDevice);
    mocks.fetchCodeList.mockResolvedValue({
      success: true, message: 'ok', reference: 'r', httpStatus: 200,
      data: { clsList: [] },
    });
    const result = await syncZraCodes(deps, 'cron');
    expect(result.status).toBe('success');
    expect(result.fetched).toBe(0);
    expect(mocks.upsertCodes).not.toHaveBeenCalled();
  });
});

describe('syncAllZraReferenceData', () => {
  it('continues to next dataset when one fails', async () => {
    mocks.findAnyActiveVsdcDevice.mockResolvedValue(fakeDevice);
    mocks.fetchCodeList.mockResolvedValue({ success: false, message: 'oops', error: 'oops', reference: 'r', httpStatus: 500 });
    mocks.fetchNotices.mockResolvedValue({ success: true, message: 'ok', reference: 'r', httpStatus: 200, data: { noticeList: [] } });
    mocks.fetchItemClassList.mockResolvedValue({ success: true, message: 'ok', reference: 'r', httpStatus: 200, data: { itemClsList: [] } });

    const results = await syncAllZraReferenceData(deps, 'cron');
    expect(results).toHaveLength(3);
    expect(results.map((r) => r.datasetType)).toEqual(['codes', 'notices', 'item_classifications']);
    expect(results[0]!.status).toBe('failed');
    expect(results[1]!.status).toBe('success');
    expect(results[2]!.status).toBe('success');
  });
});

describe('getZraCodesByClass', () => {
  it('returns trimmed DTOs', async () => {
    mocks.listCodesByClass.mockResolvedValue([
      { code: 'A', codeName: 'A-EX', codeDescription: 'desc', sortOrder: 1, userDefinedCode1: '0' },
    ]);
    const opts = await getZraCodesByClass(ctx, deps, '04');
    expect(opts).toEqual([
      { code: 'A', name: 'A-EX', description: 'desc', sortOrder: 1, userDefinedCode1: '0' },
    ]);
  });
});

describe('getZraSyncStatus', () => {
  it('flags isStale when last success is older than 48h', async () => {
    const old = new Date(Date.now() - 72 * 60 * 60 * 1000);
    mocks.getLastSuccessfulSync.mockImplementation(async (_db, type) =>
      type === 'codes' ? { startedAt: old, status: 'success' } : null
    );
    mocks.listRecentSyncLogs.mockResolvedValue([]);
    const status = await getZraSyncStatus(ctx, deps);
    expect(status.isStale.codes).toBe(true);
    expect(status.isStale.notices).toBe(true); // null is treated as stale
  });
});
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `pnpm --filter @repo/api-services test -- zra-reference-data.service`
Expected: FAIL — `zra-reference-data.service` module does not exist.

- [ ] **Step 4: Write the service**

Create `packages/api-services/src/domains/invoicing/zra/zra-reference-data.service.ts`:

```typescript
import {
  createSyncLog,
  findAnyActiveVsdcDevice,
  getLastSuccessfulSync,
  listAllCodeClasses,
  listCodesByClass,
  listItemClassifications,
  listRecentNotices,
  listRecentSyncLogs,
  deactivateCodesNotIn,
  deactivateItemClassificationsNotIn,
  updateSyncLog,
  upsertCodeClasses,
  upsertCodes,
  upsertItemClassifications,
  upsertNotices,
} from "@repo/database/repositories";
import type { ServiceContext, ServiceDependencies } from "../../../core/context";
import { ServiceError } from "../../../core/errors";
import type {
  ZraCodeClassWire,
  ZraItemClassWire,
  ZraNoticeWire,
} from "./zra-api-client";
import { ZraApiClient } from "./zra-api-client";
import type {
  ZraCodeClassOption,
  ZraCodeOption,
  ZraItemClassificationOption,
  ZraNotice,
  ZraReferenceDatasetType,
  ZraReferenceSyncLog,
  ZraSyncResult,
  ZraSyncTrigger,
} from "./zra-reference-data.types";

const STALENESS_THRESHOLD_MS = 48 * 60 * 60 * 1000;

// ============================================
// Helpers
// ============================================

function buildClientFromDevice(device: {
  baseUrl: string | null;
  apiKey: string | null;
  tpin: string | null;
  branchId: string | null;
  deviceSerialNumber: string;
}): ZraApiClient {
  return new ZraApiClient({
    baseUrl: device.baseUrl ?? "https://api-sandbox.zra.org.zm/vsdc-api/v1",
    apiKey: device.apiKey ?? undefined,
    tpin: device.tpin ?? undefined,
    branchId: device.branchId ?? undefined,
    deviceSerialNumber: device.deviceSerialNumber,
  });
}

/** Parse YYYYMMDDHHmmss to Date. Returns null on bad input. */
function parseZraTimestamp(s: string): Date | null {
  if (!/^\d{14}$/.test(s)) return null;
  const y = +s.slice(0, 4);
  const m = +s.slice(4, 6) - 1;
  const d = +s.slice(6, 8);
  const h = +s.slice(8, 10);
  const min = +s.slice(10, 12);
  const sec = +s.slice(12, 14);
  return new Date(Date.UTC(y, m, d, h, min, sec));
}

async function noDeviceFailure(
  deps: ServiceDependencies,
  datasetType: ZraReferenceDatasetType,
  trigger: ZraSyncTrigger,
  userId?: string
): Promise<ZraSyncResult> {
  const startedAt = new Date();
  const log = await createSyncLog(deps.db, {
    datasetType,
    triggeredBy: trigger,
    triggeredByUserId: userId ?? null,
    status: "failed",
    startedAt,
    completedAt: new Date(),
    durationMs: 0,
    errorMessage: "No active VSDC device available for sync",
  });
  const msg = "No active VSDC device available for sync";
  if (trigger === "manual") {
    throw new ServiceError("PRECONDITION_FAILED", msg);
  }
  return {
    datasetType,
    status: "failed",
    fetched: 0,
    upserted: 0,
    deactivated: 0,
    durationMs: 0,
    errorMessage: msg,
    syncLogId: log.id,
  };
}

// ============================================
// Sync: codes
// ============================================

export async function syncZraCodes(
  deps: ServiceDependencies,
  trigger: ZraSyncTrigger,
  userId?: string
): Promise<ZraSyncResult> {
  const device = await findAnyActiveVsdcDevice(deps.db);
  if (!device) return noDeviceFailure(deps, "codes", trigger, userId);

  const startedAt = new Date();
  const log = await createSyncLog(deps.db, {
    datasetType: "codes",
    triggeredBy: trigger,
    triggeredByUserId: userId ?? null,
    vsdcDeviceId: device.id,
    status: "success",
    startedAt,
  });

  try {
    const client = buildClientFromDevice(device);
    const apiResult = await client.fetchCodeList();
    if (!apiResult.success) {
      const errorMessage = apiResult.error ?? apiResult.message;
      await updateSyncLog(deps.db, log.id, {
        status: "failed",
        errorMessage,
        completedAt: new Date(),
        durationMs: Date.now() - startedAt.getTime(),
      });
      if (trigger === "manual") {
        throw new ServiceError("BAD_GATEWAY", `ZRA code list fetch failed: ${errorMessage}`);
      }
      return {
        datasetType: "codes", status: "failed",
        fetched: 0, upserted: 0, deactivated: 0,
        durationMs: Date.now() - startedAt.getTime(),
        errorMessage, syncLogId: log.id,
      };
    }

    const clsList = (apiResult.data?.clsList ?? []) as ZraCodeClassWire[];
    const syncedAt = new Date();

    const classRows = clsList.map((c) => ({
      codeClass: c.cdCls,
      codeClassName: c.cdClsNm,
      codeClassDescription: c.cdClsDesc,
      userDefinedName1: c.userDfnNm1,
      userDefinedName2: c.userDfnNm2,
      userDefinedName3: c.userDfnNm3,
      useYn: c.useYn,
      syncedAt,
    }));
    await upsertCodeClasses(deps.db, classRows);

    let fetched = 0;
    let upserted = 0;
    let deactivated = 0;
    const perClassStats: Record<string, { fetched: number; upserted: number; deactivated: number }> = {};

    for (const cls of clsList) {
      const rows = cls.dtlList.map((d) => ({
        codeClass: cls.cdCls,
        code: d.cd,
        codeName: d.cdNm,
        codeDescription: d.cdDesc,
        sortOrder: d.srtOrd,
        userDefinedCode1: d.userDfnCd1,
        userDefinedCode2: d.userDfnCd2,
        userDefinedCode3: d.userDfnCd3,
        useYn: d.useYn,
        syncedAt,
      }));
      const u = await upsertCodes(deps.db, rows);
      const activeCodes = cls.dtlList.filter((d) => d.useYn === "Y").map((d) => d.cd);
      const d = await deactivateCodesNotIn(deps.db, cls.cdCls, activeCodes);
      perClassStats[cls.cdCls] = { fetched: rows.length, upserted: u, deactivated: d };
      fetched += rows.length;
      upserted += u;
      deactivated += d;
    }

    await updateSyncLog(deps.db, log.id, {
      status: "success",
      recordsFetched: fetched,
      recordsUpserted: upserted,
      recordsDeactivated: deactivated,
      perClassStats,
      completedAt: new Date(),
      durationMs: Date.now() - startedAt.getTime(),
    });

    return {
      datasetType: "codes", status: "success",
      fetched, upserted, deactivated,
      durationMs: Date.now() - startedAt.getTime(),
      syncLogId: log.id,
    };
  } catch (err) {
    if (err instanceof ServiceError) throw err;
    const errorMessage = err instanceof Error ? err.message : String(err);
    await updateSyncLog(deps.db, log.id, {
      status: "failed",
      errorMessage,
      completedAt: new Date(),
      durationMs: Date.now() - startedAt.getTime(),
    });
    if (trigger === "manual") {
      throw new ServiceError("INTERNAL_SERVER_ERROR", `ZRA code sync failed: ${errorMessage}`);
    }
    return {
      datasetType: "codes", status: "failed",
      fetched: 0, upserted: 0, deactivated: 0,
      durationMs: Date.now() - startedAt.getTime(),
      errorMessage, syncLogId: log.id,
    };
  }
}

// ============================================
// Sync: item classifications
// ============================================

export async function syncZraItemClassifications(
  deps: ServiceDependencies,
  trigger: ZraSyncTrigger,
  userId?: string
): Promise<ZraSyncResult> {
  const device = await findAnyActiveVsdcDevice(deps.db);
  if (!device) return noDeviceFailure(deps, "item_classifications", trigger, userId);

  const startedAt = new Date();
  const log = await createSyncLog(deps.db, {
    datasetType: "item_classifications",
    triggeredBy: trigger,
    triggeredByUserId: userId ?? null,
    vsdcDeviceId: device.id,
    status: "success",
    startedAt,
  });

  try {
    const client = buildClientFromDevice(device);
    const apiResult = await client.fetchItemClassList();
    if (!apiResult.success) {
      const errorMessage = apiResult.error ?? apiResult.message;
      await updateSyncLog(deps.db, log.id, {
        status: "failed", errorMessage,
        completedAt: new Date(),
        durationMs: Date.now() - startedAt.getTime(),
      });
      if (trigger === "manual") {
        throw new ServiceError("BAD_GATEWAY", `ZRA item class fetch failed: ${errorMessage}`);
      }
      return {
        datasetType: "item_classifications", status: "failed",
        fetched: 0, upserted: 0, deactivated: 0,
        durationMs: Date.now() - startedAt.getTime(),
        errorMessage, syncLogId: log.id,
      };
    }

    const itemClsList = (apiResult.data?.itemClsList ?? []) as ZraItemClassWire[];
    const syncedAt = new Date();

    const rows = itemClsList.map((i) => ({
      itemClassCode: i.itemClsCd,
      itemClassName: i.itemClsNm,
      itemClassLevel: i.itemClsLvl,
      taxTypeCode: i.taxTyCd,
      majorTargetYn: i.mjrTgYn,
      useYn: i.useYn,
      syncedAt,
    }));

    const upserted = await upsertItemClassifications(deps.db, rows);
    const activeCodes = itemClsList.filter((i) => i.useYn === "Y").map((i) => i.itemClsCd);
    const deactivated = await deactivateItemClassificationsNotIn(deps.db, activeCodes);

    await updateSyncLog(deps.db, log.id, {
      status: "success",
      recordsFetched: rows.length,
      recordsUpserted: upserted,
      recordsDeactivated: deactivated,
      completedAt: new Date(),
      durationMs: Date.now() - startedAt.getTime(),
    });

    return {
      datasetType: "item_classifications", status: "success",
      fetched: rows.length, upserted, deactivated,
      durationMs: Date.now() - startedAt.getTime(),
      syncLogId: log.id,
    };
  } catch (err) {
    if (err instanceof ServiceError) throw err;
    const errorMessage = err instanceof Error ? err.message : String(err);
    await updateSyncLog(deps.db, log.id, {
      status: "failed", errorMessage,
      completedAt: new Date(),
      durationMs: Date.now() - startedAt.getTime(),
    });
    if (trigger === "manual") {
      throw new ServiceError("INTERNAL_SERVER_ERROR", `ZRA item class sync failed: ${errorMessage}`);
    }
    return {
      datasetType: "item_classifications", status: "failed",
      fetched: 0, upserted: 0, deactivated: 0,
      durationMs: Date.now() - startedAt.getTime(),
      errorMessage, syncLogId: log.id,
    };
  }
}

// ============================================
// Sync: notices
// ============================================

export async function syncZraNotices(
  deps: ServiceDependencies,
  trigger: ZraSyncTrigger,
  userId?: string
): Promise<ZraSyncResult> {
  const device = await findAnyActiveVsdcDevice(deps.db);
  if (!device) return noDeviceFailure(deps, "notices", trigger, userId);

  const startedAt = new Date();
  const log = await createSyncLog(deps.db, {
    datasetType: "notices",
    triggeredBy: trigger,
    triggeredByUserId: userId ?? null,
    vsdcDeviceId: device.id,
    status: "success",
    startedAt,
  });

  try {
    const client = buildClientFromDevice(device);
    const apiResult = await client.fetchNotices();
    if (!apiResult.success) {
      const errorMessage = apiResult.error ?? apiResult.message;
      await updateSyncLog(deps.db, log.id, {
        status: "failed", errorMessage,
        completedAt: new Date(),
        durationMs: Date.now() - startedAt.getTime(),
      });
      if (trigger === "manual") {
        throw new ServiceError("BAD_GATEWAY", `ZRA notices fetch failed: ${errorMessage}`);
      }
      return {
        datasetType: "notices", status: "failed",
        fetched: 0, upserted: 0, deactivated: 0,
        durationMs: Date.now() - startedAt.getTime(),
        errorMessage, syncLogId: log.id,
      };
    }

    const noticeList = (apiResult.data?.noticeList ?? []) as ZraNoticeWire[];
    const syncedAt = new Date();

    const rows = noticeList
      .map((n) => {
        const registeredAt = parseZraTimestamp(n.regDt);
        if (!registeredAt) return null;
        return {
          noticeNumber: n.noticeNo,
          title: n.title,
          contents: n.cont,
          registeredAt,
          detailUrl: n.dtlUrl,
          syncedAt,
        };
      })
      .filter(<T>(v: T | null): v is T => v !== null);

    const upserted = await upsertNotices(deps.db, rows);

    await updateSyncLog(deps.db, log.id, {
      status: "success",
      recordsFetched: rows.length,
      recordsUpserted: upserted,
      recordsDeactivated: 0,
      completedAt: new Date(),
      durationMs: Date.now() - startedAt.getTime(),
    });

    return {
      datasetType: "notices", status: "success",
      fetched: rows.length, upserted, deactivated: 0,
      durationMs: Date.now() - startedAt.getTime(),
      syncLogId: log.id,
    };
  } catch (err) {
    if (err instanceof ServiceError) throw err;
    const errorMessage = err instanceof Error ? err.message : String(err);
    await updateSyncLog(deps.db, log.id, {
      status: "failed", errorMessage,
      completedAt: new Date(),
      durationMs: Date.now() - startedAt.getTime(),
    });
    if (trigger === "manual") {
      throw new ServiceError("INTERNAL_SERVER_ERROR", `ZRA notices sync failed: ${errorMessage}`);
    }
    return {
      datasetType: "notices", status: "failed",
      fetched: 0, upserted: 0, deactivated: 0,
      durationMs: Date.now() - startedAt.getTime(),
      errorMessage, syncLogId: log.id,
    };
  }
}

// ============================================
// Sync: all (sequential, fault-tolerant)
// ============================================

export async function syncAllZraReferenceData(
  deps: ServiceDependencies,
  trigger: ZraSyncTrigger,
  userId?: string
): Promise<ZraSyncResult[]> {
  const results: ZraSyncResult[] = [];
  for (const fn of [
    () => syncZraCodes(deps, trigger, userId),
    () => syncZraNotices(deps, trigger, userId),
    () => syncZraItemClassifications(deps, trigger, userId),
  ]) {
    try {
      results.push(await fn());
    } catch (err) {
      // Defensive — individual sync functions already swallow on cron;
      // this catches anything they rethrow.
      const errorMessage = err instanceof Error ? err.message : String(err);
      results.push({
        datasetType: "codes", // placeholder — should never hit in practice
        status: "failed",
        fetched: 0, upserted: 0, deactivated: 0, durationMs: 0,
        errorMessage, syncLogId: null,
      });
    }
  }
  return results;
}

// ============================================
// Read functions (dropdown-facing)
// ============================================

export async function getZraCodesByClass(
  _ctx: ServiceContext,
  deps: ServiceDependencies,
  codeClass: string
): Promise<ZraCodeOption[]> {
  const rows = await listCodesByClass(deps.db, codeClass);
  return rows.map((r) => ({
    code: r.code,
    name: r.codeName,
    description: r.codeDescription,
    sortOrder: r.sortOrder,
    userDefinedCode1: r.userDefinedCode1,
  }));
}

export async function getZraCodeClasses(
  _ctx: ServiceContext,
  deps: ServiceDependencies
): Promise<ZraCodeClassOption[]> {
  const rows = await listAllCodeClasses(deps.db);
  return rows.map((r) => ({ codeClass: r.codeClass, name: r.codeClassName }));
}

export async function getZraItemClassifications(
  _ctx: ServiceContext,
  deps: ServiceDependencies,
  opts: { level?: number; search?: string; limit?: number } = {}
): Promise<ZraItemClassificationOption[]> {
  const rows = await listItemClassifications(deps.db, {
    level: opts.level,
    search: opts.search,
    limit: opts.limit ?? 100,
  });
  return rows.map((r) => ({
    itemClassCode: r.itemClassCode,
    itemClassName: r.itemClassName,
    itemClassLevel: r.itemClassLevel,
    taxTypeCode: r.taxTypeCode,
  }));
}

export async function getZraNotices(
  _ctx: ServiceContext,
  deps: ServiceDependencies,
  limit = 20
): Promise<ZraNotice[]> {
  return listRecentNotices(deps.db, limit);
}

export async function getZraSyncStatus(
  _ctx: ServiceContext,
  deps: ServiceDependencies
): Promise<{
  lastSync: Record<ZraReferenceDatasetType, { at: Date; status: string } | null>;
  recentLogs: ZraReferenceSyncLog[];
  isStale: Record<ZraReferenceDatasetType, boolean>;
}> {
  const types: ZraReferenceDatasetType[] = ["codes", "item_classifications", "notices"];
  const lastSync: Record<ZraReferenceDatasetType, { at: Date; status: string } | null> = {
    codes: null, item_classifications: null, notices: null,
  };
  const isStale: Record<ZraReferenceDatasetType, boolean> = {
    codes: true, item_classifications: true, notices: true,
  };
  const now = Date.now();
  for (const t of types) {
    const log = await getLastSuccessfulSync(deps.db, t);
    if (log) {
      lastSync[t] = { at: log.startedAt, status: log.status };
      isStale[t] = now - log.startedAt.getTime() > STALENESS_THRESHOLD_MS;
    }
  }
  const recentLogs = await listRecentSyncLogs(deps.db, 20);
  return { lastSync, recentLogs, isStale };
}
```

- [ ] **Step 5: Export from invoicing domain barrel**

Edit `packages/api-services/src/domains/invoicing/index.ts`. Find a sensible location near the existing ZRA exports and add:

```typescript
export {
    getZraCodeClasses,
    getZraCodesByClass,
    getZraItemClassifications,
    getZraNotices,
    getZraSyncStatus,
    syncAllZraReferenceData,
    syncZraCodes,
    syncZraItemClassifications,
    syncZraNotices,
} from "./zra/zra-reference-data.service";
export type {
    ZraCodeClassOption,
    ZraCodeOption,
    ZraItemClassificationOption,
    ZraReferenceDatasetType,
    ZraSyncResult,
    ZraSyncTrigger,
} from "./zra/zra-reference-data.types";
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `pnpm --filter @repo/api-services test -- zra-reference-data.service`
Expected: all assertions PASS.

- [ ] **Step 7: Commit**

```bash
git add packages/api-services/src/domains/invoicing/zra/zra-reference-data.service.ts \
        packages/api-services/src/domains/invoicing/zra/zra-reference-data.types.ts \
        packages/api-services/src/domains/invoicing/zra/__tests__/zra-reference-data.service.test.ts \
        packages/api-services/src/domains/invoicing/index.ts
git commit -m "feat(api-services): add ZRA reference data sync + read service"
```

---

## Task 5: HTTP Routes — Reference Data Endpoints

**Files:**
- Create: `apps/api-invoicing/src/routes/invoicing/zra-reference-data/routes.ts`
- Create: `apps/api-invoicing/src/routes/invoicing/zra-reference-data/handlers.ts`
- Create: `apps/api-invoicing/src/routes/invoicing/zra-reference-data/index.ts`
- Modify: `apps/api-invoicing/src/routes/invoicing/index.ts` (or whichever file composes invoicing routers — find it via the existing zra router registration)

- [ ] **Step 1: Verify how existing invoicing routers are mounted**

Run: `grep -r "zraRouter\|zra/index" apps/api-invoicing/src/routes/ --include="*.ts" -l`
Expected: identifies the file that imports and mounts the existing zra router. Use the same composition pattern when wiring the new router.

- [ ] **Step 2: Create routes.ts**

```typescript
import { createRoute, z } from "@hono/zod-openapi";
import {
  errorResponseSchema,
  successResponseSchema,
} from "@repo/backend/config";
import {
  requireAuth,
  requireBackoffice,
  requireOrg,
} from "@repo/backend/middleware/auth";
import * as HttpStatusCodes from "stoker/http-status-codes";
import { jsonContent, jsonContentRequired } from "stoker/openapi/helpers";

const tags = ["Invoicing - ZRA Reference Data"];

// ----- Reads -----

export const listCodeClassesRoute = createRoute({
  tags,
  method: "get",
  path: "/invoicing/zra/code-classes",
  summary: "List all ZRA code classes",
  middleware: [requireAuth, requireOrg],
  responses: {
    [HttpStatusCodes.OK]: jsonContent(successResponseSchema, "Code classes retrieved"),
    [HttpStatusCodes.UNAUTHORIZED]: jsonContent(errorResponseSchema, "Unauthorized"),
    [HttpStatusCodes.INTERNAL_SERVER_ERROR]: jsonContent(errorResponseSchema, "Internal Server Error"),
  },
});

export const listCodesByClassRoute = createRoute({
  tags,
  method: "get",
  path: "/invoicing/zra/codes/{codeClass}",
  summary: "List ZRA codes for a given class",
  request: { params: z.object({ codeClass: z.string().min(1) }) },
  middleware: [requireAuth, requireOrg],
  responses: {
    [HttpStatusCodes.OK]: jsonContent(successResponseSchema, "Codes retrieved"),
    [HttpStatusCodes.UNAUTHORIZED]: jsonContent(errorResponseSchema, "Unauthorized"),
    [HttpStatusCodes.INTERNAL_SERVER_ERROR]: jsonContent(errorResponseSchema, "Internal Server Error"),
  },
});

export const listItemClassificationsRoute = createRoute({
  tags,
  method: "get",
  path: "/invoicing/zra/item-classifications",
  summary: "List ZRA item classifications (filterable by level and search)",
  request: {
    query: z.object({
      level: z.coerce.number().int().min(1).max(5).optional(),
      search: z.string().min(1).optional(),
      limit: z.coerce.number().int().min(1).max(500).optional(),
    }),
  },
  middleware: [requireAuth, requireOrg],
  responses: {
    [HttpStatusCodes.OK]: jsonContent(successResponseSchema, "Item classifications retrieved"),
    [HttpStatusCodes.UNAUTHORIZED]: jsonContent(errorResponseSchema, "Unauthorized"),
    [HttpStatusCodes.INTERNAL_SERVER_ERROR]: jsonContent(errorResponseSchema, "Internal Server Error"),
  },
});

export const listNoticesRoute = createRoute({
  tags,
  method: "get",
  path: "/invoicing/zra/notices",
  summary: "List recent ZRA notices",
  request: { query: z.object({ limit: z.coerce.number().int().min(1).max(100).optional() }) },
  middleware: [requireAuth, requireOrg],
  responses: {
    [HttpStatusCodes.OK]: jsonContent(successResponseSchema, "Notices retrieved"),
    [HttpStatusCodes.UNAUTHORIZED]: jsonContent(errorResponseSchema, "Unauthorized"),
    [HttpStatusCodes.INTERNAL_SERVER_ERROR]: jsonContent(errorResponseSchema, "Internal Server Error"),
  },
});

// ----- Ops -----

export const getSyncStatusRoute = createRoute({
  tags,
  method: "get",
  path: "/invoicing/zra/reference-data/sync-status",
  summary: "Get last sync status per dataset (backoffice only)",
  middleware: [requireAuth, requireBackoffice],
  responses: {
    [HttpStatusCodes.OK]: jsonContent(successResponseSchema, "Sync status retrieved"),
    [HttpStatusCodes.UNAUTHORIZED]: jsonContent(errorResponseSchema, "Unauthorized"),
    [HttpStatusCodes.FORBIDDEN]: jsonContent(errorResponseSchema, "Forbidden"),
    [HttpStatusCodes.INTERNAL_SERVER_ERROR]: jsonContent(errorResponseSchema, "Internal Server Error"),
  },
});

export const triggerSyncRoute = createRoute({
  tags,
  method: "post",
  path: "/invoicing/zra/reference-data/sync",
  summary: "Manually trigger ZRA reference data sync (backoffice only)",
  request: {
    body: jsonContentRequired(
      z.object({
        datasets: z
          .array(z.enum(["codes", "item_classifications", "notices"]))
          .optional(),
      }),
      "Optional list of datasets to sync (omit to sync all)"
    ),
  },
  middleware: [requireAuth, requireBackoffice],
  responses: {
    [HttpStatusCodes.ACCEPTED]: jsonContent(successResponseSchema, "Sync triggered"),
    [HttpStatusCodes.UNAUTHORIZED]: jsonContent(errorResponseSchema, "Unauthorized"),
    [HttpStatusCodes.FORBIDDEN]: jsonContent(errorResponseSchema, "Forbidden"),
    [HttpStatusCodes.PRECONDITION_FAILED]: jsonContent(errorResponseSchema, "No active VSDC device"),
    [HttpStatusCodes.BAD_GATEWAY]: jsonContent(errorResponseSchema, "ZRA upstream error"),
    [HttpStatusCodes.INTERNAL_SERVER_ERROR]: jsonContent(errorResponseSchema, "Internal Server Error"),
  },
});

export type ListCodeClassesRoute = typeof listCodeClassesRoute;
export type ListCodesByClassRoute = typeof listCodesByClassRoute;
export type ListItemClassificationsRoute = typeof listItemClassificationsRoute;
export type ListNoticesRoute = typeof listNoticesRoute;
export type GetSyncStatusRoute = typeof getSyncStatusRoute;
export type TriggerSyncRoute = typeof triggerSyncRoute;
```

> **Worker note:** If `requireBackoffice` isn't exported from `@repo/backend/middleware/auth`, check `packages/backend/src/core/middleware/auth.ts` line 603 for the actual export — it exists. If the re-export path differs, update the import accordingly.

- [ ] **Step 3: Create handlers.ts**

```typescript
import {
  getZraCodeClasses,
  getZraCodesByClass,
  getZraItemClassifications,
  getZraNotices,
  getZraSyncStatus,
  syncAllZraReferenceData,
  syncZraCodes,
  syncZraItemClassifications,
  syncZraNotices,
  type ZraReferenceDatasetType,
} from "@repo/api-services/domains/invoicing";
import {
  buildServiceContext,
  buildServiceDependencies,
  handleServiceError,
} from "@repo/backend/context";
import type { AppRouteHandler } from "@repo/backend/types";
import * as HttpStatusCodes from "stoker/http-status-codes";
import type {
  GetSyncStatusRoute,
  ListCodeClassesRoute,
  ListCodesByClassRoute,
  ListItemClassificationsRoute,
  ListNoticesRoute,
  TriggerSyncRoute,
} from "./routes";

export const listCodeClassesHandler: AppRouteHandler<ListCodeClassesRoute> = async (c) => {
  try {
    const ctx = buildServiceContext(c);
    const deps = buildServiceDependencies(c);
    const data = await getZraCodeClasses(ctx, deps);
    return c.json({ success: true, data, message: "Code classes retrieved" }, HttpStatusCodes.OK);
  } catch (e) {
    return handleServiceError(c, e, "Failed to list code classes");
  }
};

export const listCodesByClassHandler: AppRouteHandler<ListCodesByClassRoute> = async (c) => {
  try {
    const ctx = buildServiceContext(c);
    const deps = buildServiceDependencies(c);
    const { codeClass } = c.req.valid("param");
    const data = await getZraCodesByClass(ctx, deps, codeClass);
    return c.json({ success: true, data, message: "Codes retrieved" }, HttpStatusCodes.OK);
  } catch (e) {
    return handleServiceError(c, e, "Failed to list codes");
  }
};

export const listItemClassificationsHandler: AppRouteHandler<ListItemClassificationsRoute> = async (c) => {
  try {
    const ctx = buildServiceContext(c);
    const deps = buildServiceDependencies(c);
    const q = c.req.valid("query");
    const data = await getZraItemClassifications(ctx, deps, {
      level: q.level, search: q.search, limit: q.limit,
    });
    return c.json({ success: true, data, message: "Item classifications retrieved" }, HttpStatusCodes.OK);
  } catch (e) {
    return handleServiceError(c, e, "Failed to list item classifications");
  }
};

export const listNoticesHandler: AppRouteHandler<ListNoticesRoute> = async (c) => {
  try {
    const ctx = buildServiceContext(c);
    const deps = buildServiceDependencies(c);
    const { limit } = c.req.valid("query");
    const data = await getZraNotices(ctx, deps, limit);
    return c.json({ success: true, data, message: "Notices retrieved" }, HttpStatusCodes.OK);
  } catch (e) {
    return handleServiceError(c, e, "Failed to list notices");
  }
};

export const getSyncStatusHandler: AppRouteHandler<GetSyncStatusRoute> = async (c) => {
  try {
    const ctx = buildServiceContext(c);
    const deps = buildServiceDependencies(c);
    const data = await getZraSyncStatus(ctx, deps);
    return c.json({ success: true, data, message: "Sync status retrieved" }, HttpStatusCodes.OK);
  } catch (e) {
    return handleServiceError(c, e, "Failed to get sync status");
  }
};

export const triggerSyncHandler: AppRouteHandler<TriggerSyncRoute> = async (c) => {
  try {
    const ctx = buildServiceContext(c);
    const deps = buildServiceDependencies(c);
    const body = await c.req.json<{ datasets?: ZraReferenceDatasetType[] }>();
    const userId = ctx.userId ?? undefined;

    // Run in background — return 202 immediately
    const task =
      !body.datasets || body.datasets.length === 0
        ? syncAllZraReferenceData(deps, "manual", userId)
        : Promise.all(
            body.datasets.map((d) =>
              d === "codes"
                ? syncZraCodes(deps, "manual", userId)
                : d === "item_classifications"
                ? syncZraItemClassifications(deps, "manual", userId)
                : syncZraNotices(deps, "manual", userId)
            )
          );

    c.executionCtx.waitUntil(
      task.catch((err) => {
        console.error("[ZRA Sync] manual trigger error", err);
      })
    );

    return c.json(
      { success: true, data: { triggered: true }, message: "Sync triggered" },
      HttpStatusCodes.ACCEPTED
    );
  } catch (e) {
    return handleServiceError(c, e, "Failed to trigger sync");
  }
};
```

- [ ] **Step 4: Create index.ts (router)**

```typescript
import { createRouter } from "@repo/backend/create-app";
import {
  getSyncStatusRoute,
  listCodeClassesRoute,
  listCodesByClassRoute,
  listItemClassificationsRoute,
  listNoticesRoute,
  triggerSyncRoute,
} from "./routes";

// eslint-disable-next-line @typescript-eslint/no-explicit-any
const lazy = (name: string) => async (c: any) => {
  const mod = await import("./handlers");
  // @ts-expect-error dynamic handler lookup
  return modname;
};

const zraReferenceDataRouter = createRouter()
  .openapi(listCodeClassesRoute, lazy("listCodeClassesHandler"))
  .openapi(listCodesByClassRoute, lazy("listCodesByClassHandler"))
  .openapi(listItemClassificationsRoute, lazy("listItemClassificationsHandler"))
  .openapi(listNoticesRoute, lazy("listNoticesHandler"))
  .openapi(getSyncStatusRoute, lazy("getSyncStatusHandler"))
  .openapi(triggerSyncRoute, lazy("triggerSyncHandler"));

export default zraReferenceDataRouter;
```

- [ ] **Step 5: Mount the router**

Open the file identified in Step 1 (the one that imports the existing `zraRouter`). Add an import:

```typescript
import zraReferenceDataRouter from "./zra-reference-data";
```

And mount it on the same router chain as `zraRouter`, e.g.:

```typescript
.route("/", zraReferenceDataRouter)
```

(use the exact same `.route(...)` pattern already used for `zraRouter`).

- [ ] **Step 6: Typecheck the worker**

Run: `pnpm --filter @repo/api-invoicing typecheck` (or whatever your invoicing app's package name is — check `apps/api-invoicing/package.json`)
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add apps/api-invoicing/src/routes/invoicing/zra-reference-data/ \
        apps/api-invoicing/src/routes/invoicing/index.ts
git commit -m "feat(api-invoicing): expose ZRA reference data endpoints (reads + manual sync)"
```

---

## Task 6: Cron Wiring — Daily Reference Sync Job

**Files:**
- Create: `apps/api-jobs/src/jobs/zra-reference-sync.ts`
- Modify: `apps/api-jobs/src/jobs/scheduler.ts`

- [ ] **Step 1: Create the job module**

`apps/api-jobs/src/jobs/zra-reference-sync.ts`:

```typescript
/**
 * ZRA Reference Data Sync Job
 *
 * Runs once daily (00:00 UTC). Pulls and caches:
 *  - Standard codes (/code/selectCodes)
 *  - Item classifications (/itemClass/selectItemsClass)
 *  - Notices (/notices/selectNotices)
 *
 * Uses the first active+initialized VSDC device available on the platform.
 * Failures in one dataset do not block the others.
 */

import { syncAllZraReferenceData } from "@repo/api-services/domains/invoicing";
import type { Env } from "@repo/api-services";
import type { DatabaseClient } from "@repo/database";

export async function runZraReferenceSyncJob(
  db: DatabaseClient,
  _env: Env
): Promise<{ results: Array<{ datasetType: string; status: string; upserted: number }> }> {
  console.log("[ZRA Sync] starting daily reference data sync");
  const startedAt = Date.now();

  const results = await syncAllZraReferenceData({ db }, "cron");

  console.log("[ZRA Sync] complete", {
    durationMs: Date.now() - startedAt,
    results: results.map((r) => ({
      datasetType: r.datasetType,
      status: r.status,
      fetched: r.fetched,
      upserted: r.upserted,
      deactivated: r.deactivated,
      error: r.errorMessage,
    })),
  });

  return {
    results: results.map((r) => ({
      datasetType: r.datasetType,
      status: r.status,
      upserted: r.upserted,
    })),
  };
}
```

- [ ] **Step 2: Wire into the scheduler**

Edit `apps/api-jobs/src/jobs/scheduler.ts`. Add the import alongside the other job imports near the top of the file:

```typescript
import { runZraReferenceSyncJob } from "./zra-reference-sync";
```

**a.** In the `runAllDevJobs` job list, add (in the array, alongside the active `outbox poll` entry):

```typescript
{
  name: "zra reference data sync",
  fn: () => runZraReferenceSyncJob(db, env),
},
```

**b.** In `runScheduledJobs`, locate the existing `case "0 0 * * *":` block (which currently runs `runWelcomeCreditExpiryJob`). Change it from a sequential single call to a `Promise.all` so the new job runs alongside, mirroring how `case "0 7 * * *":` already does it:

Replace:

```typescript
      case "0 0 * * *":
        console.log("[SCHEDULER] Running welcome credit expiry job");
        await runWelcomeCreditExpiryJob(db, env);
        break;
```

With:

```typescript
      case "0 0 * * *":
        console.log("[SCHEDULER] Running daily jobs (welcome credit expiry + ZRA reference sync)");
        await Promise.all([
          runWelcomeCreditExpiryJob(db, env),
          runZraReferenceSyncJob(db, env),
        ]);
        break;
```

`runZraReferenceSyncJob` already swallows its own errors internally (via the service-layer cron-trigger behavior) so it cannot reject the `Promise.all`.

- [ ] **Step 3: Typecheck**

Run: `pnpm --filter bumara-api-jobs typecheck` (or check the actual package name in `apps/api-jobs/package.json`)
Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add apps/api-jobs/src/jobs/zra-reference-sync.ts \
        apps/api-jobs/src/jobs/scheduler.ts
git commit -m "feat(api-jobs): daily ZRA reference data sync cron job"
```

---

## Task 7: Run Migration + Smoke Test

**Files:** none (operational task)

- [ ] **Step 1: Apply migration to local dev DB**

Run: `pnpm --filter @repo/database db:migrate`
Expected: migration applies cleanly, output mentions creating `zra_code_classes`, `zra_codes`, `zra_item_classifications`, `zra_notices`, `zra_reference_sync_log`.

- [ ] **Step 2: Verify tables exist**

Run: `pnpm --filter @repo/database db:studio` (opens browser), or alternatively connect via your usual psql/Drizzle Studio flow.
Expected: all five tables visible, empty.

- [ ] **Step 3: Manual smoke test — backoffice trigger (if a backoffice user + active VSDC device exist)**

Pre-req: at least one VSDC device with `is_initialized=true` and `device_status='active'` exists in the dev DB. If none, skip this step and rely on Task 4 service tests as the verification surface.

Start the invoicing worker (`pnpm --filter @repo/api-invoicing dev` or your equivalent) and POST as an authenticated backoffice user:

```
POST /invoicing/zra/reference-data/sync
Authorization: Bearer <backoffice token>
Content-Type: application/json
{}
```

Expected: 202 response with `{ success: true, data: { triggered: true } }`. Within ~30s the `zra_reference_sync_log` table should contain three new rows (one per dataset) with `status='success'` (or `status='failed'` if the ZRA sandbox is unreachable — that's OK for smoke testing, the pipeline ran).

- [ ] **Step 4: Verify reads**

```
GET /invoicing/zra/code-classes
GET /invoicing/zra/codes/04
GET /invoicing/zra/notices?limit=5
```

Expected: 200 with `data` arrays populated (or empty if sandbox returned no data).

- [ ] **Step 5: Commit any operational fixes uncovered during smoke test**

If smoke test exposed any bug (mistyped field, missing import), fix and commit each fix as its own commit with a clear message. No commit needed if smoke test passes cleanly.

---

## Self-Review Checklist (for the plan author)

Spec coverage:
- Schema (4 tables + sync log + jsonb perClassStats): Task 1 ✓
- API client methods + wire types: Task 3 ✓
- Repository (upsert, deactivate, list, sync log helpers, `findAnyActiveVsdcDevice`): Task 2 ✓
- Service (sync + read functions, error handling per trigger, perClassStats): Task 4 ✓
- HTTP routes (4 reads + 2 ops with `requireBackoffice`): Task 5 ✓
- Cron wiring on existing daily trigger: Task 6 ✓
- Repository tests with sample fixture from spec: Task 2 ✓
- Service tests with mocked client + sample fixture: Task 4 ✓
- Manual smoke test: Task 7 ✓

Frontend dropdown components and backoffice UI page (mentioned in spec §Frontend) are intentionally **deferred** to a separate plan — they depend on the API endpoints landing first and on UI design decisions. This plan ships the backend building blocks; frontend follow-up is called out in the spec's "Open follow-ups" section.
