---
title: "Invoicing Module: ZRA Transmission Modes"
description: "How approved sales invoices reach ZRA: the inline real-time path, and the scheduled queue-backed path with per-org cadence."
---

Every organization chooses how its smart sales invoices reach ZRA:

- **Real-time** (default): the ZRA `saveSales` call runs inline the moment a user approves an invoice.
- **Scheduled**: approval only writes a pending transmission row. A cron sweep fans pending rows onto a Cloudflare Queue, and a consumer makes the ZRA call out of band.

The choice is stored on `invoice_settings.zra_transmission_mode` (`'realtime' | 'scheduled'`), guarded by a CHECK constraint and defaulted to `realtime`, so existing orgs see no behavior change until they opt in.

<Warning>
**Implementation status on this branch.** The data model and the settings API below are
live here. The scheduled path is only partly landed: `createPendingSalesTransmission`
exists in `packages/invoicing/src/zra/smart-invoice.ts`, but `approveInvoice` still calls
`transmitInvoiceToZra` unconditionally, and `apps/api-jobs` has no sweep job, no
`SMART_INVOICE_SALES_QUEUE` binding, and no queue consumer. The cron sweep, the cadence
gate helpers, and the approve-time branch land with the `compliance-test` branch. Treat the
scheduled sections below as the design of record, not as behavior you can exercise here.
</Warning>

**Scope.** Smart sales invoices only (`transmission_type='sales'`, `document_type='sales_invoice'`). Provisional, Final, and Commercial invoices each have their own transmit orchestrator and are untouched by these modes. See [Mining export invoices](/invoicing/08-mining-export-invoices).

---

## Real-time mode

```
User approves invoice
        │
        ▼
approveInvoice(invoiceId)
        │
        ├─ sales_invoices.status = 'approved'
        ├─ recordLedgerEntry (debtor)
        ├─ deductInventoryForInvoice (side effect, non-blocking)
        │
        └─ transmitInvoiceToZra(ctx, deps, invoiceId)      ← inline
               │
               ├─ findInvoiceById / findTransmissionByInvoice
               ├─ findVsdcDevice (must be initialized)
               ├─ listInvoiceLineItems
               ├─ buildSalesPayloadFromInvoice
               │
               ├─ reuse an existing pending/failed row if present,
               │  else createTransmission({ status: 'pending', retry_count: 0 })
               │
               ├─ ZraApiClient.saveSales(payload) ─────► ZRA VSDC
               │
               ├─ success → updateTransmission({ status:'transmitted', receiptNo, sigs, qr })
               └─ failure → updateTransmission({ status:'failed',
                                                 retryCount: prior + 1,
                                                 rejectionReason })
```

**Timing.** The HTTP request blocks until `saveSales` returns: roughly 500ms to 1s against a healthy ZRA sandbox, and it hangs during a ZRA outage.

**Failure behavior.** A ZRA rejection does not block approval. The error is caught and logged, the invoice stays approved, and the failed transmission row surfaces on the invoice detail page with its `rejectionReason`. Retrying uses the legacy `handleFailedRetry` path, which applies exponential backoff (`2^retryCount` minutes) up to `MAX_RETRIES`.

---

## Scheduled mode

Two workers cooperate: the web request creates the pending row, and `api-jobs` (Cloudflare cron plus queue consumer) talks to ZRA.

### 1. Approval creates a pending row

`approveInvoice` reads the mode and, when it is `scheduled`, calls `createPendingSalesTransmission` instead of transmitting. That helper is idempotent: an existing pending row is returned as-is, and an already-validated row raises `BAD_REQUEST`. It builds the full wire payload up front and caches it in `raw_request`, then inserts a row with `transmission_status='pending'` and `retry_count=0`. No ZRA call happens, so the request returns in tens of milliseconds.

### 2. A cron sweep enqueues pending rows

The sweep runs on the `*/5 * * * *` trigger in `api-jobs`, alongside `runPaymentReconciliationJob`. It selects sweepable rows by joining transmissions to `invoice_settings`:

```sql
SELECT ... FROM zra_smart_invoice_transmissions t
INNER JOIN invoice_settings s ON s.organization_id = t.organization_id
WHERE t.transmission_status IN ('pending','failed')
  AND t.document_type     = 'sales_invoice'
  AND t.transmission_type = 'sales'
  AND s.zra_transmission_mode = 'scheduled'
  AND t.retry_count < 5
LIMIT 500
```

Each row is sent to the `smart-invoice-sales` queue as `{ salesInvoiceId, orgId, attempt }`.

