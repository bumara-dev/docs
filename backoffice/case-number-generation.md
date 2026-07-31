---
title: "Case Number Generation Strategy"
description: "Human-friendly, unique identifiers for cases in the Bumara backoffice system."
---

**Created:** 2026-02-05
**Status:** Implementation Guide

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Current State](#2-current-state)
3. [Requirements](#3-requirements)
4. [Strategies Comparison](#4-strategies-comparison)
5. [Recommended Solution](#5-recommended-solution)
6. [Implementation](#6-implementation)
7. [Examples](#7-examples)
8. [Migration Path](#8-migration-path)

---

## 1. Problem Statement

### Current Challenge
- **UUIDs** (`20000000-0000-0000-0001-000000000007`) are machine-friendly but not human-friendly
- Staff and tenants need to **communicate case references** (phone, email, Slack)
- Support tickets need **memorable case numbers**
- Reports and exports need **sortable, readable identifiers**

### What We Need
A **human-readable case number** that is:
- Short (8-15 characters)
- Memorable
- Unique across the system
- Sortable (chronologically or logically)
- Informative (hints at case type/regulator)
- No collisions (guaranteed uniqueness)

---

## 2. Current State

### Database Schema (submission_jobs)
```typescript
{
  id: UUID (primary key),          // e.g., "uuid-uuid-uuid"
  organizationId: TEXT,
  filingId: UUID | null,
  serviceRequestId: UUID | null,
  regulatorKey: TEXT,              // "zra", "pacra", "napsa", etc.
  status: ENUM,
  priority: ENUM,
  ...
}
```

### Issues
- ❌ UUIDs not suitable for verbal communication
- ❌ No human-friendly reference number
- ❌ Hard to search/filter by case reference
- ❌ Poor UX when staff says "UUID-4a3b..."

---

## 3. Requirements

### Functional Requirements

| Req | Description | Priority |
|-----|-------------|----------|
| FR-1 | Must be unique across all cases | CRITICAL |
| FR-2 | Must be short (≤15 characters) | HIGH |
| FR-3 | Must be easy to communicate verbally | HIGH |
| FR-4 | Should indicate case type or context | MEDIUM |
| FR-5 | Should be sortable chronologically | MEDIUM |
| FR-6 | Must handle high volume (10k+ cases/year) | HIGH |
| FR-7 | Must prevent collisions in distributed system | CRITICAL |

### Non-Functional Requirements
- **Performance**: Generation must be fast (&lt;10ms)
- **Scalability**: Support millions of cases
- **Reliability**: No duplicates, even with concurrent requests
- **Maintainability**: Simple to understand and debug

---

## 4. Strategies Comparison

### Strategy 1: Sequential Global Counter
```
Format: CASE-000001, CASE-000002, etc.
Example: CASE-123456
```

**Pros:**
- ✅ Simple and short
- ✅ Easy to remember
- ✅ Chronologically sortable

**Cons:**
- ❌ Requires database sequence or counter
- ❌ Single point of contention (locking)
- ❌ No context about case type
- ❌ Predictable (security concern)

**Use Case:** Low-volume systems (&lt;1000 cases/day)

---

### Strategy 2: Year-Based Sequential
```
Format: CASE-YYYY-NNNNNN
Example: CASE-2024-001234
```

**Pros:**
- ✅ Chronologically organized by year
- ✅ Counter resets annually (shorter numbers)
- ✅ Easy to archive by year

**Cons:**
- ❌ Still requires sequence per year
- ❌ No context about case type/regulator
- ❌ Longer than simple sequential

**Use Case:** Medium-volume systems with annual cycles

---

### Strategy 3: Prefix-Based with Sequential
```
Format: {PREFIX}-NNNNNN
Example: VAT-123456, PACRA-789012
```

**Pros:**
- ✅ Context about case type/regulator
- ✅ Easy filtering by prefix
- ✅ Familiar pattern (invoice numbers, tickets)

**Cons:**
- ❌ Requires separate sequence per prefix
- ❌ More complex logic
- ❌ Prefix selection can be ambiguous

**Use Case:** Systems with clear categorization

---

### Strategy 4: Timestamp-Based
```
Format: CASE-YYMMDD-XXXXX
Example: CASE-240215-A3B7C
```

**Pros:**
- ✅ Date embedded in number
- ✅ Chronologically sortable
- ✅ No sequence needed (use timestamp + random)

**Cons:**
- ❌ Longer format
- ❌ Random suffix hard to remember
- ❌ Possible collisions if not careful

**Use Case:** Distributed systems

---

### Strategy 5: Hybrid (Type + Year + Sequential)
```
Format: {TYPE}{YY}{MMMMMM}
Example: F24001234 (Filing), S24005678 (Service Request)
```

**Pros:**
- ✅ Compact (9 characters)
- ✅ Type + year + sequence
- ✅ No separators needed
- ✅ Chronologically sortable within type

**Cons:**
- ❌ Less readable without separators
- ❌ Requires parsing to understand

**Use Case:** High-volume systems with type classification

---

### Strategy 6: Snowflake-Style (Recommended for Bumara)
```
Format: BUM-{BASE36_ENCODED_ID}
Example: BUM-A1B2C3D, BUM-X9Y8Z7W
```

**Pros:**
- ✅ Guaranteed unique (database-backed)
- ✅ Short (11-13 characters total)
- ✅ No sequence/counter needed
- ✅ Distributed-friendly
- ✅ Uses existing UUID as source

**Cons:**
- ❌ Not strictly chronological (but can be with timestamp-based UUIDs)
- ❌ Less context about case type

**Use Case:** High-volume, distributed systems (like Bumara)

---

## 5. Recommended Solution

### **Hybrid: Type-Year-Sequential (Strategy 5)**

**Why This Works for Bumara:**

1. **Human-Friendly**: Easy to communicate ("Filing 2024-001234")
2. **Contextual**: Prefix indicates case type
3. **Sortable**: Year + sequence provides chronological order
4. **Scalable**: Can handle 999,999 cases per type per year
5. **Familiar**: Similar to invoice/receipt numbers

### Format Specification

```
{TYPE_CODE}{YY}{NNNNNN}

Where:
- TYPE_CODE: Single letter (1 char)
  - F = Filing
  - S = Service Request
  - J = Submission Job (if exposed separately)

- YY: Two-digit year (2 chars)
  - 24 = 2024, 25 = 2025, etc.

- NNNNNN: Six-digit sequential number (6 chars)
  - Resets to 000001 each year per type
  - Max: 999,999 per type per year

Total Length: 9 characters (no separators)
```

### Examples

```
F24000001  → First filing of 2024
F24000002  → Second filing of 2024
F24123456  → 123,456th filing of 2024
S24000001  → First service request of 2024
S24000002  → Second service request of 2024
F25000001  → First filing of 2025 (counter reset)
```

### Alternative: With Separators (More Readable)

```
{TYPE_CODE}-{YY}-{NNNNNN}

Examples:
F-24-000001
S-24-000001
F-25-000001

Total Length: 11 characters (with separators)
```

---

## 6. Implementation

### 6.1 Database Schema Changes

#### Add Case Number Fields

```sql
-- Add case_number column to submission_jobs
ALTER TABLE submission_jobs
ADD COLUMN case_number TEXT UNIQUE;

-- Add case_number column to filings (if exposed directly)
ALTER TABLE filings
ADD COLUMN case_number TEXT UNIQUE;

-- Add case_number column to service_requests (if exposed directly)
ALTER TABLE service_requests
ADD COLUMN case_number TEXT UNIQUE;

-- Add index for fast lookups
CREATE INDEX idx_submission_jobs_case_number
ON submission_jobs(case_number);
```

#### Add Sequence Counter Table

```sql
CREATE TABLE case_number_sequences (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  entity_type TEXT NOT NULL,        -- 'filing', 'service_request', 'submission_job'
  year INTEGER NOT NULL,             -- 2024, 2025, etc.
  last_sequence INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  UNIQUE(entity_type, year)
);

-- Index for fast lookups
CREATE INDEX idx_case_sequences_type_year
ON case_number_sequences(entity_type, year);
```

### 6.2 Sequence Generation Function

```typescript
// packages/database/src/repositories/case-numbers.ts

import { db } from '../index';
import { caseNumberSequences } from '../schema/system/case-number-sequences';
import { eq, and } from 'drizzle-orm';

export type CaseEntityType = 'filing' | 'service_request' | 'submission_job';

interface GenerateCaseNumberResult {
  caseNumber: string;
  sequence: number;
}

/**
 * Generate next case number for a given entity type.
 * Thread-safe via database transaction.
 */
export async function generateCaseNumber(
  entityType: CaseEntityType
): Promise<GenerateCaseNumberResult> {

  const currentYear = new Date().getFullYear();
  const yearShort = currentYear.toString().slice(-2); // "24" for 2024

  // Type code mapping
  const typeCodeMap: Record<CaseEntityType, string> = {
    filing: 'F',
    service_request: 'S',
    submission_job: 'J',
  };

  const typeCode = typeCodeMap[entityType];

  // Use transaction for atomic increment
  const result = await db.transaction(async (tx) => {
    // Try to get existing sequence for this year
    const existing = await tx
      .select()
      .from(caseNumberSequences)
      .where(
        and(
          eq(caseNumberSequences.entityType, entityType),
          eq(caseNumberSequences.year, currentYear)
        )
      )
      .for('update') // Row-level lock to prevent concurrent updates
      .limit(1);

    let sequence: number;

    if (existing.length > 0) {
      // Increment existing sequence
      sequence = existing[0].lastSequence + 1;

      await tx
        .update(caseNumberSequences)
        .set({
          lastSequence: sequence,
          updatedAt: new Date(),
        })
        .where(eq(caseNumberSequences.id, existing[0].id));
    } else {
      // Create new sequence for this year
      sequence = 1;

      await tx.insert(caseNumberSequences).values({
        entityType,
        year: currentYear,
        lastSequence: sequence,
      });
    }

    // Format case number: {TYPE}{YY}{NNNNNN}
    const caseNumber = `${typeCode}${yearShort}${sequence.toString().padStart(6, '0')}`;

    return { caseNumber, sequence };
  });

  return result;
}

/**
 * Parse a case number back to its components.
 */
export function parseCaseNumber(caseNumber: string): {
  typeCode: string;
  year: number;
  sequence: number;
} | null {

  // Format: {TYPE}{YY}{NNNNNN}
  const match = caseNumber.match(/^([FSJ])(\d{2})(\d{6})$/);

  if (!match) return null;

  const [, typeCode, yearShort, sequenceStr] = match;
  const year = 2000 + parseInt(yearShort, 10);
  const sequence = parseInt(sequenceStr, 10);

  return { typeCode, year, sequence };
}

/**
 * Validate case number format.
 */
export function isValidCaseNumber(caseNumber: string): boolean {
  return /^[FSJ]\d{8}$/.test(caseNumber);
}
```

### 6.3 Update Submission Job Creation

```typescript
// packages/backend/src/modules/compliance/cases/handlers.ts

import { generateCaseNumber } from '@repo/database/repositories/case-numbers';

export async function createSubmissionJobHandler(c: Context) {
  const { filingId, serviceRequestId } = c.req.valid('json');

  // Determine entity type
  const entityType = filingId ? 'filing' : 'service_request';

  // Generate unique case number
  const { caseNumber } = await generateCaseNumber(entityType);

  // Create submission job with case number
  const job = await db.insert(submissionJobs).values({
    organizationId,
    filingId,
    serviceRequestId,
    caseNumber,  // ← NEW: Add case number
    status: 'queued',
    requestedAt: new Date(),
    requestedByUserId: userId,
  }).returning();

  return c.json({
    success: true,
    data: job[0],
  });
}
```

### 6.4 Update API Response Schema

```typescript
// packages/backend/src/modules/compliance/cases/routes.ts

const caseItemSchema = z.object({
  id: z.string().uuid(),
  caseNumber: z.string(),  // ← NEW: Add case number field
  caseType: caseTypeEnum,
  organizationId: z.string(),
  organizationName: z.string().nullable(),
  regulatorKey: z.string().nullable(),
  // ... rest of fields
});
```

### 6.5 Display in UI

```tsx
// apps/backoffice/app/(authenticated)/(home)/(general)/cases/_components/case-header.tsx

export function CaseHeader({ caseDetail }: CaseHeaderProps) {
  return (
    <div>
      {/* Breadcrumb */}
      <Breadcrumb>
        <BreadcrumbList>
          <BreadcrumbItem>
            <BreadcrumbLink href="/cases">Cases</BreadcrumbLink>
          </BreadcrumbItem>
          <BreadcrumbSeparator />
          <BreadcrumbItem>
            <BreadcrumbPage>{caseDetail.caseNumber}</BreadcrumbPage>
          </BreadcrumbItem>
        </BreadcrumbList>
      </Breadcrumb>

      {/* Title */}
      <div className="mt-2 flex items-center gap-3">
        <h1 className="text-2xl font-bold">
          Case {caseDetail.caseNumber}
        </h1>
        <StatusBadge status={caseDetail.status} />
      </div>

      {/* Subtitle */}
      <p className="text-muted-foreground">
        {caseDetail.organizationName} • {caseDetail.obligationName || caseDetail.serviceName}
      </p>
    </div>
  );
}
```

### 6.6 Search by Case Number

```typescript
// apps/backoffice/lib/queries/cases/index.ts

export interface ListCasesParams {
  caseNumber?: string;  // ← NEW: Add case number filter
  type?: CaseType;
  status?: string;
  // ... other filters
}

// Backend handler
export const listCasesHandler: AppRouteHandler<ListCasesRoute> = async (c) => {
  const query = c.req.valid('query');

  const conditions = [];

  // Filter by case number (exact or partial match)
  if (query.caseNumber) {
    conditions.push(
      or(
        eq(submissionJobs.caseNumber, query.caseNumber),
        sql`${submissionJobs.caseNumber} ILIKE ${`%${query.caseNumber}%`}`
      )
    );
  }

  // ... rest of query
};
```

---

## 7. Examples

### Real-World Usage

#### Example 1: Staff Communication
```
Staff A: "I'm working on case F24123456"
Staff B: "Great! That's the VAT return for Acme Corp, right?"
Staff A: "Yes, filing from 2024, case number 123,456"
```

#### Example 2: Support Ticket
```
Ticket Subject: Payment verification needed for F24098765
Description: Case F24098765 (Acme Corp VAT Return Q4 2024)
             requires payment verification before submission.
```

#### Example 3: Report Export
```csv
Case Number,Organization,Type,Status,Due Date
F24000001,Acme Corp,Filing,accepted,2024-01-31
F24000002,Beta Ltd,Filing,submitted,2024-01-31
S24000001,Gamma Inc,Service Request,in_progress,2024-02-15
F24000003,Delta Co,Filing,queued,2024-02-28
```

#### Example 4: Email Notification
```
Subject: Case F24123456 Submitted Successfully

Dear Acme Corp,

Your VAT Return (Case F24123456) has been successfully
submitted to ZRA on your behalf.

Case Details:
- Case Number: F24123456
- Type: VAT Return Q4 2024
- Submitted: 2024-02-15 14:30 CAT
- Status: Awaiting regulator approval

You can track the status in your dashboard.
```

---

## 8. Migration Path

### Phase 1: Add Schema (Non-Breaking)
```sql
-- Add column (nullable initially)
ALTER TABLE submission_jobs
ADD COLUMN case_number TEXT;

-- Add sequence table
CREATE TABLE case_number_sequences (...);
```

### Phase 2: Backfill Existing Cases
```typescript
// Migration script: backfill-case-numbers.ts

async function backfillCaseNumbers() {
  // Get all submission jobs without case numbers, ordered by creation date
  const jobs = await db
    .select()
    .from(submissionJobs)
    .where(isNull(submissionJobs.caseNumber))
    .orderBy(asc(submissionJobs.createdAt));

  for (const job of jobs) {
    const entityType = job.filingId ? 'filing' : 'service_request';
    const jobYear = new Date(job.createdAt).getFullYear();

    // Generate case number for the year of creation
    const { caseNumber } = await generateCaseNumberForYear(
      entityType,
      jobYear
    );

    // Update job
    await db
      .update(submissionJobs)
      .set({ caseNumber })
      .where(eq(submissionJobs.id, job.id));

    console.log(`Backfilled: ${job.id} → ${caseNumber}`);
  }
}
```

### Phase 3: Make Required
```sql
-- After backfill complete, make NOT NULL
ALTER TABLE submission_jobs
ALTER COLUMN case_number SET NOT NULL;

-- Add unique constraint
ALTER TABLE submission_jobs
ADD CONSTRAINT unique_case_number UNIQUE (case_number);
```

### Phase 4: Update UI
- Display case numbers in all lists
- Add search by case number
- Update breadcrumbs to show case number
- Update notifications to include case number

### Phase 5: Deprecate UUID-Only References
- Encourage using case number in communication
- Keep UUID as primary key (database)
- Use case number as display reference (UI/UX)

---

## Alternative Approaches

### Option A: Use UUID as Base (Simpler)
```typescript
// Use first 8 chars of UUID, encode to base32
function generateCaseNumber(uuid: string): string {
  const short = uuid.replace(/-/g, '').slice(0, 16);
  const base32 = encodeBase32(short);
  return `BUM-${base32}`;
}

// Example: BUM-A1B2C3D4
```

**Pros:** No sequence table needed
**Cons:** Less context, harder to remember

### Option B: Regulator-Specific Prefixes
```typescript
const prefixMap = {
  'zra': 'ZRA',
  'pacra': 'PAC',
  'napsa': 'NAP',
  'nhima': 'NHI',
};

// Example: ZRA-24-001234 (ZRA case, 2024, sequence 1234)
```

**Pros:** Clear regulator context
**Cons:** More complex, requires regulator mapping

---

## Summary & Recommendation

### ✅ Recommended: Type-Year-Sequential

**Format:** `{TYPE}{YY}{NNNNNN}`
**Example:** `F24000001`, `S24000002`

**Implementation Checklist:**
- [ ] Add `case_number` column to `submission_jobs`
- [ ] Create `case_number_sequences` table
- [ ] Implement `generateCaseNumber()` function
- [ ] Update submission job creation logic
- [ ] Backfill existing cases
- [ ] Update API schemas
- [ ] Update UI to display case numbers
- [ ] Add search by case number
- [ ] Update documentation

**Benefits:**
- ✅ Human-friendly (9 chars)
- ✅ Type context (F/S prefix)
- ✅ Chronological (year + sequence)
- ✅ Scalable (999,999 per type/year)
- ✅ Unique (database-backed sequence)
- ✅ Simple to implement

**Timeline:**
- Week 1: Schema + backend implementation
- Week 2: Backfill existing data
- Week 3: UI updates
- Week 4: Testing + rollout

---

**Questions or feedback?** Update this document or discuss with the team.
