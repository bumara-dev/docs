---
title: "Directory Module"
description: "Organizations (Tenants) documentation."
---

## Table of Contents

1. [Overview](#1-overview)
2. [Organizations List](#2-organizations-list)
3. [Organization Profile](#3-organization-profile)
4. [Search](#4-search)
5. [Permissions](#5-permissions)

---

## 1. Overview

**Route:** `/orgs` (list), `/orgs/:id` (profile)  
**Purpose:** View and search tenant organizations.

### 1.1 Scope

The Directory module provides:
- Search and filter organizations
- View organization compliance profile
- Access organization's cases, documents, and activity
- Quick navigation to related data

### 1.2 Non-goals (MVP)

- ❌ Edit organization data (managed by tenant or admin)
- ❌ Create new organizations
- ❌ Billing/subscription management
- ❌ User management (use Clerk dashboard)

---

## 2. Organizations List

**Route:** `/orgs`

### 2.1 Page Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│ Organizations                                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│ ┌───────────────────────────────────────────────────────────────┐   │
│ │ 🔍 Search by name, TPIN, or registration number...           │   │
│ └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│ Filters: [Plan ▼] [Compliance Score ▼] [Regulator ▼] [Risk ▼]      │
│                                                                      │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ Organization    │ TPIN       │ Plan    │ Score │ Cases │ Risk  │ │
│ ├─────────────────┼────────────┼─────────┼───────┼───────┼───────┤ │
│ │ ABC Company Ltd │ 1234567890 │ Pro     │  85%  │  3    │ Low   │ │
│ │ XYZ Corporation │ 0987654321 │ Starter │  45%  │  8    │ High  │ │
│ │ 123 Industries  │ 5555555555 │ Pro     │  92%  │  1    │ Low   │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│ Showing 1-20 of 156 organizations                    [<] 1 2 3 [>]  │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 Table Columns

| Column | Description | Sortable |
|--------|-------------|----------|
| Organization | Name with logo | Yes |
| TPIN | Tax ID | Yes |
| Plan | Subscription tier | Yes |
| Compliance Score | Health percentage | Yes |
| Active Cases | Open cases count | Yes |
| Open Tickets | Unresolved tickets | Yes |
| Risk | Risk flag (Low/Medium/High) | Yes |
| Last Activity | Recent action date | Yes |

### 2.3 Compliance Score

Calculated based on:
- Overdue filings (negative)
- On-time submissions (positive)
- Open tickets (negative)
- Missing documents (negative)

```typescript
function calculateComplianceScore(org: Organization): number {
  let score = 100;
  
  // Deduct for overdue filings
  score -= org.overdueFilings * 10;
  
  // Deduct for open tickets
  score -= org.openTickets * 5;
  
  // Deduct for missing required docs
  score -= org.missingDocs * 3;
  
  // Add for on-time history
  score += Math.min(org.onTimeSubmissions, 20);
  
  return Math.max(0, Math.min(100, score));
}
```

### 2.4 Risk Flags

| Risk Level | Criteria | Color |
|------------|----------|-------|
| Low | Score ≥ 80, no overdue | Green |
| Medium | Score 50-79 or 1 overdue | Yellow |
| High | Score &lt; 50 or 2+ overdue | Red |

---

## 3. Organization Profile

**Route:** `/orgs/:id`

### 3.1 Profile Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│ ← Back to Organizations                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ 🏢  ABC Company Limited                                         │ │
│ │     TPIN: 1234567890 • REG: 123456                              │ │
│ │     Plan: Pro • Since: Jan 2024                                 │ │
│ │                                                          [Edit] │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐    │
│ │ Compliance  │ │ Active      │ │ Open        │ │ Due This    │    │
│ │ Score       │ │ Cases       │ │ Tickets     │ │ Month       │    │
│ │    85%      │ │    3        │ │    1        │ │    5        │    │
│ └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘    │
│                                                                      │
│ [Overview] [Cases] [Tickets] [Documents] [Activity] [Contacts]      │
│                                                                      │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │                                                                 │ │
│ │               Tab content here                                  │ │
│ │                                                                 │ │
│ └─────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 Profile Header

| Field | Description |
|-------|-------------|
| Name | Organization legal name |
| TPIN | Tax Payer Identification Number |
| Registration | PACRA registration number |
| Plan | Subscription tier |
| Member Since | When they joined Bumara |

### 3.3 Stats Cards

| Stat | Description | Clickable |
|------|-------------|-----------|
| Compliance Score | Current health % | No |
| Active Cases | Open filings + requests | Yes → Cases tab |
| Open Tickets | Unresolved tickets | Yes → Tickets tab |
| Due This Month | Upcoming deadlines | Yes → Cases tab |

### 3.4 Profile Tabs

#### Overview Tab

| Section | Content |
|---------|---------|
| Organization Details | Full legal info, industry, address |
| Regulator Connections | ZRA, PACRA, NAPSA, NHIMA status |
| Obligations | Active obligations list |
| Upcoming Deadlines | Next 5 due dates |

**Regulator Connections Display:**

```
Regulator Connections
────────────────────────────────────────
✓ ZRA - Connected
  └─ TPIN: 1234567890
  └─ Last sync: 2 hours ago

✓ PACRA - Connected
  └─ Reg #: 123456
  
✗ NAPSA - Not connected
  └─ [Connect]
  
✗ NHIMA - Not connected
  └─ [Connect]
────────────────────────────────────────
```

#### Cases Tab

Filtered view of `/cases` for this organization:
- All cases for this org
- Same filters as Cases page
- Quick actions available

#### Tickets Tab

Filtered view of `/tickets` for this organization:
- All tickets for this org
- Status filters
- SLA indicators

#### Documents Tab

Filtered view of `/documents` for this organization:
- All documents for this org
- Tag filters
- Upload (staff can add documents)

#### Activity Tab

Organization-specific audit timeline:
- Recent events
- Compliance changes
- Payment activity
- Ticket updates

#### Contacts Tab

Organization contact information:

| Field | Description |
|-------|-------------|
| Primary Contact | Main person for compliance |
| Email | Contact email |
| Phone | Contact phone |
| Secondary Contacts | Additional team members |

---

## 4. Search

### 4.1 Search Fields

Search matches against:
- Organization name (partial match)
- TPIN (exact or prefix)
- Registration number (exact or prefix)
- Primary contact email

### 4.2 Search API

```typescript
interface OrgSearchQuery {
  query: string;        // Search term
  plan?: string[];      // Filter by plan
  minScore?: number;    // Minimum compliance score
  maxScore?: number;    // Maximum compliance score
  hasOverdue?: boolean; // Has overdue items
  regulator?: string[]; // Has connection to regulator
  page?: number;
  pageSize?: number;
  sort?: 'name' | 'score' | 'cases' | 'activity';
  order?: 'asc' | 'desc';
}
```

### 4.3 Autocomplete

For organization fields in other forms:

```tsx
<OrganizationAutocomplete
  value={selectedOrg}
  onChange={setSelectedOrg}
  placeholder="Search organization..."
/>
```

Returns top 10 matches as user types.

---

## 5. Permissions

### 5.1 Access Rules

| Action | member | manager | admin |
|--------|:------:|:-------:|:-----:|
| View org list | ✅ | ✅ | ✅ |
| View org profile | ✅ | ✅ | ✅ |
| View org cases | ✅ | ✅ | ✅ |
| View org documents | ✅ | ✅ | ✅ |
| Edit org details | ❌ | ❌ | ✅ |
| Manage connections | ❌ | ❌ | ✅ |

### 5.2 Data Visibility

All backoffice staff can view all organizations.

Sensitive fields (if any) may be masked:
- Bank account details
- Personal ID numbers
- Full TPIN (show last 4 only in list)

---

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `GET /orgs/search` | GET | Search organizations |
| `GET /orgs/:id` | GET | Get organization profile |
| `GET /orgs/:id/cases` | GET | Get org's cases |
| `GET /orgs/:id/tickets` | GET | Get org's tickets |
| `GET /orgs/:id/documents` | GET | Get org's documents |
| `GET /orgs/:id/activity` | GET | Get org's audit events |
| `GET /orgs/:id/contacts` | GET | Get org's contacts |

### Search Response

```typescript
interface OrgSearchResponse {
  data: OrganizationSummary[];
  meta: {
    total: number;
    page: number;
    pageSize: number;
    totalPages: number;
  };
}

interface OrganizationSummary {
  id: string;
  name: string;
  tpin: string;
  registrationNumber?: string;
  plan: string;
  complianceScore: number;
  activeCases: number;
  openTickets: number;
  riskLevel: 'low' | 'medium' | 'high';
  lastActivity: Date;
}
```

### Profile Response

```typescript
interface OrganizationProfile {
  id: string;
  name: string;
  tpin: string;
  registrationNumber?: string;
  
  // Details
  industry?: string;
  address?: {
    street: string;
    city: string;
    province: string;
  };
  
  // Subscription
  plan: {
    id: string;
    name: string;
    tier: string;
  };
  memberSince: Date;
  
  // Stats
  complianceScore: number;
  activeCases: number;
  openTickets: number;
  dueThisMonth: number;
  overdueItems: number;
  
  // Connections
  regulatorConnections: {
    regulator: string;
    connected: boolean;
    identifier?: string;
    lastSync?: Date;
  }[];
  
  // Contacts
  primaryContact?: {
    name: string;
    email: string;
    phone?: string;
  };
}
```

---

## File Locations

**Current Implementation:**

| Component | Location |
|-----------|----------|
| Organizations page | `apps/backoffice/app/(authenticated)/(home)/(general)/orgs/page.tsx` |
| Organizations schema | `packages/database/src/schema/core/organizations.ts` |

**To Create:**

| Component | Location |
|-----------|----------|
| Org profile page | `apps/backoffice/app/(authenticated)/(home)/(general)/orgs/[id]/page.tsx` |
| Org profile components | `apps/backoffice/components/orgs/` |
| API routes | `packages/backend/src/modules/orgs/routes.ts` |
| Search service | `packages/api-services/src/domains/orgs/search.service.ts` |

---

## Related Documentation

- [Operations Module](/backoffice/modules/operations) - Cases for organizations
- [Finance Module](/backoffice/modules/finance) - Payments by organization
- [Implementation Plan](/backoffice/implementation-plan) - Build steps

