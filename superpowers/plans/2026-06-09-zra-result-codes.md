---
title: "ZRA Result Codes Implementation Plan"
description: "For agentic workers: REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this..."
---

**Goal:** Add a typed catalog of ZRA result codes with three-category classification (success / informational / error), enrich `ZraApiResponse` with the parsed code/message/date/category, and persist these on every transmission, sync log, and customer-sync record.

**Architecture:** A `const` catalog + 2 helpers + 4 new `ZraApiResponse` fields are added to `zra-api-client.ts`. `makeRequest` uses the catalog to drive its success/failure branching, restoring correct behavior for `001` (success) and `801–805` (informational). Three tables get the same three new columns; three services pass the new fields through their existing `updateTransmission` / `updateSyncLog` / `updateCustomerZraSyncStatus` calls.

**Tech Stack:** TypeScript, Drizzle ORM (Postgres), Cloudflare Workers (Hono), Vitest with `vi.hoisted` mocks.

**Spec:** [docs/superpowers/specs/2026-06-09-zra-result-codes-design.md](/superpowers/specs/2026-06-09-zra-result-codes-design)

**Note on commits:** The user is handling commits manually at the end. Tasks DO NOT commit.

---

## File Structure

**Modify API client (single file does a lot of work):**
- `packages/api-services/src/domains/invoicing/zra/zra-api-client.ts` — add types, catalog, 2 helpers, 4 response fields, update `makeRequest`

