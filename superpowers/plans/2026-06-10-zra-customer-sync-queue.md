---
title: "ZRA Customer Sync — Queue-Backed Implementation Plan"
description: "For agentic workers: REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this..."
---

**Goal:** Replace `c.executionCtx.waitUntil(syncCustomerToZra(...))` in `createCustomerHandler` and `updateCustomerHandler` with a fire-and-forget enqueue onto a dedicated Cloudflare Queue (`zra-customer-sync`), processed by `api-jobs` on its own fresh DB connection with built-in retries and a DLQ. Adds a centralized `runAfterResponse(c, promise, label?)` helper.

**Architecture:** Producer-side in `api-invoicing` enqueues `{ customerId, orgId, attempt }`; consumer in `api-jobs` opens one DB client per batch, calls the existing `syncCustomerToZra` service, then classifies failures — transient → throw (Queues retries up to 5; lands in DLQ); permanent → ack. The producer goes through a new `runAfterResponse` helper in `@repo/backend/context` so other handlers (invoice / item sync) can adopt the same pattern.

**Tech Stack:** Cloudflare Workers (Hono + zod-openapi), Cloudflare Queues, Drizzle ORM (Neon serverless Postgres), TypeScript, Vitest with `vi.hoisted` mocks.

**Spec:** [docs/superpowers/specs/2026-06-10-zra-customer-sync-queue-design.md](/superpowers/specs/2026-06-10-zra-customer-sync-queue-design)

**Note on commits:** Each task ends with staging a coherent slice but does not run `git commit`. The user is handling commits manually at the end.

---

## File Structure

**Modify shared backend context:**
- `packages/backend/src/core/context/run-after-response.ts` — new helper (file)
- `packages/backend/src/core/context/run-after-response.test.ts` — unit tests
- `packages/backend/src/core/context/index.ts` — new barrel re-exporting `service-context.ts` + the new helper
- `packages/backend/package.json` — `./context` export points to the new barrel

**Modify shared service types:**
- `packages/api-services/src/core/context/context.ts` — extend `Env` with `ZRA_CUSTOMER_SYNC_QUEUE`; export `ZraCustomerSyncMessage`
- `packages/api-services/src/core/context/index.ts` — re-export `ZraCustomerSyncMessage` (if not already wildcarded)

**Modify ZRA customer-sync service:**
- `packages/api-services/src/domains/invoicing/zra/zra-customer-sync.service.ts` — add `classifyCustomerSyncFailure`
- `packages/api-services/src/domains/invoicing/zra/__tests__/zra-customer-sync.service.test.ts` — add classifier tests
- `packages/api-services/src/domains/invoicing/index.ts` — export `classifyCustomerSyncFailure`

**New consumer in api-jobs:**
- `apps/api-jobs/src/queues/zra-customer-sync-processor.ts` — `processZraCustomerSyncQueue` + `isZraCustomerSyncMessage`
- `apps/api-jobs/src/queues/__tests__/zra-customer-sync-processor.test.ts` — processor tests
- `apps/api-jobs/src/index.ts` — extend `queue()` router with the new branch

**Wrangler bindings:**
- `apps/api-invoicing/wrangler.toml` — add producer binding (every env)
- `apps/api-jobs/wrangler.toml` — add producer + consumer bindings (every env), with DLQ in production

**Handler integration:**
- `apps/api-invoicing/src/routes/invoicing/customers/handlers.ts` — revert the `Pool` import block; replace `waitUntil(syncCustomerToZra(...))` with `runAfterResponse(c, env.QUEUE.send(...))` in both create and update handlers

---

## Task 1: Shared helper — `runAfterResponse`

**Files:**
- Create: `packages/backend/src/core/context/run-after-response.ts`
- Create: `packages/backend/src/core/context/run-after-response.test.ts`
- Create: `packages/backend/src/core/context/index.ts`
- Modify: `packages/backend/package.json`

- [ ] **Step 1: Write the failing test file**

Create `packages/backend/src/core/context/run-after-response.test.ts` with:

```typescript
import { describe, expect, it, vi } from "vitest";
import { runAfterResponse } from "./run-after-response";

function makeCtx(opts: {
  withExecutionCtx?: boolean;
  withLogger?: boolean;
} = {}) {
  const waitUntil = vi.fn();
  const loggerError = vi.fn();
  const ctx = {
    executionCtx: opts.withExecutionCtx === false ? undefined : { waitUntil },
    get: vi.fn((key: string) => {
      if (key === "logger" && opts.withLogger) {
        return { error: loggerError };
      }
      return undefined;
    }),
  };
  return { ctx: ctx as never, waitUntil, loggerError };
}

describe("runAfterResponse", () => {
  it("schedules the promise via executionCtx.waitUntil when available", () => {
    const { ctx, waitUntil } = makeCtx({ withExecutionCtx: true });
    const promise = Promise.resolve("ok");

    runAfterResponse(ctx, promise);

    expect(waitUntil).toHaveBeenCalledTimes(1);
    // The promise passed to waitUntil is the .catch()-wrapped version, not the raw promise
    expect(waitUntil.mock.calls[0]![0]).toBeInstanceOf(Promise);
  });

  it("logs via c.get('logger') on rejection when a logger is present", async () => {
    const { ctx, waitUntil, loggerError } = makeCtx({
      withExecutionCtx: true,
      withLogger: true,
    });
    const promise = Promise.reject(new Error("boom"));

    runAfterResponse(ctx, promise, "[test] failed");

    // Drain the catch handler
    await waitUntil.mock.calls[0]![0];

    expect(loggerError).toHaveBeenCalledTimes(1);
    expect(loggerError.mock.calls[0]![0]).toMatchObject({ err: expect.any(Error) });
    expect(loggerError.mock.calls[0]![1]).toBe("[test] failed");
  });

  it("falls back to console.error when no logger is present", async () => {
    const { ctx, waitUntil } = makeCtx({ withExecutionCtx: true, withLogger: false });
    const consoleError = vi.spyOn(console, "error").mockImplementation(() => undefined);
    const promise = Promise.reject(new Error("boom"));

    runAfterResponse(ctx, promise, "[test] failed");
    await waitUntil.mock.calls[0]![0];

    expect(consoleError).toHaveBeenCalledWith("[test] failed", expect.any(Error));
    consoleError.mockRestore();
  });

  it("no-ops gracefully when executionCtx is absent (test env)", async () => {
    const { ctx } = makeCtx({ withExecutionCtx: false });
    const consoleError = vi.spyOn(console, "error").mockImplementation(() => undefined);
    const promise = Promise.reject(new Error("boom"));

    // Must not throw synchronously and must still attach a catch so the rejection
    // does not surface as an unhandled rejection.
    runAfterResponse(ctx, promise, "[test] failed");

    // Drain the microtask queue so the .catch fires.
    await new Promise((r) => setTimeout(r, 0));

    // It still logs the rejection via console.error fallback.
    expect(consoleError).toHaveBeenCalledWith("[test] failed", expect.any(Error));
    consoleError.mockRestore();
  });

  it("uses the default label when none is provided", async () => {
    const { ctx, waitUntil } = makeCtx({ withExecutionCtx: true, withLogger: false });
    const consoleError = vi.spyOn(console, "error").mockImplementation(() => undefined);

    runAfterResponse(ctx, Promise.reject(new Error("x")));
    await waitUntil.mock.calls[0]![0];

    expect(consoleError).toHaveBeenCalledWith(
      "[runAfterResponse] background task failed",
      expect.any(Error)
    );
    consoleError.mockRestore();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm --filter @repo/backend exec vitest run run-after-response`
Expected: FAIL — `Cannot find module './run-after-response'`.

- [ ] **Step 3: Write the helper**

Create `packages/backend/src/core/context/run-after-response.ts`:

```typescript
import type { Context } from "hono";

type WaitUntilCtx = {
  executionCtx?: { waitUntil: (p: Promise<unknown>) => void };
  get: (key: string) => unknown;
};

type MinimalLogger = {
  error?: (meta: Record<string, unknown>, message: string) => void;
};

/**
 * Schedule a promise to run after the response is sent.
 *
 * - Uses Cloudflare Workers' `executionCtx.waitUntil` when available.
 * - Catches rejections so they never surface as unhandled rejections, logging
 *   via `c.get("logger")` if present, falling back to `console.error`.
 * - In environments without `executionCtx` (Node tests, local dev without the
 *   Cloudflare runtime), the promise still runs because the caller already
 *   constructed it; this helper just guarantees the rejection is logged.
 *
 * Use this for any post-response work: queue enqueues, fire-and-forget logging,
 * metrics, etc. Do not use it for work the response correctness depends on.
 */
export function runAfterResponse<T>(
  c: Context,
  promise: Promise<T>,
  label = "[runAfterResponse] background task failed"
): void {
  const ctx = c as unknown as WaitUntilCtx;
  const logger = ctx.get("logger") as MinimalLogger | undefined;

  const wrapped = promise.catch((err: unknown) => {
    if (logger?.error) {
      logger.error({ err }, label);
      return;
    }
    console.error(label, err);
  });

  if (ctx.executionCtx?.waitUntil) {
    ctx.executionCtx.waitUntil(wrapped);
  }
  // No executionCtx: the .catch above still runs on the microtask queue,
  // so the rejection is logged and not surfaced.
}
```

- [ ] **Step 4: Create the context barrel**

Create `packages/backend/src/core/context/index.ts`:

```typescript
export * from "./service-context";
export * from "./run-after-response";
```

- [ ] **Step 5: Update `@repo/backend` export to point at the barrel**

In `packages/backend/package.json`, change the line:

```json
"./context": "./src/core/context/service-context.ts",
```

to:

```json
"./context": "./src/core/context/index.ts",
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `pnpm --filter @repo/backend exec vitest run run-after-response`
Expected: all 5 tests PASS.

- [ ] **Step 7: Typecheck**

Run: `pnpm --filter @repo/backend typecheck`
Expected: passes (preexisting errors in unrelated files in the wider monorepo are OK).

- [ ] **Step 8: Stage (do not commit)**

Leave the new helper, the test, the barrel, and the `package.json` change staged in the working tree.

---

## Task 2: Extend `Env` with the new queue and message type

**Files:**
- Modify: `packages/api-services/src/core/context/context.ts`

- [ ] **Step 1: Add the message interface and Env field**

Open `packages/api-services/src/core/context/context.ts`. Above the existing `Env` interface (around line 21), beside the other queue message interfaces, add:

```typescript
export interface ZraCustomerSyncMessage {
  customerId: string;
  orgId: string;
  /**
   * Producer-side attempt counter. Informational only — Cloudflare's
   * `message.attempts` is authoritative for retry decisions.
   */
  attempt: number;
}
```

Then in the existing `Env` interface, add the new queue binding next to `DELIVERY_QUEUE`:

```typescript
export interface Env {
  MYBROWSER: Fetcher;
  // Database
  DATABASE_URL: string;

