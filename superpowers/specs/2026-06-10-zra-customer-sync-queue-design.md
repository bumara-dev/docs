---
title: "ZRA Customer Sync — Queue-Backed Design"
description: "Date: 2026-06-10 Status: Approved, pending implementation plan Supersedes (partially): 2026-06-08-zra-customer-sync-design.md — Task 5 (waitUntil block..."
---

**Date:** 2026-06-10
**Status:** Approved, pending implementation plan
**Supersedes (partially):** [2026-06-08-zra-customer-sync-design.md](/superpowers/specs/2026-06-08-zra-customer-sync-design) — Task 5 (`waitUntil` block in create/update handlers) is replaced by a Cloudflare Queue producer/consumer flow.

## Background

The initial ZRA customer sync (spec dated 2026-06-08) fires `syncCustomerToZra` via `c.executionCtx.waitUntil(...)` inside `createCustomerHandler` / `updateCustomerHandler`. In practice this raced against the database middleware's own `c.executionCtx.waitUntil(pool.end())` and threw `Cannot use a pool after calling end on the pool` mid-`UPDATE`. A targeted fix — giving the background sync its own DB pool — addressed the symptom but left the fundamental constraints in place:

- `waitUntil` has a ~30s cap.
- `waitUntil` has no retries.
- Failures are invisible outside server logs.

ZRA submissions need to *eventually succeed*. The right substrate is a queue with retries and a DLQ — a substrate the project already uses for `notification-outbox` and `notification-delivery`. This spec defines the queue-backed replacement and a small reusable handler helper (`runAfterResponse`) that will also serve future invoice / item sync flows.

## Goals

1. Replace `waitUntil(syncCustomerToZra(...))` in `createCustomerHandler` and `updateCustomerHandler` with a fire-and-forget enqueue onto a dedicated Cloudflare Queue.
2. Process the queue in `api-jobs` on its own fresh DB connection, with built-in retries on transient failures and DLQ landing on persistent failures.
3. Provide a centralized `runAfterResponse(c, promise, label?)` helper for any post-response work, applied first to the queue enqueue.
4. Keep the customer mutation independent of ZRA / queue availability — a 5xx from Cloudflare Queues must not make the customer create fail.
5. Preserve the existing manual sync endpoint (`POST /invoicing/customers/:id/sync-zra`) as the user-driven retry path.

## Non-Goals

- A reconciliation cron for rows stuck at `zra_sync_status = null` after enqueue failure (left for a follow-up spec; manual sync endpoint covers the gap meanwhile).
- Operator UI / endpoint to replay messages from the DLQ.
- Invoice / item / vendor sync queues. The design generalizes to per-entity queues so adding them is mechanical, but each gets its own spec.
- Changes to the `ZraApiClient` or `syncCustomerToZra` service signature beyond adding a small failure classifier.

## Architecture

```
                                                  +----------------------------+
POST /invoicing/customers                         |  api-jobs worker            |
        |                                         |                            |
        v                                         |  queue() handler routes by |
+--------------------+                            |  message type, runs        |
| api-invoicing      |                            |  processZraCustomerSync... |
| createCustomer     |                            |                            |
| (writes customer   |  send(...)                 |  createDbClient(env...)    |
|  row)              | -----------------------+   |  loop messages:            |
| runAfterResponse(  |                        |   |    syncCustomerToZra()     |
|   c,               |   ZRA_CUSTOMER_SYNC_   |   |    classify failure        |
|   env.QUEUE.send())|   QUEUE (Cloudflare)   +-->|    throw -> retry          |
| returns 201        | ---------------------->|   |    else   -> ack           |
+--------------------+                        |   |                            |
                                              |   |  DLQ after max_retries (5) |
                                              |   +----------------------------+
                                              |
                                       +------v---------+
                                       | DLQ            |
                                       | zra-customer-  |
                                       | sync-prod-dlq  |
                                       +----------------+
```

Per-entity queue chosen over a generic ZRA sync queue: each ZRA entity (customers, invoices, items) gets its own queue, DLQ, retry budget, and back-pressure characteristics. The producer/consumer pattern below is the template each future entity will copy.

## Components

### 1. `runAfterResponse(c, promise, label?)` — shared helper

**Location:** `packages/backend/src/core/context/run-after-response.ts`. Re-exported from `@repo/backend/context`.

**Signature:**

```ts
export function runAfterResponse<T>(
  c: Context<AppBindings>,
  promise: Promise<T>,
  label = "[runAfterResponse] background task failed"
): void;
```

**Behavior:**

