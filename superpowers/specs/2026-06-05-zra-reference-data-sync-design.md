---
title: "ZRA Reference Data Sync — Design"
description: "Design: cache ZRA reference data — standard codes, item classifications, notices — so ERP dropdowns match ZRA, synced daily."
---

**Date:** 2026-06-05
**Status:** Approved
**Owner:** Invoicing domain

## Summary

Pull and cache ZRA's reference data (standard codes, item classifications, notices) into our database so ERP dropdowns (currency, country, tax type, unit of measurement, item classification, etc.) match ZRA exactly. Sync runs daily via the existing `api-jobs` Cloudflare Worker, with a backoffice-gated manual trigger for ad-hoc refresh.

## Motivation

The current ZRA Smart Invoice integration sends sales/purchase data referencing ZRA codes (tax types `A`/`B`/`C1`/`D`, currency codes, payment types, etc.), but the application's dropdown options are hardcoded. This causes drift when ZRA updates their code list, blocks proper item classification (`itemClsCd` is required on every line item), and makes UI dropdowns inconsistent across organizations. ZRA exposes the data via three documented endpoints; we just need to cache it.

## Scope

**In scope:**
- Pull and cache: standard codes (`/code/selectCodes`), item classifications (`/itemClass/selectItemsClass`), notices (`/notices/selectNotices`).
- Schedule daily sync via existing cron worker.
- Backoffice manual sync endpoint and UI.
- Reusable frontend dropdown components for codes and item classifications.

**Out of scope:**
- Branch list (`/branch/selectBhfList`) — org-specific, not global reference data.
- Wiring every existing dropdown in the codebase to use ZRA data — separate plan per surface.
- Multi-language support for code names (only English `cdNm` is stored).
- Inferring item classification hierarchy from code prefixes.

## Approach

**Storage:** Dedicated global tables (no `organizationId` — reference data is shared across all orgs). Full-snapshot replacement each sync (not incremental) — simpler, no drift risk, datasets are small enough.

**Sync trigger:** Scheduled cron (existing daily `0 0 * * *` trigger in `api-jobs`) + manual button in backoffice (gated by `requireBackoffice`).

**Auth source for ZRA calls:** First active+initialized VSDC device on the platform (any organization). ZRA codes are global, so any valid device works.

**Deactivation model:** Never delete. Rows ZRA stops returning get `useYn='N'`; rows that reappear get `useYn='Y'` restored. Preserves historical data and supports rollback by re-syncing.

## Database schema

New schema file: `packages/database/src/schema/zra/zra-reference-data.ts`.

### `zra_code_classes`

Lookup table for code class metadata (one row per class).

| Column | Type | Notes |
|---|---|---|
| `code_class` | text PK | e.g. `"04"` |
| `code_class_name` | text NOT NULL | e.g. `"Tax Type"` |
| `code_class_description` | text NULL | from `cdClsDesc` |
| `user_defined_name_1` | text NULL | class-level label for `userDfnCd1` (e.g. `"Tax Rate"`) |
| `user_defined_name_2` | text NULL | |
| `user_defined_name_3` | text NULL | |
| `use_yn` | text NOT NULL DEFAULT `'Y'` | |
| `synced_at` | timestamp NOT NULL | |
| ...`timestamps` | | created_at / updated_at |

### `zra_codes`

Flat table of all code detail rows, grouped by `codeClass`.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | defaultRandom |
| `code_class` | text NOT NULL | FK → `zra_code_classes.code_class` |
| `code` | text NOT NULL | from `cd` |
| `code_name` | text NOT NULL | from `cdNm` (English) |
| `code_description` | text NULL | from `cdDesc` |
| `sort_order` | integer NOT NULL DEFAULT 0 | from `srtOrd` |
| `user_defined_code_1` | text NULL | class-specific meaning (e.g. tax rate `"18"` for class `"04"`) |
| `user_defined_code_2` | text NULL | |
| `user_defined_code_3` | text NULL | |
| `use_yn` | text NOT NULL DEFAULT `'Y'` | flipped to `'N'` if not in latest sync |
| `synced_at` | timestamp NOT NULL | |
| ...`timestamps` | | |