  // Queues
  OUTBOX_QUEUE: Queue<OutboxQueueMessage>;
  DELIVERY_QUEUE: Queue<DeliveryQueueMessage>;
  ZRA_CUSTOMER_SYNC_QUEUE: Queue<ZraCustomerSyncMessage>;

  NEXT_PUBLIC_APP_URL: string;
  NEXT_PUBLIC_APP_BACKOFFICE_URL: string;

  // Providers
  RESEND_TOKEN: string;
  RESEND_FROM: string;
  WHATSAPP_PHONE_NUMBER_ID: string;
  WHATSAPP_ACCESS_TOKEN: string;
}
```

- [ ] **Step 2: Verify the type is exported from the package barrel**

The existing barrel at `packages/api-services/src/index.ts` already re-exports from `./core/context` (search the file for `export * from "./core/context"` or similar). If `ZraCustomerSyncMessage` is not picked up by `import type { ZraCustomerSyncMessage } from "@repo/api-services"`, add a named re-export at the appropriate barrel. If the existing message types (`OutboxQueueMessage`, `DeliveryQueueMessage`) are already importable from `@repo/api-services`, the new type rides on the same export and no extra change is needed.

To verify, run:

```
pnpm --filter @repo/api-services exec tsc --noEmit src/core/context/context.ts
```

Expected: file compiles. If `OutboxQueueMessage` is already exposed via the package root, `ZraCustomerSyncMessage` is too.

- [ ] **Step 3: Typecheck**

Run: `pnpm --filter @repo/api-services typecheck`
Expected: passes for `context.ts`. Preexisting errors elsewhere are OK.

- [ ] **Step 4: Stage (do not commit)**

---

## Task 3: Add `classifyCustomerSyncFailure` to the service

**Files:**
- Modify: `packages/api-services/src/domains/invoicing/zra/zra-customer-sync.service.ts`
- Modify: `packages/api-services/src/domains/invoicing/zra/__tests__/zra-customer-sync.service.test.ts`
- Modify: `packages/api-services/src/domains/invoicing/index.ts`

- [ ] **Step 1: Append failing classifier tests to the existing test file**

Open `packages/api-services/src/domains/invoicing/zra/__tests__/zra-customer-sync.service.test.ts`. Add a new `describe` block at the bottom of the file (after the existing `describe('syncCustomerToZra', ...)` closes):

```typescript
import { classifyCustomerSyncFailure } from '../zra-customer-sync.service';