- Detects `c.executionCtx?.waitUntil` (cast via `c as { executionCtx?: { waitUntil: (p: Promise<unknown>) => void } }`).
- When present: schedules `promise.catch(err => logger?.error?.({ err }, label) ?? console.error(label, err))`.
- When absent (test / Node env): attaches the same `.catch(...)` but does not block; the promise still runs because the caller already constructed it.
- Uses `c.get("logger")` if available, falling back to `console.error`.

**Why a `Promise` (not `() => Promise`):** the caller already builds the promise — keeping the helper concrete keeps the call sites short. The lazy variant was rejected as more verbose at every site for no observed benefit.

### 2. `Env` extension and message type

**Location:** `packages/api-services/src/core/context/context.ts`.

```ts
export interface ZraCustomerSyncMessage {
  customerId: string;
  orgId: string;
  attempt: number; // producer-side counter; informational only — Cloudflare's
                   // own delivery-count is authoritative for retries
}

export interface Env {
  // ...existing fields...
  ZRA_CUSTOMER_SYNC_QUEUE: Queue<ZraCustomerSyncMessage>;
}
```

### 3. Wrangler bindings

**`apps/api-invoicing/wrangler.toml`** (producer) — add to every env (`[dev]`, `[env.staging]`, `[env.production]`):

```toml
[[env.<env>.queues.producers]]
binding = "ZRA_CUSTOMER_SYNC_QUEUE"
queue   = "zra-customer-sync"          # zra-customer-sync-prod in [env.production]
```

The root (default-env) `[dev]` block also needs `[[queues.producers]]` with `queue = "zra-customer-sync"` to match the existing root-level `[browser]` binding pattern.

**`apps/api-jobs/wrangler.toml`** (consumer + producer):

```toml
[[env.<env>.queues.producers]]
binding = "ZRA_CUSTOMER_SYNC_QUEUE"
queue   = "zra-customer-sync"          # zra-customer-sync-prod in [env.production]

[[env.<env>.queues.consumers]]
queue          = "zra-customer-sync"   # zra-customer-sync-prod in [env.production]
max_batch_size = 10
max_retries    = 5

# Production only:
# dead_letter_queue = "zra-customer-sync-prod-dlq"
```

Producer binding inside `api-jobs` is included so a future cron-based reconciliation worker (out of scope) can re-enqueue stuck rows without crossing worker boundaries.

### 4. Consumer processor

**Location:** `apps/api-jobs/src/queues/zra-customer-sync-processor.ts`.

**Exports:**

```ts
export function isZraCustomerSyncMessage(msg: unknown): msg is ZraCustomerSyncMessage;

export async function processZraCustomerSyncQueue(
  batch: MessageBatch<ZraCustomerSyncMessage>,
  env: Env,
  ctx: ExecutionContext
): Promise<void>;
```

**Body shape:**

```ts
// One DB client for the whole batch — same pattern as outbox-processor.ts
const db = createDbClient({ databaseUrl: env.DATABASE_URL });

for (const message of batch.messages) {
  const { customerId, orgId, attempt } = message.body;
  const ctx: ServiceContext = {
    ...systemServiceContext({ env, source: 'zra-customer-sync-queue' }),
    orgId,
  };
  const deps: ServiceDependencies = { db };

  try {
    const result = await syncCustomerToZra(ctx, deps, customerId);
    if (result.status === "failed") {
      const classification = classifyCustomerSyncFailure(result);
      if (classification === "transient") {
        // Throw to make Queues retry; row already marked 'failed' by the service.
        throw new Error(`[ZRA-CUST-SYNC] transient failure: ${result.error}`);
      }
    }
    message.ack();
  } catch (err) {
    if (err instanceof ServiceError && err.code === "NOT_FOUND") {
      // Customer deleted since enqueue — nothing to do.
      console.warn("[ZRA-CUST-SYNC] customer not found, acking", { customerId, orgId });
      message.ack();
      continue;
    }
    // Any other thrown error -> let Queues redeliver. Log for visibility.
    console.error("[ZRA-CUST-SYNC] processing error, will retry", {
      customerId,
      orgId,
      attempt,
      deliveryAttempt: message.attempts,
      err,
    });
    throw err;
  }
}
```

### 5. `classifyCustomerSyncFailure` helper

**Location:** Same file as the service — `packages/api-services/src/domains/invoicing/zra/zra-customer-sync.service.ts`.

```ts
export function classifyCustomerSyncFailure(
  result: CustomerSyncResult
): "transient" | "permanent" {
  if (result.status === "skipped_no_device") return "permanent";
  if (result.status === "synced") return "permanent"; // shouldn't be called
  const err = result.error ?? "";
  if (/HTTP 5\d\d|ETIMEDOUT|ECONNRESET|ECONNREFUSED|fetch failed|HTTP 429/i.test(err)) {
    return "transient";
  }
  return "permanent";
}
```

