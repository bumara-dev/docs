---
title: "Inventory Module"
description: "Stock tracking for Zambian SMEs with audit-safe ledger, operations, and low-stock alerts."
---

The Inventory Module enables businesses to track stock accurately across one or more locations, with an immutable stock ledger, simple operations (receive, issue, adjust, transfer, count), and low-stock visibility with alerts.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          INVENTORY MODULE                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │                         TENANT APP (apps/app)                         │  │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────┐   │  │
│   │  │  Dashboard  │  │   Items     │  │   Stock     │  │ Operations │   │  │
│   │  │  /inventory │  │   /items    │  │   /stock    │  │ adj/xfer/  │   │  │
│   │  │             │  │             │  │   /movements│  │ count      │   │  │
│   │  └─────────────┘  └─────────────┘  └─────────────┘  └────────────┘   │  │
│   └──────────────────────────────────────────────────────────────────────┘  │
│                                      │                                       │
│                                      ▼                                       │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │                    HONO API (packages/backend)                        │  │
│   │  /inventory/items │ /inventory/stock │ /inventory/adjustments │ ...  │  │
│   └──────────────────────────────────────────────────────────────────────┘  │
│                                      │                                       │
│                                      ▼                                       │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │                 SERVICES (packages/api-services)                      │  │
│   │  inventory.service.ts │ stock-ledger.service.ts │ stock-ops.service  │  │
│   └──────────────────────────────────────────────────────────────────────┘  │
│                                      │                                       │
│                                      ▼                                       │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │                   DATABASE (packages/database)                        │  │
│   │  ┌─────────────────┐  ┌──────────────────┐  ┌────────────────────┐   │  │
│   │  │ inventory_items │  │ stock_movements  │  │  stock_balances    │   │  │
│   │  │ inventory_units │  │ (immutable       │  │  (cached/derived)  │   │  │
│   │  │ locations       │  │  ledger)         │  │                    │   │  │
│   │  └─────────────────┘  └──────────────────┘  └────────────────────┘   │  │
│   │                                                                       │  │
│   │  ┌──────────────────────────────────────────────────────────────┐    │  │
│   │  │  Operations: adjustments │ transfers │ counts (with lines)   │    │  │
│   │  └──────────────────────────────────────────────────────────────┘    │  │
│   └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                        INTEGRATION POINTS                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                   │
│  │  Documents   │    │    Tasks     │    │ Notifications│                   │
│  │  (Evidence)  │    │  (Reorder)   │    │  (Alerts)    │                   │
│  │              │    │              │    │              │                   │
│  │ Attach GRNs, │    │ Auto-create  │    │ Low stock,   │                   │
│  │ count sheets │    │ reorder task │    │ count posted │                   │
│  └──────────────┘    └──────────────┘    └──────────────┘                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Core Principle:** Stock movements form an immutable ledger. Balances are derived/cached for performance. All operations create audit trails.

---

## Module Scope

### MVP (Phase 1) — TO BE IMPLEMENTED

- [ ] Items (products), categories, units of measure (UoM)
- [ ] Locations (warehouse/store)
- [ ] Stock Ledger (`stock_movements`) + real-time balance (`stock_balances`)
- [ ] Stock Adjustment (increase/decrease with reason)
- [ ] Stock Transfer (location A → B)
- [ ] Stock Count (cycle/full count with variance posting)
- [ ] Low stock thresholds + notifications
- [ ] Attachments/evidence via Documents module
- [ ] Permissions & audit logs
- [ ] Idempotent posting (prevent double-post on retry)

### Phase 2 (Future)

- [ ] Suppliers + Purchases (PO → Receive → Supplier Invoice)
- [ ] Sales integration (Invoice → issue stock)
- [ ] Costing/valuation improvements (moving average, FIFO)
- [ ] Barcode scanning + SKU enforcement
- [ ] Batches/expiry dates (pharma/food)

---

## Documentation Index