describe('classifyCustomerSyncFailure', () => {
  const transientStrings = [
    'HTTP 500 internal server error',
    'HTTP 502 bad gateway',
    'HTTP 503 service unavailable',
    'HTTP 504 gateway timeout',
    'HTTP 429 too many requests',
    'ETIMEDOUT',
    'ECONNRESET',
    'ECONNREFUSED',
    'fetch failed',
  ];

  const permanentStrings = [
    'HTTP 400 bad request',
    'HTTP 401 unauthorized',
    'HTTP 403 forbidden',
    'HTTP 404 not found',
    'HTTP 422 unprocessable entity',
    'invalid TPIN',
  ];

  for (const err of transientStrings) {
    it(`treats "${err}" as transient`, () => {
      expect(classifyCustomerSyncFailure({ status: 'failed', error: err })).toBe('transient');
    });
  }

  for (const err of permanentStrings) {
    it(`treats "${err}" as permanent`, () => {
      expect(classifyCustomerSyncFailure({ status: 'failed', error: err })).toBe('permanent');
    });
  }

  it('treats skipped_no_device as permanent', () => {
    expect(classifyCustomerSyncFailure({ status: 'skipped_no_device' })).toBe('permanent');
  });

  it('treats synced as permanent (defensive; should never be called)', () => {
    expect(classifyCustomerSyncFailure({ status: 'synced' })).toBe('permanent');
  });

  it('treats undefined error string as permanent (no transient signal)', () => {
    expect(classifyCustomerSyncFailure({ status: 'failed' })).toBe('permanent');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm --filter @repo/api-services exec vitest run zra-customer-sync.service`
Expected: FAIL — `classifyCustomerSyncFailure is not a function` or import error.

- [ ] **Step 3: Implement the classifier**

Open `packages/api-services/src/domains/invoicing/zra/zra-customer-sync.service.ts`. Append at the end of the file (after the `syncCustomerToZra` export):

```typescript
const TRANSIENT_RE =
  /HTTP 5\d\d|ETIMEDOUT|ECONNRESET|ECONNREFUSED|fetch failed|HTTP 429/i;

/**
 * Decide whether a CustomerSyncResult represents a transient failure
 * (worth retrying) or a permanent one (ack immediately).
 *
 * - 5xx, network resets/timeouts, and 429 are transient.
 * - 4xx, validation errors, skipped_no_device, and missing error strings
 *   are permanent.
 *
 * NOT_FOUND ServiceErrors from syncCustomerToZra throw rather than return,
 * and the caller handles them separately (see api-jobs consumer).
 */
export function classifyCustomerSyncFailure(
  result: CustomerSyncResult
): "transient" | "permanent" {
  if (result.status !== "failed") return "permanent";
  const err = result.error ?? "";
  return TRANSIENT_RE.test(err) ? "transient" : "permanent";
}
```

- [ ] **Step 4: Re-export from the invoicing barrel**

Open `packages/api-services/src/domains/invoicing/index.ts`. Find the existing export from `./zra/zra-customer-sync.service` (added by the previous plan):

```typescript
export {
  type CustomerSyncResult,
  syncCustomerToZra,
} from "./zra/zra-customer-sync.service";
```

Extend it to include the classifier:

```typescript
export {
  classifyCustomerSyncFailure,
  type CustomerSyncResult,
  syncCustomerToZra,
} from "./zra/zra-customer-sync.service";
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `pnpm --filter @repo/api-services exec vitest run zra-customer-sync.service`
Expected: all existing tests still pass + the new classifier tests pass.

- [ ] **Step 6: Typecheck**

Run: `pnpm --filter @repo/api-services typecheck`
Expected: passes for the edited files.

- [ ] **Step 7: Stage (do not commit)**

---

## Task 4: Consumer processor in `api-jobs`

**Files:**
- Create: `apps/api-jobs/src/queues/zra-customer-sync-processor.ts`
- Create: `apps/api-jobs/src/queues/__tests__/zra-customer-sync-processor.test.ts`

- [ ] **Step 1: Write the failing test file**

Create `apps/api-jobs/src/queues/__tests__/zra-customer-sync-processor.test.ts`:

```typescript
import { beforeEach, describe, expect, it, vi } from "vitest";

const mocks = vi.hoisted(() => ({
  createDbClient: vi.fn(),
  syncCustomerToZra: vi.fn(),
  classifyCustomerSyncFailure: vi.fn(),
}));

vi.mock("@repo/database", () => ({
  createDbClient: mocks.createDbClient,
}));

vi.mock("@repo/api-services/domains/invoicing", () => ({
  syncCustomerToZra: mocks.syncCustomerToZra,
  classifyCustomerSyncFailure: mocks.classifyCustomerSyncFailure,
}));

// ServiceError class — re-imported from the real module so `instanceof`
// checks in the consumer line up with real-world throws.
import { ServiceError } from "@repo/api-services";

import {
  isZraCustomerSyncMessage,
  processZraCustomerSyncQueue,
} from "../zra-customer-sync-processor";

type MockMessage = {
  body: { customerId: string; orgId: string; attempt: number };
  attempts: number;
  ack: ReturnType<typeof vi.fn>;
};

function makeMessage(customerId: string, orgId = "org-1"): MockMessage {
  return {
    body: { customerId, orgId, attempt: 1 },
    attempts: 1,
    ack: vi.fn(),
  };
}

const env = { DATABASE_URL: "postgres://test" } as never;
const execCtx = {} as never;

beforeEach(() => {
  mocks.createDbClient.mockReset().mockReturnValue({});
  mocks.syncCustomerToZra.mockReset();
  mocks.classifyCustomerSyncFailure.mockReset();
});

describe("isZraCustomerSyncMessage", () => {
  it("accepts a well-formed message", () => {
    expect(
      isZraCustomerSyncMessage({ customerId: "c", orgId: "o", attempt: 1 })
    ).toBe(true);
  });

  it("rejects messages with the wrong shape", () => {
    expect(isZraCustomerSyncMessage(null)).toBe(false);
    expect(isZraCustomerSyncMessage({})).toBe(false);
    expect(isZraCustomerSyncMessage({ customerId: "c" })).toBe(false);
    expect(isZraCustomerSyncMessage({ outboxEventId: "x" })).toBe(false);
  });
});

describe("processZraCustomerSyncQueue", () => {
  it("acks a successful message", async () => {
    const msg = makeMessage("c1");
    mocks.syncCustomerToZra.mockResolvedValue({ status: "synced" });

    await processZraCustomerSyncQueue(
      { messages: [msg] } as never,
      env,
      execCtx
    );

    expect(msg.ack).toHaveBeenCalledTimes(1);
  });

  it("opens a single DB client for the whole batch", async () => {
    const msgs = [makeMessage("c1"), makeMessage("c2"), makeMessage("c3")];
    mocks.syncCustomerToZra.mockResolvedValue({ status: "synced" });

    await processZraCustomerSyncQueue(
      { messages: msgs } as never,
      env,
      execCtx
    );

    expect(mocks.createDbClient).toHaveBeenCalledTimes(1);
  });

  it("re-throws on transient failure (does not ack)", async () => {
    const msg = makeMessage("c1");
    mocks.syncCustomerToZra.mockResolvedValue({
      status: "failed",
      error: "HTTP 503",
    });
    mocks.classifyCustomerSyncFailure.mockReturnValue("transient");

    await expect(
      processZraCustomerSyncQueue({ messages: [msg] } as never, env, execCtx)
    ).rejects.toThrow(/transient/i);

    expect(msg.ack).not.toHaveBeenCalled();
  });

  it("acks on permanent failure", async () => {
    const msg = makeMessage("c1");
    mocks.syncCustomerToZra.mockResolvedValue({
      status: "failed",
      error: "HTTP 400 invalid TPIN",
    });
    mocks.classifyCustomerSyncFailure.mockReturnValue("permanent");

    await processZraCustomerSyncQueue(
      { messages: [msg] } as never,
      env,
      execCtx
    );

    expect(msg.ack).toHaveBeenCalledTimes(1);
  });

  it("acks on ServiceError NOT_FOUND (customer deleted since enqueue)", async () => {
    const msg = makeMessage("c1");
    mocks.syncCustomerToZra.mockRejectedValue(
      new ServiceError("NOT_FOUND", "Customer not found")
    );

    await processZraCustomerSyncQueue(
      { messages: [msg] } as never,
      env,
      execCtx
    );

    expect(msg.ack).toHaveBeenCalledTimes(1);
  });

  it("processes a mixed batch independently — success acks, transient throws, permanent acks", async () => {
    const m1 = makeMessage("c1");
    const m2 = makeMessage("c2");
    const m3 = makeMessage("c3");

    mocks.syncCustomerToZra
      .mockResolvedValueOnce({ status: "synced" })
      .mockResolvedValueOnce({ status: "failed", error: "HTTP 503" })
      .mockResolvedValueOnce({ status: "failed", error: "HTTP 400" });

    mocks.classifyCustomerSyncFailure
      .mockReturnValueOnce("transient")
      .mockReturnValueOnce("permanent");

    await expect(
      processZraCustomerSyncQueue(
        { messages: [m1, m2, m3] } as never,
        env,
        execCtx
      )
    ).rejects.toThrow(/transient/i);

    // m1 acked before m2 threw
    expect(m1.ack).toHaveBeenCalledTimes(1);
    // m2 not acked
    expect(m2.ack).not.toHaveBeenCalled();
    // m3 never reached (loop stops at the throw); Queues will redeliver
    // the whole remaining slice. The consumer does not catch-and-continue
    // past a transient throw — that's the documented behavior.
    expect(m3.ack).not.toHaveBeenCalled();
  });

  it("no-ops on an empty batch", async () => {
    await processZraCustomerSyncQueue(
      { messages: [] } as never,
      env,
      execCtx
    );
    expect(mocks.createDbClient).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm --filter bumara-api-jobs exec vitest run zra-customer-sync-processor`
Expected: FAIL — `Cannot find module '../zra-customer-sync-processor'`.

If the filter name `bumara-api-jobs` is wrong, look up the actual `"name"` field in `apps/api-jobs/package.json` and use that. The processor tests must run inside the `api-jobs` workspace because that's where the test file lives.

- [ ] **Step 3: Implement the processor**

Create `apps/api-jobs/src/queues/zra-customer-sync-processor.ts`:

```typescript
import type {
  ExecutionContext,
  MessageBatch,
} from "@cloudflare/workers-types";
import {
  type Env,
  ServiceError,
  type ZraCustomerSyncMessage,
} from "@repo/api-services";
import {
  classifyCustomerSyncFailure,
  syncCustomerToZra,
} from "@repo/api-services/domains/invoicing";
import { createDbClient } from "@repo/database";

/**
 * Narrowing type-guard used by the root `queue()` router in `api-jobs/index.ts`
 * to dispatch the first message in a batch to the correct processor.
 */
export function isZraCustomerSyncMessage(
  msg: unknown
): msg is ZraCustomerSyncMessage {
  if (msg === null || typeof msg !== "object") return false;
  const m = msg as Record<string, unknown>;
  return (
    typeof m.customerId === "string" &&
    typeof m.orgId === "string" &&
    typeof m.attempt === "number"
  );
}

/**
 * Process a batch of ZRA customer-sync messages.
 *
 * Per-batch DB client (no per-request middleware pool to race against).
 * Per-message error policy:
 *  - success           -> ack
 *  - failed/transient  -> throw (Cloudflare Queues redelivers, then DLQ)
 *  - failed/permanent  -> ack (row already marked `failed` by the service)
 *  - NOT_FOUND throw   -> ack (customer deleted since enqueue)
 *  - other throw       -> re-throw (transient; let Queues redeliver)
 */
export async function processZraCustomerSyncQueue(
  batch: MessageBatch<ZraCustomerSyncMessage>,
  env: Env,
  _ctx: ExecutionContext
): Promise<void> {
  console.log("[ZRA-CUST-SYNC] queue handler invoked", {
    messageCount: batch.messages.length,
    timestamp: new Date().toISOString(),
  });

  if (batch.messages.length === 0) {
    return;
  }

  const db = createDbClient({ databaseUrl: env.DATABASE_URL });

  for (const message of batch.messages) {
    const { customerId, orgId, attempt } = message.body;

    try {
      const result = await syncCustomerToZra(
        {
          ...systemServiceContext({ env, source: 'zra-customer-sync-queue' }),
          orgId,
        },
        { db },
        customerId
      );

      if (result.status === "failed") {
        const classification = classifyCustomerSyncFailure(result);
        if (classification === "transient") {
          // Don't ack: throw so Queues redelivers. Row was already marked
          // `failed` by the service.
          throw new Error(
            `[ZRA-CUST-SYNC] transient failure for ${customerId}: ${result.error ?? "unknown"}`
          );
        }
      }

      // synced, skipped_no_device, or permanent failed -> ack.
      message.ack();
    } catch (err) {
      if (err instanceof ServiceError && err.code === "NOT_FOUND") {
        console.warn("[ZRA-CUST-SYNC] customer not found, acking", {
          customerId,
          orgId,
        });
        message.ack();
        continue;
      }
      console.error("[ZRA-CUST-SYNC] processing error, will retry", {
        customerId,
        orgId,
        attempt,
        deliveryAttempt: message.attempts,
        err: err instanceof Error ? err.message : String(err),
      });
      throw err;
    }
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm --filter bumara-api-jobs exec vitest run zra-customer-sync-processor`
Expected: all tests PASS.

- [ ] **Step 5: Typecheck**

Run: `pnpm --filter bumara-api-jobs typecheck`
Expected: passes for the new file.

- [ ] **Step 6: Stage (do not commit)**

---

## Task 5: Wire the processor into the `api-jobs` queue router

**Files:**
- Modify: `apps/api-jobs/src/index.ts`

- [ ] **Step 1: Extend the queue handler**

Open `apps/api-jobs/src/index.ts`. Replace the existing imports + `queue` method as follows.

Update the import section at the top to add the new types and processor:

```typescript
import type {
  ExecutionContext,
  MessageBatch,
  ScheduledEvent,
} from "@cloudflare/workers-types";
import type {
  DeliveryQueueMessage,
  Env,
  OutboxQueueMessage,
  ZraCustomerSyncMessage,
} from "@repo/api-services";
import app from "./app";
import {
  isDeliveryQueueMessage,
  isOutboxQueueMessage,
  processDeliveryQueue,
} from "./queues/delivery-processor";
import { processOutboxQueue } from "./queues/outbox-processor";
import {
  isZraCustomerSyncMessage,
  processZraCustomerSyncQueue,
} from "./queues/zra-customer-sync-processor";
```

Replace the `queue` method body to add the new branch:

```typescript
  async queue(
    batch: MessageBatch<
      OutboxQueueMessage | DeliveryQueueMessage | ZraCustomerSyncMessage
    >,
    env: Env,
    ctx: ExecutionContext
  ) {
    console.log("[QUEUE] Handler invoked with batch:", {
      messageCount: batch.messages.length,
      timestamp: new Date().toISOString(),
    });

    if (batch.messages.length === 0) {
      console.log("[QUEUE] Empty batch");
      return;
    }

    const firstMessage = batch.messages[0]?.body;

    if (isOutboxQueueMessage(firstMessage)) {
      console.log("[QUEUE] Routing to outbox processor");
      await processOutboxQueue(
        batch as MessageBatch<OutboxQueueMessage>,
        env,
        ctx
      );
    } else if (isDeliveryQueueMessage(firstMessage)) {
      console.log("[QUEUE] Routing to delivery processor");
      await processDeliveryQueue(
        batch as MessageBatch<DeliveryQueueMessage>,
        env,
        ctx
      );
    } else if (isZraCustomerSyncMessage(firstMessage)) {
      console.log("[QUEUE] Routing to ZRA customer sync processor");
      await processZraCustomerSyncQueue(
        batch as MessageBatch<ZraCustomerSyncMessage>,
        env,
        ctx
      );
    } else {
      console.error("[QUEUE] Unknown message type:", firstMessage);
    }
  },
```

- [ ] **Step 2: Typecheck**

Run: `pnpm --filter bumara-api-jobs typecheck`
Expected: passes for `index.ts`.

- [ ] **Step 3: Stage (do not commit)**

---

## Task 6: Wrangler bindings

**Files:**
- Modify: `apps/api-invoicing/wrangler.toml`
- Modify: `apps/api-jobs/wrangler.toml`

- [ ] **Step 1: Add producer binding in `api-invoicing`**

Open `apps/api-invoicing/wrangler.toml`. After the `[dev]` block (around line 18) and before `# Staging environment`, add a default-env producer binding (this serves `wrangler dev` from the root):

```toml
[[queues.producers]]
binding = "ZRA_CUSTOMER_SYNC_QUEUE"
queue = "zra-customer-sync"
```

In `[env.staging]`, before the next env block, add:

```toml
[[env.staging.queues.producers]]
binding = "ZRA_CUSTOMER_SYNC_QUEUE"
queue = "zra-customer-sync"
```

In `[env.production]`, at the end of that env block, add:

```toml
[[env.production.queues.producers]]
binding = "ZRA_CUSTOMER_SYNC_QUEUE"
queue = "zra-customer-sync-prod"
```

- [ ] **Step 2: Add producer + consumer bindings in `api-jobs`**

Open `apps/api-jobs/wrangler.toml`. In each env block (`[env.dev]`, `[env.staging]`, `[env.production]`), add a producer and a consumer for the new queue.

In `[env.dev]` (after the existing `notification-delivery` consumer block, around line 30):

```toml
[[env.dev.queues.producers]]
binding = "ZRA_CUSTOMER_SYNC_QUEUE"
queue = "zra-customer-sync"

[[env.dev.queues.consumers]]
queue = "zra-customer-sync"
max_batch_size = 10
max_retries = 5
```

In `[env.staging]` (after the existing `notification-delivery` consumer block, around line 70):

```toml
[[env.staging.queues.producers]]
binding = "ZRA_CUSTOMER_SYNC_QUEUE"
queue = "zra-customer-sync"

[[env.staging.queues.consumers]]
queue = "zra-customer-sync"
max_batch_size = 10
max_retries = 5
```

In `[env.production]` (after the existing `notification-delivery-prod` consumer block with DLQ, around line 105):

```toml
[[env.production.queues.producers]]
binding = "ZRA_CUSTOMER_SYNC_QUEUE"
queue = "zra-customer-sync-prod"

[[env.production.queues.consumers]]
queue = "zra-customer-sync-prod"
max_batch_size = 10
max_retries = 5
dead_letter_queue = "zra-customer-sync-prod-dlq"
```

- [ ] **Step 3: Validate wrangler config syntactically**

Run from `apps/api-invoicing`:

```
pnpm exec wrangler --env dev deploy --dry-run --outdir /tmp/wrangler-dry-invoicing
```

Expected: dry-run succeeds, no TOML parse errors. (If the command needs different syntax in this repo's wrangler version, the minimum check is `pnpm exec wrangler types --env dev` succeeding.)

Run from `apps/api-jobs`:

```
pnpm exec wrangler --env dev deploy --dry-run --outdir /tmp/wrangler-dry-jobs
```

Expected: dry-run succeeds.

- [ ] **Step 4: Create the queues in Cloudflare (deferred to operator)**

Before deploy to any environment, the `zra-customer-sync` (dev/staging) and `zra-customer-sync-prod` + `zra-customer-sync-prod-dlq` queues must exist in the Cloudflare account. This is operator work, but it must happen before the first deploy that includes Task 7.

Example commands (run by operator):

```
wrangler queues create zra-customer-sync
wrangler queues create zra-customer-sync-prod
wrangler queues create zra-customer-sync-prod-dlq
```

- [ ] **Step 5: Stage (do not commit)**

---

## Task 7: Replace `waitUntil` in customer handlers with the queue enqueue

**Files:**
- Modify: `apps/api-invoicing/src/routes/invoicing/customers/handlers.ts`

This task **reverts** the targeted pool-race fix (the `Pool` import + `bgPool` block) and replaces both `waitUntil(syncCustomerToZra(...))` blocks with `runAfterResponse(c, env.ZRA_CUSTOMER_SYNC_QUEUE.send(...))`. It also restores the real `createCustomer(...)` call and `customer.id` that were previously commented out / hardcoded — without the queue indirection there's no reason to keep the placeholder UUID, and the new flow requires the real customer.id.

- [ ] **Step 1: Replace imports**

Open `apps/api-invoicing/src/routes/invoicing/customers/handlers.ts`. Replace the current import block at the top (lines 1–35 in the current file) with:

```typescript
import {
  createCustomer,
  type CreateCustomerInput,
  deleteCustomer,
  generateCustomerStatementPdf,
  getCustomer,
  getCustomerAging,
  getCustomerStatement,
  listCustomers,
  syncCustomerToZra,
  type UpdateCustomerInput,
  updateCustomer,
} from "@repo/api-services/domains/invoicing";
import {
  buildServiceContext,
  buildServiceDependencies,
  handleServiceError,
  runAfterResponse,
} from "@repo/backend/context";
import type { AppRouteHandler } from "@repo/backend/types";
import * as HttpStatusCodes from "stoker/http-status-codes";
import type {
  CreateCustomerRoute,
  DeleteCustomerRoute,
  GetCustomerAgingRoute,
  GetCustomerRoute,
  GetCustomerStatementPdfRoute,
  GetCustomerStatementRoute,
  ListCustomersRoute,
  SyncCustomerToZraRoute,
  UpdateCustomerRoute,
} from "./routes";
```

Note what's gone: `Pool`, `DatabaseClient`, `drizzle`, `* as schema`, and the `biome-ignore` for the namespace import. The replacement adds `runAfterResponse` and `createCustomer`.

- [ ] **Step 2: Replace `createCustomerHandler`**

Replace the entire body of `createCustomerHandler` (the export starting `export const createCustomerHandler: AppRouteHandler<CreateCustomerRoute> = async (c) => {`) with:

```typescript
export const createCustomerHandler: AppRouteHandler<
  CreateCustomerRoute
> = async (c) => {
  try {
    const ctx = buildServiceContext(c);
    const deps = buildServiceDependencies(c);
    const body = await c.req.json<CreateCustomerInput>();
    const customer = await createCustomer(ctx, deps, body);

    const env = c.env as { ZRA_CUSTOMER_SYNC_QUEUE?: { send: (msg: unknown) => Promise<void> } };
    if (env.ZRA_CUSTOMER_SYNC_QUEUE && ctx.orgId) {
      runAfterResponse(
        c,
        env.ZRA_CUSTOMER_SYNC_QUEUE.send({
          customerId: customer.id,
          orgId: ctx.orgId,
          attempt: 1,
        }),
        "[ZRA Customer Sync] enqueue failed (create)"
      );
    }

    return c.json(
      {
        success: true,
        data: customer,
        message: "Customer created successfully",
      },
      HttpStatusCodes.CREATED
    );
  } catch (error) {
    return handleServiceError(c, error, "Failed to create customer");
  }
};
```

The presence-check on `env.ZRA_CUSTOMER_SYNC_QUEUE` keeps the handler safe in test envs and during the brief window between Task 7 merging and Task 6's wrangler config being deployed.

- [ ] **Step 3: Replace `updateCustomerHandler`**

Replace the entire body of `updateCustomerHandler` with:

```typescript
export const updateCustomerHandler: AppRouteHandler<
  UpdateCustomerRoute
> = async (c) => {
  try {
    const ctx = buildServiceContext(c);
    const deps = buildServiceDependencies(c);
    const { customerId } = c.req.valid("param");
    const body = await c.req.json<UpdateCustomerInput>();
    const customer = await updateCustomer(ctx, deps, customerId, body);

    const env = c.env as { ZRA_CUSTOMER_SYNC_QUEUE?: { send: (msg: unknown) => Promise<void> } };
    if (env.ZRA_CUSTOMER_SYNC_QUEUE && ctx.orgId) {
      runAfterResponse(
        c,
        env.ZRA_CUSTOMER_SYNC_QUEUE.send({
          customerId: customer.id,
          orgId: ctx.orgId,
          attempt: 1,
        }),
        "[ZRA Customer Sync] enqueue failed (update)"
      );
    }

    return c.json(
      {
        success: true,
        data: customer,
        message: "Customer updated successfully",
      },
      HttpStatusCodes.OK
    );
  } catch (error) {
    return handleServiceError(c, error, "Failed to update customer");
  }
};
```

Leave `syncCustomerToZraHandler` (the manual sync route, around line 243 in the current file) untouched. The manual endpoint stays synchronous for user-driven retries.

- [ ] **Step 4: Typecheck**

Run: `pnpm --filter api-invoicing typecheck`
Expected: passes for `customers/handlers.ts`. Preexisting errors in unrelated files are OK.

- [ ] **Step 5: Stage (do not commit)**

---

## Task 8: End-to-end smoke (deferred to operator)

After Task 6's queues are created in Cloudflare and all tasks are deployed to staging:

- [ ] **Step 1: Create a customer through staging**

Issue a `POST /invoicing/customers` against the staging api-invoicing worker. Expect `201` with a customer in the response body. Confirm the response returned in under 200ms (no waiting for ZRA).

- [ ] **Step 2: Verify the row eventually reaches `synced` (or `failed`)**

Within ~30s, query the customer row:

```sql
SELECT id, zra_sync_status, zra_synced_at, zra_sync_attempted_at, zra_sync_error
FROM customers WHERE id = '<new-customer-id>';
```

Expected: `zra_sync_status` is `synced` (org has a VSDC device) or `failed` (with an error string) or `skipped_no_device`. Not null.

- [ ] **Step 3: Verify retry path with a forced transient failure**

(Optional, operator-only.) Temporarily block the ZRA sandbox URL at the network edge or point the sandbox URL at a 503-returning endpoint, then create a customer. Expect: the row goes to `failed`, the message is redelivered up to 5 times, and after exhaustion lands in `zra-customer-sync-prod-dlq` (in prod) or just stops retrying (staging — no DLQ configured).

- [ ] **Step 4: Verify enqueue-failure path**

(Optional.) Temporarily revoke the worker's queue binding by editing `wrangler.toml` to point at a non-existent queue, deploy to dev, create a customer. Expect: `201` returns normally; log line `[ZRA Customer Sync] enqueue failed (create)` appears; row stays `zra_sync_status = null`. Restore the binding afterwards.

---

## Self-Review Checklist (for the plan author)

**Spec coverage:**
- `runAfterResponse` helper, package re-export, tests: Task 1 ✓
- `Env` extension + `ZraCustomerSyncMessage` interface: Task 2 ✓
- `classifyCustomerSyncFailure` + tests + barrel re-export: Task 3 ✓
- Consumer processor + type-guard + tests: Task 4 ✓
- Router wiring in `api-jobs/index.ts`: Task 5 ✓
- Wrangler bindings (producer in `api-invoicing`, producer + consumer in `api-jobs`, DLQ in prod): Task 6 ✓
- Handler replacement (revert Pool fix, add `runAfterResponse(c, env.QUEUE.send(...))`, restore real `createCustomer`): Task 7 ✓
- End-to-end operator smoke: Task 8 (deferred to operator) ✓

**Placeholder scan:** No "TBD" or "implement later". Operator steps in Task 6 (queue creation) and Task 8 (e2e smoke) are explicitly deferred, which is intentional, not placeholder behavior.

**Type consistency:**
- `ZraCustomerSyncMessage` fields (`customerId`, `orgId`, `attempt`) match across the type definition (Task 2), the type-guard (Task 4), the producer call (Task 7), and tests (Tasks 1 and 4).
- `CustomerSyncResult.status` is `'synced' | 'failed' | 'skipped_no_device'` — the classifier (Task 3) and the consumer (Task 4) both branch on this union; `classifyCustomerSyncFailure` covers all three.
- `runAfterResponse(c, promise, label?)` signature is consistent across the test (Task 1), the helper (Task 1), and both call sites (Task 7).

**Ordering:**
- Task 1 (helper) must come before Task 7 (handler uses it).
- Task 2 (Env type) must come before Task 4 (processor imports the type) and before Task 7 (handler accesses `env.ZRA_CUSTOMER_SYNC_QUEUE`).
- Task 3 (classifier) must come before Task 4 (processor imports it).
- Task 4 (processor) must come before Task 5 (router imports it).
- Task 6 (wrangler) and Task 7 (handler change) can be merged in either order — the handler's presence-check on `env.ZRA_CUSTOMER_SYNC_QUEUE` allows handler-first deploys.

**Reversibility:**
- If anything goes wrong post-merge, reverting Task 7 alone restores the targeted pool-race fix (still safer than the original `waitUntil` race).
- Reverting Task 6 stops messages flowing without code changes.