### Cadence

The Cloudflare cron tick is fixed at every 5 minutes. What varies per org is which ticks actually enqueue that org's rows, controlled by `invoice_settings.zra_transmission_cadence`:

- **`interval`** (default): the sweep fires when `now >= last_sweep_at + zra_transmission_interval_minutes`. Options are 5, 15, 30, and 60 minutes. Existing orgs default to 5 and behave as before.
- **`daily`**: the sweep fires on the first tick at or after `zra_transmission_daily_time` in Africa/Lusaka time (`HH:MM`, stored as a string and evaluated in memory as UTC+2).

`zra_transmission_last_sweep_at` is written by the sweep after enqueuing and anchors both gates on the next tick.

### 3. The queue consumer calls ZRA

The consumer creates one DB client per batch, not per message, and calls `transmitInvoiceToZra` for each message. That reuses the pending row from step 1 rather than inserting a duplicate: it refreshes `raw_request`, sets status back to `pending`, and calls ZRA.

**Ack policy.** Every message is acked, whether ZRA succeeded, rejected, or threw. The sweep is the retry loop: a `failed` row with `retry_count < 5` reappears on the next qualifying tick. The Cloudflare DLQ is a safety net for genuinely unrecoverable message-level errors only.

**Retry cap.** Once `retry_count >= 5` the sweep filter stops re-picking the row. The invoice detail page shows the last `rejectionReason` and a manual Retry button, which uses the legacy path and resets counters.

---

## Data model

| Table | Column | Purpose |
|-------|--------|---------|
| `invoice_settings` | `zra_transmission_mode` | `'realtime' \| 'scheduled'`, NOT NULL, default `'realtime'`, CHECK constrained. |
| `invoice_settings` | `zra_transmission_cadence` | `'interval' \| 'daily'`, CHECK constrained. Only meaningful in scheduled mode. |
| `invoice_settings` | `zra_transmission_interval_minutes` | Interval gate: 5, 15, 30, or 60. |
| `invoice_settings` | `zra_transmission_daily_time` | Daily gate: `HH:MM` in Africa/Lusaka. |
| `invoice_settings` | `zra_transmission_last_sweep_at` | Anchor for both gates, written by the sweep. |
| `zra_smart_invoice_transmissions` | existing columns | Reused unchanged: `transmission_status`, `transmission_type`, `document_type`, `retry_count`, `raw_request`, `rejection_reason`. |

A partial index on `transmission_status` (`WHERE transmission_status IN ('pending','failed')`) keeps the sweep query cheap.

## Files that carry the flow

| Concern | File |
|---------|------|
| Settings columns | `packages/database/src/schema/invoicing/invoice-settings.ts` |
| Settings service | `packages/invoicing/src/settings/service.ts` |
| Approve branch | `packages/invoicing/src/invoices/service.ts` (`approveInvoice`) |
| Pending helper and transmit | `packages/invoicing/src/zra/smart-invoice.ts` |
| HTTP routes | `apps/api-invoicing/src/modules/settings/routes.ts` (`GET`/`PUT /invoicing/settings`) |
| Scheduler wiring | `apps/api-jobs/src/jobs/scheduler.ts` |

## Operations

- **Flip an org.** Update `zra_transmission_mode` through `PUT /invoicing/settings`. No downtime.
- **Find stuck rows.** `SELECT * FROM zra_smart_invoice_transmissions WHERE transmission_status='failed' AND retry_count >= 5;` These are never re-picked by the sweep. Reset `retry_count = 0` or use the invoice's Retry button.
- **Observe the sweep.** Cloudflare Worker logs for `bumara-api-jobs`: `[SWEEP-ZRA] found sweepable rows` carries the count, and the consumer logs `[SMART-SALES-QUEUE] handler invoked` plus `[SMART-SALES-QUEUE] transmit failed` on ZRA errors.
- **Deployment prerequisite.** The Cloudflare Queue objects must exist per environment before any org is switched to scheduled mode (`wrangler queues create <name>`).

## Known follow-ups

- No `ORDER BY` on the sweep, so more than 500 pending rows means DB-arbitrary starvation.
- Overlapping ticks (a sweep taking longer than 5 minutes) can enqueue a row twice; there is no claim or dedup mechanism yet.
- The partial index covers `transmission_status` alone; a compound index would prune more aggressively.
- Genuine reconciliation against ZRA (`selectTrnsSalesList` and diffing) is a separate piece of work.
- Provisional, Final, and Commercial invoices could adopt the same mode switch, but each needs its own orchestrator change.