| Document | Description |
|----------|-------------|
| [01-overview.md](/inventory/01-overview) | Problem statement, goals, personas, glossary |
| [02-architecture.md](/inventory/02-architecture) | System design, directory structure, integration diagrams |
| [03-data-model.md](/inventory/03-data-model) | Database tables, enums, indexes, concurrency strategy |
| [04-api-spec.md](/inventory/04-api-spec) | API endpoint contracts (Hono RPC + Zod) |
| [05-workflows.md](/inventory/05-workflows) | State machines for operations (adjust, transfer, count) |
| [06-ui-ux.md](/inventory/06-ui-ux) | Page structure, components, forms, permissions UI |
| [07-events-notifications.md](/inventory/07-events-notifications) | Event catalog, notification routing, alerts |
| [08-testing.md](/inventory/08-testing) | Test strategy, fixtures, scenarios |
| [09-rollout-migrations.md](/inventory/09-rollout-migrations) | Migration plan, feature flags, observability |

### Related Documentation

| Document | Description |
|----------|-------------|
| [Documents Module](/modules/documents) | File attachments for evidence |
| [Tasks Module](/modules/tasks) | Task auto-creation patterns |
| [Notifications Module](/modules/notifications) | Alert delivery |
| [Database Schema](/ARCHITECTURE/DATABASE_SCHEMA) | Full database architecture |
| [API Setup](/API-SETUP) | Hono backend setup guide |

---

## Quick Reference

### Item Statuses

| Status | Description |
|--------|-------------|
| `active` | Available for stock operations |
| `archived` | Hidden from lists, no new movements allowed |

### Movement Types

| Type | Description | Source |
|------|-------------|--------|
| `ADJUSTMENT_IN` | Stock increase via adjustment | Adjustment |
| `ADJUSTMENT_OUT` | Stock decrease via adjustment | Adjustment |
| `TRANSFER_OUT` | Stock leaves source location | Transfer |
| `TRANSFER_IN` | Stock arrives at destination | Transfer |
| `COUNT_VARIANCE_IN` | Positive variance from count | Count |
| `COUNT_VARIANCE_OUT` | Negative variance from count | Count |

### Operation Statuses

| Operation | Statuses |
|-----------|----------|
| Adjustment | `draft` → `posted` \| `void` |
| Transfer | `draft` → `in_transit` → `received` \| `void` |
| Count | `draft` → `in_progress` → `completed` → `posted` \| `void` |

### Location Types

| Type | Description |
|------|-------------|
| `warehouse` | Primary storage facility |
| `store` | Retail/sales location |
| `van` | Mobile/delivery vehicle |
| `other` | Custom location type |

---

## Key File Locations (TO BE IMPLEMENTED)

| Purpose | Location |
|---------|----------|
| Database schema | `packages/database/src/schema/inventory/` |
| Backend routes | `packages/backend/src/modules/inventory/` |
| Services | `packages/inventory/src/` |
| Frontend pages | `apps/app/app/(authenticated)/(modules)/inventory/` |
| React Query hooks | `apps/app/lib/queries/inventory/` |
| Module config | `apps/app/config/modules.ts` (already registered) |
| Secondary sidebar | `apps/app/config/secondary-sidebar/inventory-sidebar.ts` |

---

## Environment Variables

```bash
# No new environment variables required for MVP
# Inventory uses existing:
# - DATABASE_URL (Postgres/Neon)
# - NEXT_PUBLIC_API_URL (Backend API)
# - Clerk auth (existing)
# - Knock notifications (existing)
```

---

## Getting Started

1. Read [01-overview.md](/inventory/01-overview) to understand goals and scope
2. Review [03-data-model.md](/inventory/03-data-model) for database design
3. Check [04-api-spec.md](/inventory/04-api-spec) for API contracts
4. Follow [05-workflows.md](/inventory/05-workflows) for operation flows
5. Use [09-rollout-migrations.md](/inventory/09-rollout-migrations) for implementation steps

---

## Open Questions

1. **Negative stock policy**: Should negative stock be blocked by default, or configurable per org?
2. **Costing in MVP**: Should `unit_cost` on movements be required or optional?
3. **Reserved quantity**: Include in MVP for sales integration, or defer to Phase 2?
4. **Backoffice access**: Should backoffice staff have read-only inventory access for support?