Indexes:
- `UNIQUE (code_class, code)`
- `INDEX (code_class, use_yn, sort_order)` — for dropdown reads

### `zra_item_classifications`

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | defaultRandom |
| `item_class_code` | text NOT NULL | from `itemClsCd` (e.g. `"14111400"`) |
| `item_class_name` | text NOT NULL | from `itemClsNm` |
| `item_class_level` | integer NOT NULL | from `itemClsLvl` (1–5, leaves at 5) |
| `tax_type_code` | text NULL | from `taxTyCd` (e.g. `"B"`) — applicable tax category for items in this class |
| `major_target_yn` | text NULL | from `mjrTgYn` (preserved as-is) |
| `use_yn` | text NOT NULL DEFAULT `'Y'` | |
| `synced_at` | timestamp NOT NULL | |
| ...`timestamps` | | |

Indexes:
- `UNIQUE (item_class_code)`
- `INDEX (item_class_level, use_yn)` — for leaf-only dropdown queries
- `INDEX USING gin (to_tsvector('english', item_class_name))` — for search

### `zra_notices`

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | defaultRandom |
| `notice_number` | integer NOT NULL | from `noticeNo` |
| `title` | text NOT NULL | |
| `contents` | text NOT NULL | from `cont` |
| `registered_at` | timestamp NOT NULL | from `regDt` (parsed from YYYYMMDDHHmmss) |
| `detail_url` | text NULL | from `dtlUrl` |
| `synced_at` | timestamp NOT NULL | |
| ...`timestamps` | | |

Indexes:
- `UNIQUE (notice_number)`
- `INDEX (registered_at DESC)` — for "recent notices" reads

### `zra_reference_sync_log`

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | defaultRandom |
| `dataset_type` | text NOT NULL | `'codes'` \| `'item_classifications'` \| `'notices'` |
| `triggered_by` | text NOT NULL | `'cron'` \| `'manual'` |
| `triggered_by_user_id` | text NULL | Clerk user ID when manual |
| `vsdc_device_id` | uuid NULL | FK → `zra_vsdc_devices.id` (nullable; null = no device available) |
| `status` | text NOT NULL | `'success'` \| `'failed'` \| `'partial'` |
| `records_fetched` | integer NOT NULL DEFAULT 0 | |
| `records_upserted` | integer NOT NULL DEFAULT 0 | |
| `records_deactivated` | integer NOT NULL DEFAULT 0 | |
| `per_class_stats` | jsonb NULL | only for `dataset_type='codes'`: `{ "04": { fetched, upserted, deactivated }, ... }` |
| `error_message` | text NULL | |
| `duration_ms` | integer NULL | |
| `started_at` | timestamp NOT NULL | |
| `completed_at` | timestamp NULL | null while in-flight |
| ...`timestamps` | | |

Indexes:
- `INDEX (dataset_type, started_at DESC)`
- `INDEX (status, started_at DESC)`

### Migration

New Drizzle migration generated via existing `pnpm db:generate` flow. Migration runs the table creates in a single transaction. No data backfill required.

## API client additions

File: `packages/api-services/src/domains/invoicing/zra/zra-api-client.ts`.

Add to `ZRA_API_ENDPOINTS`:
```ts
CODE_LIST: "/code/selectCodes",            // already declared
ITEM_CLASS_LIST: "/itemClass/selectItemsClass",
NOTICES: "/notices/selectNotices",
```

