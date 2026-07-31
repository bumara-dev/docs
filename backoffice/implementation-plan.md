---
title: "Bumara Backoffice Implementation Plan"
description: "Step-by-step build guide based on current repository state."
---

**Status:** Phase 1-2 Implemented ✅  
**Last Updated:** 2026-01-28  
**Estimated Duration:** 2 weeks  
**Prerequisites:** Database schemas exist, auth infrastructure complete

---

## Implementation Status

> [!NOTE]
> This section tracks the current implementation state. For full details, see the [architecture.md](/backoffice/architecture) doc.

### ✅ Completed (Phase 1 & 2)

| Component | Status | Location |
|-----------|--------|----------|
| **Backend Data Layer** | ✅ Done | `packages/backend/src/modules/compliance/cases/` |
| **Queue API** | ✅ Done | `GET /backoffice/queue`, `GET /backoffice/queue/counts` |
| **Cases API** | ✅ Done | `GET/POST /backoffice/cases/*` |
| **Tickets API** | ✅ Done | `GET/POST /backoffice/tickets/*` |
| **React Query Hooks (Cases)** | ✅ Done | `apps/backoffice/lib/queries/cases/index.ts` |
| **React Query Hooks (Queue)** | ✅ Done | `apps/backoffice/lib/queries/queue/index.ts` |
| **React Query Hooks (Tickets)** | ✅ Done | `apps/backoffice/lib/queries/tickets/index.ts` |
| **Case Detail Page** | ✅ Done | `apps/backoffice/app/.../cases/[id]/` |
| **Tickets Page (Real API)** | ✅ Done | `apps/backoffice/app/.../tickets/` |

### Files Created/Updated

**Backend Handlers:**
- handlers.ts - Case operations with atomic claims, queue counts, status transitions

**React Query Hooks:**
- lib/queries/cases/index.ts - `useCases`, `useCase`, `useClaimCase`, `useAssignCase`, `useTransitionCase`, `useOverrideGate`
- lib/queries/queue/index.ts - `useQueueCounts`, `useQueueItems`
- lib/queries/tickets/index.ts - `useTickets`, `useTicket`, `useCreateTicket`, `useResolveTicket`

**Case Detail Page Components:**
- page.tsx - Server component with Suspense
- case-detail-client.tsx - Main client component
- case-header.tsx - Status, actions, assignment
- readiness-checklist.tsx - Gates with override modal
- case-overview.tsx - Org info, regulator fields
- case-timeline.tsx - Audit events

**Tickets Page:**
- tickets-client.tsx - Real API integration with SLA stats

### 🔄 In Progress / Remaining

| Component | Status | Notes |
|-----------|--------|-------|
| Payment Verification UI | ⬜ Not Started | Phase 3 |
| Payout Workflow UI | ⬜ Not Started | Phase 3 |
| Evidence Upload Integration | ⬜ Not Started | Phase 3 |
| Organization Profile Page | ⬜ Not Started | Phase 4 |
| Create Ticket Modal | ⬜ Not Started | Wire up button |
| Regulator-specific field configs | ⬜ Not Started | ZRA, PACRA, etc. |

---

## Table of Contents