**New tests:**
- `packages/api-services/src/domains/invoicing/zra/__tests__/zra-result-codes.test.ts` — catalog + helper unit tests
- `packages/api-services/src/domains/invoicing/zra/__tests__/zra-api-client.test.ts` — `makeRequest` categorization integration tests (new file; the API client wasn't directly tested before)

**Modify schemas (3 tables, same 3 columns each):**
- `packages/database/src/schema/invoicing/zra-smart-invoice.ts` — add 3 columns to `zraSmartInvoiceTransmissions`
- `packages/database/src/schema/invoicing/zra-reference-data.ts` — add 3 columns to `zraReferenceSyncLog`
- `packages/database/src/schema/invoicing/customers.ts` — add 3 columns

**Modify services:**
- `packages/api-services/src/domains/invoicing/zra-smart-invoice.service.ts` — pass new fields in 3 `updateTransmission` calls (lines 81, 99, 404)
- `packages/api-services/src/domains/invoicing/zra/zra-reference-data.service.ts` — pass new fields in `updateSyncLog` calls inside `syncZraCodes`, `syncZraItemClassifications`, `syncZraNotices`
- `packages/api-services/src/domains/invoicing/zra/zra-customer-sync.service.ts` — pass new fields in `updateCustomerZraSyncStatus` calls

**Update existing service tests** (one new assertion each):
- `packages/api-services/src/domains/invoicing/zra/__tests__/zra-reference-data.service.test.ts`
- `packages/api-services/src/domains/invoicing/zra/__tests__/zra-customer-sync.service.test.ts`

**Migration (auto-generated):** `packages/database/drizzle/XXXX_<name>.sql` — should contain 9 ALTER TABLE ADD COLUMN statements (3 tables × 3 columns).

---

## Task 1: Catalog + Helpers + Types

**Files:**
- Modify: `packages/api-services/src/domains/invoicing/zra/zra-api-client.ts`
- Create: `packages/api-services/src/domains/invoicing/zra/__tests__/zra-result-codes.test.ts`

- [ ] **Step 1: Write the failing test**

Create `packages/api-services/src/domains/invoicing/zra/__tests__/zra-result-codes.test.ts`:

```typescript
import { describe, expect, it } from 'vitest';
import {
  ZRA_RESULT_CODES,
  categorizeZraResultCode,
  getZraResultCodeMeta,
} from '../zra-api-client';

describe('ZRA result code catalog', () => {
  it('classifies success codes', () => {
    expect(categorizeZraResultCode('000')).toBe('success');
    expect(categorizeZraResultCode('001')).toBe('success');
  });

  it('classifies informational codes (801-805)', () => {
    for (const cd of ['801', '802', '803', '804', '805']) {
      expect(categorizeZraResultCode(cd)).toBe('informational');
    }
  });

  it('classifies error codes', () => {
    expect(categorizeZraResultCode('834')).toBe('error');
    expect(categorizeZraResultCode('884')).toBe('error');
    expect(categorizeZraResultCode('924')).toBe('error');
    expect(categorizeZraResultCode('930')).toBe('error');
    expect(categorizeZraResultCode('999')).toBe('error');
  });

  it('classifies unknown codes as error with descriptive message', () => {
    const meta = getZraResultCodeMeta('999_NEW');
    expect(meta.category).toBe('error');
    expect(meta.description).toContain('Unknown ZRA code 999_NEW');
    expect(meta.code).toBe('999_NEW');
  });

  it('classifies missing code as error with descriptive message', () => {
    const meta = getZraResultCodeMeta(undefined);
    expect(meta.category).toBe('error');
    expect(meta.description).toContain('No result code returned');
    expect(meta.code).toBe('');
  });

  it('returns full metadata for known codes', () => {
    expect(getZraResultCodeMeta('000')).toMatchObject({
      code: '000', system: 'Server', category: 'success',
    });
    expect(getZraResultCodeMeta('924').description).toContain('CIS Invoice number');
    expect(getZraResultCodeMeta('834').system).toBe('Client');
  });

  it('catalog contains all 41 documented codes', () => {
    // 000, 001 + 801-805 + the 34 error codes from the spec
    expect(Object.keys(ZRA_RESULT_CODES).length).toBe(41);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `pnpm --filter @repo/api-services exec vitest run zra-result-codes`
Expected: FAIL — `categorizeZraResultCode is not a function` (or similar — the exports don't exist yet).

- [ ] **Step 3: Add types, catalog, and helpers to `zra-api-client.ts`**

Open `packages/api-services/src/domains/invoicing/zra/zra-api-client.ts`. Find the constants section (around line 230, just before `ZRA_API_ENDPOINTS`). Insert the following block BEFORE `ZRA_API_ENDPOINTS`:

```typescript
// ============================================
// ZRA Result Codes
// ============================================

export type ZraResultCategory = "success" | "informational" | "error";
export type ZraResultSystem = "Server" | "Client";

export interface ZraResultCodeMeta {
  code: string;
  system: ZraResultSystem;
  category: ZraResultCategory;
  description: string;
}

export const ZRA_RESULT_CODES: Record<string, ZraResultCodeMeta> = {
  "000": { code: "000", system: "Server", category: "success", description: "It is succeeded" },
  "001": { code: "001", system: "Server", category: "success", description: "There is no search result" },
  "801": { code: "801", system: "Client", category: "informational", description: "There is no data to retransmit." },
  "802": { code: "802", system: "Client", category: "informational", description: "There is data that has not been transferred. After transfer is possible." },
  "803": { code: "803", system: "Client", category: "informational", description: "This is a report that transfer is complete." },
  "804": { code: "804", system: "Client", category: "informational", description: "There is no data to send for the report." },
  "805": { code: "805", system: "Client", category: "informational", description: "Corresponding retransmission data exists." },
  "834": { code: "834", system: "Client", category: "error", description: "SalesType and ReceiptType must be NS-NR-ND-TS-TR-TD-CS-CR-CD-PS check your inputs" },
  "836": { code: "836", system: "Client", category: "error", description: "Your Sequences have been altered, Connect to ZRA API to get Sequences." },
  "838": { code: "838", system: "Client", category: "error", description: "Connection to API is not established: check connection." },
  "884": { code: "884", system: "Client", category: "error", description: "Invalid customer TPIN was provided" },
  "891": { code: "891", system: "Client", category: "error", description: "An error occurred while Request URL is created." },
  "892": { code: "892", system: "Client", category: "error", description: "An error occurred while Request Header data is created." },
  "893": { code: "893", system: "Client", category: "error", description: "An error occurred while Request Body data is created." },
  "894": { code: "894", system: "Client", category: "error", description: "An error regarding server communication occurred." },
  "895": { code: "895", system: "Client", category: "error", description: "An error regarding unallowed Request Method occurred." },
  "896": { code: "896", system: "Client", category: "error", description: "An error regarding Request Status occurred." },
  "899": { code: "899", system: "Client", category: "error", description: "An error regarding Client occurred." },
  "900": { code: "900", system: "Server", category: "error", description: "There is no Header information" },
  "901": { code: "901", system: "Server", category: "error", description: "It is not valid device" },
  "902": { code: "902", system: "Server", category: "error", description: "This device is installed" },
  "903": { code: "903", system: "Server", category: "error", description: "Only VSDC device can be verified." },
  "910": { code: "910", system: "Server", category: "error", description: "Request parameter error" },
  "911": { code: "911", system: "Server", category: "error", description: "There is no request full text" },
  "912": { code: "912", system: "Server", category: "error", description: "There is a request Method error." },
  "913": { code: "913", system: "Client", category: "error", description: "Code value error among request parameters." },
  "921": { code: "921", system: "Server", category: "error", description: "Sales or sales invoice data which is declared cannot be received." },
  "922": { code: "922", system: "Server", category: "error", description: "Sales invoice data can be received after receiving the sales data." },
  "924": { code: "924", system: "Client", category: "error", description: "CIS Invoice number already exists." },
  "930": { code: "930", system: "Server", category: "error", description: "The specified invoice could not be found. Please verify [orgInvcNo] and try again" },
  "931": { code: "931", system: "Server", category: "error", description: "The credit note amount exceeds the original invoice amount for item:" },
  "932": { code: "932", system: "Server", category: "error", description: "The item specified in the credit note does not exist on the original invoice. [itemCd]" },
  "934": { code: "934", system: "Server", category: "error", description: "The quantity specified in the credit note exceeds the quantity in the original invoice." },
  "935": { code: "935", system: "Server", category: "error", description: "The credit note contains information that does not match the original invoice data" },
  "990": { code: "990", system: "Server", category: "error", description: "The maximum number of views are exceeded" },
  "991": { code: "991", system: "Server", category: "error", description: "There is an error during registration" },
  "992": { code: "992", system: "Server", category: "error", description: "There is an error during modification" },
  "993": { code: "993", system: "Server", category: "error", description: "There is an error during deletion" },
  "994": { code: "994", system: "Server", category: "error", description: "There is an overlapped Data" },
  "995": { code: "995", system: "Server", category: "error", description: "There is no downloaded file" },
  "999": { code: "999", system: "Server", category: "error", description: "There is an unknown error. Please ask it administrator" },
};

export function getZraResultCodeMeta(cd: string | undefined | null): ZraResultCodeMeta {
  if (!cd) {
    return { code: "", system: "Client", category: "error", description: "No result code returned" };
  }
  const meta = ZRA_RESULT_CODES[cd];
  if (meta) return meta;
  return {
    code: cd,
    system: "Server",
    category: "error",
    description: `Unknown ZRA code ${cd}`,
  };
}

export function categorizeZraResultCode(cd: string | undefined | null): ZraResultCategory {
  return getZraResultCodeMeta(cd).category;
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `pnpm --filter @repo/api-services exec vitest run zra-result-codes`
Expected: all 7 tests PASS.

- [ ] **Step 5: Typecheck**

Run: `pnpm --filter @repo/api-services typecheck`
Expected: zero new errors. (Pre-existing errors elsewhere are OK.)

- [ ] **Step 6: DO NOT COMMIT.** Leave changes in working tree.

---

## Task 2: Enrich `ZraApiResponse` + update `makeRequest`

**Files:**
- Modify: `packages/api-services/src/domains/invoicing/zra/zra-api-client.ts`
- Create: `packages/api-services/src/domains/invoicing/zra/__tests__/zra-api-client.test.ts`

- [ ] **Step 1: Write the failing test**

Create `packages/api-services/src/domains/invoicing/zra/__tests__/zra-api-client.test.ts`:

```typescript
import { afterEach, beforeEach, describe, expect, it, vi } from 'vitest';
import { ZraApiClient } from '../zra-api-client';

const originalFetch = global.fetch;

function mockFetchOnce(body: object, status = 200) {
  global.fetch = vi.fn().mockResolvedValue(
    new Response(JSON.stringify(body), {
      status,
      headers: { 'Content-Type': 'application/json' },
    })
  ) as unknown as typeof fetch;
}

describe('ZraApiClient.makeRequest result categorization', () => {
  let client: ZraApiClient;

  beforeEach(() => {
    client = new ZraApiClient({
      baseUrl: 'https://api-sandbox.zra.org.zm/vsdc-api/v1',
      tpin: '1000000000',
      branchId: '000',
      deviceSerialNumber: 'SN1',
    });
  });

  afterEach(() => {
    global.fetch = originalFetch;
  });

  it('treats resultCd=000 as success with all fields populated', async () => {
    mockFetchOnce({
      resultCd: '000',
      resultMsg: 'It is succeeded',
      resultDt: '20260609120000',
      data: { foo: 'bar' },
    });
    const res = await client.fetchCodeList();
    expect(res.success).toBe(true);
    expect(res.category).toBe('success');
    expect(res.resultCd).toBe('000');
    expect(res.resultMsg).toBe('It is succeeded');
    expect(res.resultDt).toBe('20260609120000');
  });

  it('treats resultCd=001 (no search result) as success', async () => {
    mockFetchOnce({ resultCd: '001', resultMsg: 'There is no search result' });
    const res = await client.fetchCodeList();
    expect(res.success).toBe(true);
    expect(res.category).toBe('success');
    expect(res.resultCd).toBe('001');
  });

  it('treats resultCd=802 (informational) as success with informational category', async () => {
    mockFetchOnce({ resultCd: '802', resultMsg: 'pending transfer' });
    const res = await client.fetchCodeList();
    expect(res.success).toBe(true);
    expect(res.category).toBe('informational');
    expect(res.resultCd).toBe('802');
  });

  it('treats resultCd=924 as error with full fields populated', async () => {
    mockFetchOnce({
      resultCd: '924',
      resultMsg: 'CIS Invoice number already exists.',
      resultDt: '20260609120000',
    });
    const res = await client.fetchCodeList();
    expect(res.success).toBe(false);
    expect(res.category).toBe('error');
    expect(res.resultCd).toBe('924');
    expect(res.resultMsg).toContain('CIS Invoice');
    expect(res.error).toContain('CIS Invoice');
  });

  it('treats unknown resultCd as error with fallback description', async () => {
    mockFetchOnce({ resultCd: '999_UNKNOWN' });
    const res = await client.fetchCodeList();
    expect(res.success).toBe(false);
    expect(res.category).toBe('error');
    expect(res.resultCd).toBe('999_UNKNOWN');
    expect(res.error).toContain('Unknown ZRA code 999_UNKNOWN');
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `pnpm --filter @repo/api-services exec vitest run zra-api-client.test`
Expected: FAIL — assertions on `category` field fail because the field doesn't exist on the response object yet.

- [ ] **Step 3: Add new fields to `ZraApiResponse`**

In `zra-api-client.ts`, find the `ZraApiResponse` interface (around line 27) and update to:

```typescript
export interface ZraApiResponse<T = Record<string, unknown>> {
  success: boolean;
  message: string;
  data?: T;
  error?: string;
  reference: string;
  httpStatus: number;
  rawResponse?: string;
  /** Raw ZRA resultCd, if present in the response body */
  resultCd?: string;
  /** Raw ZRA resultMsg, unmodified */
  resultMsg?: string;
  /** Raw ZRA resultDt (YYYYMMDDHHmmss), unmodified */
  resultDt?: string;
  /** Computed category: success | informational | error */
  category?: ZraResultCategory;
}
```

- [ ] **Step 4: Update `makeRequest` to populate the new fields**

In `zra-api-client.ts`, find the `makeRequest` method's response-handling block (around lines 683–722, the `isZraSuccess` branch and the error branch). Replace that block with:

```typescript
        // Parse ZRA-specific response fields
        const resultCd = responseData["resultCd"] as string | undefined;
        const resultMsg = responseData["resultMsg"] as string | undefined;
        const resultDt = responseData["resultDt"] as string | undefined;
        const category = categorizeZraResultCode(resultCd);

        // Success or informational → success=true
        if (category === "success" || category === "informational") {
          return {
            success: true,
            message:
              resultMsg ??
              (responseData["message"] as string) ??
              "Operation successful",
            data: (responseData["data"] ?? responseData) as T,
            reference,
            httpStatus: response.status,
            rawResponse: responseText,
            resultCd,
            resultMsg,
            resultDt,
            category,
          };
        }

        // ZRA error code OR 4xx → do not retry
        if (resultCd || (response.status >= 400 && response.status < 500)) {
          const meta = resultCd ? getZraResultCodeMeta(resultCd) : undefined;
          return {
            success: false,
            message:
              resultMsg ??
              (responseData["message"] as string) ??
              "API request failed",
            error:
              resultMsg ??
              meta?.description ??
              (responseData["error"] as string) ??
              `HTTP ${response.status}`,
            reference,
            httpStatus: response.status,
            rawResponse: responseText,
            resultCd,
            resultMsg,
            resultDt,
            category: "error",
          };
        }

        // 5xx with no resultCd — retry (unchanged)
        lastError = new Error(
          `HTTP ${response.status}: ${responseText.substring(0, 200)}`
        );
```

Important: the helpers `categorizeZraResultCode` and `getZraResultCodeMeta` are defined in the same file (added in Task 1) so no import changes needed.

The previous `const isZraSuccess = resultCd === "000" || (...)` line is REMOVED and replaced by `category === "success" || category === "informational"`.

- [ ] **Step 5: Run tests to verify they pass**

Run: `pnpm --filter @repo/api-services exec vitest run zra-api-client.test`
Expected: all 5 tests PASS.

Also re-run Task 1 tests: `pnpm --filter @repo/api-services exec vitest run zra-result-codes`
Expected: still 7 PASS.

- [ ] **Step 6: Run the existing service tests to verify no regressions**

Run: `pnpm --filter @repo/api-services exec vitest run zra-reference-data.service zra-customer-sync.service`
Expected: all tests still pass (the new fields are additive — existing assertions ignore them).

- [ ] **Step 7: Typecheck**

Run: `pnpm --filter @repo/api-services typecheck`
Expected: zero new errors.

- [ ] **Step 8: DO NOT COMMIT.**

---

## Task 3: Schema — Add `zra_result_*` Columns to 3 Tables

**Files:**
- Modify: `packages/database/src/schema/invoicing/zra-smart-invoice.ts`
- Modify: `packages/database/src/schema/invoicing/zra-reference-data.ts`
- Modify: `packages/database/src/schema/invoicing/customers.ts`

- [ ] **Step 1: Add columns to `zra_smart_invoice_transmissions`**

In `packages/database/src/schema/invoicing/zra-smart-invoice.ts`, find the `zraSmartInvoiceTransmissions` pgTable definition. Find the `rawResponse: text('raw_response')` line. Right after it (before the `...timestamps` spread), add:

```typescript
    // Raw payloads for debugging
    rawRequest: text('raw_request'), // JSON
    rawResponse: text('raw_response'), // JSON

    // ZRA result code metadata (from resultCd/resultMsg/resultDt)
    zraResultCd: text('zra_result_cd'),
    zraResultMsg: text('zra_result_msg'),
    zraResultDt: text('zra_result_dt'),

    ...timestamps,
```

(The `rawRequest` and `rawResponse` lines already exist — they're shown for positional context. Only add the 4 new lines: the comment and the 3 columns.)

- [ ] **Step 2: Add columns to `zra_reference_sync_log`**

In `packages/database/src/schema/invoicing/zra-reference-data.ts`, find the `zraReferenceSyncLog` pgTable definition. Find the `errorMessage: text('error_message')` line. After the `completedAt` line (which is the last field before `...timestamps`), add the same 4 lines:

```typescript
    completedAt: timestamp('completed_at', { mode: 'date' }),

    // ZRA result code metadata (from resultCd/resultMsg/resultDt)
    zraResultCd: text('zra_result_cd'),
    zraResultMsg: text('zra_result_msg'),
    zraResultDt: text('zra_result_dt'),

    ...timestamps,
```

- [ ] **Step 3: Add columns to `customers`**

In `packages/database/src/schema/invoicing/customers.ts`, find the ZRA sync block (added in the previous customer-sync feature — `zraSyncStatus`, `zraSyncedAt`, `zraSyncAttemptedAt`, `zraSyncError`). After `zraSyncError`, before the next block (`// Additional information` or whatever comes next), add:

```typescript
    zraSyncError: text('zra_sync_error'),                                       // last error message; null on success

    // ZRA result code metadata (from resultCd/resultMsg/resultDt)
    zraResultCd: text('zra_result_cd'),
    zraResultMsg: text('zra_result_msg'),
    zraResultDt: text('zra_result_dt'),
```

(The `zraSyncError` line already exists — shown for positional context.)

- [ ] **Step 4: Typecheck**

Run: `pnpm --filter @repo/database typecheck`
Expected: zero new errors. Existing errors elsewhere are OK.

- [ ] **Step 5: Generate the migration**

Run: `pnpm --filter @repo/database db:generate`
Expected: new SQL file in `packages/database/drizzle/` with exactly 9 `ALTER TABLE ... ADD COLUMN` statements (3 tables × 3 columns).

**If the migration contains unrelated drift** (inventory `last_sale_price`, payroll `payslip_*` drops, etc.), hand-edit the generated SQL to keep only the 9 ZRA result-code ADD COLUMN statements. The reference-data and customer-sync features hit this same drift; the trimming pattern is established.

- [ ] **Step 6: DO NOT COMMIT.** Leave the generated SQL, snapshot, and journal updates in the working tree.

---

## Task 4: Service Updates — Pass Result Code Fields Through

**Files:**
- Modify: `packages/api-services/src/domains/invoicing/zra-smart-invoice.service.ts`
- Modify: `packages/api-services/src/domains/invoicing/zra/zra-reference-data.service.ts`
- Modify: `packages/api-services/src/domains/invoicing/zra/zra-customer-sync.service.ts`

This task adds the same three `apiResult` field passthroughs (`zraResultCd`, `zraResultMsg`, `zraResultDt`) to every `updateTransmission` / `updateSyncLog` / `updateCustomerZraSyncStatus` call in the three services. The patch object types are inferred from the schema (updated in Task 3), so once the schema lands no signature changes are needed.

- [ ] **Step 1: Update `zra-smart-invoice.service.ts` — `transmitInvoiceToZra`**

In `packages/api-services/src/domains/invoicing/zra-smart-invoice.service.ts`, find `transmitInvoiceToZra`. There's an `updateTransmission` call (around line 399–407). Update the `set` object to include the new fields:

```typescript
  const updated = await updateTransmission(deps.db, transmission.id, {
    transmissionStatus: apiResult.success ? "transmitted" : "failed",
    transmittedAt: apiResult.success ? new Date() : null,
    rawResponse: apiResult.rawResponse ?? JSON.stringify(apiResult),
    zraReceiptNumber: apiResult.success ? receiptNumber : null,
    rejectionReason: apiResult.success
      ? null
      : (apiResult.error ?? apiResult.message),
    zraResultCd: apiResult.resultCd ?? null,
    zraResultMsg: apiResult.resultMsg ?? null,
    zraResultDt: apiResult.resultDt ?? null,
  });
```

- [ ] **Step 2: Update `zra-smart-invoice.service.ts` — `handleSuccessfulRetry`**

In the same file, find `handleSuccessfulRetry` (around line 72). Update its `updateTransmission` call:

```typescript
async function handleSuccessfulRetry(
  deps: ServiceDependencies,
  transmissionId: string,
  retryCount: number,
  apiResult: ZraApiResponse
) {
  await updateTransmission(deps.db, transmissionId, {
    retryCount,
    transmissionStatus: "transmitted",
    transmittedAt: new Date(),
    rawResponse: apiResult.rawResponse ?? JSON.stringify(apiResult),
    zraReceiptNumber:
      ((apiResult.data as Record<string, unknown>)?.rcptNo as string) ?? null,
    rejectionReason: null,
    zraResultCd: apiResult.resultCd ?? null,
    zraResultMsg: apiResult.resultMsg ?? null,
    zraResultDt: apiResult.resultDt ?? null,
  });
}
```

- [ ] **Step 3: Update `zra-smart-invoice.service.ts` — `handleFailedRetry`**

In the same file, find `handleFailedRetry` (around line 89). Update its `updateTransmission` call:

```typescript
async function handleFailedRetry(
  deps: ServiceDependencies,
  transmissionId: string,
  retryCount: number,
  apiResult: ZraApiResponse
) {
  const backoffMs = 2 ** retryCount * 60 * 1000;
  await updateTransmission(deps.db, transmissionId, {
    retryCount,
    nextRetryAt: new Date(Date.now() + backoffMs),
    transmissionStatus: retryCount >= MAX_RETRIES ? "failed" : "pending",
    rawResponse: apiResult.rawResponse ?? JSON.stringify(apiResult),
    rejectionReason: apiResult.error ?? apiResult.message,
    zraResultCd: apiResult.resultCd ?? null,
    zraResultMsg: apiResult.resultMsg ?? null,
    zraResultDt: apiResult.resultDt ?? null,
  });
}
```

- [ ] **Step 4: Update `zra-reference-data.service.ts` — `syncZraCodes` (success branch)**

In `packages/api-services/src/domains/invoicing/zra/zra-reference-data.service.ts`, find `syncZraCodes`. There's an `updateSyncLog` call at the end of the try block (success path, around line 209). Update its `set` to include the new fields:

```typescript
    await updateSyncLog(deps.db, log.id, {
      status: "success",
      recordsFetched: fetched,
      recordsUpserted: upserted,
      recordsDeactivated: deactivated,
      perClassStats,
      completedAt: new Date(),
      durationMs: Date.now() - startedAt.getTime(),
      zraResultCd: apiResult.resultCd ?? null,
      zraResultMsg: apiResult.resultMsg ?? null,
      zraResultDt: apiResult.resultDt ?? null,
    });
```

- [ ] **Step 5: Update `zra-reference-data.service.ts` — `syncZraCodes` (ZRA-failure branch)**

In the same function, find the earlier `updateSyncLog` call inside `if (!apiResult.success)` (around line 134). Update its `set`:

```typescript
      await updateSyncLog(deps.db, log.id, {
        status: "failed",
        errorMessage,
        completedAt: new Date(),
        durationMs: Date.now() - startedAt.getTime(),
        zraResultCd: apiResult.resultCd ?? null,
        zraResultMsg: apiResult.resultMsg ?? null,
        zraResultDt: apiResult.resultDt ?? null,
      });
```

The catch-block `updateSyncLog` (around line 231, which handles thrown errors mid-sync) does NOT get the new fields — at that point we don't have an `apiResult` in scope. Leave the catch block alone.

- [ ] **Step 6: Update `zra-reference-data.service.ts` — `syncZraItemClassifications` (both branches)**

Apply the same pattern to `syncZraItemClassifications`:
- In its success-path `updateSyncLog` call, add the three `zraResultCd/Msg/Dt: apiResult.resultCd/Msg/Dt ?? null` lines.
- In its ZRA-failure-path `updateSyncLog` call (the one inside `if (!apiResult.success)`), add the same three lines.
- Do NOT modify its catch-block `updateSyncLog`.

- [ ] **Step 7: Update `zra-reference-data.service.ts` — `syncZraNotices` (both branches)**

Apply the same pattern to `syncZraNotices`:
- Success-path `updateSyncLog` (whether status is `"success"` or `"partial"`): add the three fields.
- ZRA-failure-path `updateSyncLog`: add the three fields.
- Do NOT modify its catch-block `updateSyncLog`.

- [ ] **Step 8: Update `zra-customer-sync.service.ts` — success and failure branches**

In `packages/api-services/src/domains/invoicing/zra/zra-customer-sync.service.ts`, find the success-path `updateCustomerZraSyncStatus` call (the one that sets `zraSyncStatus: "synced"`, around line 69). Update:

```typescript
  if (apiResult.success) {
    await updateCustomerZraSyncStatus(deps.db, customerId, orgId, {
      zraSyncStatus: "synced",
      zraSyncedAt: new Date(),
      zraSyncAttemptedAt: new Date(),
      zraSyncError: null,
      zraResultCd: apiResult.resultCd ?? null,
      zraResultMsg: apiResult.resultMsg ?? null,
      zraResultDt: apiResult.resultDt ?? null,
    });
    return { status: "synced" };
  }
```

And the failure-path call (the one that sets `zraSyncStatus: "failed"`, around line 79):

```typescript
  await updateCustomerZraSyncStatus(deps.db, customerId, orgId, {
    zraSyncStatus: "failed",
    zraSyncAttemptedAt: new Date(),
    zraSyncError: errorMessage,
    zraResultCd: apiResult.resultCd ?? null,
    zraResultMsg: apiResult.resultMsg ?? null,
    zraResultDt: apiResult.resultDt ?? null,
  });
```

The `noDeviceFailure` skip path (status `"skipped_no_device"`) does NOT get the new fields — no ZRA call was made.

- [ ] **Step 9: Update the patch type in `updateCustomerZraSyncStatus`**

In `packages/database/src/repositories/invoicing-customers.ts`, find `updateCustomerZraSyncStatus`. The patch parameter currently lists only the four sync-state fields. Extend it to accept the three new ones:

```typescript
export async function updateCustomerZraSyncStatus(
  db: DatabaseClient,
  customerId: string,
  organizationId: string,
  patch: {
    zraSyncStatus?: 'synced' | 'failed' | 'skipped_no_device';
    zraSyncedAt?: Date | null;
    zraSyncAttemptedAt?: Date;
    zraSyncError?: string | null;
    zraResultCd?: string | null;
    zraResultMsg?: string | null;
    zraResultDt?: string | null;
  }
): Promise<void> {
  // body unchanged
  await db
    .update(customers)
    .set({
      ...patch,
      updatedAt: new Date(),
    })
    .where(
      and(
        eq(customers.id, customerId),
        eq(customers.organizationId, organizationId)
      )
    );
}
```

- [ ] **Step 10: Typecheck**

Run: `pnpm --filter @repo/api-services typecheck` and `pnpm --filter @repo/database typecheck`
Expected: zero new errors. The `updateTransmission` and `updateSyncLog` repo functions accept `Partial<XInsert>` already — no signature changes needed for them.

- [ ] **Step 11: DO NOT COMMIT.**

---

## Task 5: Update Existing Service Tests

**Files:**
- Modify: `packages/api-services/src/domains/invoicing/zra/__tests__/zra-reference-data.service.test.ts`
- Modify: `packages/api-services/src/domains/invoicing/zra/__tests__/zra-customer-sync.service.test.ts`

The existing tests use `expect.objectContaining` for the patch assertions, so they continue to pass even after the service adds new fields. This task adds ONE new assertion per test file to lock in the new behavior.

- [ ] **Step 1: Update the reference-data service "happy path" test**

In `packages/api-services/src/domains/invoicing/zra/__tests__/zra-reference-data.service.test.ts`, find the test `it('transforms the sample tax-type response correctly', ...)` (or whichever existing test asserts on a successful `syncZraCodes` run). Update the mocked `fetchCodeList` resolved value to include the new fields:

```typescript
mocks.fetchCodeList.mockResolvedValue({
  success: true,
  message: 'ok',
  reference: 'r',
  httpStatus: 200,
  resultCd: '000',
  resultMsg: 'It is succeeded',
  resultDt: '20260609120000',
  category: 'success',
  data: { clsList: [ /* existing fixture */ ] },
});
```

Then add a new assertion AFTER the existing `expect(result.status).toBe('success')`:

```typescript
// New: verify resultCd flowed through to the log
expect(mocks.updateSyncLog).toHaveBeenLastCalledWith(
  deps.db,
  'log-1',
  expect.objectContaining({
    zraResultCd: '000',
    zraResultMsg: 'It is succeeded',
    zraResultDt: '20260609120000',
  })
);
```

- [ ] **Step 2: Update the customer-sync service "synced" test**

In `packages/api-services/src/domains/invoicing/zra/__tests__/zra-customer-sync.service.test.ts`, find the test `it('returns synced and clears error on successful ZRA call', ...)`. Update the mocked `saveBranchCustomer` resolved value:

```typescript
mocks.saveBranchCustomer.mockResolvedValue({
  success: true,
  message: 'ok',
  reference: 'r',
  httpStatus: 200,
  resultCd: '000',
  resultMsg: 'It is succeeded',
  resultDt: '20260609120000',
  category: 'success',
});
```

Then update the existing assertion on `updateCustomerZraSyncStatus` to include the new fields:

```typescript
expect(mocks.updateCustomerZraSyncStatus).toHaveBeenCalledWith(
  deps.db,
  'cust-1',
  'org-1',
  expect.objectContaining({
    zraSyncStatus: 'synced',
    zraSyncedAt: expect.any(Date),
    zraSyncAttemptedAt: expect.any(Date),
    zraSyncError: null,
    zraResultCd: '000',
    zraResultMsg: 'It is succeeded',
    zraResultDt: '20260609120000',
  })
);
```

- [ ] **Step 3: Run all updated tests**

Run: `pnpm --filter @repo/api-services exec vitest run zra-reference-data.service zra-customer-sync.service`
Expected: all tests PASS.

- [ ] **Step 4: Run the full ZRA test suite for one final check**

Run: `pnpm --filter @repo/api-services exec vitest run zra`
Expected: catalog tests (Task 1), api-client tests (Task 2), reference-data service tests (Task 5), customer-sync service tests (Task 5) all PASS.

- [ ] **Step 5: DO NOT COMMIT.**

---

## Task 6: Apply Migration + Smoke Test (deferred to user)

The migration generated in Task 3 needs to apply to whatever DB `DATABASE_URL` points at. This step is **deferred to the user** because it requires credentials and is destructive (alters production-shape tables).

To apply once ready:

```
pnpm --filter @repo/database db:migrate
```

Smoke test:
1. Trigger a successful sync (e.g. manually create a customer with an active VSDC device).
2. `SELECT zra_result_cd, zra_result_msg, zra_result_dt FROM customers WHERE id = '<new customer id>'` — expect `'000'`, the human-readable success message, and a 14-char timestamp.
3. Trigger a failing sync (e.g. create a customer when ZRA sandbox is unreachable — VSDC device with bad credentials).
4. `SELECT zra_result_cd, zra_sync_error FROM customers ...` — expect a populated error code (or null if the request never reached ZRA) and a populated `zra_sync_error`.

---

## Self-Review Checklist (for the plan author)

**Spec coverage:**
- Catalog of 41 codes, `getZraResultCodeMeta`, `categorizeZraResultCode`: Task 1 ✓
- New `ZraApiResponse` fields (`resultCd`, `resultMsg`, `resultDt`, `category`): Task 2 ✓
- `makeRequest` uses category for success/error branching: Task 2 ✓
- 3 tables × 3 new columns: Task 3 ✓
- 3 services updated to pass new fields: Task 4 ✓
- Catalog tests, `makeRequest` tests, updated service tests: Tasks 1, 2, 5 ✓
- Migration apply + smoke test: Task 6 (deferred to user) ✓

**Placeholder scan:** No "TBD" / "TODO" / "similar to". Task 6 is explicitly deferred (operational work).

**Type consistency:**
- `ZraResultCategory` union (`'success' | 'informational' | 'error'`) — same in catalog, types, tests, and `makeRequest`.
- `ZraResultCodeMeta` shape — same fields used in catalog and `getZraResultCodeMeta` return.
- New column names (`zraResultCd`, `zraResultMsg`, `zraResultDt`) — same in 3 schema files, 3 services, repo helper type, and tests.
- `apiResult.resultCd ?? null` — same null-coalesce pattern in every service callsite (matches the Drizzle insert/update behavior for nullable columns).
