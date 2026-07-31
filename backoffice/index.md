---
title: "Bumara Backoffice Documentation"
description: "Internal operations platform for delivering managed compliance services to Zambian businesses."
---

## Quick Navigation

| Document | Description |
|----------|-------------|
| [Specification](/backoffice/specification) | Full product spec with roles, concepts, state machines |
| [Architecture](/backoffice/architecture) | System design, data model, auth integration |
| Gap Analysis | Current vs target implementation status |
| [Implementation Plan](/backoffice/implementation-plan) | Step-by-step build guide with file locations |

### Module Documentation

| Module | Description |
|--------|-------------|
| [Operations](/backoffice/modules/operations) | Inbox, Cases, Tickets & SLA |
| [Finance](/backoffice/modules/finance) | Payments & Payouts workflows |
| [Documents](/backoffice/modules/documents) | Evidence, S3 storage, tagging |
| [Directory](/backoffice/modules/directory) | Organizations (tenants) |
| [Admin](/backoffice/admin) | Staff management, roles, configuration |

---

## Current Implementation Status

> Last updated: 2026-02-02

| Feature | Status | Notes |
|---------|--------|-------|
| **Backend API** | ✅ Done | Cases, Queue, Tickets, Payouts, Bulk Operations, Saved Filters, Workload |
| **React Query Hooks** | ✅ Done | `useCases`, `useQueueCounts`, `useTickets`, `useWorkloadStats`, etc. |
| **Case Detail Page** | ✅ Done | Header, readiness gates, overview, timeline, blockers panel |
| **Tickets Page** | ✅ Done | Real API with SLA stats |
| **Inbox Queue** | ✅ Done | Tab filters, counts, claim action, bulk operations |
| **Payment Verification** | ✅ Done | Modal with proof document upload, timeline events |
| **Payout Approval** | ✅ Done | Dual control, threshold-based approval, approval queue UI |
| **Bulk Operations** | ✅ Done | Bulk claim, assign, transition with batch processing |
| **Saved Filters** | ✅ Done | CRUD endpoints, default filter, filter management UI |
| **Workload Dashboard** | ✅ Done | Agent stats, load indicators, completion rates |
| **State Machine** | ✅ Done | Centralized transitions, submission blocking, gate overrides |
| Organization Profiles | ⬜ Pending | Phase 4 |

See [Implementation Plan](/backoffice/implementation-plan) for detailed status.

## Purpose

Bumara Backoffice is the **internal operations application** used by Bumara staff to deliver managed compliance services. Unlike self-serve platforms, Bumara operates as a managed service where:

- Tenants prepare data and documents
- Tenants request submission
- **Backoffice staff** verify readiness, pay regulators, and submit manually
- Evidence is captured for every action
- Full audit trail is maintained

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           BUMARA PLATFORM                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────┐                    ┌──────────────────┐          │
│  │   TENANT APP     │                    │   BACKOFFICE     │          │
│  │  (apps/app)      │                    │  (apps/backoffice)│          │
│  │                  │                    │                  │          │
│  │  • Dashboard     │                    │  • Inbox/Queue   │          │
│  │  • Obligations   │   Request          │  • Cases         │          │
│  │  • Tasks         │  Submission        │  • Tickets       │          │
│  │  • Documents     │ ──────────────────►│  • Payments      │          │
│  │  • Payments      │                    │  • Documents     │          │
│  └────────┬─────────┘                    │  • Organizations │          │
│           │                              │  • Admin         │          │
│           │                              └────────┬─────────┘          │
│           │                                       │                    │
│           │              ┌────────────────────────┘                    │
│           │              │                                             │
│           ▼              ▼                                             │
│  ┌─────────────────────────────────────────────────────────────┐      │
│  │                    SHARED BACKEND                            │      │
│  │                   (packages/backend)                         │      │
│  │                                                              │      │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │      │
│  │  │  Auth/RBAC   │  │  Compliance  │  │   Payments   │       │      │
│  │  │  Middleware  │  │   Services   │  │   Services   │       │      │
│  │  └──────────────┘  └──────────────┘  └──────────────┘       │      │
│  └─────────────────────────────────────────────────────────────┘      │
│                              │                                         │
│                              ▼                                         │
│  ┌─────────────────────────────────────────────────────────────┐      │
│  │                    SHARED DATABASE                           │      │
│  │                  (packages/database)                         │      │
│  │                                                              │      │
│  │  organizations │ filings │ service_requests │ tasks         │      │
│  │  submission_jobs │ payment_requests │ regulator_payouts     │      │
│  │  tickets │ documents │ audit_logs │ back_office_agents      │      │
│  └─────────────────────────────────────────────────────────────┘      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                         EXTERNAL SERVICES                                │
├──────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                   │
│  │    Clerk     │  │         AWS  │  │  Regulators  │                   │
│  │    (Auth)    │  │  S3 (Files)  │  │  (Manual)    │                   │
│  └──────────────┘  └──────────────┘  └──────────────┘                   │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Core Operational Model