Add wire types:
```ts
interface ZraCodeDetailWire {
  cd: string; cdNm: string; cdDesc: string | null;
  useYn: "Y" | "N"; srtOrd: number;
  userDfnCd1: string | null; userDfnCd2: string | null; userDfnCd3: string | null;
}
interface ZraCodeClassWire {
  cdCls: string; cdClsNm: string; cdClsDesc: string | null; useYn: "Y" | "N";
  userDfnNm1: string | null; userDfnNm2: string | null; userDfnNm3: string | null;
  dtlList: ZraCodeDetailWire[];
}
interface ZraItemClassWire {
  itemClsCd: string; itemClsNm: string; itemClsLvl: number;
  taxTyCd: string | null; mjrTgYn: "Y" | "N" | null; useYn: "Y" | "N";
}
interface ZraNoticeWire {
  noticeNo: number; title: string; cont: string;
  regDt: string; dtlUrl: string | null;
}
```

Add three methods on `ZraApiClient` (all reuse `makeRequest`, all default `lastReqDt` to `"20180101000000"` for full snapshot):

```ts
fetchCodeList(lastReqDt?: string): Promise<ZraApiResponse<{ clsList: ZraCodeClassWire[] }>>
fetchItemClassList(lastReqDt?: string): Promise<ZraApiResponse<{ itemClsList: ZraItemClassWire[] }>>
fetchNotices(lastReqDt?: string): Promise<ZraApiResponse<{ noticeList: ZraNoticeWire[] }>>
```

Each builds request body `{ tpin: this.config.tpin, bhfId: this.config.branchId, lastReqDt }`.

## Repository

New file: `packages/database/src/repositories/zra-reference-data.ts`.

```ts
// Code classes + codes
upsertCodeClasses(db, classes: ZraCodeClassInsert[]): Promise<number>
upsertCodes(db, codes: ZraCodeInsert[]): Promise<number>
deactivateCodesNotIn(db, codeClass: string, activeCodes: string[]): Promise<number>
listCodesByClass(db, codeClass: string): Promise<ZraCode[]>           // useYn='Y' only
listAllCodeClasses(db): Promise<ZraCodeClass[]>                       // useYn='Y' only

// Item classifications
upsertItemClassifications(db, items: ZraItemClassificationInsert[]): Promise<number>
deactivateItemClassificationsNotIn(db, activeCodes: string[]): Promise<number>
listItemClassifications(db, opts?: {
  level?: number; search?: string; limit?: number
}): Promise<ZraItemClassification[]>

// Notices
upsertNotices(db, notices: ZraNoticeInsert[]): Promise<number>
listRecentNotices(db, limit: number): Promise<ZraNotice[]>

// Sync log
createSyncLog(db, log: ZraReferenceSyncLogInsert): Promise<ZraReferenceSyncLog>
updateSyncLog(db, id: string, patch: Partial<ZraReferenceSyncLog>): Promise<ZraReferenceSyncLog>
listRecentSyncLogs(db, limit: number): Promise<ZraReferenceSyncLog[]>
getLastSuccessfulSync(db, datasetType: string): Promise<ZraReferenceSyncLog | null>

// Helper added to existing zra repo
findAnyActiveVsdcDevice(db): Promise<ZraVsdcDevice | null>  // first device WHERE is_initialized=true AND device_status='active', ordered by created_at ASC for determinism
```

**Upsert strategy:** `onConflictDoUpdate` keyed on the natural unique constraint. Always sets `syncedAt = now()` and `useYn = 'Y'` on conflict (restoring previously-deactivated rows that ZRA started returning again).

**Deactivation:** `UPDATE ... SET use_yn='N' WHERE <scope> AND <natural_key> NOT IN (<activeBatch>)`. For codes the scope is per `codeClass`; for item classifications it's all rows. Notices are append-only — no deactivation.

## Service

New file: `packages/api-services/src/domains/invoicing/zra/zra-reference-data.service.ts`.

### Sync functions

```ts
type SyncTrigger = 'cron' | 'manual';
type DatasetType = 'codes' | 'item_classifications' | 'notices';

interface SyncResult {
  datasetType: DatasetType;
  status: 'success' | 'failed' | 'partial';
  fetched: number;
  upserted: number;
  deactivated: number;
  durationMs: number;
  errorMessage?: string;
  syncLogId: string;
}

syncZraCodes(deps, trigger: SyncTrigger, userId?: string): Promise<SyncResult>
syncZraItemClassifications(deps, trigger, userId?): Promise<SyncResult>
syncZraNotices(deps, trigger, userId?): Promise<SyncResult>
syncAllZraReferenceData(deps, trigger, userId?): Promise<SyncResult[]>
```