1. [Phase 1: Backend API Foundation](#phase-1-backend-api-foundation)
2. [Phase 2: Core Operations UI](#phase-2-core-operations-ui)
3. [Phase 3: Finance & Evidence](#phase-3-finance--evidence)
4. [Phase 4: Directory & Admin](#phase-4-directory--admin)
5. [Phase 5: Polish & Testing](#phase-5-polish--testing)

---

## Phase 1: Backend API Foundation

**Duration:** Week 1  
**Goal:** Create backoffice-specific API endpoints with state machine validation  
**Status:** ✅ Completed

### Step 1.1: State Machine Service ✅

State machine validation implemented in handlers.

**Status:** ✅ Implemented in handlers.ts

**Acceptance Criteria:**
- [x] State machine validates all filing transitions
- [x] State machine validates all submission job transitions
- [x] Gate checking returns accurate blockers
- [x] Override with reason is supported for managers

---

### Step 1.2: Backoffice Cases API ✅

**Status:** ✅ Implemented

**Endpoints Created:**
- `GET /backoffice/cases` - List cases with filters
- `GET /backoffice/cases/:id` - Case detail with gates
- `POST /backoffice/cases/:id/claim` - Atomic case claim
- `POST /backoffice/cases/:id/assign` - Assign to agent (manager+)
- `POST /backoffice/cases/:id/transition` - Status transition
- `POST /backoffice/cases/:id/override-gate` - Override readiness gate

**Acceptance Criteria:**
- [x] GET `/backoffice/cases` returns paginated list
- [x] GET `/backoffice/cases/:id` returns case with gates
- [x] POST `/backoffice/cases/:id/claim` is atomic (no double-claim)
- [x] POST `/backoffice/cases/:id/assign` requires manager role
- [x] POST `/backoffice/cases/:id/transition` validates state machine
- [x] All endpoints create audit logs

---

### Step 1.3: Queue Endpoints ✅

**Status:** ✅ Implemented

**Endpoints Created:**
- `GET /backoffice/queue` - Queue items with tab filtering
- `GET /backoffice/queue/counts` - Efficient count queries

**Acceptance Criteria:**
- [x] Queue counts are accurate
- [x] "Assigned to me" filter works
- [x] "Unassigned" filter works
- [x] Overdue items are flagged

---

### Step 1.4: Tickets API ✅

**Status:** ✅ Implemented

**Endpoints Created:**
- `GET /backoffice/tickets` - List tickets with filters
- `GET /backoffice/tickets/:id` - Ticket detail with notes
- `POST /backoffice/tickets` - Create ticket
- `POST /backoffice/tickets/:id/resolve` - Resolve ticket
- `POST /backoffice/tickets/:id/notes` - Add note

**Acceptance Criteria:**
- [x] Tickets can be created with required items
- [x] Tickets link to cases
- [x] Ticket status transitions are tracked
- [x] Internal notes are separate from tenant-visible messages

---

### Step 1.5: Payouts API ✅

**Status:** ✅ Implemented

**Endpoints Created:**
- `GET /backoffice/payouts` - List payouts
- `POST /backoffice/payouts` - Create payout
- `POST /backoffice/payouts/:id/approve` - Approve payout
- `POST /backoffice/payouts/:id/mark-paid` - Mark as paid

---

## Phase 2: Core Operations UI

**Duration:** Week 1  
**Goal:** Replace mock data with real API integration  
**Status:** ✅ Completed

### Step 2.1: API Client Hooks ✅

**Status:** ✅ Implemented

**Files Created:**
- `apps/backoffice/lib/queries/cases/index.ts`
- `apps/backoffice/lib/queries/queue/index.ts` (updated)
- `apps/backoffice/lib/queries/tickets/index.ts`

**Hooks Available:**
```typescript
// Cases
useCases(filters)
useCase(id)
useClaimCase()
useAssignCase()
useTransitionCase()
useOverrideGate()

// Queue
useQueueCounts()
useQueueItems(params)

// Tickets
useTickets(params)
useTicket(id)
useCreateTicket()
useResolveTicket()
useAddTicketNote()
```

**Acceptance Criteria:**
- [x] All hooks handle loading/error states
- [x] Mutations invalidate relevant queries
- [x] TypeScript types are inferred from API

---

### Step 2.2: Update Inbox Page ✅

**Status:** ✅ Already using real queue data via `useQueueItems` and `useQueueCounts`

**Acceptance Criteria:**
- [x] Queue tabs show accurate counts
- [x] Table displays real case data
- [x] Claim button works (atomic)
- [x] Empty/loading states are handled
- [x] Filters work correctly

---

### Step 2.3: Create Case Detail Page ✅

**Status:** ✅ Implemented

**File:** `apps/backoffice/app/(authenticated)/(home)/(general)/cases/[id]/page.tsx`

**Components Created:**
- `case-detail-client.tsx` - Main client with tabs (Overview, Tasks, Documents, Payments, Tickets)
- `case-header.tsx` - Status badge, due date, assignment, quick actions
- `readiness-checklist.tsx` - Gates with pass/fail, blockers, override modal
- `case-overview.tsx` - Organization info, regulator details, key dates
- `case-timeline.tsx` - Audit events from timeline API

**Acceptance Criteria:**
- [x] Header shows status badge, due date, assignment
- [x] Checklist shows all gates with pass/fail
- [x] Blockers are clearly displayed
- [x] Quick actions work (claim, submit, transition)
- [x] Timeline shows audit events
- [ ] All tabs display relevant data (Tasks, Documents, Payments tabs are placeholders)

---

### Step 2.5: Update Tickets Page ✅

**Status:** ✅ Implemented

**File:** `apps/backoffice/app/(authenticated)/(home)/(general)/tickets/_components/tickets-client.tsx`

**Acceptance Criteria:**
- [x] Ticket list shows real data
- [x] SLA statistics display (OK, At Risk, Breached)
- [x] Status filters work
- [x] Loading/error states handled
- [ ] Create ticket modal (not wired yet)

---

## Phase 3: Finance & Evidence

**Duration:** Week 2  
**Goal:** Implement payment verification and evidence upload  
**Status:** ⬜ Not Started

### Step 3.1: Payment Verification UI

⬜ Not implemented yet. Use existing payout endpoints.

### Step 3.2: Payout Workflow UI

⬜ Not implemented yet. Backend endpoints exist.

### Step 3.3: Evidence Upload Integration

⬜ Not implemented yet.

---

## Phase 4: Directory & Admin

**Duration:** Week 2  
**Goal:** Complete remaining modules  
**Status:** ⬜ Not Started

---

## Phase 5: Polish & Testing

**Duration:** Week 2  
**Goal:** Complete the MVP  
**Status:** ⬜ Not Started

---

## Endpoint Checklist

### Backoffice Queue API

| Method | Endpoint | Status |
|--------|----------|--------|
| GET | `/backoffice/queue` | ✅ |
| GET | `/backoffice/queue/counts` | ✅ |

### Backoffice Cases API

| Method | Endpoint | Status |
|--------|----------|--------|
| GET | `/backoffice/cases` | ✅ |
| GET | `/backoffice/cases/:id` | ✅ |
| POST | `/backoffice/cases/:id/claim` | ✅ |
| POST | `/backoffice/cases/:id/assign` | ✅ |
| POST | `/backoffice/cases/:id/transition` | ✅ |
| POST | `/backoffice/cases/:id/override-gate` | ✅ |

### Tickets API

| Method | Endpoint | Status |
|--------|----------|--------|
| GET | `/backoffice/tickets` | ✅ |
| GET | `/backoffice/tickets/:id` | ✅ |
| POST | `/backoffice/tickets` | ✅ |
| POST | `/backoffice/tickets/:id/resolve` | ✅ |
| POST | `/backoffice/tickets/:id/notes` | ✅ |

### Payouts API

| Method | Endpoint | Status |
|--------|----------|--------|
| GET | `/backoffice/payouts` | ✅ |
| POST | `/backoffice/payouts` | ✅ |
| POST | `/backoffice/payouts/:id/approve` | ✅ |
| POST | `/backoffice/payouts/:id/mark-paid` | ✅ |

### Payments API

| Method | Endpoint | Status |
|--------|----------|--------|
| GET | `/payments/tenant` | ⬜ |
| POST | `/payments/tenant/:id/verify` | ⬜ |

### Documents API

| Method | Endpoint | Status |
|--------|----------|--------|
| POST | `/documents/upload-url` | ⬜ |
| POST | `/documents/:id/attach` | ⬜ |
| GET | `/documents` | ⬜ |

---

## File Locations Summary

### Backend (packages/backend/src/modules/compliance/)

```
cases/
├── routes.ts           # Case & queue routes
├── handlers.ts         # Database handlers ✅ IMPLEMENTED
├── schemas.ts          # Zod schemas
└── index.ts

tickets/
├── routes.ts           # Ticket routes
├── handlers.ts         # Ticket handlers
└── schemas.ts

payouts/
├── routes.ts           # Payout routes
├── handlers.ts         # Payout handlers
└── schemas.ts
```

### Frontend Query Hooks (apps/backoffice/lib/queries/)

```
cases/
└── index.ts            # ✅ IMPLEMENTED

queue/
└── index.ts            # ✅ UPDATED

tickets/
└── index.ts            # ✅ IMPLEMENTED
```

### Frontend Pages (apps/backoffice/app/)

```
(authenticated)/(home)/(general)/
├── inbox/
│   └── _components/
│       └── inbox-content.tsx  # Uses useQueueItems
├── cases/
│   ├── page.tsx               # Cases list
│   └── [id]/
│       ├── page.tsx           # ✅ IMPLEMENTED
│       └── _components/
│           ├── case-detail-client.tsx  # ✅
│           ├── case-header.tsx         # ✅
│           ├── readiness-checklist.tsx # ✅
│           ├── case-overview.tsx       # ✅
│           └── case-timeline.tsx       # ✅
└── tickets/
    ├── page.tsx               # ✅ UPDATED
    └── _components/
        └── tickets-client.tsx # ✅ IMPLEMENTED
```

---

## Related Documentation

- [Architecture](/backoffice/architecture) - System design
- [Specification](/backoffice/specification) - Requirements
- [Admin Guide](/backoffice/admin) - Staff management

---

## How to Test

1. **Seed Test Data:**
   ```bash
   cd packages/database
   npx tsx src/seeds/seed-inbox-test-data.ts
   ```

2. **Start Dev Servers:**
   ```bash
   pnpm dev
   ```

3. **Test in Browser (http://localhost:3001):**
   - **Inbox** → Verify queue counts display
   - Click a case → Should load case detail page
   - **Tickets** → Should show real ticket data with SLA stats

4. **Test API Endpoints:**
   ```bash
   # Queue counts
   curl http://localhost:8787/api/v1/backoffice/queue/counts
   
   # Cases list
   curl http://localhost:8787/api/v1/backoffice/cases
   
   # Case detail
   curl http://localhost:8787/api/v1/backoffice/cases/{id}
   ```