```
OBLIGATION
    │
    ▼
FILING / SERVICE REQUEST ──► TASKS (Tenant completes)
    │
    ▼
SUBMISSION JOB ◄────────────── Tenant clicks "Request Submission"
    │
    ├──► PAYMENT REQUEST (Tenant → Bumara)
    │         │
    │         ▼ Verify
    │
    ├──► REGULATOR PAYOUT (Bumara → Regulator)
    │         │
    │         ▼ Pay + Verify
    │
    ▼
SUBMISSION (Manual by Backoffice)
    │
    ▼
EVIDENCE + OUTCOME
    │
    ▼
AUDIT LOG (Every step)
```

---

## Running Locally

### Prerequisites

- Node.js 20+
- pnpm
- PostgreSQL database
- Clerk account with backoffice organization configured
- Environment variables set (see `.env.example`)

### Development Server

```bash
# From repo root
pnpm install
pnpm dev:backoffice

# Or from apps/backoffice
cd apps/backoffice
pnpm dev
```

The backoffice runs at `http://localhost:3001` (separate from tenant app).

### Required Environment Variables

```bash
# Clerk
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_...
CLERK_SECRET_KEY=sk_...
CLERK_INTERNAL_ORG_ID=org_...           # Backoffice organization ID
ALLOWED_EMAIL_DOMAINS=bumara.com        # Staff email domain

# Database
DATABASE_URL=postgres://...

# Backend API
NEXT_PUBLIC_API_URL=http://localhost:8787
```

---

## New Features (MVP)

### Payout Approval with Dual Control

Payouts require approval based on amount thresholds:
- &lt; ZMW 5,000: No approval required (auto-approved)
- ZMW 5,000 - 50,000: Manager approval required
- > ZMW 50,000: Manager approval required

**Dual Control**: The creator of a payout cannot approve it (four-eyes principle).

### Bulk Operations

Staff can select multiple cases and perform bulk actions:
- Bulk claim (up to 50 cases)
- Bulk assign to specific agent
- Bulk transition (with state machine validation)

### Saved Filters

Staff can save filter configurations for quick access:
- Save current filter view
- Set default filter per type
- Manage (rename, delete) saved filters

### Workload Dashboard

View agent workload statistics:
- Current load vs capacity
- Completed today/this week
- Average completion time
- Overdue and due-soon case counts

### Submission Blocking

Cases cannot proceed to submission unless:
- All required tasks are complete
- Required documents are uploaded
- Payment is verified (if required)
- Payout is verified (if required)
- Authorization is valid (if required)

Managers can override blocked gates with a reason.

---

## Key File Locations

