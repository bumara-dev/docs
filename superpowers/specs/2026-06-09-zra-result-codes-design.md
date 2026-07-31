---
title: "ZRA Result Codes — Design"
description: "Design: replace binary ZRA result handling with a three-category typed code catalog, persisted on every transmission and sync record."
---

**Date:** 2026-06-09
**Status:** Approved
**Owner:** Invoicing domain

## Summary

Replace the existing binary success/failure handling of ZRA's `resultCd` with a three-category model (success / informational / error) backed by a typed code catalog. Persist `resultCd`, `resultMsg`, and `resultDt` on every transmission, sync log, and customer-sync record so admins can audit exactly what ZRA returned for each call.

## Motivation

The current `ZraApiClient.makeRequest` treats only `resultCd === "000"` as success; every other code becomes a generic failure. ZRA's spec actually distinguishes three classes of codes:

- `000` (success), `001` ("no search result" — a successful empty response, not an error)
- `801–805` (informational client-state flags about transmission/retransmission state)
- All other `8xx` / `9xx` codes (real errors)

Misclassifying `001` and `801–805` as errors causes spurious failed logs for normal outcomes (e.g. fetching notices from a newly-registered device returns `001` legitimately). Persisting only `error` strings also discards the raw codes admins need to debug specific ZRA rejections.

## Scope

**In scope:**
- TypeScript catalog of all 36 documented ZRA result codes with system / category / description.
- Helpers `getZraResultCodeMeta` and `categorizeZraResultCode`.
- `ZraApiResponse` gains `resultCd`, `resultMsg`, `resultDt`, `category` fields.
- `makeRequest` populates the new fields and uses category to determine `success`.
- Three tables get the same three new columns and three services pass them through.
- Tests for catalog, `makeRequest` categorization, and updated service tests.

**Out of scope:**
- Backfill of historical records (existing rows stay null on the new columns).
- UI surface for the new fields (admin dashboard work is a separate plan).
- Auto-retry behavior based on specific codes (e.g. retry on `805 — Corresponding retransmission data exists`). Current retry policy stays: 5xx + network errors retry, all `resultCd` responses do not.

## Approach

**Catalog as a `const` map** in `zra-api-client.ts`. Static, no DB lookup, ~40 entries. Updating requires a code change, which is the right friction for a vendor-controlled list.

**Additive change to `ZraApiResponse`** — no fields removed or renamed. Callers that ignore the new fields keep working; callers that opt in get richer info.

**Category drives `success`** — `success === true` when category is `success` OR `informational`. This fixes `001`/`801–805` misclassification automatically across every caller, no per-call-site changes needed.

**Persist on every record** — success and failure both write `resultCd/Msg/Dt`. Existing `rawResponse` columns stay (full JSON for deep debugging); the new columns are for cheap indexed audit.

## Code catalog

New constant `ZRA_RESULT_CODES` in `packages/api-services/src/domains/invoicing/zra/zra-api-client.ts`. Full list (categorization established up front; do not re-litigate during implementation):

| Code | System | Category | Description |
|---|---|---|---|
| 000 | Server | success | It is succeeded |
| 001 | Server | success | There is no search result |
| 801 | Client | informational | There is no data to retransmit. |
| 802 | Client | informational | There is data that has not been transferred. After transfer is possible. |
| 803 | Client | informational | This is a report that transfer is complete. |
| 804 | Client | informational | There is no data to send for the report. |
| 805 | Client | informational | Corresponding retransmission data exists. |
| 834 | Client | error | SalesType and ReceiptType must be NS-NR-ND-TS-TR-TD-CS-CR-CD-PS check your inputs |
| 836 | Client | error | Your Sequences have been altered, Connect to ZRA API to get Sequences. |
| 838 | Client | error | Connection to API is not established: check connection. |
| 884 | Client | error | Invalid customer TPIN was provided |
| 891 | Client | error | An error occurred while Request URL is created. |
| 892 | Client | error | An error occurred while Request Header data is created. |
| 893 | Client | error | An error occurred while Request Body data is created. |
| 894 | Client | error | An error regarding server communication occurred. |
| 895 | Client | error | An error regarding unallowed Request Method occurred. |
| 896 | Client | error | An error regarding Request Status occurred. |
| 899 | Client | error | An error regarding Client occurred. |
| 900 | Server | error | There is no Header information |
| 901 | Server | error | It is not valid device |
| 902 | Server | error | This device is installed |
| 903 | Server | error | Only VSDC device can be verified. |
| 910 | Server | error | Request parameter error |
| 911 | Server | error | There is no request full text |
| 912 | Server | error | There is a request Method error. |
| 913 | Client | error | Code value error among request parameters. |
| 921 | Server | error | Sales or sales invoice data which is declared cannot be received. |
| 922 | Server | error | Sales invoice data can be received after receiving the sales data. |
| 924 | Client | error | CIS Invoice number already exists. |
| 930 | Server | error | The specified invoice could not be found. Please verify [orgInvcNo] and try again |
| 931 | Server | error | The credit note amount exceeds the original invoice amount for item: |
| 932 | Server | error | The item specified in the credit note does not exist on the original invoice. [itemCd] |
| 934 | Server | error | The quantity specified in the credit note exceeds the quantity in the original invoice. |
| 935 | Server | error | The credit note contains information that does not match the original invoice data |
| 990 | Server | error | The maximum number of views are exceeded |
| 991 | Server | error | There is an error during registration |
| 992 | Server | error | There is an error during modification |
| 993 | Server | error | There is an error during deletion |
| 994 | Server | error | There is an overlapped Data |
| 995 | Server | error | There is no downloaded file |
| 999 | Server | error | There is an unknown error. Please ask it administrator |

