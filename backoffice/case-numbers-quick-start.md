---
title: "Case Numbers: Quick Start Guide"
description: "Get up and running with human-friendly case numbers in 30 minutes."
---

## What You're Implementing

**Before:** Cases identified by UUID (`20000000-0000-0000-0001-000000000007`)
**After:** Cases identified by case number (`F24000001`, `S24000123`)

**Format:** `{TYPE}{YY}{NNNNNN}`
- `F` = Filing, `S` = Service Request
- `24` = Year 2024
- `000001` = Sequential number (resets each year per type)

---

## Step-by-Step Implementation

### Step 1: Run Database Migration (5 mins)

```bash
# Navigate to database package
cd packages/database

# Run the migration
psql $DATABASE_URL -f migrations/add-case-numbers.sql

# Verify tables created
psql $DATABASE_URL -c "SELECT * FROM case_number_sequences LIMIT 5;"
```

### Step 2: Backfill Existing Cases (10 mins)

```bash
# Run backfill script
pnpm tsx scripts/backfill-case-numbers.ts

# Expected output:
# 🚀 Starting case number backfill...
# ✓ abc123... → F24000001 (filing, 2024, seq: 1)
# ✓ def456... → F24000002 (filing, 2024, seq: 2)
# ✓ ghi789... → S24000001 (service_request, 2024, seq: 1)
# 🎉 Backfill completed successfully!
```

### Step 3: Make Case Number Required (2 mins)

```sql
-- After successful backfill
ALTER TABLE submission_jobs
ALTER COLUMN case_number SET NOT NULL;

ALTER TABLE submission_jobs
ADD CONSTRAINT unique_submission_jobs_case_number
UNIQUE (case_number);
```

### Step 4: Update Backend to Generate Case Numbers (10 mins)

**File:** `packages/backend/src/modules/compliance/cases/handlers.ts`

```typescript
import { generateCaseNumber } from '@repo/database/repositories/case-numbers';
import db from '@repo/database';

// In your create submission job handler
export async function createSubmissionJob(data) {
  // Determine entity type
  const entityType = data.filingId ? 'filing' : 'service_request';

  // Generate case number
  const { caseNumber } = await generateCaseNumber(db, entityType);

  // Create job with case number
  const job = await db.insert(submissionJobs).values({
    ...data,
    caseNumber,  // ← Add this
    status: 'queued',
  }).returning();

  return job[0];
}
```

### Step 5: Update API Schema (3 mins)

**File:** `packages/backend/src/modules/compliance/cases/routes.ts`

```typescript
const caseItemSchema = z.object({
  id: z.string().uuid(),
  caseNumber: z.string(),  // ← Add this field
  caseType: caseTypeEnum,
  // ... rest of fields
});
```

### Step 6: Update Frontend Types (2 mins)

**File:** `apps/backoffice/lib/queries/cases/index.ts`

```typescript
export interface CaseItem {
  id: string;
  caseNumber: string;  // ← Add this field
  caseType: CaseType;
  organizationName: string | null;
  // ... rest of fields
}
```

### Step 7: Display in UI (5 mins)

**File:** `apps/backoffice/app/(authenticated)/(home)/(general)/cases/[id]/_components/case-header.tsx`

```tsx
import { CaseNumberHeader } from '@/components/cases/case-number-badge';

export function CaseHeader({ caseDetail }) {
  return (
    <div>
      {/* Display case number */}
      <CaseNumberHeader caseNumber={caseDetail.caseNumber} />

      {/* Or use badge in tables */}
      <CaseNumberBadge
        caseNumber={caseDetail.caseNumber}
        showIcon
        showCopyButton
      />
    </div>
  );
}
```

---

## Testing

### Test Case Number Generation

```typescript
// Test script (packages/database/scripts/test-case-numbers.ts)
import { generateCaseNumber } from '../src/repositories/case-numbers';
import { db } from '../src';

async function test() {
  // Generate filing case numbers
  const filing1 = await generateCaseNumber(db, 'filing');
  console.log('Filing 1:', filing1);
  // Expected: { caseNumber: 'F24000001', sequence: 1, year: 2024 }

  const filing2 = await generateCaseNumber(db, 'filing');
  console.log('Filing 2:', filing2);
  // Expected: { caseNumber: 'F24000002', sequence: 2, year: 2024 }

  // Generate service request case numbers
  const service1 = await generateCaseNumber(db, 'service_request');
  console.log('Service 1:', service1);
  // Expected: { caseNumber: 'S24000001', sequence: 1, year: 2024 }
}

test().then(() => process.exit(0));
```

Run test:
```bash
pnpm tsx packages/database/scripts/test-case-numbers.ts
```

### Test UI Component

```tsx
// In Storybook or test page
<CaseNumberBadge caseNumber="F24000001" />
<CaseNumberBadge caseNumber="S24000123" variant="secondary" />
<CaseNumberHeader caseNumber="F24001234" />
```

---

## Real-World Examples

