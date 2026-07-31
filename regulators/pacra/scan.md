---
title: "PACRA Workspace - Codebase Scan Results"
description: "Pre-implementation scan for the PACRA Tenant Workspace UI"
---

This document summarizes existing infrastructure, patterns, and gaps discovered during the codebase scan.

---

## 1. Tenant App Structure

### Directory: `apps/app`

The tenant-facing application uses **Next.js App Router** with the following structure:

```
apps/app/
├── app/(authenticated)/(general)/regulators/
│   ├── pacra/page.tsx      # Stub - needs implementation
│   ├── zra/page.tsx
│   └── napsa/page.tsx
├── components/
│   ├── layout/sidebar/     # Sidebar navigation
│   └── modules/regulators/ # Regulator UI components
├── features/pacra/         # PACRA-specific features
├── lib/queries/            # React Query hooks
└── config/secondary-sidebar/
```

### Routing Convention

- Base regulator path: `/regulators/:key` (e.g., `/regulators/pacra`)
- Sub-routes follow: `/regulators/pacra/{section}`
- Layout groups: `(authenticated)` for protected routes

---

## 2. Existing PACRA Implementation

### 2.1 Page Scaffold

**File:** `apps/app/app/(authenticated)/(general)/regulators/pacra/page.tsx`

```typescript
// Current state: Stub only
export default function PacraPage() {
  return <div>PacraPage</div>;
}
```

**Status:** Needs full implementation with workspace layout and overview content.

### 2.2 Workspace Sidebar

**File:** `apps/app/config/secondary-sidebar/regulator-sidebar.ts`

PACRA_MENU is already defined with all required sections:

| Section | Route | Status |
|---------|-------|--------|
| Overview | `/regulators/pacra` | Stub |
| Obligations | `/regulators/pacra/obligations` | Not created |
| Filings | `/regulators/pacra/filings` | Not created |
| Service Requests | `/regulators/pacra/service-requests` | Not created |
| Tasks | `/regulators/pacra/tasks` | Not created |
| Payments | `/regulators/pacra/payments` | Not created |
| Documents | `/regulators/pacra/documents` | Not created |
| Timeline | `/regulators/pacra/timeline` | Not created |
| Reports | `/regulators/pacra/reports` | Not created |
| Authorization | `/regulators/pacra/authorization` | Not created |

**File:** `apps/app/components/layout/sidebar/secondary-sidebar.tsx`

The secondary sidebar dynamically selects menu based on pathname:
- `/regulators/pacra*` → `PACRA_MENU`
- `/regulators/zra*` → `ZRA_MENU`
- `/regulators/napsa*` → `NAPSA_MENU`

---

## 3. Data Fetching Patterns

### 3.1 Regulator Connections (Real API)

**File:** `apps/app/lib/queries/regulators/connections.ts`

| Hook | Endpoint | Status |
|------|----------|--------|
| `useRegulatorConnections(regulatorId?)` | `GET /connections` | Working |
| `useCreateConnection()` | `POST /connections` | Working |
| `useUpdateConnection()` | `PATCH /connections/:id` | Working |
| `useDeleteConnection()` | `DELETE /connections/:id` | Working |

**Reusable:** Yes - can check PACRA connection state.

### 3.2 Obligations (Mock Data)

**File:** `apps/app/lib/queries/pacra/obligations.ts`

| Hook | Status | Notes |
|------|--------|-------|
| `usePacraObligations()` | Mock | Uses `mockGetPacraObligations()` |
| `usePacraComplianceSummary()` | Mock | Uses `mockGetPacraComplianceSummary()` |
| `usePacraObligation(id)` | Mock | Uses `mockGetPacraObligation()` |
| `usePacraObligationFilings(id)` | Mock | Uses `mockGetPacraObligationFilings()` |

**Action Required:** Replace mock with real API endpoint.

### 3.3 Filings (Mock Data)

**File:** `apps/app/lib/queries/filings/filings.ts`

| Hook | Status | Notes |
|------|--------|-------|
| `useFilings(filters)` | Mock | Uses `mockGetFilings()` |

**Action Required:** Create real API endpoint and update hook.

### 3.4 Service Requests (Mock Data)

**File:** `apps/app/features/pacra/requests/queries.ts`

| Hook | Status | Notes |
|------|--------|-------|
| `usePacraServiceRequests(filters)` | Mock | Inline mock data |
| `usePacraServiceCatalog()` | Mock | Returns static catalog |