| Purpose | Location |
|---------|----------|
| Backoffice app | `apps/backoffice/` |
| Navigation config | `apps/backoffice/config/nav.ts` |
| Role guards | `apps/backoffice/lib/guards/require-role.ts` |
| **Query Hooks (Cases)** | `apps/backoffice/lib/queries/cases/index.ts` |
| **Query Hooks (Queue)** | `apps/backoffice/lib/queries/queue/index.ts` |
| **Query Hooks (Tickets)** | `apps/backoffice/lib/queries/tickets/index.ts` |
| **Query Hooks (Workload)** | `apps/backoffice/lib/queries/workload/index.ts` |
| **Case Detail Page** | `apps/backoffice/app/.../cases/[id]/` |
| **Workload Dashboard** | `apps/backoffice/app/.../reports/workload/` |
| Auth middleware | `packages/backend/src/core/middleware/auth.ts` |
| **Cases Backend Handlers** | `packages/backend/src/modules/compliance/cases/handlers.ts` |
| **Bulk Operations** | `packages/backend/src/modules/compliance/bulk/` |
| **Saved Filters** | `packages/backend/src/modules/compliance/saved-filters/` |
| **Workload** | `packages/backend/src/modules/compliance/workload/` |
| **State Machine** | `packages/api-services/src/domains/compliance/state-machine.ts` |
| **Readiness Gates** | `packages/api-services/src/domains/compliance/gates.ts` |
| Database schema | `packages/database/src/schema/` |
| Compliance schema | `packages/database/src/schema/compliance/` |
| Backend API routes | `packages/backend/src/modules/` |
| API services | `packages/api-services/src/domains/` |

---

## Staff Roles

| Role | Level | Capabilities |
|------|-------|--------------|
| `member` | 1 | View, claim, execute cases, create tickets, upload evidence |
| `manager` | 2 | + Reassign, override gates, approve high-value payouts, bulk operations |
| `admin` | 3 | + Staff management, system configuration, all access |

See [Specification > Permission Matrix](/backoffice/specification#33-permission-matrix-mvp) for details.

---

## API Endpoints Reference

### Cases & Queue

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/backoffice/queue` | List queue items with tab filters |
| GET | `/backoffice/queue/counts` | Get counts for each queue tab |
| GET | `/backoffice/cases` | List all cases with filters |
| GET | `/backoffice/cases/:id` | Get case detail with gates |
| POST | `/backoffice/cases/:id/claim` | Claim case for current agent |
| POST | `/backoffice/cases/:id/assign` | Assign case to specific agent |
| POST | `/backoffice/cases/:id/transition` | Transition case status |
| POST | `/backoffice/cases/:id/override-gate` | Override a failing gate |

### Bulk Operations

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/backoffice/bulk/claim` | Bulk claim up to 50 cases |
| POST | `/backoffice/bulk/assign` | Bulk assign cases to agent |
| POST | `/backoffice/bulk/transition` | Bulk transition case statuses |

### Payouts

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/backoffice/payouts` | List payouts with filters |
| GET | `/backoffice/payouts/:id` | Get payout detail |
| POST | `/backoffice/payouts` | Create new payout |
| POST | `/backoffice/payouts/:id/pay` | Mark payout as paid |
| POST | `/backoffice/payouts/:id/approve` | Approve/reject payout |
| POST | `/backoffice/payouts/:id/verify` | Verify payout completion |

### Saved Filters

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/backoffice/saved-filters` | List saved filters for user |
| POST | `/backoffice/saved-filters` | Create saved filter |
| PATCH | `/backoffice/saved-filters/:id` | Update saved filter |
| DELETE | `/backoffice/saved-filters/:id` | Delete saved filter |
| POST | `/backoffice/saved-filters/:id/set-default` | Set as default filter |

### Workload

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/backoffice/workload` | Get agent workload statistics |

---

## Related Documentation

- Tenant App Docs (if exists)
- Database Schema
- API Reference
- [Tasks Module](/modules/tasks)

---

## Contributing

When adding features to the backoffice:

1. Follow the operational model (gated execution, evidence required)
2. Add audit logging for all mutations
3. Implement server-side authorization
4. Update relevant documentation
5. Add acceptance criteria tests

See [Implementation Plan](/backoffice/implementation-plan) for current build priorities.