### Filing VAT Return
```
Case Number: F24000123
Meaning: Filing, Year 2024, 123rd filing of the year
Display: "Case F24000123" or "F-24-000123"
```

### Service Request (PACRA Name Search)
```
Case Number: S24000045
Meaning: Service Request, Year 2024, 45th service request
Display: "Case S24000045" or "S-24-000045"
```

### Timeline Progression
```
F24000001 → Created Jan 1, 2024 (first filing)
F24000002 → Created Jan 2, 2024 (second filing)
...
F24123456 → Created Dec 15, 2024 (123,456th filing)
F25000001 → Created Jan 1, 2025 (counter resets for new year)
```

---

## Troubleshooting

### Issue: Duplicate case numbers
```
Error: duplicate key value violates unique constraint "unique_submission_jobs_case_number"
```

**Solution:** This shouldn't happen if using the transaction-based generation. If it does:
1. Check if multiple app instances are running
2. Verify row-level locking is working: `SELECT * FROM case_number_sequences FOR UPDATE;`
3. Check sequence table for gaps

### Issue: Case numbers not appearing in UI
```
Error: caseDetail.caseNumber is undefined
```

**Solution:**
1. Check API response includes `caseNumber` field
2. Verify frontend types include `caseNumber`
3. Ensure backfill completed successfully
4. Check database: `SELECT id, case_number FROM submission_jobs LIMIT 10;`

### Issue: Sequence numbers jumping
```
Expected: F24000001, F24000002, F24000003
Actual:   F24000001, F24000005, F24000009
```

**Cause:** Backfill script ran multiple times or sequence table corrupted

**Solution:**
```sql
-- Check sequence state
SELECT * FROM case_number_sequences
WHERE entity_type = 'filing' AND year = 2024;

-- Reset if needed (CAUTION: only before production)
UPDATE case_number_sequences
SET last_sequence = 0
WHERE entity_type = 'filing' AND year = 2024;
```

---

## Useful Queries

### Find cases by case number
```sql
SELECT * FROM submission_jobs
WHERE case_number = 'F24000123';
```

### Find cases by partial case number
```sql
SELECT * FROM submission_jobs
WHERE case_number ILIKE 'F24%'
ORDER BY case_number;
```

### Get case count by type and year
```sql
SELECT
  entity_type,
  year,
  last_sequence as total_cases
FROM case_number_sequences
ORDER BY year DESC, entity_type;
```

### Find missing case numbers (gaps in sequence)
```sql
WITH sequences AS (
  SELECT
    generate_series(1, (
      SELECT last_sequence
      FROM case_number_sequences
      WHERE entity_type = 'filing' AND year = 2024
    )) AS seq
)
SELECT
  'F24' || LPAD(seq::TEXT, 6, '0') AS missing_case_number
FROM sequences
WHERE NOT EXISTS (
  SELECT 1 FROM submission_jobs
  WHERE case_number = 'F24' || LPAD(seq::TEXT, 6, '0')
);
```

---

## API Examples

### Create case with case number
```http
POST /backoffice/cases
Content-Type: application/json

{
  "filingId": "uuid-here",
  "organizationId": "uuid-here"
}

Response:
{
  "success": true,
  "data": {
    "id": "uuid",
    "caseNumber": "F24000123",  ← Auto-generated
    "status": "queued",
    ...
  }
}
```

### Search by case number
```http
GET /backoffice/cases?caseNumber=F24000123

Response:
{
  "success": true,
  "data": [{
    "id": "uuid",
    "caseNumber": "F24000123",
    ...
  }],
  "meta": { ... }
}
```

---

## Files Created

✅ **Schema:**
- `packages/database/src/schema/system/case-number-sequences.ts`

✅ **Repository:**
- `packages/database/src/repositories/case-numbers.ts`

✅ **Migration:**
- `packages/database/migrations/add-case-numbers.sql`

✅ **Scripts:**
- `packages/database/scripts/backfill-case-numbers.ts`

✅ **Components:**
- `apps/backoffice/components/cases/case-number-badge.tsx`

✅ **Documentation:**
- `docs/backoffice/case-number-generation.md` (full spec)
- `docs/backoffice/case-numbers-quick-start.md` (this file)

---

## Next Steps

1. ✅ Run migration
2. ✅ Backfill existing cases
3. ✅ Update backend to generate case numbers
4. ✅ Update API schemas
5. ✅ Update frontend to display case numbers
6. ⬜ Add case number search to filters
7. ⬜ Update email templates to use case numbers
8. ⬜ Update notifications to include case numbers
9. ⬜ Train staff on new case number format
10. ⬜ Update support documentation

---

## Timeline

- **Day 1:** Database migration + backfill (Steps 1-3)
- **Day 2:** Backend implementation (Step 4-5)
- **Day 3:** Frontend implementation (Steps 6-7)
- **Day 4:** Testing + rollout
- **Day 5:** Staff training + documentation updates

**Total:** ~1 week from start to production

---

**Questions?** Check the [full documentation](/backoffice/case-number-generation) or ask the team!