Living next to the service keeps failure-mode knowledge in one place — the consumer doesn't need to know how ZRA error strings look.

### 6. `api-jobs/src/index.ts` routing

Extend the union and add a branch:

```ts
async queue(
  batch: MessageBatch<OutboxQueueMessage | DeliveryQueueMessage | ZraCustomerSyncMessage>,
  env: Env,
  ctx: ExecutionContext
) {
  if (batch.messages.length === 0) return;
  const first = batch.messages[0]?.body;

  if (isOutboxQueueMessage(first)) {
    await processOutboxQueue(batch as MessageBatch<OutboxQueueMessage>, env, ctx);
  } else if (isDeliveryQueueMessage(first)) {
    await processDeliveryQueue(batch as MessageBatch<DeliveryQueueMessage>, env, ctx);
  } else if (isZraCustomerSyncMessage(first)) {
    await processZraCustomerSyncQueue(batch as MessageBatch<ZraCustomerSyncMessage>, env, ctx);
  } else {
    console.error("[QUEUE] Unknown message type:", first);
  }
}
```

### 7. Handlers in `apps/api-invoicing/src/routes/invoicing/customers/handlers.ts`

Revert the `Pool` / `drizzle` / `schema` imports introduced by the targeted fix. The new shape:

```ts
import { runAfterResponse } from "@repo/backend/context";

export const createCustomerHandler = async (c) => {
  try {
    const ctx = buildServiceContext(c);
    const deps = buildServiceDependencies(c);
    const body = await c.req.json<CreateCustomerInput>();
    const customer = await createCustomer(ctx, deps, body);

    runAfterResponse(
      c,
      c.env.ZRA_CUSTOMER_SYNC_QUEUE.send({
        customerId: customer.id,
        orgId: ctx.orgId!,
        attempt: 1,
      }),
      "[ZRA Customer Sync] enqueue failed (create)"
    );

    return c.json({ success: true, data: customer, message: "Customer created successfully" }, HttpStatusCodes.CREATED);
  } catch (error) {
    return handleServiceError(c, error, "Failed to create customer");
  }
};
```

`updateCustomerHandler` mirrors this exactly (different label).

**Note:** the previous spec's Task 5 left a hardcoded UUID and a commented-out `createCustomer` call in the handler. The new handler restores the real `createCustomer` and uses `customer.id`. Reverts the targeted pool-race fix (`Pool` import + `bgPool` block) entirely.

## Data Flow

**Happy path:**

1. `POST /invoicing/customers` → handler runs `createCustomer` → row inserted with `zra_sync_status = null`.
2. Handler calls `runAfterResponse(c, env.ZRA_CUSTOMER_SYNC_QUEUE.send({...}))`. Send is single-digit ms.
3. Handler returns 201. Request-scoped pool ends normally — no DB work in flight.
4. Cloudflare invokes `api-jobs` `queue()`. Router → `processZraCustomerSyncQueue`. One DB connection per batch.
5. `syncCustomerToZra` runs → `findCustomerById`, payload build, `ZraApiClient.saveBranchCustomer`, mark `synced`. Returns `{ status: 'synced' }`. Consumer acks.

**Transient failure (5xx / 429 / network):**