Public types and helpers:

```typescript
export type ZraResultCategory = "success" | "informational" | "error";
export type ZraResultSystem = "Server" | "Client";

export interface ZraResultCodeMeta {
  code: string;
  system: ZraResultSystem;
  category: ZraResultCategory;
  description: string;
}

export const ZRA_RESULT_CODES: Record<string, ZraResultCodeMeta> = { /* full table above */ };

export function getZraResultCodeMeta(cd: string | undefined | null): ZraResultCodeMeta;
export function categorizeZraResultCode(cd: string | undefined | null): ZraResultCategory;
```

**Unknown code policy:** `getZraResultCodeMeta("999_NEW")` returns `{ code: "999_NEW", system: "Server", category: "error", description: "Unknown ZRA code 999_NEW" }`. Logged with the raw value so admins can spot codes ZRA introduces.

**Missing code policy:** `getZraResultCodeMeta(undefined)` returns `{ code: "", system: "Client", category: "error", description: "No result code returned" }`. Equivalent to a network failure.

## `ZraApiResponse` changes

Additive, no removals. New fields:

```typescript
export interface ZraApiResponse<T = Record<string, unknown>> {
  // existing fields unchanged
  success: boolean;
  message: string;
  data?: T;
  error?: string;
  reference: string;
  httpStatus: number;
  rawResponse?: string;

  // NEW (all optional)
  resultCd?: string;             // e.g. "000", "924", "999"
  resultMsg?: string;            // raw resultMsg from ZRA, unmodified
  resultDt?: string;             // raw resultDt from ZRA (YYYYMMDDHHmmss)
  category?: ZraResultCategory;  // "success" | "informational" | "error"
}
```

## `makeRequest` changes

In `packages/api-services/src/domains/invoicing/zra/zra-api-client.ts`, the existing `makeRequest` private method updates:

```typescript
// After parsing responseData...
const resultCd = responseData["resultCd"] as string | undefined;
const resultMsg = responseData["resultMsg"] as string | undefined;
const resultDt = responseData["resultDt"] as string | undefined;
const category = categorizeZraResultCode(resultCd);

if (category === "success" || category === "informational") {
  return {
    success: true,
    message: resultMsg ?? (responseData["message"] as string) ?? "Operation successful",
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

// category === "error" OR 4xx with no resultCd
if (resultCd || (response.status >= 400 && response.status < 500)) {
  const meta = resultCd ? getZraResultCodeMeta(resultCd) : undefined;
  return {
    success: false,
    message: resultMsg ?? (responseData["message"] as string) ?? "API request failed",
    error: resultMsg ?? meta?.description ?? (responseData["error"] as string) ?? `HTTP ${response.status}`,
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
```

The 5xx retry path and network/timeout retry path are unchanged; they don't have a `resultCd` to populate.

## Schema changes

Three identical column additions on three tables.

### `zra_smart_invoice_transmissions`

| Column | Type | Notes |
|---|---|---|
| `zra_result_cd` | text NULL | Last `resultCd` returned by ZRA |
| `zra_result_msg` | text NULL | Last `resultMsg` |
| `zra_result_dt` | text NULL | Last `resultDt` (YYYYMMDDHHmmss string) |