**Action Required:** Create real API endpoints.

### 3.5 Tasks (Real API)

**File:** `apps/app/features/pacra/tasks/queries.ts`

| Hook | Endpoint | Status |
|------|----------|--------|
| `usePacraTasks(filters)` | `GET /tasks` | Working |
| `useTask(id)` | `GET /tasks/:id` | Working |
| `useUpdateTaskStatus()` | `PATCH /tasks/:id/status` | Working |
| `useSkipTask()` | `POST /tasks/:id/skip` | Working |
| `useReopenTask()` | `POST /tasks/:id/reopen` | Working |
| `useAddTaskComment()` | `POST /tasks/:id/comments` | Working |
| `useTaskComments(taskId)` | `GET /tasks/:id/comments` | Working |
| `useFilingReadiness(filingId)` | `GET /filings/:id/readiness` | Working |

**Reusable:** Yes - fully functional.

---

## 4. Backend API Modules

### 4.1 Existing Modules

**Location:** `packages/backend/src/modules/`

| Module | Path | Status |
|--------|------|--------|
| PACRA Connect | `/pacra/connect` | Complete |
| PACRA Profile | `/pacra/profile` | Complete |
| Regulator Connections | `/connections` | Complete |
| Tasks | `/tasks` | Complete |
| Audit Logs | `/audit-logs` | Complete |
| Dashboard | `/dashboard` | Complete |

### 4.2 Missing Modules

| Module | Required Endpoints |
|--------|-------------------|
| Obligations | `GET /obligations`, `GET /obligations/:id` |
| Filings | `GET /filings`, `GET /filings/:id` |
| Service Requests | `GET /service-requests`, `GET /service-requests/:id`, `POST /service-requests` |
| Documents | `GET /documents`, `POST /documents`, `POST /documents/:id/attach` |
| Timeline | `GET /timeline` |
| Authorization | `GET/POST/PATCH /authorization` |

---

## 5. Database Schema

### 5.1 Existing Tables (Reusable)

**Location:** `packages/database/src/schema/`

| Table | File | Notes |
|-------|------|-------|
| `organization_regulator_connections` | `core/organization-regulator-connections.ts` | Connection state |
| `org_obligations` | `compliance/org-obligations.ts` | Org-specific obligations |
| `filings` | `compliance/filings.ts` | Filing instances |
| `service_requests` | `compliance/service-requests.ts` | Service request instances |
| `tasks` | `compliance/tasks.ts` | Task checklist items |
| `documents` | `compliance/documents.ts` | Document metadata |
| `audit_logs` | `system/audit-logs.ts` | Audit trail |
| `pacra_profiles` | `pacra/profiles.ts` | PACRA-specific profile data |
| `obligation_templates` | `compliance/obligation-templates.ts` | Global templates |
| `service_templates` | `compliance/service-templates.ts` | Service templates |

### 5.2 Missing Schema

| Table | Purpose |
|-------|---------|
| `regulator_authorizations` | Store authorization representatives |

---

## 6. API Services

### 6.1 Existing Services

**Location:** `packages/api-services/src/domains/`

| Service | File | Status |
|---------|------|--------|
| Activation Engine | `activation/activation.service.ts` | Complete |
| PACRA Connect | `pacra/pacra-connect.service.ts` | Complete |
| Tasks | `tasks/tasks.service.ts` | Complete |
| Audit Log | `audit/audit-log.service.ts` | Complete |

### 6.2 Missing Services

| Service | Purpose |
|---------|---------|
| `obligations.service.ts` | List/get obligations by regulator |
| `filings.service.ts` | List/get filings by regulator |
| `service-requests.service.ts` | CRUD for service requests |
| `documents.service.ts` | Document management |
| `timeline.service.ts` | Timeline event queries |
| `authorization.service.ts` | Authorization management |

---

## 7. UI Components

### 7.1 Existing Components (Reusable)

**PACRA Tasks:**
- `apps/app/features/pacra/tasks/pacra-tasks-client.tsx` - Full task list UI
- `apps/app/features/pacra/tasks/components/task-details-sheet.tsx`
- `apps/app/features/pacra/tasks/components/task-status-badge.tsx`
- `apps/app/features/pacra/tasks/components/due-date-chip.tsx`