1. Steps 1–4 as above.
2. Service returns `{ status: 'failed', error: 'HTTP 503' }`. Row marked `failed` with error string (service's never-throws invariant preserved).
3. Consumer's classifier → `'transient'` → throws. Queues schedules redelivery with exponential backoff.
4. After 5 attempts, message lands in `zra-customer-sync-prod-dlq`. Row remains `failed`. Operator replays via manual `POST /invoicing/customers/:id/sync-zra` or (future) DLQ replay tool.

**Permanent failure (4xx, `NOT_FOUND`, `skipped_no_device`):**

1. Steps 1–4 as above.
2. Service returns `{ status: 'failed', error: 'HTTP 400 invalid TPIN' }` or `{ status: 'skipped_no_device' }`, or throws `NOT_FOUND`.
3. Classifier → `'permanent'` (or `NOT_FOUND` caught explicitly). Consumer acks. Row stays `failed` / `skipped_no_device`. UI shows `zra_sync_error`.

**Enqueue failure:**

1. `env.ZRA_CUSTOMER_SYNC_QUEUE.send(...)` rejects.
2. `runAfterResponse`'s `.catch` logs `[ZRA Customer Sync] enqueue failed (create) { err }`.
3. Response is already 201. Row stays at `zra_sync_status = null`. Picked up later by manual sync endpoint.

## Edge Cases

- **Idempotency:** ZRA's `saveBrancheCustomers` is an upsert on `(tpin, custNo)`. Re-delivery is safe — repeats only refresh `zra_synced_at`.
- **Update fires during pending sync:** Both messages drain. Each consume reads current customer state via `findCustomerById`, so the later message reflects the latest customer state; ZRA upsert reconciles. No locking.
- **Customer deleted between enqueue and consume:** `syncCustomerToZra` throws `NOT_FOUND`. Consumer catches, acks. No update needed.
- **Batch partial failures:** Loop is per-message. One throw doesn't abort siblings — Queues retries only the throwing message.
- **Missing `orgId` on ctx (cross-tenant edge):** Producer always populates `orgId` from `ctx.orgId` at enqueue time, which is required by `requireOrganizationContext` upstream. The consumer builds its `ServiceContext` directly from the message — no auth middleware involved.

## Error Handling Summary

| Source                              | Action                                                      |
|-------------------------------------|-------------------------------------------------------------|
| `env.QUEUE.send` rejects            | `runAfterResponse` logs; response unaffected; row stays null|
| ZRA HTTP 5xx / 429 / network        | Service marks row `failed`; consumer throws; Queues retries |
| ZRA HTTP 4xx / validation           | Service marks row `failed`; consumer acks                   |
| `findCustomerById` returns null     | Service throws `NOT_FOUND`; consumer acks                   |
| `skipped_no_device`                 | Service marks row; consumer acks                            |
| Unexpected error in consumer        | Caught at loop boundary; re-thrown; Queues retries          |
| Exhausted retries                   | Message lands in DLQ; row remains `failed`                  |

## Testing

**Consumer (`apps/api-jobs/src/queues/__tests__/zra-customer-sync-processor.test.ts`):**

- Processes a success message and calls `message.ack()`.
- Re-throws on transient failure (HTTP 503) — `ack` not called for that message.
- Acks on permanent failure (HTTP 400).
- Acks on `NOT_FOUND` ServiceError without throwing.
- Mixed batch: success / transient / permanent — independent per-message outcomes.
- `createDbClient` called exactly once per batch.

**Classifier (extend `packages/api-services/src/domains/invoicing/zra/__tests__/zra-customer-sync.service.test.ts`):**

- 5xx codes (500, 502, 503, 504) → `'transient'`.
- 429 → `'transient'`.
- Network strings (`ETIMEDOUT`, `ECONNRESET`, `ECONNREFUSED`, `fetch failed`) → `'transient'`.
- 4xx codes (400, 401, 403, 404, 422) → `'permanent'`.
- `skipped_no_device` → `'permanent'`.
- Empty / undefined error → `'permanent'` (no signal it's transient).

**Helper (`packages/backend/src/core/context/__tests__/run-after-response.test.ts`):**

- Passes promise to `executionCtx.waitUntil` when present.
- Logs via `c.get("logger")` if available; falls back to `console.error`.
- Catches rejection so unhandled rejections never escape.
- No-ops gracefully when `executionCtx` is absent.

**Handlers:** no new tests beyond what the previous spec already added. The enqueue is one line; the substantive behavior is covered by the processor tests.

## Self-Review Checklist

- **Placeholders:** none — all sections concrete.
- **Internal consistency:** `Env.ZRA_CUSTOMER_SYNC_QUEUE` is referenced by producer, message type, consumer routing, and handler. Producer-side queue name (`zra-customer-sync` / `zra-customer-sync-prod`) is consistent.
- **Scope:** focused on a single user-visible flow (customer create / update → ZRA sync). The `runAfterResponse` helper is in-scope because the only safe way to enqueue from a request handler is through this helper; it's not a separate refactor.
- **Ambiguity:** error classification is a regex-based whitelist. Documented explicitly so the consumer's behavior is predictable.
- **Reversibility:** if Queues itself misbehaves, reverting is one handler edit (call `syncCustomerToZra` directly with the pool fix) plus removing the queue binding. The service signature is unchanged.

## Supersession Notes

- The 2026-06-08 spec's Task 5 (`waitUntil` block) is replaced by this design.
- The 2026-06-08 spec's Task 6 (manual sync endpoint) is **unchanged** and remains the user-driven retry path.
- The 2026-06-08 spec's Task 7 (apply migration) is unchanged.
- The targeted pool-race fix in `customers/handlers.ts` (`Pool` import + `bgPool` block) is reverted by this design.