Each sync function follows this skeleton:
1. `findAnyActiveVsdcDevice(db)` → if null, write failed sync log and return/throw per trigger.
2. Insert sync log row with `status='success'`, `started_at=now()`, placeholder zeros.
3. Build `ZraApiClient` from device, call fetcher.
4. On non-success response: update log to `failed`, return/throw per trigger.
5. Transform wire payload → DB inserts.
6. Run upsert. For codes, loop per class and accumulate `perClassStats`.
7. Run deactivation. Accumulate `recordsDeactivated`.
8. Update sync log with final counts, `status`, `completed_at`, `duration_ms`.
9. Catch any throw mid-flight: update log to `failed` with `error.message`, then rethrow if `manual`, swallow if `cron`.

`syncAllZraReferenceData` calls each sync function in sequence (codes → notices → item classifications) wrapped in try/catch so one failure doesn't stop the next.

### Read functions

```ts
getZraCodesByClass(ctx, deps, codeClass: string): Promise<ZraCodeOption[]>
getZraCodeClasses(ctx, deps): Promise<ZraCodeClassOption[]>
getZraItemClassifications(ctx, deps, opts: {
  level?: number; search?: string; limit?: number
}): Promise<ZraItemClassificationOption[]>
getZraNotices(ctx, deps, limit?: number): Promise<ZraNotice[]>
getZraSyncStatus(ctx, deps): Promise<{
  lastSync: Record<DatasetType, { at: Date; status: string } | null>;
  recentLogs: ZraReferenceSyncLog[];
  isStale: Record<DatasetType, boolean>;  // true if last successful > 48h ago
}>
```

DTO shapes (`ZraCodeOption`, etc.) are trimmed for dropdown use (`{ code, name, sortOrder }`), not raw DB rows.

## HTTP routes

Added to existing `apps/api-invoicing` router under `/zra/...`:

| Method | Path | Auth | Handler |
|---|---|---|---|
| GET | `/zra/code-classes` | `requireAuth` | `getZraCodeClasses` |
| GET | `/zra/codes/:codeClass` | `requireAuth` | `getZraCodesByClass` |
| GET | `/zra/item-classifications` | `requireAuth` | `getZraItemClassifications` (query: `level`, `search`, `limit`) |
| GET | `/zra/notices` | `requireAuth` | `getZraNotices` (query: `limit`) |
| GET | `/zra/reference-data/sync-status` | `requireAuth, requireBackoffice` | `getZraSyncStatus` |
| POST | `/zra/reference-data/sync` | `requireAuth, requireBackoffice` | `syncAllZraReferenceData` via `ctx.waitUntil`, returns log IDs |

Manual sync body: `{ datasets?: DatasetType[] }` — omit to sync all. Returns 202 with `{ syncLogIds: string[] }`. Status polled via `GET /sync-status`.

## Cron wiring

Edit apps/api-jobs/src/scheduled.ts, `handleDailyJobs`:

```ts
async function handleDailyJobs(env: Env): Promise<void> {
  console.log('[Scheduled] Running daily jobs...');
  await runZraReferenceDataSync(env);
}

async function runZraReferenceDataSync(env: Env): Promise<void> {
  const deps = buildServiceDeps(env);
  try {
    const results = await syncAllZraReferenceData(deps, 'cron');
    console.log('[ZRA Sync] Daily reference data sync complete', results);
  } catch (err) {
    console.error('[ZRA Sync] Daily sync threw', err);
  }
}
```

No new cron entry needed — `0 0 * * *` is already configured in `apps/api-jobs/wrangler.toml` under both `[triggers]` and `[env.production.triggers]`.

## Frontend

### Reusable dropdown components

New directory: `apps/app/components/features/invoicing/zra/`.

