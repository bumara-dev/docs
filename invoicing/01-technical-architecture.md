---
title: "Invoicing Module — Technical Architecture"
description: "The invoicing module's architecture: tech stack, package structure, multi-tenancy, request flow, and ZRA Smart Invoice compliance."
---

## System Overview

The invoicing module is a full-stack, multi-tenant invoicing system built on a monorepo architecture. It handles the complete invoicing lifecycle — from quotation to payment reconciliation — with ZRA Smart Invoice compliance for Zambian tax regulations.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Frontend (Next.js)                       │
│  apps/app                                                       │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │
│  │  Pages   │ │Components│ │  Hooks   │ │ Fetchers │           │
│  │ (Routes) │ │  (UI)    │ │(Queries) │ │ (API)    │           │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘           │
│       └─────────────┴────────────┴─────────────┘                │
│                          │ HTTP/REST                            │
└──────────────────────────┼──────────────────────────────────────┘
                           │
┌──────────────────────────┼──────────────────────────────────────┐
│                  API Worker (Hono/Cloudflare)                   │
│  apps/api-invoicing                                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────┐                │
│  │  Routes  │ │ Handlers │ │  OpenAPI Schema  │                │
│  │ (Hono)   │ │ (Logic)  │ │  (Zod Validation)│                │
│  └────┬─────┘ └────┬─────┘ └────────┬─────────┘                │
│       └─────────────┴────────────────┘                          │
│                          │                                      │
└──────────────────────────┼──────────────────────────────────────┘
                           │
┌──────────────────────────┼──────────────────────────────────────┐
│                    Service Layer                                │
│  packages/invoicing/src                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │
│  │ Invoice  │ │ Payment  │ │   PDF    │ │   ZRA    │           │
│  │ Services │ │ Services │ │ Services │ │ Services │           │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘           │
│       └─────────────┴────────────┴─────────────┘                │
│                          │                                      │
└──────────────────────────┼──────────────────────────────────────┘
                           │
┌──────────────────────────┼──────────────────────────────────────┐
│                    Data Layer                                   │
│  packages/database                                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                        │
│  │  Schema  │ │  Repos   │ │Migrations│                        │
│  │(Drizzle) │ │ (Queries)│ │  (SQL)   │                        │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘                        │
│       └─────────────┴────────────┘                              │
│                          │                                      │
│                    PostgreSQL                                   │
└─────────────────────────────────────────────────────────────────┘
```

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 15, React 19, TanStack Query, Tailwind CSS, shadcn/ui |
| API | Hono (OpenAPI), Cloudflare Workers |
| Validation | Zod |
| ORM | Drizzle ORM |
| Database | PostgreSQL (Neon) |
| Auth | Clerk (multi-tenant) |
| PDF | HTML-to-PDF rendering (Cloudflare Workers) |
| Email | Resend |
| ZRA | VSDC API v1 (REST) |

## Package Structure

```
bumara/
├── apps/
│   ├── app/                          # the single Next.js app
│   │   ├── app/invoicing/            # route tree
│   │   └── zones/invoicing/          # the invoicing zone
│   │       ├── modules/              # sales-invoices, credit-notes, customers,
│   │       │                         # products, purchases, recurring, reports,
│   │       │                         # settings, zra, ...
│   │       ├── components/           # zone-wide shared components
│   │       └── lib/                  # hooks, fetchers, query keys
│   └── api-invoicing/                # Hono worker (thin HTTP edge)
│       └── src/modules/<slice>/      # routes.ts, handlers.ts, index.ts
├── packages/
│   ├── invoicing/src/                # invoicing domain logic, one dir per slice
│   │   ├── invoices/  credit-notes/  debit-notes/  customers/  vendors/
│   │   ├── commercial-invoices/      # mining export chain
│   │   ├── zra/                      # ZRA client, payload builders, transmit
│   │   ├── settings/  sequences/  reports/  recurring/  imports/
│   │   └── shared/  ports/
│   ├── database/                     # schema + repositories
│   │   └── src/
│   │       ├── schema/invoicing/
│   │       └── repositories/
│   └── notifications/                # email adapter
└── docs/invoicing/                   # this documentation
```

## Multi-Tenancy

Every request is scoped to an organization via Clerk auth middleware:
1. `requireAuth` — validates JWT, extracts userId
2. `requireOrg` — extracts orgId from Clerk session
3. `requireOrganizationContext(ctx)` — enforces org context in service layer
4. All DB queries filter by `organizationId`

## Request Flow

```
Browser → Next.js Page → Hook (useQuery/useMutation)
  → Fetcher (fetch + auth headers)
  → Hono Route (validation via Zod)
  → Handler (buildServiceContext + buildServiceDependencies)
  → Service Function (business logic + audit logging)
  → Repository Function (Drizzle query)
  → PostgreSQL
```

## Key Design Patterns

- **Repository Pattern**: DB access abstracted into typed repository functions
- **Service Layer**: Business logic isolated from HTTP layer, receives `ctx` + `deps`
- **Lazy Loading**: Route handlers loaded dynamically to reduce cold start
- **Audit Trail**: All mutations log to `audit_logs` table
- **Ledger System**: Double-entry accounting via `account_transactions` table
- **Inventory Link**: Invoice creation/deletion triggers stock movements