**PACRA Service Requests:**
- `apps/app/features/pacra/requests/pacra-service-requests-client.tsx`
- `apps/app/features/pacra/requests/components/request-list.tsx`
- `apps/app/features/pacra/requests/components/request-filters.tsx`
- `apps/app/features/pacra/requests/components/service-cards.tsx`

**Shared Widgets:**
- `apps/app/components/widgets/empty-state.tsx`
- `apps/app/components/widgets/skeleton-loaders.tsx`
- `apps/app/components/widgets/module-header.tsx`

### 7.2 Missing Components

| Component | Purpose |
|-----------|---------|
| `RegulatorWorkspaceLayout` | Connection check + nav wrapper |
| `RegulatorConnectCta` | Connect prompt for disconnected state |
| `OverviewStats` | Dashboard statistics cards |
| `ObligationCard` | Obligation list item |
| `FilingCard` | Filing list item |
| `FilingDetail` | Filing detail sections |
| `DocumentList` | Document management UI |
| `DocumentUpload` | Document upload component |
| `TimelineList` | Activity timeline display |
| `AuthorizationForm` | Authorization capture form |

---

## 8. Activation Engine Integration

### 8.1 PACRA Connect Flow

**File:** `packages/backend/src/modules/pacra/connect/handlers.ts`

The connect flow:
1. Validates input against `pacraConnectConfigSchema`
2. Creates/updates PACRA profile
3. Creates/updates regulator connection
4. Calls `activateRegulatorTemplates()` to create obligations/filings/tasks
5. Returns activation summary

**Endpoint:** `POST /api/pacra/connect`

### 8.2 Template-Driven Data

All workspace data is template-driven:
- Obligations created from `obligation_templates`
- Filings created with computed due dates
- Tasks created from `taskTemplateConfigs`
- Document requirements from `docRequirementConfigs`

**Key Fields:**
- `templateId` - Links to template for runbook lookup
- `templateKey` - Stable identifier for idempotency
- `periodKey` - Period identifier (e.g., `FY2024`)

---

## 9. Summary of Actions Required

### Backend (packages/backend + packages/api-services)

1. Create `regulator_authorizations` schema
2. Create compliance module with:
   - Obligations routes/handlers/service
   - Filings routes/handlers/service
   - Service Requests routes/handlers/service
   - Documents routes/handlers/service
   - Timeline routes/handlers/service
   - Authorization routes/handlers/service

### Frontend (apps/app)

1. Create regulator config system
2. Create `RegulatorWorkspaceLayout` component
3. Implement all PACRA workspace pages:
   - Overview (with real stats)
   - Obligations (with real API)
   - Filings list + detail
   - Service Requests (connect to real API)
   - Tasks (use existing)
   - Documents
   - Timeline
   - Authorization
   - Payments (placeholder)
   - Reports (placeholder)

### Documentation

1. Create workspace documentation at `docs/regulators/pacra/workspace.md`

---

## 10. File Reference

### Key Files to Modify

```
apps/app/app/(authenticated)/(general)/regulators/pacra/page.tsx
apps/app/lib/queries/pacra/obligations.ts
apps/app/lib/queries/filings/filings.ts
apps/app/features/pacra/requests/queries.ts
packages/backend/src/modules/index.ts
```

### Key Files to Create

```
# Schema
packages/database/src/schema/compliance/regulator-authorizations.ts

# Backend Module
packages/backend/src/modules/compliance/
├── index.ts
├── obligations/
├── filings/
├── service-requests/
├── documents/
├── timeline/
└── authorization/

# API Services
packages/api-services/src/domains/compliance/
├── obligations.service.ts
├── filings.service.ts
├── service-requests.service.ts
├── documents.service.ts
├── timeline.service.ts
└── authorization.service.ts

# Frontend Config
apps/app/features/regulators/config.ts

# Frontend Components
apps/app/features/regulators/components/
├── workspace-layout.tsx
├── connect-cta.tsx
├── overview-stats.tsx
└── ...

# Frontend Routes
apps/app/app/(authenticated)/(general)/regulators/pacra/
├── layout.tsx
├── obligations/page.tsx
├── filings/page.tsx
├── filings/[filingId]/page.tsx
├── service-requests/page.tsx
├── service-requests/[id]/page.tsx
├── tasks/page.tsx
├── documents/page.tsx
├── timeline/page.tsx
├── authorization/page.tsx
├── payments/page.tsx
└── reports/page.tsx

# Documentation
docs/regulators/pacra/workspace.md
```