- `<ZraCodeSelect codeClass="04" value={...} onChange={...} />` — generic dropdown for any code class. Fetches via existing Tanstack Query setup, client-cache TTL 1 hour.
- `<ZraItemClassificationCombobox value={...} onSelect={...} />` — async-search combobox with debounced server search (`level=5` default for leaf nodes only).

Both consume thin hooks:
```ts
useZraCodes(codeClass: string)            // → GET /zra/codes/:codeClass
useZraItemClassifications(opts)            // → GET /zra/item-classifications
```

### Backoffice sync UI

New page: `apps/backoffice/app/(authenticated)/zra/reference-data/page.tsx`.
- Top: per-dataset cards showing last sync timestamp, status, counts, "Sync now" button.
- Stale warning banner if any dataset is `isStale=true`.
- Bottom: recent sync log table (last 20 runs), row click opens drawer with `perClassStats` JSON.

### Wiring into existing forms

Out of scope for this plan — separate plan per surface. This plan only ships the building blocks. Document the wiring tasks as a follow-up TODO list in the implementation plan.

## Error handling

| Scenario | Behavior |
|---|---|
| No active+initialized VSDC device | Cron: log failed, swallow. Manual: throw `ServiceError("PRECONDITION_FAILED", "No active VSDC device available for sync")`. |
| ZRA returns `resultCd !== "000"` | Log failed with `resultMsg` in `error_message`. Cron: swallow. Manual: throw `ServiceError("BAD_GATEWAY", ...)`. |
| Network/timeout (after client retries exhausted) | Same as above. |
| DB upsert throws mid-sync | Catch, update log to `failed` with `error.message`, rethrow per trigger rules. |
| Partial: codes sync, one class fails | Log `status='partial'`, succeeded classes persisted, failed class noted in `per_class_stats.<class>.error`. |
| ZRA returns empty list | Valid result — all existing rows of that scope get `useYn='N'`. No crash. |

**Invariant:** No data is ever hard-deleted. Deactivation via `useYn='N'` is the only removal mechanism; re-running the sync restores rows ZRA starts returning again.

## Testing

**Repository tests** (`packages/database/__tests__/zra-reference-data.repo.test.ts`):
- Upsert is idempotent (same input twice → same row count, `synced_at` advances).
- Deactivation flips `useYn='N'` on rows not in the latest batch, scoped per `code_class` for codes.
- Restoration: row previously deactivated, re-appearing in batch, comes back as `useYn='Y'`.
- Filter queries respect `useYn='Y'`.

**Service tests** (`packages/api-services/__tests__/zra-reference-data.service.test.ts`) with mocked `ZraApiClient`:
- Successful full sync → correct counts in returned `SyncResult` and persisted log row.
- ZRA returns empty list → all existing rows deactivated, log `success`, no crash.
- ZRA returns `resultCd="999"` → log `failed`, error persisted, throws on manual / swallows on cron.
- No active VSDC device → log `failed`, no API call attempted.
- Real fixture from the sample JSON in this spec: `tax type` class transformed correctly to 4 codes with `userDefinedCode1` set to the rate.
- `syncAllZraReferenceData`: codes sync fails, item classifications still runs.

**Integration test** (one): manual-sync route with mocked ZRA API → verifies DB state, sync log row, and 202 response shape.

**No live ZRA sandbox test in CI** — too flaky. Manual smoke-test checklist included with the implementation plan.

## Observability

- Primary debugging surface: `zra_reference_sync_log` table + backoffice sync UI.
- Cron logs prefix: `[ZRA Sync]` for Cloudflare Logpush filtering.
- Staleness alert: `getZraSyncStatus` flags `isStale=true` when last successful sync > 48h ago (2× cron interval); backoffice UI shows banner.

## Open follow-ups (not in this plan)

- Wire individual dropdowns (sales invoice line item editor, customer/supplier forms, etc.) to use the new components — one plan per surface.
- Consider exposing notices in the main app dashboard.
- Branch list sync (`/branch/selectBhfList`) if/when needed for multi-branch orgs.