### `zra_reference_sync_log`

Same three columns. No new indexes.

### `customers`

Same three columns, alongside the existing ZRA sync state columns added in the previous customer-sync feature.

No backfill — existing rows stay null. Future rows populate via the service-layer changes below.

## Service changes

### `zra-smart-invoice.service.ts`

`transmitInvoiceToZra` already calls `updateTransmission` on success and failure. Add the new fields:

```typescript
await updateTransmission(deps.db, transmission.id, {
  // existing fields
  zraResultCd: apiResult.resultCd ?? null,
  zraResultMsg: apiResult.resultMsg ?? null,
  zraResultDt: apiResult.resultDt ?? null,
});
```

Same change in `handleSuccessfulRetry` and `handleFailedRetry` helpers.

### `zra-reference-data.service.ts`

`syncZraCodes`, `syncZraItemClassifications`, `syncZraNotices` each call `updateSyncLog` on success and failure. Add the same three fields to each call from the `apiResult` they hold.

### `zra-customer-sync.service.ts`

`updateCustomerZraSyncStatus` patch type extends to accept the new fields. Service passes them through on both success and failure paths.

## Repository changes

The three update functions (`updateTransmission`, `updateSyncLog`, `updateCustomerZraSyncStatus`) accept `Partial` patches. The new optional fields surface automatically once the schema types update — no signature changes. Inferred Drizzle insert types pick up the new columns.

## Testing

### Catalog tests
New file `packages/api-services/src/domains/invoicing/zra/__tests__/zra-result-codes.test.ts`:
- `categorizeZraResultCode("000")` → `"success"`
- `categorizeZraResultCode("001")` → `"success"`
- `categorizeZraResultCode("802")` → `"informational"`
- `categorizeZraResultCode("924")` → `"error"`
- `categorizeZraResultCode("999_NEW")` → `"error"` (unknown)
- `categorizeZraResultCode(undefined)` → `"error"`
- `getZraResultCodeMeta("924").description` contains "CIS Invoice number"

### `makeRequest` integration tests
New tests in an existing or new test file for `zra-api-client.ts`. Mock `fetch` to return:
- `{ resultCd: "000", resultMsg: "ok", resultDt: "20260609120000", data: {...} }` → `success: true, category: "success"`, all three fields populated
- `{ resultCd: "001", resultMsg: "no results" }` → `success: true, category: "success"` (the helper coerces 001 to success)
- `{ resultCd: "802", resultMsg: "pending transfer" }` → `success: true, category: "informational"`
- `{ resultCd: "924", resultMsg: "exists" }` → `success: false, category: "error"`, `error` populated
- `{ resultCd: "999_UNKNOWN" }` → `success: false, category: "error"`, description falls back to "Unknown ZRA code 999_UNKNOWN"

### Service tests
Updated:
- `zra-reference-data.service.test.ts` — one new assertion in the "happy path" test: log entry receives `zraResultCd: "000"`.
- `zra-customer-sync.service.test.ts` — same: success test asserts `zraResultCd: "000"` on the patch.

## Error handling

| Scenario | Behavior |
|---|---|
| Unknown `resultCd` (not in catalog) | `category: "error"`, description `"Unknown ZRA code <cd>"`. Logged. Persisted as the raw code. |
| Missing `resultCd` | `category: "error"`, description `"No result code returned"`. Behavior identical to a 4xx with no body. |
| Informational code on a flow expecting data | Caller already handles empty array gracefully (existing reference-data sync code). New `category` field lets future callers branch if needed, but no current call site needs changes. |
| `resultCd` is present but malformed (e.g. not a string) | Cast to string is unsafe; current code uses `as string | undefined`. If the cast yields a non-string at runtime, `categorizeZraResultCode` falls back to error. Safe. |

**Invariant:** No existing call site's semantic changes except where `001` or `801–805` was previously misclassified as failure — those now correctly become success / informational.

## Observability

- `console.error` calls in existing services already log `apiResult`; the new fields ride along automatically.
- The new persisted columns are queryable: `SELECT zra_result_cd, COUNT(*) FROM zra_smart_invoice_transmissions GROUP BY zra_result_cd` lets ops spot trends.
- No new logging is added in this plan.

## Open follow-ups (not in this plan)

- Admin UI surface for resultCd / description on transmission and customer detail pages.
- Code-specific retry policies (e.g. retry `805` differently from `924`).
- Backfill script to populate the new columns on historical rows by re-parsing `raw_response`.
