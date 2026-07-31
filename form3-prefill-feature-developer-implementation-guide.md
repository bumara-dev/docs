---
title: "Form 3 Pre-fill Feature — Developer Implementation Guide"
description: "Subtitle: Bumara Compliance Platform | PACRA Service Request Module Version: 3.0.0 Status: Draft — Pending Technical Review Date: 2026-03-05"
---

**Subtitle:** Bumara Compliance Platform | PACRA Service Request Module
**Version:** 3.0.0
**Status:** Draft — Pending Technical Review
**Date:** 2026-03-05

## Table of Contents

- [0. Stack, Runtime, and Wiring](#0-stack-runtime-and-wiring)
- [1. Product & Platform Context](#1-product--platform-context)
- [2. Architecture Decision Records](#2-architecture-decision-records)
- [3. Complete Data Model](#3-complete-data-model)
- [4. The Generation Service](#4-the-generation-service)
- [5. The Form 3 Wizard (Frontend)](#5-the-form-3-wizard-frontend)
- [6. Database Schema](#6-database-schema)
- [7. API Endpoints](#7-api-endpoints)
- [8. Implementation Sequence](#8-implementation-sequence)
- [9. Error Handling & Edge Cases](#9-error-handling--edge-cases)
- [10. Testing Strategy](#10-testing-strategy)
- [11. Security Considerations](#11-security-considerations)
- [12. Deployment & Configuration](#12-deployment--configuration)
- [13. Glossary](#13-glossary)
- [14. Known Limitations & Future Work](#14-known-limitations--future-work)

---

## 0. Stack, Runtime, and Wiring

### 0.7 Local Development Quick Start

Follow these steps to have a running local environment within 30 minutes.

**1. Clone the repo and install dependencies:**

```bash
git clone <repo-url> && cd bumara
npm install
```

**2. Copy `.env.example` to `.env.local` and fill in required values:**

```bash
cp .env.example .env.local
```

Required values:

```
DATABASE_URL=postgresql://...          # Neon dev branch connection string
CLERK_SECRET_KEY=sk_test_...
CLERK_PUBLISHABLE_KEY=pk_test_...
CLOUDFLARE_ACCOUNT_ID=...
R2_ACCESS_KEY_ID=...
R2_SECRET_ACCESS_KEY=...
R2_BUCKET_NAME=bumara-documents-dev
```

**3. Run database migrations:**

```bash
npx drizzle-kit push
```

**4. Start the API workers (compliance + submissions):**

The Form 3 feature spans two workers. Start them in separate terminals:

```bash
# Terminal 1 — compliance worker (service requests, tasks)
cd apps/api-compliance && npm run dev

# Terminal 2 — submissions worker (documents, uploads)
cd apps/api-submissions && npm run dev
```

**5. Start the Next.js frontend:**

```bash
cd apps/app && npm run dev
```

**6. Test DOCX generation locally (without R2):**

Tests pass the template `ArrayBuffer` directly, bypassing R2:

```bash
npm test -- --filter="form3"
```

**7. Test with a real template against local R2:**

```bash
npx wrangler r2 object put bumara-templates-dev/templates/form3-template-v1.docx \
  --file assets/templates/form3-template-v1.docx \
  --local
```

> **Note:** The template DOCX must exist before calling `GET /form3.docx`. Template prep is a one-time task documented in Section 4. For local development before template prep is done, use test fixtures.

### 0.1 Technology Stack

| Layer | Technology | Notes |
|-------|-----------|-------|
| Frontend | Next.js App Router (TypeScript, RSC) | `apps/app` — React 19, react-hook-form, Zod, TanStack Query |
| API | Hono 4.x on Cloudflare Workers | Multi-worker: `api-compliance`, `api-submissions`, `api-regulators` |
| Database | PostgreSQL (Neon) | `@neondatabase/serverless` driver |
| ORM | Drizzle ORM | `drizzle-orm/neon-http` for Workers |
| Auth | Clerk | Organisation-scoped; `organizationId === Clerk orgId` |
| Storage | Cloudflare R2 | S3-compatible; accessed via Workers R2 binding or SDK |
| Testing | Vitest (unit/integration), Playwright (E2E) | |
| Runtime | Cloudflare Workers | **NO Node.js APIs** (`fs`, `path`, `Buffer`, `process`) in production |

### 0.2 Critical Runtime Constraints

Every code example in this document conforms to these constraints.

**CONSTRAINT 1 — No filesystem.**
`fs`, `path`, `__dirname`, `process.cwd()` do not exist in Workers.

- **Production:** `env.DOCX_TEMPLATES.get(env.FORM3_TEMPLATE_OBJECT_KEY)` (R2 binding)
- **Tests:** Pass `ArrayBuffer` directly to `generateForm3Docx(data, buffer, partials)`

Never write `fs.readFileSync` in any code example.

**CONSTRAINT 2 — Hono route syntax only.**

```typescript
// CORRECT — Hono
app.get('/path', ...middleware, async (c) => {
  return c.json({ data }, 200)
})

**CONSTRAINT 3 — Neon serverless DB connection.**

Workers cannot use standard `pg` TCP connections. Use the existing `buildServiceDependencies(c)` pattern which provides a pre-configured `db` instance:

```typescript
import { buildServiceContext, buildServiceDependencies } from '@repo/backend/core/context'

// Inside route handler:
const ctx = buildServiceContext(c)   // { userId, orgId, roles, ... }
const deps = buildServiceDependencies(c)  // { db, logger, now, env }
```

**CONSTRAINT 4 — JSZip output type.**

```typescript
// Workers — returns ArrayBuffer
const arrayBuffer = await zip.generateAsync({ type: 'arraybuffer' })
return new Response(arrayBuffer, { headers: { 'Content-Type': '...' } })

// Tests — returns Buffer
const buffer = await zip.generateAsync({ type: 'nodebuffer' })
```

No `Buffer.from()` conversion is needed or valid in Workers.

**CONSTRAINT 5 — Multipart upload.**

Workers use the Web Fetch API `FormData`. Do NOT use `multer`, `busboy`, or any Node.js multipart library:

```typescript
const formData = await c.req.formData()
const file = formData.get('file') as File
const arrayBuffer = await file.arrayBuffer()
```

### 0.3 Middleware Stack

The Bumara backend uses a standardised middleware chain defined in `packages/backend/src/core/middleware/auth.ts`.

Every protected route passes through:

1. **`requireAuth`** — Validates Clerk JWT, returns 401 if unauthenticated
2. **`requireOrg`** — Returns 403 if no active organisation context

```typescript
// packages/backend/src/core/middleware/auth.ts (existing)
export const ClerkOrgRoles = {
  ADMIN: "org:admin",
  MANAGER: "org:manager",
  MEMBER: "org:member",
  BACKOFFICE_ADMIN: "org:backoffice_admin",
  BACKOFFICE_MANAGER: "org:backoffice_manager",
  BACKOFFICE_MEMBER: "org:backoffice_member",
}
```

Route handlers access tenant context via the service context builder:

```typescript
// In any route handler
const ctx = buildServiceContext(c)
const orgId = ctx.orgId  // string — always present after requireOrg
```

**Backoffice middleware** — for internal staff routes:

```typescript
// packages/backend/src/core/middleware/auth.ts (existing pattern)
function requireBackoffice(): MiddlewareHandler {
  return async (c, next) => {
    const ctx = buildServiceContext(c)
    if (ctx.orgId !== c.env.BACKOFFICE_ORG_ID) {
      return c.json({ code: 'FORBIDDEN', message: 'Backoffice access required' }, 403)
    }
    await next()
  }
}
```

### 0.4 Tenant Isolation — organizationId Scoping

**Every database query touching tenant-scoped tables MUST filter by `organizationId`.** Failure to do so is a data isolation bug.

The existing codebase enforces this via the service layer:

```typescript
// packages/api-services/src/core/context.ts (existing)
export function requireOrganizationContext(ctx: ServiceContext) {
  if (!ctx.orgId) throw new ServiceError('FORBIDDEN', 'No organization context')
  return { orgId: ctx.orgId }
}
```

Concrete usage in all queries:

```typescript
// SELECT — always filter by organizationId
const record = await deps.db.query.serviceRequests.findFirst({
  where: and(
    eq(serviceRequests.organizationId, ctx.orgId),
    eq(serviceRequests.id, id)
  ),
})

// UPDATE — always filter by organizationId
await deps.db.update(serviceRequests)
  .set({ status: 'in_progress' })
  .where(and(
    eq(serviceRequests.organizationId, ctx.orgId),
    eq(serviceRequests.id, id)
  ))
```

**Code review rule:** No PR touching tenant-scoped tables may merge without `organizationId` filtering present on every query in the changeset.

### 0.5 Environment Configuration

Worker environment variables for the Form 3 feature:

```toml
# wrangler.toml additions for the api-submissions worker
[[r2_buckets]]
binding = "DOCX_TEMPLATES"
bucket_name = "bumara-templates-prod"

[vars]
FORM3_TEMPLATE_OBJECT_KEY = "templates/form3-template-v1.docx"
FORM3_TEMPLATE_VERSION = "v1"
FORM3_MIN_NOMINAL_CAPITAL_ZMW = "5000"
FORM3_MAX_UPLOAD_SIZE_BYTES = "10485760"
FORM3_DOWNLOAD_RATE_LIMIT = "10"
```

Add to the existing env schema (`apps/api-submissions/src/env.ts`):

```typescript
// Additions to the existing envSchema
FORM3_TEMPLATE_OBJECT_KEY: z.string().default("templates/form3-template-v1.docx"),
FORM3_TEMPLATE_VERSION: z.string().default("v1"),
FORM3_MIN_NOMINAL_CAPITAL_ZMW: z.string().default("5000"),
FORM3_MAX_UPLOAD_SIZE_BYTES: z.string().default("10485760"),
FORM3_DOWNLOAD_RATE_LIMIT: z.string().default("10"),
```

### 0.6 Project File Structure

Files created or modified for this feature, placed within the existing monorepo structure:

```
bumara/
├── packages/
│   ├── database/
│   │   └── src/schema/
│   │       └── pacra/company-registration/  # EXISTING — directors, shareholders, etc.
│   │           ├── company-details.ts       # EXISTING — Part A schema
│   │           ├── directors.ts             # EXISTING — Part B schema
│   │           ├── shareholders.ts          # EXISTING — Part C schema
│   │           ├── beneficiaries.ts         # EXISTING — Part D schema
│   │           ├── guarantors.ts            # EXISTING — Part E schema
│   │           ├── secretaries.ts           # EXISTING — Part F schema
│   │           ├── declarations.ts          # EXISTING — Part G schema
│   │           └── applicant.ts             # EXISTING — Part H schema
│   │
│   └── api-services/
│       └── src/domains/
│           ├── compliance/
│           │   └── service-requests.service.ts  # MODIFY — add Form 3 creation logic
│           └── form3/                           # NEW — Form 3 generation domain
│               ├── form3.types.ts               # All TypeScript types
│               ├── form3.validate.ts            # validateForm3Data, validateCompanyName
│               ├── form3.errors.ts              # Custom error classes
│               ├── form3.status.ts              # evaluateServiceRequestReadiness
│               ├── form3-generator.service.ts   # generateForm3Docx and helpers
│               └── form3-generator.utils.ts     # escapeXml, formatDate, checkbox
│
├── apps/
│   ├── api-submissions/
│   │   └── src/routes/
│   │       └── service-requests/                # NEW — Form 3 API routes
│   │           ├── index.ts                     # POST /api/service-requests
│   │           └── [id]/
│   │               ├── form3-draft.ts           # GET + PATCH /form3-draft
│   │               ├── form3-docx.ts            # GET /form3.docx
│   │               └── tasks/
│   │                   └── [taskId]/
│   │                       └── upload.ts        # POST /tasks/:taskId/upload
│   │
│   └── app/
│       ├── features/
│       │   └── pacra/
│       │       └── components/
│       │           └── form3/                   # NEW — Form 3 wizard UI
│       │               ├── Form3Wizard.tsx       # Root wizard component
│       │               ├── steps/
│       │               │   ├── StepPartA.tsx
│       │               │   ├── StepDirectors.tsx
│       │               │   ├── StepShareholders.tsx
│       │               │   ├── StepBeneficialOwnership.tsx
│       │               │   ├── StepGuarantors.tsx
│       │               │   ├── StepSecretary.tsx
│       │               │   ├── StepDeclaration.tsx
│       │               │   ├── StepLodgingPerson.tsx
│       │               │   ├── StepReview.tsx
│       │               │   └── StepDownload.tsx
│       │               └── shared/
│       │                   ├── PersonCard.tsx
│       │                   ├── PersonFields.tsx
│       │                   └── AddressFields.tsx
│       └── lib/queries/
│           └── form3/                           # NEW — React Query hooks
│               ├── types.ts
│               ├── fetchers/
│               │   └── form3.ts
│               └── hooks/
│                   ├── use-form3-draft.ts
│                   ├── use-form3-save.ts
│                   ├── use-form3-download.ts
│                   └── index.ts
│
├── assets/
│   └── templates/
│       └── partials/                            # NEW — OOXML partial snippets
│           ├── director-row.xml
│           ├── shareholder-row.xml
│           ├── beneficial-owner-row.xml
│           └── guarantor-row.xml
│
├── scripts/
│   └── validate-placeholders.ts                 # NEW — Template validation script
│
└── drizzle/
    └── migrations/
        └── 0012_form3_data.sql                  # NEW — Migration for form3 columns
```

---

## 1. Product & Platform Context

### 1.1 Platform Summary

Bumara is a managed compliance operations platform for Zambian SMEs. It is **NOT** a self-serve filing tool. Tenants provide inputs and payments; Bumara backoffice staff execute regulatory submissions on their behalf.

**Regulators served:** PACRA, ZRA, NAPSA, NHIMA, Local Government.

#### Core Concepts

**Obligations** — Recurring statutory duties.
- Table: `org_obligations` (links `obligation_templates` to an organisation)
- Example: "PACRA Annual Returns" — must be filed every year by the anniversary of incorporation.

**Filings** — Period-based instances generated from Obligations.
- Table: `filings` — unique on `(organizationId, obligationId, periodKey)`
- Example: "2025 Annual Return" for company XYZ, due 2025-06-15.

**Service Requests** — One-off regulator actions.
- Table: `service_requests` — scoped by `organizationId`, linked to `service_templates`
- Example: "Company Incorporation" — a one-time PACRA submission to register a new company.

**Tasks** — Tenant-provided gating inputs.
- Table: `tasks` — linked to either a `filingId` OR `serviceRequestId` (XOR constraint)
- Types: `upload_document`, `fill_form`, `review_approve`, `payment_action`, `info_request`
- Example: "Upload signed Form 3" — tenant must provide the physically signed document.

**Submission Jobs** — Backoffice execution records.
- Table: `submission_jobs` — created when a filing/request reaches `ready_for_submission`
- Example: Backoffice agent submits Form 3 to PACRA portal, records tracking number.

**Payments** — Two-step flow: Tenant → Bumara (platform fee), Bumara → Regulator (statutory fee).
- Tables: `payment_requests` (tenant→Bumara), `regulator_payouts` (Bumara→regulator)
- Rule: Money never flows directly from tenant to regulator.

**Tickets** — SLA-tracked communication between tenant and backoffice.
- Table: `tickets` — types: `data_request`, `payment_request`, `clarification`, `correction`

**Documents** — Immutable evidence store.
- Table: `documents` — kinds: `source`, `workpaper`, `submission`, `payment_proof`, `outcome_proof`, `certificate`
- R2 storage key format: `documents/{organizationId}/{entityId}/{documentId}/{filename}`

**Timeline Events** — Forward-only audit trail.
- Table: `timeline_events` — every status change, document upload, payment, and assignment.

#### Hard Rules

| Rule | Enforcement |
|------|------------|
| Data isolation | ALL queries filter by `organizationId`. No cross-tenant reads or writes. |
| Audit trail | ALL state changes produce `timeline_events` rows. Forward-only. |
| Money flow | Tenant → Bumara → Regulator. Never direct. |
| Task authority | Tasks are completed by tenants ONLY. Backoffice cannot mark tenant tasks as done. |

### 1.2 Feature Context — Form 3 Pre-fill

Form 3 is the official PACRA "Application for Incorporation" form.
**Mandate:** Companies Act 2017 (Act No. 10 of 2017), Regulation 4, Sections 12, 13, and 94.

#### End-to-End Flow

```
┌──────────────┐    ┌──────────────┐    ┌───────────────┐    ┌──────────────┐
│ 1. Initiate  │───>│ 2. Fill      │───>│ 3. Download   │───>│ 4. Upload    │
│ Service Req  │    │ Wizard       │    │ Pre-filled    │    │ Signed DOCX  │
│ POST /api/   │    │ PATCH draft  │    │ GET .docx     │    │ POST upload  │
└──────────────┘    └──────────────┘    └───────────────┘    └──────────────┘
                                                                     │
                    ┌──────────────┐    ┌───────────────┐            │
                    │ 7. Backoffice│<───│ 6. Submission │<───────────┘
                    │ Submits to   │    │ Job Created   │    ┌──────────────┐
                    │ PACRA        │    │ (readiness OK)│<───│ 5. Payment   │
                    └──────────────┘    └───────────────┘    │ Verified     │
                                                             └──────────────┘
```

1. **Tenant initiates incorporation** → `POST /api/v1/service-requests`
   Creates a Service Request (`type: company_incorporation`) and a `DOCUMENT_UPLOAD` Task.

2. **Tenant fills Form 3 Wizard** (8 parts, multi-step) → `PATCH /api/v1/service-requests/:id/form3-draft`
   Auto-saves on each step change. Wizard is idempotent: reopening restores the draft.

3. **Tenant downloads pre-filled DOCX** → `GET /api/v1/service-requests/:id/form3.docx`
   Bumara generates Form 3 with tenant data. Tenant prints, obtains wet signatures.

4. **Tenant uploads signed document** → `POST /api/v1/service-requests/:id/tasks/:taskId/upload`
   Task transitions to `done`. Service request readiness is re-evaluated.

5. **Payment is verified** (existing Payments module — out of scope for this guide).

6. **Readiness check passes** → Submission Job created by backoffice.

7. **Backoffice submits to PACRA.** Records reference number and certificate.
   Service request transitions to `accepted`. Certificate document stored.

---

## 2. Architecture Decision Records

### ADR-1: Template Placeholder Replacement

**Status:** Accepted
**Date:** 2026-03-05

#### Context

The PACRA Form 3 DOCX has complex table layouts, Bookman Old Style fonts, a PACRA logo (`rId5 → media/image1.jpeg`), FFE599-yellow section headers, and precise column widths. Any approach that rebuilds the layout programmatically risks producing a document that deviates visually from the official form.

#### Decision

Modify the original DOCX once during template prep to inject `{{PLACEHOLDER}}` tokens into cell text nodes. At generation time, load the template via JSZip, replace tokens with tenant data in `word/document.xml`, and return the result. The document structure (XML elements, styles, images) is never rebuilt — only `<w:t>` text content is replaced.

#### Rationale

Preserves 100% visual fidelity to the official PACRA form. Changes are limited to text content, so fonts, logos, column widths, and shading are inherited from the original.

#### Consequences

- Template prep is a manual, one-time workflow (see Section 4.3).
- Any PACRA form revision requires re-running template prep.
- The `validate-placeholders.ts` script must pass before committing a template.

#### Alternatives Considered

1. **`docx` npm library (rebuild from scratch)** — High maintenance; hard to match official layout pixel-for-pixel.
2. **HTML → PDF** — PACRA requires `.docx` format, not PDF.
3. **LibreOffice headless conversion** — Not available in Cloudflare Workers runtime.

---

### ADR-2: Two-layer Run-split Defence

**Status:** Accepted
**Date:** 2026-03-05

#### Context

Word splits continuous text across multiple `<w:r>` runs. A placeholder like `{{COMPANY_NAME}}` can appear in `document.xml` as:

```xml
<w:r><w:t>{{</w:t></w:r>
<w:r><w:t>COMPANY_NAME</w:t></w:r>
<w:r><w:t>}}</w:t></w:r>
```

String replacement against raw XML silently fails for any split placeholder.

#### Decision

Two-layer defence — not one layer, not the other alone:

**Layer 1 (primary):** Template prep enforcement + `validate-placeholders.ts` script. The script parses every `<w:p>` paragraph, extracts all `<w:t>` text nodes, concatenates them, and verifies that every `{{...}}` token appears within a single `<w:r>` run. It fails CI if any token is split.

**Layer 2 (safety net):** `normalizePlaceholders(xml)` runs before every replacement. It scans each `<w:p>` block, detects adjacent `<w:r>` runs whose collective text contains `{{...}}`, and merges them into a single run inheriting `<w:rPr>` from the first run.

#### Rationale

Layer 1 catches splits at development time (fast, deterministic). Layer 2 catches splits caused by someone opening the template in Word and re-saving (runtime safety net). Neither alone is sufficient.

#### Consequences

- Template prep workflow must include running `validate-placeholders.ts`.
- `normalizePlaceholders` adds ~5ms overhead per generation — acceptable.

---

### ADR-3: Dynamic OOXML Cloning for Repeatable Sections

**Status:** Accepted
**Date:** 2026-03-05

#### Context

Parts B (Directors), C (Shareholders), D (Beneficial Ownership), and E (Guarantors) are repeatable. The number of people per section is unknown at template prep time.

#### Decision

Each repeatable section uses a block-level placeholder (e.g. `{{DIRECTOR_SECTIONS}}`). At generation time, the generator:

1. Loads the partial XML snippet from `assets/templates/partials/director-row.xml`
2. Clones the snippet N times (once per director)
3. Replaces per-field placeholders (`{{DIR_FIRST_NAME}}`, etc.) within each clone
4. Joins all clones and replaces the block-level placeholder in `document.xml`

Partial XML snippets are extracted from the real template during template prep and stored in `assets/templates/partials/*.xml`. They are **never hand-written in TypeScript**.

#### Rationale

- Partials are real OOXML, ensuring visual consistency with the template.
- Adding a new director is a matter of cloning + replacing, not constructing XML.

#### Consequences

- Partial files must be updated if the Form 3 layout changes.
- The `validate-placeholders.ts` script must also validate partials.

---

### ADR-4: Signed Upload Modelled as a Task

**Status:** Accepted
**Date:** 2026-03-05

#### Context

The tenant must upload a physically signed document before backoffice can submit to PACRA. This is a tenant-provided input that gates submission.

#### Decision

Model as a Task with `taskType: "upload_document"` and `required: true`, created at the same time as the Service Request. The task's `completionTrigger` is `"doc:form3_signed"`. Re-upload (corrected signature) updates the existing task's linked document — it does NOT create a duplicate task.

#### Rationale

Aligns with the existing compliance task/readiness system. The `tasks` table already has a `(serviceRequestId, templateKey)` unique constraint for idempotency.

#### Consequences

- Re-uploads must mark the old document as superseded (`status: 'archived'`).
- `evaluateServiceRequestReadiness` must check this task's status.

---

### ADR-5: XML Escaping Boundary

**Status:** Accepted
**Date:** 2026-03-05

#### Context

User data may contain XML special characters (`&`, `<`, `>`, `"`, `'`). Inserting these unescaped into `document.xml` produces a file that Word refuses to open.

#### Decision

Define two categories of replacement value, enforced by TypeScript types:

```typescript
type XmlFieldValue = string   // Result of escapeXml(). Applied to ALL user data.
type XmlBlock = string        // Raw OOXML. NEVER pass through escapeXml().
```

The replacements map has type `Record<string, XmlFieldValue | XmlBlock>`.

The generator builds this map in two separate steps:
- **Step A — field values:** `escapeXml(value)` before adding to the map.
- **Step B — section blocks:** Inject the raw XML string WITHOUT calling `escapeXml`.

`applyReplacements` itself does NOT call `escapeXml`. Callers are responsible for escaping values before adding them to the map.

#### Rationale

Eliminates the "when does escaping happen?" ambiguity that causes double-escaping bugs. The type distinction makes the contract unambiguous and statically checkable.

#### Consequences

- `applyReplacements` must document: "values must be pre-escaped."
- All call sites building the replacements map must escape field values.
- Code review must verify escaping at the call site, not in the utility.

---

### ADR-6: declarantCapacity as a Union Type

**Status:** Accepted
**Date:** 2026-03-05

#### Context

Part G requires the declarant to be either "First Director" OR "First Secretary" — mutually exclusive. Two boolean fields allow invalid states.

#### Decision

Use a union type:

```typescript
type DeclarantCapacity = "first_director" | "first_secretary"
```

The DOCX generator maps this to two checkboxes:

```typescript
DECL_IS_DIRECTOR: checkbox(data.partG.declarantCapacity === "first_director")
DECL_IS_SECRETARY: checkbox(data.partG.declarantCapacity === "first_secretary")
```

This matches the existing `declarantRoleEnum` in `packages/database/src/schema/enums.ts`.

---

### ADR-7: financialYearEnd as Two Distinct Fields

**Status:** Accepted
**Date:** 2026-03-05

#### Context

`firstFinancialYearEnd` is a specific date needed for Form 3. `recurringYearEndMonthDay` is a pattern (`MM-DD`) needed for Obligations scheduling. A single field cannot serve both purposes.

#### Decision

Two fields on PartA:

```typescript
firstFinancialYearEnd: string     // YYYY-MM-DD — appears on Form 3
recurringYearEndMonthDay: string  // MM-DD — used by Obligations module for scheduling
```

---

### ADR-8: Service Request Idempotency

**Status:** Accepted
**Date:** 2026-03-05

#### Context

A user who double-clicks "Start Incorporation" would create duplicate service requests, producing duplicate tasks, duplicate wizard state, and potentially duplicate PACRA submissions.

#### Decision

`POST /api/v1/service-requests` for `company_incorporation` must be idempotent by tenant:

1. Before creating, check for an existing service request with `name = 'company_incorporation'` AND `status NOT IN ('accepted', 'cancelled', 'waived')` for the same `organizationId`.
2. If one exists, return it with HTTP 200 and `{ created: false }`.
3. Only create a new record if none exists in an active state.

Response contract:

```typescript
// 200: Existing record returned
{ serviceRequest: ServiceRequest, tasks: Task[], created: false }

// 201: New record created
{ serviceRequest: ServiceRequest, tasks: Task[], created: true }
```

This means "start incorporation" is safe to call multiple times; only the first call creates a new record.

---

## 3. Complete Data Model

All types are defined in `packages/api-services/src/domains/form3/form3.types.ts`.

### 3.1 Custom Error Classes

File: `packages/api-services/src/domains/form3/form3.errors.ts`

```typescript
export class Form3Error extends Error {
  constructor(message: string, public readonly code: string) {
    super(message)
    this.name = 'Form3Error'
  }
}

export class Form3TemplateNotFoundError extends Form3Error {
  constructor(objectKey: string) {
    super(`Template not found in R2: ${objectKey}`, 'TEMPLATE_NOT_FOUND')
  }
}

export class Form3TemplateLoadError extends Form3Error {
  constructor(objectKey: string, cause: unknown) {
    super(`Failed to load template from R2: ${objectKey}`, 'TEMPLATE_LOAD_ERROR')
    this.cause = cause
  }
}

export class Form3GenerationError extends Form3Error {
  constructor(message: string, cause?: unknown) {
    super(message, 'GENERATION_ERROR')
    this.cause = cause
  }
}

export class Form3ValidationError extends Form3Error {
  constructor(public readonly fieldErrors: Record<string, string>) {
    super('Form3 data validation failed', 'VALIDATION_ERROR')
  }
}
```

### 3.2 Service Request Status Machine

File: `packages/api-services/src/domains/form3/form3.status.ts`

The existing `serviceRequestStatusEnum` in the codebase defines:

```typescript
// packages/database/src/schema/enums.ts (EXISTING)
export const serviceRequestStatusEnum = pgEnum("service_request_status", [
  "pending_data",            // Wizard not yet complete — Form 3 maps to this
  "in_progress",             // Data submitted, processing — maps to "pending_signature"
  "ready_for_submission",    // All gates passed
  "awaiting_payment",        // Payment pending
  "submission_in_progress",  // Backoffice actively submitting
  "submitted",               // Sent to PACRA
  "accepted",                // PACRA accepted
  "needs_correction",        // PACRA returned
  "waived",                  // Obligation waived
  "cancelled",               // Cancelled
])
```

**Form 3 status mapping** (how Form 3 lifecycle maps to existing enum values):

| Form 3 Lifecycle Stage | DB Status | Trigger |
|------------------------|-----------|---------|
| Wizard created, not complete | `pending_data` | `POST /service-requests` |
| DOCX first generated | `in_progress` | `GET /form3.docx` |
| Signed document uploaded | `awaiting_payment` | `POST /tasks/:taskId/upload` |
| Payment verified | `ready_for_submission` | Payment webhook |
| Submission Job created | `submission_in_progress` | Backoffice action |
| Submitted to PACRA | `submitted` | Backoffice confirms |
| PACRA accepted | `accepted` | Backoffice records |
| PACRA returned | `needs_correction` | Backoffice records |

**Valid transitions (directed graph):**

```
pending_data ──────────> in_progress           (on: DOCX first generated)
in_progress ───────────> in_progress           (on: DOCX re-generated, no change)
in_progress ───────────> awaiting_payment      (on: signed document uploaded)
awaiting_payment ──────> ready_for_submission  (on: payment verified)
awaiting_payment ──────> in_progress           (on: re-upload corrected doc)
ready_for_submission ──> submission_in_progress (on: Submission Job created)
submission_in_progress > submitted             (on: backoffice confirms submission)
submitted ─────────────> accepted              (on: PACRA acceptance)
submitted ─────────────> needs_correction      (on: PACRA correction notice)
needs_correction ──────> in_progress           (on: corrected DOCX re-generated)
```

**Readiness evaluation function:**

```typescript
// packages/api-services/src/domains/form3/form3.status.ts

import type { NodePgDatabase } from 'drizzle-orm/node-postgres'
import { and, eq } from 'drizzle-orm'
import { tasks, serviceRequests } from '@repo/database/schema'

type ServiceRequestStatus =
  | "pending_data" | "in_progress" | "ready_for_submission"
  | "awaiting_payment" | "submission_in_progress" | "submitted"
  | "accepted" | "needs_correction" | "waived" | "cancelled"

export interface ReadinessResult {
  isReady: boolean
  newStatus: ServiceRequestStatus
  reason?: string
}

export async function evaluateServiceRequestReadiness(
  serviceRequestId: string,
  organizationId: string,
  db: NodePgDatabase<any>
): Promise<ReadinessResult> {
  // 1. Fetch all tasks for the service request
  const srTasks = await db.select().from(tasks).where(
    and(
      eq(tasks.organizationId, organizationId),
      eq(tasks.serviceRequestId, serviceRequestId)
    )
  )

  // 2. Check required tasks with upload_document type
  const uploadTasks = srTasks.filter(t => t.taskType === 'upload_document' && t.required)
  const hasIncompleteUploadTask = uploadTasks.some(t => t.status !== 'done')
  if (hasIncompleteUploadTask) {
    return {
      isReady: false,
      newStatus: 'in_progress',
      reason: 'Signed Form 3 document has not been uploaded'
    }
  }

  // 3. Check all other required tasks
  const otherRequired = srTasks.filter(t => t.taskType !== 'upload_document' && t.required)
  const hasIncompleteOther = otherRequired.some(t => t.status !== 'done')
  if (hasIncompleteOther) {
    return {
      isReady: false,
      newStatus: 'in_progress',
      reason: 'Not all required tasks are completed'
    }
  }

  // 4. Check payment status (delegate to existing payment readiness module)
  // Import from packages/api-services/src/domains/compliance/readiness-core.ts
  const paymentReady = await checkPaymentReadiness(serviceRequestId, organizationId, db)
  if (!paymentReady) {
    return {
      isReady: false,
      newStatus: 'awaiting_payment',
      reason: 'Payment has not been verified'
    }
  }

  // 5. All gates passed
  return { isReady: true, newStatus: 'ready_for_submission' }
}

async function checkPaymentReadiness(
  serviceRequestId: string,
  organizationId: string,
  db: NodePgDatabase<any>
): Promise<boolean> {
  // Uses existing readiness-core.ts pattern:
  // loadPaymentSnapshot() + summarizePaymentReadiness()
  const { summarizePaymentReadiness, loadPaymentSnapshot } = await import(
    '../compliance/readiness-core'
  )
  const snapshot = await loadPaymentSnapshot('service_request', serviceRequestId, organizationId, db)
  const result = summarizePaymentReadiness(snapshot)
  return result.met
}
```

**Call this function after:**
- `POST /tasks/:taskId/upload` (task completion)
- Payment verification webhook (handled by Payments module)

### 3.3 Enums and Primitive Types

```typescript
// packages/api-services/src/domains/form3/form3.types.ts

/**
 * CompanyType — maps to PACRA Form 3 Part A "Type of Company"
 * Each value corresponds to a checkbox on the physical form.
 */
export type CompanyType =
  | "private_shares"       // Private Company Limited by Shares (most common)
  | "private_guarantee"    // Private Company Limited by Guarantee
  | "public_limited"       // Public Limited Company
  | "unlimited_private"    // Unlimited Private Company

/**
 * CompanyCategory — optional sub-classification required for regulated industries.
 * Only applicable when the company operates in banking, insurance, or financial services.
 */
export type CompanyCategory =
  | "local_bank"
  | "foreign_bank"
  | "insurance"
  | "reinsurance"
  | "bureau_de_change"
  | "financial_institution"
  | "other"

/**
 * ArticlesType — whether company uses PACRA's standard articles or custom ones.
 * Maps to existing articlesTypeEnum in packages/database/src/schema/enums.ts
 */
export type ArticlesType = "Standard" | "Non-Standard"

/** Gender — PACRA form field */
export type Gender = "male" | "female"

/**
 * IdentityType — type of identification document.
 * Maps to existing identityEnum: "nrc" | "passport" | "drivers_license" | "permit"
 */
export type IdentityType = "nrc" | "passport" | "drivers_license" | "permit"

/** ShareClass — classification of shares issued */
export type ShareClass = "ordinary" | "preferential" | "other"

/**
 * DeclarantCapacity — Part G mutually exclusive checkbox (ADR-6).
 * Maps to existing declarantRoleEnum (using relevant subset).
 */
export type DeclarantCapacity = "first_director" | "company_secretary"
```

### 3.4 Shared Structures

```typescript
/**
 * Physical address as structured by PACRA forms.
 * All fields required — PACRA rejects incomplete addresses.
 */
export interface Address {
  plot: string             // Plot/Stand number
  street: string           // Street name
  area: string             // Area/suburb
  town: string             // Town/city
  province: ZambiaProvince // Province (Zambian provinces only)
  country: string          // Default: "Zambia"
}

/** Zambian provinces — used for address validation */
export type ZambiaProvince =
  | "Central" | "Copperbelt" | "Eastern" | "Luapula"
  | "Lusaka" | "Muchinga" | "Northern" | "North-Western"
  | "Southern" | "Western"

/** Postal address — P.O. Box format */
export interface PostalAddress {
  postBox: string
  area: string
  town: string
  province: ZambiaProvince
}

/** Phone numbers — mobile is required, landline optional */
export interface PhoneNumbers {
  mobile: string           // Must match: /^\+260\d{9}$/ (Zambian format)
  landline?: string
}

/**
 * PersonParticulars — base interface for all individuals on the form.
 * Used by Directors, Shareholders (individual), Beneficial Owners,
 * Guarantors, Secretary (individual), and Lodging Person.
 */
export interface PersonParticulars {
  firstName: string
  surname: string
  gender: Gender
  dateOfBirth: string          // YYYY-MM-DD
  nationality: string
  identityType: IdentityType
  identityNumber: string
  isResidentInZambia: boolean  // V11: if true, address.country must be "Zambia"
  email: string
  mobile: string               // +260XXXXXXXXX
  landline?: string
  physicalAddress: Address
}

/**
 * BodyCorporate — for corporate shareholders, corporate secretary,
 * and corporate beneficial owners.
 */
export interface BodyCorporate {
  corporateName: string
  incorporationNumber: string  // Registration number
  jurisdiction: string         // Country of incorporation
  registeredOffice: Address
  contactEmail: string
  contactPhone: PhoneNumbers
}
```

### 3.5 Part A through Part H

```typescript
// ─── Part A: Company Details ─────────────────────────────────────

export interface PartA {
  companyName: string                    // V26: no trailing "(In Liquidation)" or "(In Receivership)"
  approvedNameReference?: string         // PACRA name clearance reference (optional)
  companyType: CompanyType
  companyCategory?: CompanyCategory      // Required only for regulated industries
  companyCategoryOther?: string          // Free text if category === "other"
  principalBusinessActivity: string      // V: min 10 chars
  otherActivities?: string
  articlesRestrictBusiness: boolean      // Does the articles of association restrict business?
  articlesType: ArticlesType

  // Capital structure
  nominalCapitalZmw: number             // V7: >= FORM3_MIN_NOMINAL_CAPITAL_ZMW (5000)
  numberOfShares: number                // V: nominalCapitalZmw / parValue === numberOfShares
  shareClass: ShareClass
  shareClassOther?: string              // Free text if shareClass === "other"
  parValue: number                      // Price per share in ZMW

  // Addresses
  registeredOffice: Address
  postalAddress: PostalAddress
  principalPlaceOfBusiness?: Address    // If different from registered office

  // Contact
  contactEmail: string
  contactPhone: string                  // +260XXXXXXXXX
  contactLandline?: string

  // Financial year (ADR-7)
  firstFinancialYearEnd: string         // YYYY-MM-DD — appears on Form 3
  recurringYearEndMonthDay: string      // MM-DD — used by Obligations module

  // Investment (optional)
  pledgedInvestment?: number            // ZMW
}

// ─── Part B: Directors ───────────────────────────────────────────

export interface Director extends PersonParticulars {
  occupation: string
  postalAddress: PostalAddress
  position: string                      // e.g. "Director", "Managing Director"
  appointmentDate: string               // YYYY-MM-DD
}

export interface PartB {
  directors: Director[]
  // V1: directors.length >= 2 for private_shares, private_guarantee, unlimited_private
  // V2: directors.length >= 3 for public_limited
}

// ─── Part C: Shareholders ────────────────────────────────────────

export interface ShareAllocation {
  shareClass: ShareClass
  shareClassOther?: string
  numberOfShares: number
  amountPaidZmw?: number
}

export interface IndividualShareholder extends PersonParticulars {
  kind: "individual"
  postalAddress: PostalAddress
  allocations: ShareAllocation[]
}

export interface CorporateShareholder {
  kind: "corporate_body"
  body: BodyCorporate
  representativeName: string
  representativeIdentityType: IdentityType
  representativeIdentityNumber: string
  allocations: ShareAllocation[]
}

export type Shareholder = IndividualShareholder | CorporateShareholder

export interface PartC {
  shareholders: Shareholder[]
  // V3: Required for private_shares, public_limited, unlimited_private
}

// ─── Part D: Beneficial Owners ───────────────────────────────────

export interface BeneficialOwner extends PersonParticulars {
  postalAddress: PostalAddress
  occupation: string
  ownershipPercent: number              // 0–100
  ownershipNature: "direct" | "indirect" | "both"
  controlDetails?: string              // How control is exercised
  isPep: boolean                       // Politically Exposed Person
  pepDetails?: string                  // Required if isPep === true
  linkedShareholderRef?: string        // Optional cross-reference
}

export interface PartD {
  beneficialOwners: BeneficialOwner[]
  // V3: Required for private_shares, public_limited, unlimited_private
}

// ─── Part E: Guarantors ──────────────────────────────────────────

export interface Guarantor extends PersonParticulars {
  postalAddress: PostalAddress
  occupation: string
  guaranteedAmountZmw: number
}

export interface PartE {
  guarantors: Guarantor[]
  // V4: Required for private_guarantee only
}

// ─── Part F: Company Secretary ───────────────────────────────────

export interface CompanySecretaryIndividual extends PersonParticulars {
  type: "individual"
  postalAddress: PostalAddress
}

export interface CompanySecretaryFirm {
  type: "firm"
  body: BodyCorporate
  contactPersonName: string
}

export type CompanySecretary = CompanySecretaryIndividual | CompanySecretaryFirm

export interface PartF {
  secretary: CompanySecretary
}

// ─── Part G: Declaration ─────────────────────────────────────────

export interface PartG {
  declarantName: string
  declarantCapacity: DeclarantCapacity  // ADR-6: mutually exclusive
  declarationDate?: string             // YYYY-MM-DD — auto-filled on generation
}

// ─── Part H: Lodging Person ──────────────────────────────────────

export interface PartH extends PersonParticulars {
  postalAddress: PostalAddress
  capacity: string                     // e.g. "First Director", "Company Secretary"
}
```

### 3.6 Root Form3Data Interface

```typescript
export interface Form3Data {
  serviceRequestId: string
  organizationId: string
  templateVersion?: string              // e.g. "v1" — must match env.FORM3_TEMPLATE_VERSION
  generatedAt?: string                  // ISO timestamp — set on DOCX generation
  incorporationDate?: string            // YYYY-MM-DD — set by backoffice after acceptance
  lastSavedAt?: string                  // ISO timestamp — for optimistic locking

  partA: PartA
  partB: PartB
  partC?: PartC                         // Required based on companyType (V3)
  partD?: PartD                         // Required based on companyType (V3)
  partE?: PartE                         // Required for private_guarantee (V4)
  partF: PartF
  partG: PartG
  partH: PartH
}
```

### 3.7 Validation Rules (V1–V26)

| Rule | Field/Scope | Condition | Error |
|------|-------------|-----------|-------|
| V1 | `partB.directors` | `length >= 2` for private_shares, private_guarantee, unlimited_private | "At least 2 directors required" |
| V2 | `partB.directors` | `length >= 3` for public_limited | "Public companies require at least 3 directors" |
| V3 | `partC`, `partD` | Required when companyType is private_shares, public_limited, or unlimited_private | "Shareholders and beneficial owners required for this company type" |
| V4 | `partE` | Required when companyType is private_guarantee | "Guarantors required for company limited by guarantee" |
| V5 | `partA.firstFinancialYearEnd` | If `incorporationDate` defined: `firstFinancialYearEnd <= incorporationDate + 18 months` | "Financial year end too far from incorporation date" |
| V6 | All date fields | Must be valid `YYYY-MM-DD` format | "Invalid date format" |
| V7 | `partA.nominalCapitalZmw` | `>= parseInt(env.FORM3_MIN_NOMINAL_CAPITAL_ZMW)` (default: 5000) | "Nominal capital below minimum" |
| V8 | `partA.nominalCapitalZmw` | `nominalCapitalZmw === numberOfShares * parValue` | "Capital/shares/par value mismatch" |
| V9 | `partA.contactPhone` | Matches `/^\+260\d{9}$/` | "Invalid Zambian phone number" |
| V10 | All `PersonParticulars.mobile` | Matches `/^\+260\d{9}$/` | "Invalid Zambian mobile number" |
| V11 | `PersonParticulars` | If `isResidentInZambia === true`, then `physicalAddress.country === "Zambia"` | "Resident must have Zambian address" |
| V12 | `partA.principalBusinessActivity` | `length >= 10` | "Business activity description too short" |
| V13 | `partG.declarantCapacity` | Must be one of the union values | "Invalid declarant capacity" |
| V14 | `partG.declarantName` | Must match a name in `partB.directors` (if "first_director") or `partF.secretary` (if "company_secretary") | "Declarant must be a listed director or secretary" |
| V15 | `Shareholder.allocations` | `sum(numberOfShares) === partA.numberOfShares` across all shareholders | "Total shares allocated must equal total shares issued" |
| V16 | `BeneficialOwner.ownershipPercent` | Each value 0–100; total across all owners &lt;= 100 | "Invalid ownership percentage" |
| V17 | `partA.companyName` | `length >= 3 && length <= 100` | "Company name must be 3–100 characters" |
| V18 | `partB.directors[*].appointmentDate` | Must not be in the future | "Appointment date cannot be future" |
| V19 | `partA.recurringYearEndMonthDay` | Matches `/^\d{2}-\d{2}$/` and is a valid month-day | "Invalid recurring year end format" |
| V20 | `Guarantor.guaranteedAmountZmw` | `> 0` | "Guaranteed amount must be positive" |
| V21 | All email fields | Valid email format | "Invalid email address" |
| V22 | `PersonParticulars.dateOfBirth` | Person must be >= 18 years old | "Person must be at least 18 years old" |
| V23 | `partC.shareholders` | `length >= 1` when partC is present | "At least one shareholder required" |
| V24 | `partD.beneficialOwners` | `length >= 1` when partD is present | "At least one beneficial owner required" |
| V25 | `partE.guarantors` | `length >= 1` when partE is present | "At least one guarantor required" |
| V26 | `partA.companyName` | Must NOT contain trailing "(In Liquidation)" or "(In Receivership)" | "Reserved PACRA suffix not allowed at incorporation" |

---

## 4. The Generation Service

### 4.1 Utility Functions

File: `packages/api-services/src/domains/form3/form3-generator.utils.ts`

#### escapeXml

Escapes XML special characters in user data before inserting into OOXML.

```typescript
/**
 * Escapes XML special characters. Must be called on ALL user-provided string
 * values before they are added to the replacements map.
 *
 * @example escapeXml('AT&T "quotes"')  → 'AT&amp;T &quot;quotes&quot;'
 * @example escapeXml(undefined)         → ''
 */
export function escapeXml(value: string | undefined | null): string {
  if (!value) return ''
  return value
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&apos;')
}
```

| Input | Output |
|-------|--------|
| `AT&T` | `AT&amp;T` |
| `<script>` | `&lt;script&gt;` |
| `He said "hello"` | `He said &quot;hello&quot;` |
| `O'Brien` | `O&apos;Brien` |
| `undefined` | `""` |
| `null` | `""` |

#### formatDate

```typescript
/**
 * Converts ISO date string to PACRA display format: "15th March 2026"
 * Returns empty string for undefined/invalid input.
 */
export function formatDate(isoDate: string | undefined): string {
  if (!isoDate) return ''
  const date = new Date(isoDate)
  if (isNaN(date.getTime())) return ''

  const day = date.getDate()
  const suffix = getDaySuffix(day)
  const month = date.toLocaleString('en-GB', { month: 'long' })
  const year = date.getFullYear()
  return `${day}${suffix} ${month} ${year}`
}

function getDaySuffix(day: number): string {
  if (day >= 11 && day <= 13) return 'th'
  switch (day % 10) {
    case 1: return 'st'
    case 2: return 'nd'
    case 3: return 'rd'
    default: return 'th'
  }
}
```

#### checkbox

```typescript
/**
 * Returns the OOXML checkbox representation.
 * ☒ for checked (U+2612), ☐ for unchecked (U+2610).
 */
export function checkbox(condition: boolean): string {
  return condition ? '☒' : '☐'
}
```

#### escapeRegex

```typescript
/**
 * Escapes a string for safe use inside new RegExp().
 * Prevents injection via placeholder key names.
 */
export function escapeRegex(str: string): string {
  return str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
}
```

#### normalizePlaceholders

```typescript
/**
 * Safety net (ADR-2, Layer 2). Merges split <w:r> runs within <w:p> paragraphs
 * where the collective text contains a {{...}} placeholder pattern.
 *
 * Before:
 *   <w:r><w:rPr>...</w:rPr><w:t>{{</w:t></w:r>
 *   <w:r><w:rPr>...</w:rPr><w:t>COMPANY_NAME</w:t></w:r>
 *   <w:r><w:rPr>...</w:rPr><w:t>}}</w:t></w:r>
 *
 * After:
 *   <w:r><w:rPr>...</w:rPr><w:t>{{COMPANY_NAME}}</w:t></w:r>
 *
 * Inherits <w:rPr> from the FIRST run in the merged sequence.
 */
export function normalizePlaceholders(xml: string): string {
  // Process each paragraph independently
  return xml.replace(/<w:p[ >][\s\S]*?<\/w:p>/g, (paragraph) => {
    // Extract all runs with their text
    const runPattern = /<w:r>([\s\S]*?)<\/w:r>/g
    const runs: Array<{ full: string; rPr: string; text: string }> = []
    let match: RegExpExecArray | null

    while ((match = runPattern.exec(paragraph)) !== null) {
      const inner = match[1]
      const rPrMatch = inner.match(/<w:rPr>[\s\S]*?<\/w:rPr>/)
      const textMatch = inner.match(/<w:t[^>]*>([\s\S]*?)<\/w:t>/)
      runs.push({
        full: match[0],
        rPr: rPrMatch ? rPrMatch[0] : '',
        text: textMatch ? textMatch[1] : '',
      })
    }

    // Concatenate all text and check for placeholders
    const combined = runs.map(r => r.text).join('')
    if (!combined.includes('{{')) return paragraph

    // Check if any single run already contains the full placeholder
    const hasSplit = !runs.some(r => /\{\{[^}]+\}\}/.test(r.text))
    if (!hasSplit) return paragraph

    // Merge: create single run with combined text, inheriting first run's rPr
    const firstRPr = runs[0]?.rPr ?? ''
    const mergedRun = `<w:r>${firstRPr}<w:t xml:space="preserve">${combined}</w:t></w:r>`

    // Replace all original runs with the single merged run
    let result = paragraph
    runs.forEach((run, i) => {
      result = result.replace(run.full, i === 0 ? mergedRun : '')
    })
    return result
  })
}
```

#### applyReplacements

```typescript
/**
 * Replaces all {{PLACEHOLDER}} tokens in the XML string with their values.
 *
 * IMPORTANT: Values in the replacements map must be PRE-ESCAPED by the caller.
 * This function does NOT call escapeXml(). See ADR-5.
 *
 * After all replacements, logs a warning for any remaining {{...}} tokens.
 */
export function applyReplacements(
  xml: string,
  replacements: Record<string, string>,
  serviceRequestId?: string
): string {
  let result = xml
  for (const [key, value] of Object.entries(replacements)) {
    const pattern = new RegExp(escapeRegex(`{{${key}}}`), 'g')
    result = result.replace(pattern, value)
  }

  // Warn about unresolved placeholders
  const unresolvedPattern = /\{\{([^}]+)\}\}/g
  let unresolvedMatch: RegExpExecArray | null
  while ((unresolvedMatch = unresolvedPattern.exec(result)) !== null) {
    console.warn({
      event: 'unresolved_placeholder',
      token: unresolvedMatch[1],
      serviceRequestId,
    })
  }

  return result
}
```

### 4.2 Section Builder Functions

File: `packages/api-services/src/domains/form3/form3-generator.service.ts`

#### buildPartAReplacements

```typescript
import { escapeXml, formatDate, checkbox } from './form3-generator.utils'
import type { PartA, Form3Data } from './form3.types'

export function buildPartAReplacements(partA: PartA): Record<string, string> {
  return {
    // Company identity
    COMPANY_NAME: escapeXml(partA.companyName),
    APPROVED_NAME_REF: escapeXml(partA.approvedNameReference),
    PRINCIPAL_ACTIVITY: escapeXml(partA.principalBusinessActivity),
    OTHER_ACTIVITIES: escapeXml(partA.otherActivities),

    // Company type checkboxes
    IS_PRIVATE_SHARES: checkbox(partA.companyType === 'private_shares'),
    IS_PRIVATE_GUARANTEE: checkbox(partA.companyType === 'private_guarantee'),
    IS_PUBLIC_LIMITED: checkbox(partA.companyType === 'public_limited'),
    IS_UNLIMITED_PRIVATE: checkbox(partA.companyType === 'unlimited_private'),

    // Category (if applicable)
    COMPANY_CATEGORY: escapeXml(partA.companyCategory ?? ''),
    COMPANY_CATEGORY_OTHER: escapeXml(partA.companyCategoryOther ?? ''),

    // Articles
    ARTICLES_RESTRICT: checkbox(partA.articlesRestrictBusiness),
    IS_STANDARD_ARTICLES: checkbox(partA.articlesType === 'Standard'),
    IS_NON_STANDARD_ARTICLES: checkbox(partA.articlesType === 'Non-Standard'),

    // Capital/shares
    NOMINAL_CAPITAL: escapeXml(partA.nominalCapitalZmw.toFixed(2)),
    NUMBER_OF_SHARES: escapeXml(String(partA.numberOfShares)),
    SHARE_CLASS: escapeXml(partA.shareClass),
    SHARE_CLASS_OTHER: escapeXml(partA.shareClassOther ?? ''),
    PAR_VALUE: escapeXml(partA.parValue.toFixed(2)),

    // Registered office address
    REG_PLOT: escapeXml(partA.registeredOffice.plot),
    REG_STREET: escapeXml(partA.registeredOffice.street),
    REG_AREA: escapeXml(partA.registeredOffice.area),
    REG_TOWN: escapeXml(partA.registeredOffice.town),
    REG_PROVINCE: escapeXml(partA.registeredOffice.province),
    REG_COUNTRY: escapeXml(partA.registeredOffice.country),

    // Postal address
    POSTAL_BOX: escapeXml(partA.postalAddress.postBox),
    POSTAL_AREA: escapeXml(partA.postalAddress.area),
    POSTAL_TOWN: escapeXml(partA.postalAddress.town),
    POSTAL_PROVINCE: escapeXml(partA.postalAddress.province),

    // Contact
    CONTACT_EMAIL: escapeXml(partA.contactEmail),
    CONTACT_PHONE: escapeXml(partA.contactPhone),
    CONTACT_LANDLINE: escapeXml(partA.contactLandline ?? ''),

    // Financial year (ADR-7)
    FIRST_FY_END: formatDate(partA.firstFinancialYearEnd),
    RECURRING_FY_END: escapeXml(partA.recurringYearEndMonthDay),

    // Investment
    PLEDGED_INVESTMENT: escapeXml(partA.pledgedInvestment?.toFixed(2) ?? ''),
  }
}
```

#### buildPartFGHReplacements

```typescript
export function buildPartFGHReplacements(data: Form3Data): Record<string, string> {
  const { partF, partG, partH } = data
  const replacements: Record<string, string> = {}

  // Part F — Secretary
  if (partF.secretary.type === 'individual') {
    const sec = partF.secretary
    replacements.SEC_TYPE = escapeXml('Individual')
    replacements.SEC_FIRST_NAME = escapeXml(sec.firstName)
    replacements.SEC_SURNAME = escapeXml(sec.surname)
    replacements.SEC_GENDER = escapeXml(sec.gender)
    replacements.SEC_DOB = formatDate(sec.dateOfBirth)
    replacements.SEC_NATIONALITY = escapeXml(sec.nationality)
    replacements.SEC_ID_TYPE = escapeXml(sec.identityType)
    replacements.SEC_ID_NUMBER = escapeXml(sec.identityNumber)
    replacements.SEC_EMAIL = escapeXml(sec.email)
    replacements.SEC_MOBILE = escapeXml(sec.mobile)
    replacements.SEC_LANDLINE = escapeXml(sec.landline ?? '')
    replacements.SEC_ADDRESS_PLOT = escapeXml(sec.physicalAddress.plot)
    replacements.SEC_ADDRESS_STREET = escapeXml(sec.physicalAddress.street)
    replacements.SEC_ADDRESS_TOWN = escapeXml(sec.physicalAddress.town)
    replacements.SEC_ADDRESS_PROVINCE = escapeXml(sec.physicalAddress.province)
    // Corporate fields empty
    replacements.SEC_CORP_NAME = ''
    replacements.SEC_CORP_REG_NO = ''
    replacements.SEC_CORP_JURISDICTION = ''
  } else {
    const sec = partF.secretary
    replacements.SEC_TYPE = escapeXml('Firm/Body Corporate')
    replacements.SEC_CORP_NAME = escapeXml(sec.body.corporateName)
    replacements.SEC_CORP_REG_NO = escapeXml(sec.body.incorporationNumber)
    replacements.SEC_CORP_JURISDICTION = escapeXml(sec.body.jurisdiction)
    replacements.SEC_EMAIL = escapeXml(sec.body.contactEmail)
    replacements.SEC_MOBILE = escapeXml(sec.body.contactPhone.mobile)
    replacements.SEC_LANDLINE = escapeXml(sec.body.contactPhone.landline ?? '')
    replacements.SEC_ADDRESS_PLOT = escapeXml(sec.body.registeredOffice.plot)
    replacements.SEC_ADDRESS_STREET = escapeXml(sec.body.registeredOffice.street)
    replacements.SEC_ADDRESS_TOWN = escapeXml(sec.body.registeredOffice.town)
    replacements.SEC_ADDRESS_PROVINCE = escapeXml(sec.body.registeredOffice.province)
    replacements.SEC_CONTACT_PERSON = escapeXml(sec.contactPersonName)
    // Individual fields empty
    replacements.SEC_FIRST_NAME = ''
    replacements.SEC_SURNAME = ''
    replacements.SEC_GENDER = ''
    replacements.SEC_DOB = ''
    replacements.SEC_NATIONALITY = ''
    replacements.SEC_ID_TYPE = ''
    replacements.SEC_ID_NUMBER = ''
  }

  // Part G — Declaration (ADR-6)
  replacements.DECL_NAME = escapeXml(partG.declarantName)
  replacements.DECL_IS_DIRECTOR = checkbox(partG.declarantCapacity === 'first_director')
  replacements.DECL_IS_SECRETARY = checkbox(partG.declarantCapacity === 'company_secretary')
  replacements.DECL_DATE = formatDate(partG.declarationDate ?? new Date().toISOString().slice(0, 10))

  // Part H — Lodging Person
  replacements.LODGER_FIRST_NAME = escapeXml(partH.firstName)
  replacements.LODGER_SURNAME = escapeXml(partH.surname)
  replacements.LODGER_ID_TYPE = escapeXml(partH.identityType)
  replacements.LODGER_ID_NUMBER = escapeXml(partH.identityNumber)
  replacements.LODGER_EMAIL = escapeXml(partH.email)
  replacements.LODGER_MOBILE = escapeXml(partH.mobile)
  replacements.LODGER_CAPACITY = escapeXml(partH.capacity)
  replacements.LODGER_ADDRESS_PLOT = escapeXml(partH.physicalAddress.plot)
  replacements.LODGER_ADDRESS_STREET = escapeXml(partH.physicalAddress.street)
  replacements.LODGER_ADDRESS_TOWN = escapeXml(partH.physicalAddress.town)
  replacements.LODGER_ADDRESS_PROVINCE = escapeXml(partH.physicalAddress.province)

  // Metadata
  replacements.GENERATED_DATE = formatDate(new Date().toISOString().slice(0, 10))
  replacements.SERVICE_REQ_ID = escapeXml(data.serviceRequestId)
  replacements.TEMPLATE_VERSION = escapeXml(data.templateVersion ?? 'v1')

  return replacements
}
```

#### buildDirectorSection / buildShareholderSection / buildBeneficialOwnerSection / buildGuarantorSection

```typescript
export function buildDirectorSection(director: Director, partialXml: string): string {
  const replacements: Record<string, string> = {
    DIR_FIRST_NAME: escapeXml(director.firstName),
    DIR_SURNAME: escapeXml(director.surname),
    DIR_GENDER: escapeXml(director.gender),
    DIR_DOB: formatDate(director.dateOfBirth),
    DIR_NATIONALITY: escapeXml(director.nationality),
    DIR_ID_TYPE: escapeXml(director.identityType),
    DIR_ID_NUMBER: escapeXml(director.identityNumber),
    DIR_OCCUPATION: escapeXml(director.occupation),
    DIR_POSITION: escapeXml(director.position),
    DIR_APPOINTMENT_DATE: formatDate(director.appointmentDate),
    DIR_EMAIL: escapeXml(director.email),
    DIR_MOBILE: escapeXml(director.mobile),
    DIR_LANDLINE: escapeXml(director.landline ?? ''),
    DIR_ADDRESS_PLOT: escapeXml(director.physicalAddress.plot),
    DIR_ADDRESS_STREET: escapeXml(director.physicalAddress.street),
    DIR_ADDRESS_TOWN: escapeXml(director.physicalAddress.town),
    DIR_ADDRESS_PROVINCE: escapeXml(director.physicalAddress.province),
    DIR_POSTAL_BOX: escapeXml(director.postalAddress.postBox),
    DIR_POSTAL_TOWN: escapeXml(director.postalAddress.town),
    DIR_IS_RESIDENT: checkbox(director.isResidentInZambia),
  }
  return applyReplacements(partialXml, replacements)
}

export function buildShareholderSection(shareholder: Shareholder, partialXml: string): string {
  const replacements: Record<string, string> = {}

  if (shareholder.kind === 'individual') {
    replacements.SH_KIND = escapeXml('Individual')
    replacements.SH_FIRST_NAME = escapeXml(shareholder.firstName)
    replacements.SH_SURNAME = escapeXml(shareholder.surname)
    replacements.SH_GENDER = escapeXml(shareholder.gender)
    replacements.SH_DOB = formatDate(shareholder.dateOfBirth)
    replacements.SH_NATIONALITY = escapeXml(shareholder.nationality)
    replacements.SH_ID_TYPE = escapeXml(shareholder.identityType)
    replacements.SH_ID_NUMBER = escapeXml(shareholder.identityNumber)
    replacements.SH_EMAIL = escapeXml(shareholder.email)
    replacements.SH_MOBILE = escapeXml(shareholder.mobile)
    // Corporate fields empty
    replacements.SH_CORP_NAME = ''
    replacements.SH_CORP_REG_NO = ''
    replacements.SH_CORP_JURISDICTION = ''
  } else {
    replacements.SH_KIND = escapeXml('Body Corporate')
    replacements.SH_CORP_NAME = escapeXml(shareholder.body.corporateName)
    replacements.SH_CORP_REG_NO = escapeXml(shareholder.body.incorporationNumber)
    replacements.SH_CORP_JURISDICTION = escapeXml(shareholder.body.jurisdiction)
    replacements.SH_FIRST_NAME = ''
    replacements.SH_SURNAME = ''
    replacements.SH_GENDER = ''
    replacements.SH_DOB = ''
  }

  // Share allocations (first allocation — simplified for v1)
  const alloc = shareholder.allocations[0]
  if (alloc) {
    replacements.SH_SHARE_CLASS = escapeXml(alloc.shareClass)
    replacements.SH_NUM_SHARES = escapeXml(String(alloc.numberOfShares))
    replacements.SH_AMOUNT_PAID = escapeXml(alloc.amountPaidZmw?.toFixed(2) ?? '')
  }

  return applyReplacements(partialXml, replacements)
}

export function buildBeneficialOwnerSection(owner: BeneficialOwner, partialXml: string): string {
  const replacements: Record<string, string> = {
    BO_FIRST_NAME: escapeXml(owner.firstName),
    BO_SURNAME: escapeXml(owner.surname),
    BO_GENDER: escapeXml(owner.gender),
    BO_DOB: formatDate(owner.dateOfBirth),
    BO_NATIONALITY: escapeXml(owner.nationality),
    BO_ID_TYPE: escapeXml(owner.identityType),
    BO_ID_NUMBER: escapeXml(owner.identityNumber),
    BO_OCCUPATION: escapeXml(owner.occupation),
    BO_OWNERSHIP_PCT: escapeXml(String(owner.ownershipPercent)),
    BO_OWNERSHIP_NATURE: escapeXml(owner.ownershipNature),
    BO_CONTROL_DETAILS: escapeXml(owner.controlDetails ?? ''),
    BO_IS_PEP: checkbox(owner.isPep),
    BO_PEP_DETAILS: escapeXml(owner.pepDetails ?? ''),
    BO_ADDRESS_PLOT: escapeXml(owner.physicalAddress.plot),
    BO_ADDRESS_STREET: escapeXml(owner.physicalAddress.street),
    BO_ADDRESS_TOWN: escapeXml(owner.physicalAddress.town),
    BO_ADDRESS_PROVINCE: escapeXml(owner.physicalAddress.province),
  }
  // v1 limitation: sub-tables for body corporate beneficial owners
  // are not dynamically generated. Use flat fields.
  return applyReplacements(partialXml, replacements)
}

export function buildGuarantorSection(guarantor: Guarantor, partialXml: string): string {
  const replacements: Record<string, string> = {
    GU_FIRST_NAME: escapeXml(guarantor.firstName),
    GU_SURNAME: escapeXml(guarantor.surname),
    GU_GENDER: escapeXml(guarantor.gender),
    GU_DOB: formatDate(guarantor.dateOfBirth),
    GU_NATIONALITY: escapeXml(guarantor.nationality),
    GU_ID_TYPE: escapeXml(guarantor.identityType),
    GU_ID_NUMBER: escapeXml(guarantor.identityNumber),
    GU_OCCUPATION: escapeXml(guarantor.occupation),
    GU_AMOUNT: escapeXml(guarantor.guaranteedAmountZmw.toFixed(2)),
    GU_EMAIL: escapeXml(guarantor.email),
    GU_MOBILE: escapeXml(guarantor.mobile),
    GU_ADDRESS_PLOT: escapeXml(guarantor.physicalAddress.plot),
    GU_ADDRESS_STREET: escapeXml(guarantor.physicalAddress.street),
    GU_ADDRESS_TOWN: escapeXml(guarantor.physicalAddress.town),
    GU_ADDRESS_PROVINCE: escapeXml(guarantor.physicalAddress.province),
  }
  return applyReplacements(partialXml, replacements)
}
```

### 4.3 Main Generation Function

```typescript
import JSZip from 'jszip'
import { Form3GenerationError, Form3TemplateNotFoundError } from './form3.errors'
import type { Form3Data } from './form3.types'
import {
  normalizePlaceholders,
  applyReplacements,
} from './form3-generator.utils'
import {
  buildPartAReplacements,
  buildPartFGHReplacements,
  buildDirectorSection,
  buildShareholderSection,
  buildBeneficialOwnerSection,
  buildGuarantorSection,
} from './form3-generator.service'

export type Partials = {
  directorRow: string
  shareholderRow: string
  beneficialOwnerRow: string
  guarantorRow: string
}

export type PartialName = "director" | "shareholder" | "beneficial-owner" | "guarantor"

/**
 * Loads the DOCX template from R2 storage.
 * @throws Form3TemplateNotFoundError if the object key returns null
 * @throws Form3TemplateLoadError on R2 read failure
 */
export async function loadTemplate(
  r2Bucket: R2Bucket,
  objectKey: string
): Promise<ArrayBuffer> {
  const object = await r2Bucket.get(objectKey)
  if (!object) {
    throw new Form3TemplateNotFoundError(objectKey)
  }
  return object.arrayBuffer()
}

/**
 * Loads a partial XML snippet from R2 storage.
 * @throws Form3TemplateNotFoundError if the partial is missing
 */
export async function loadPartial(
  name: PartialName,
  r2Bucket: R2Bucket
): Promise<string> {
  const key = `templates/partials/${name}-row.xml`
  const object = await r2Bucket.get(key)
  if (!object) {
    throw new Form3TemplateNotFoundError(key)
  }
  return object.text()
}

/**
 * Generates the pre-filled Form 3 DOCX from tenant data.
 *
 * @param data - Complete Form3Data (must pass validation first)
 * @param templateBuffer - DOCX template as ArrayBuffer (from R2 or test fixture)
 * @param partials - XML partial snippets for repeatable sections
 * @returns ArrayBuffer of the generated DOCX file
 * @throws Form3GenerationError if JSZip cannot process the template
 */
export async function generateForm3Docx(
  data: Form3Data,
  templateBuffer: ArrayBuffer,
  partials: Partials
): Promise<ArrayBuffer> {
  // 1. Load template into JSZip
  let zip: JSZip
  try {
    zip = await JSZip.loadAsync(templateBuffer)
  } catch (err) {
    throw new Form3GenerationError('Failed to parse DOCX template as ZIP', err)
  }

  // 2. Extract word/document.xml
  const docXmlFile = zip.file('word/document.xml')
  if (!docXmlFile) {
    throw new Form3GenerationError('Template missing word/document.xml')
  }
  let xml = await docXmlFile.async('string')

  // 3. Normalize split placeholders (ADR-2, Layer 2)
  xml = normalizePlaceholders(xml)

  // 4. Build section XML blocks (raw OOXML — NOT escaped)
  const directorsXml = data.partB.directors
    .map(d => buildDirectorSection(d, partials.directorRow))
    .join('\n')

  const shareholdersXml = (data.partC?.shareholders ?? [])
    .map(s => buildShareholderSection(s, partials.shareholderRow))
    .join('\n')

  const beneficialOwnersXml = (data.partD?.beneficialOwners ?? [])
    .map(o => buildBeneficialOwnerSection(o, partials.beneficialOwnerRow))
    .join('\n')

  const guarantorsXml = (data.partE?.guarantors ?? [])
    .map(g => buildGuarantorSection(g, partials.guarantorRow))
    .join('\n')

  // 5. Build field replacements (ALL values pre-escaped via escapeXml)
  const fieldReplacements: Record<string, string> = {
    ...buildPartAReplacements(data.partA),
    ...buildPartFGHReplacements(data),
  }

  // 6. Add section block replacements WITHOUT escaping (raw OOXML)
  const blockReplacements: Record<string, string> = {
    DIRECTOR_SECTIONS: directorsXml,
    SHAREHOLDER_SECTIONS: shareholdersXml,
    BENEFICIAL_OWNER_SECTIONS: beneficialOwnersXml,
    GUARANTOR_SECTIONS: guarantorsXml,
  }

  // 7. Merge into single map and apply (ADR-5: caller-escaped values)
  const allReplacements = { ...fieldReplacements, ...blockReplacements }
  xml = applyReplacements(xml, allReplacements, data.serviceRequestId)

  // 8. Write modified XML back to the ZIP
  zip.file('word/document.xml', xml)

  // 9. Generate output ArrayBuffer (Workers-compatible)
  return zip.generateAsync({
    type: 'arraybuffer',
    compression: 'DEFLATE',
    compressionOptions: { level: 6 },
  })
}
```

### 4.4 Template Prep Workflow

This is a **one-time** process performed before the first deployment.

**Step 1:** Obtain the official PACRA Form 3 DOCX from the regulator.

**Step 2:** Open in Word (or LibreOffice). For each data field on the form, replace the existing content with a `{{PLACEHOLDER_NAME}}` token. Use the keys from the placeholder mapping table below.

**Critical rule:** Each placeholder must be typed in a single operation. Do NOT type `{{`, then move the cursor, then type the key. Word will split the text across runs.

**Step 3:** Save the modified template as `assets/templates/form3-template-v1.docx`.

**Step 4:** Extract repeatable section snippets. For each repeatable section (directors, shareholders, beneficial owners, guarantors):
1. Open `word/document.xml` with a text editor
2. Locate the table row/block for that section
3. Copy the raw OOXML into the corresponding partial file
4. Replace the extracted block with the block-level placeholder

**Step 5:** Run the validation script:

```bash
npx tsx scripts/validate-placeholders.ts assets/templates/form3-template-v1.docx
```

The script must:
- Parse every `<w:p>` paragraph in `word/document.xml`
- Verify that every `{{...}}` token exists within a single `<w:r>` run (not split)
- Verify that all partial files do NOT contain block-level placeholders
- Exit with code 0 on success, code 1 on failure with details

**Step 6:** Upload to R2:

```bash
npx wrangler r2 object put bumara-templates-prod/templates/form3-template-v1.docx \
  --file assets/templates/form3-template-v1.docx
```

### 4.5 Placeholder Mapping Table

| Key | Type | Source Field |
|-----|------|-------------|
| **Part A — Company Details** | | |
| `COMPANY_NAME` | Field | `partA.companyName` |
| `APPROVED_NAME_REF` | Field | `partA.approvedNameReference` |
| `PRINCIPAL_ACTIVITY` | Field | `partA.principalBusinessActivity` |
| `OTHER_ACTIVITIES` | Field | `partA.otherActivities` |
| `IS_PRIVATE_SHARES` | Field | `checkbox(companyType === 'private_shares')` |
| `IS_PRIVATE_GUARANTEE` | Field | `checkbox(companyType === 'private_guarantee')` |
| `IS_PUBLIC_LIMITED` | Field | `checkbox(companyType === 'public_limited')` |
| `IS_UNLIMITED_PRIVATE` | Field | `checkbox(companyType === 'unlimited_private')` |
| `NOMINAL_CAPITAL` | Field | `partA.nominalCapitalZmw` |
| `NUMBER_OF_SHARES` | Field | `partA.numberOfShares` |
| `PAR_VALUE` | Field | `partA.parValue` |
| `SHARE_CLASS` | Field | `partA.shareClass` |
| `FIRST_FY_END` | Field | `formatDate(partA.firstFinancialYearEnd)` |
| `REG_PLOT` ... `REG_COUNTRY` | Field | `partA.registeredOffice.*` |
| `POSTAL_BOX` ... `POSTAL_PROVINCE` | Field | `partA.postalAddress.*` |
| `CONTACT_EMAIL` / `CONTACT_PHONE` | Field | `partA.contactEmail/contactPhone` |
| **Block-level (repeatable sections)** | | |
| `DIRECTOR_SECTIONS` | Block | Cloned from `director-row.xml` |
| `SHAREHOLDER_SECTIONS` | Block | Cloned from `shareholder-row.xml` |
| `BENEFICIAL_OWNER_SECTIONS` | Block | Cloned from `beneficial-owner-row.xml` |
| `GUARANTOR_SECTIONS` | Block | Cloned from `guarantor-row.xml` |
| **Part F — Secretary** | | |
| `SEC_FIRST_NAME` ... `SEC_ID_NUMBER` | Field | `partF.secretary.*` (individual) |
| `SEC_CORP_NAME` ... `SEC_CORP_JURISDICTION` | Field | `partF.secretary.body.*` (firm) |
| **Part G — Declaration** | | |
| `DECL_NAME` | Field | `partG.declarantName` |
| `DECL_IS_DIRECTOR` | Field | `checkbox(declarantCapacity === 'first_director')` |
| `DECL_IS_SECRETARY` | Field | `checkbox(declarantCapacity === 'company_secretary')` |
| `DECL_DATE` | Field | `formatDate(partG.declarationDate)` |
| **Part H — Lodging Person** | | |
| `LODGER_FIRST_NAME` ... `LODGER_CAPACITY` | Field | `partH.*` |
| **Metadata** | | |
| `GENERATED_DATE` | Field | `formatDate(now)` |
| `SERVICE_REQ_ID` | Field | `data.serviceRequestId` |
| `TEMPLATE_VERSION` | Field | `data.templateVersion` |

---

## 5. The Form 3 Wizard (Frontend)

File location: `apps/app/features/pacra/components/form3/`

### 5.1 Component Hierarchy and Prop Types

```typescript
// apps/app/features/pacra/components/form3/Form3Wizard.tsx

import type { Form3Data } from '@repo/api-services/domains/form3/form3.types'

export type WizardStepId =
  | 'partA' | 'directors' | 'shareholders' | 'beneficialOwnership'
  | 'guarantors' | 'secretary' | 'declaration' | 'lodgingPerson'
  | 'review' | 'download'

interface WizardStepDef {
  id: WizardStepId
  label: string
  description: string
  icon: LucideIcon
  isConditional?: boolean
  showWhen?: (data: Partial<Form3Data>) => boolean
}

interface Form3WizardProps {
  serviceRequestId: string
  organizationId: string
}
```

### 5.2 Wizard State Shape and Initialisation

```typescript
interface WizardState {
  formData: Partial<Form3Data>
  stepIndex: number
  completedSteps: Set<WizardStepId>
  saveStatus: 'idle' | 'saving' | 'saved' | 'error'
  lastSavedAt: string | null
  downloadState: DownloadState
  isDirty: boolean
}

type DownloadState =
  | { status: 'NOT_DOWNLOADED' }
  | { status: 'DOWNLOADING' }
  | { status: 'DOWNLOADED'; downloadedAt: string }
  | { status: 'UPLOADING'; progress: number }
  | { status: 'UPLOAD_SUCCESS'; documentId: string }
  | { status: 'UPLOAD_ERROR'; error: string }

const initialWizardState: WizardState = {
  formData: {},
  stepIndex: 0,
  completedSteps: new Set(),
  saveStatus: 'idle',
  lastSavedAt: null,
  downloadState: { status: 'NOT_DOWNLOADED' },
  isDirty: false,
}
```

### 5.3 Wizard Hydration on Mount

When `Form3Wizard` mounts, it calls `GET /form3-draft` to restore in-progress state. Without this, a returning user loses all entered data.

```typescript
'use client'
import { useEffect, useState, useCallback } from 'react'
import { useAuth } from '@repo/auth/client'
import { useForm3Draft } from '@/lib/queries/form3/hooks/use-form3-draft'

export function Form3Wizard({ serviceRequestId, organizationId }: Form3WizardProps) {
  const [wizardState, setWizardState] = useState<WizardState>(initialWizardState)
  const { data: draftData, isLoading } = useForm3Draft(serviceRequestId)

  // Hydrate from saved draft on mount
  useEffect(() => {
    if (draftData?.form3Data) {
      setWizardState(prev => ({
        ...prev,
        formData: draftData.form3Data,
        lastSavedAt: draftData.lastSavedAt,
        stepIndex: determineRestoredStepIndex(draftData.form3Data),
        completedSteps: determineCompletedSteps(draftData.form3Data),
      }))
    }
  }, [draftData])

  // ... render
}
```

**Helper: `determineRestoredStepIndex`**

Returns the index of the first step whose part is absent or empty. This advances the user to where they left off.

```typescript
function determineRestoredStepIndex(formData: Partial<Form3Data>): number {
  const steps = getSteps(formData)
  for (let i = 0; i < steps.length; i++) {
    const step = steps[i]
    switch (step.id) {
      case 'partA': if (!formData.partA) return i; break
      case 'directors': if (!formData.partB?.directors?.length) return i; break
      case 'shareholders': if (!formData.partC?.shareholders?.length) return i; break
      case 'beneficialOwnership': if (!formData.partD?.beneficialOwners?.length) return i; break
      case 'guarantors': if (!formData.partE?.guarantors?.length) return i; break
      case 'secretary': if (!formData.partF) return i; break
      case 'declaration': if (!formData.partG) return i; break
      case 'lodgingPerson': if (!formData.partH) return i; break
      case 'review': return i  // Always stop at review if all parts filled
      case 'download': return i
    }
  }
  return 0
}
```

**Helper: `determineCompletedSteps`**

```typescript
function determineCompletedSteps(formData: Partial<Form3Data>): Set<WizardStepId> {
  const completed = new Set<WizardStepId>()
  if (formData.partA) completed.add('partA')
  if (formData.partB?.directors?.length) completed.add('directors')
  if (formData.partC?.shareholders?.length) completed.add('shareholders')
  if (formData.partD?.beneficialOwners?.length) completed.add('beneficialOwnership')
  if (formData.partE?.guarantors?.length) completed.add('guarantors')
  if (formData.partF) completed.add('secretary')
  if (formData.partG) completed.add('declaration')
  if (formData.partH) completed.add('lodgingPerson')
  return completed
}
```

### 5.4 Unsaved Changes Warning

```typescript
useEffect(() => {
  const handleBeforeUnload = (e: BeforeUnloadEvent) => {
    if (wizardState.isDirty) {
      e.preventDefault()
      e.returnValue = ''  // Modern browsers show generic warning
    }
  }
  window.addEventListener('beforeunload', handleBeforeUnload)
  return () => window.removeEventListener('beforeunload', handleBeforeUnload)
}, [wizardState.isDirty])
```

`isDirty` is set to `true` when `formData` changes and reset to `false` after a successful auto-save (`PATCH /form3-draft` returns 200).

### 5.5 getSteps Function

Returns the visible wizard steps based on the selected company type. Steps for guarantors are hidden unless `companyType === 'private_guarantee'`. Steps for shareholders/beneficial owners are hidden for `private_guarantee`.

```typescript
import {
  Building2, Users, FileText, UserCheck, Shield,
  Briefcase, PenTool, User, CheckCircle, Download,
} from 'lucide-react'

export function getSteps(formData: Partial<Form3Data>): WizardStepDef[] {
  const companyType = formData.partA?.companyType
  const allSteps: WizardStepDef[] = [
    { id: 'partA', label: 'Company Details', description: 'Part A', icon: Building2 },
    { id: 'directors', label: 'Directors', description: 'Part B', icon: Users },
    {
      id: 'shareholders', label: 'Shareholders', description: 'Part C', icon: FileText,
      isConditional: true,
      showWhen: (d) => d.partA?.companyType !== 'private_guarantee',
    },
    {
      id: 'beneficialOwnership', label: 'Beneficial Owners', description: 'Part D', icon: UserCheck,
      isConditional: true,
      showWhen: (d) => d.partA?.companyType !== 'private_guarantee',
    },
    {
      id: 'guarantors', label: 'Guarantors', description: 'Part E', icon: Shield,
      isConditional: true,
      showWhen: (d) => d.partA?.companyType === 'private_guarantee',
    },
    { id: 'secretary', label: 'Secretary', description: 'Part F', icon: Briefcase },
    { id: 'declaration', label: 'Declaration', description: 'Part G', icon: PenTool },
    { id: 'lodgingPerson', label: 'Lodging Person', description: 'Part H', icon: User },
    { id: 'review', label: 'Review', description: 'Review all data', icon: CheckCircle },
    { id: 'download', label: 'Download & Upload', description: 'Get your form', icon: Download },
  ]

  return allSteps.filter(step => {
    if (!step.isConditional) return true
    return step.showWhen?.(formData) ?? true
  })
}
```

### 5.6 Auto-save Strategy

Auto-save triggers on two occasions:
1. **Debounced change** — 1500ms after the last `formData` change.
2. **Step advance** — Immediately when the user clicks "Next" to advance to the next step.

```typescript
import { useMemo, useRef, useCallback } from 'react'
import { useForm3Save } from '@/lib/queries/form3/hooks/use-form3-save'

function useAutoSave(
  serviceRequestId: string,
  wizardState: WizardState,
  setWizardState: React.Dispatch<React.SetStateAction<WizardState>>
) {
  const { mutateAsync: saveDraft } = useForm3Save(serviceRequestId)
  const debounceRef = useRef<ReturnType<typeof setTimeout>>()

  const save = useCallback(async () => {
    setWizardState(prev => ({ ...prev, saveStatus: 'saving' }))
    try {
      const result = await saveDraft({
        data: wizardState.formData,
        currentLastSavedAt: wizardState.lastSavedAt,
      })
      setWizardState(prev => ({
        ...prev,
        saveStatus: 'saved',
        lastSavedAt: result.lastSavedAt,
        isDirty: false,
      }))
    } catch (error: any) {
      if (error.status === 409) {
        // Conflict — another session updated the draft
        setWizardState(prev => ({ ...prev, saveStatus: 'error' }))
        // Show conflict banner, reload draft
      } else {
        setWizardState(prev => ({ ...prev, saveStatus: 'error' }))
      }
    }
  }, [wizardState.formData, wizardState.lastSavedAt, saveDraft, setWizardState])

  // Debounced save on data change
  const scheduleSave = useCallback(() => {
    if (debounceRef.current) clearTimeout(debounceRef.current)
    debounceRef.current = setTimeout(save, 1500)
  }, [save])

  // Immediate save on step advance
  const saveNow = useCallback(() => {
    if (debounceRef.current) clearTimeout(debounceRef.current)
    save()
  }, [save])

  return { scheduleSave, saveNow }
}
```

**Save status display** — shown in the wizard sidebar footer:

```typescript
function SaveStatusIndicator({ status, lastSavedAt }: {
  status: WizardState['saveStatus']
  lastSavedAt: string | null
}) {
  switch (status) {
    case 'saving': return <span className="text-muted-foreground text-xs">Saving...</span>
    case 'saved': return (
      <span className="text-muted-foreground text-xs">
        Saved {lastSavedAt ? formatRelativeTime(lastSavedAt) : ''}
      </span>
    )
    case 'error': return <span className="text-destructive text-xs">Save failed — retrying</span>
    default: return null
  }
}
```

**Optimistic locking for 409 conflicts:**

The `PATCH /form3-draft` request includes `currentLastSavedAt`. If the server's `form3_last_saved_at` differs, it returns 409. The client handles this by:

1. Displaying a conflict banner: "Your draft was updated in another session."
2. Reloading the draft from the server.
3. Discarding local changes.

### 5.7 Company Type Change Handling

When the user changes `companyType` in Part A, conditional parts may appear or disappear. The wizard must:

1. **Show/hide steps** via `getSteps()` re-evaluation.
2. **Clear invalidated data** — if switching FROM `private_guarantee` to `private_shares`, clear `partE` (guarantors). If switching TO `private_guarantee`, clear `partC` (shareholders) and `partD` (beneficial owners).
3. **Reset `completedSteps`** for any cleared parts.

```typescript
function handleCompanyTypeChange(
  newType: CompanyType,
  setWizardState: React.Dispatch<React.SetStateAction<WizardState>>
) {
  setWizardState(prev => {
    const updated = { ...prev, isDirty: true }
    updated.formData = { ...prev.formData }
    if (updated.formData.partA) {
      updated.formData.partA = { ...updated.formData.partA, companyType: newType }
    }

    if (newType === 'private_guarantee') {
      // Remove shareholders and beneficial owners
      delete updated.formData.partC
      delete updated.formData.partD
      updated.completedSteps = new Set([...prev.completedSteps]
        .filter(s => s !== 'shareholders' && s !== 'beneficialOwnership'))
    } else {
      // Remove guarantors
      delete updated.formData.partE
      updated.completedSteps = new Set([...prev.completedSteps]
        .filter(s => s !== 'guarantors'))
    }
    return updated
  })
}
```

### 5.8 Download Step States

The Download step (`StepDownload.tsx`) manages a state machine for the download-then-upload flow:

| State | UI | Actions Available |
|-------|-----|-------------------|
| `NOT_DOWNLOADED` | "Generate & Download" button | Click to download |
| `DOWNLOADING` | Spinner + "Generating..." | None (disabled) |
| `DOWNLOADED` | Success checkmark + "Upload Signed Copy" button | Upload or re-download |
| `UPLOADING` | Progress bar | None (disabled) |
| `UPLOAD_SUCCESS` | Green checkmark + document ID | Re-upload allowed |
| `UPLOAD_ERROR` | Error message + "Retry" button | Retry upload |

```typescript
function StepDownload({ serviceRequestId, taskId, wizardState, setWizardState }: StepDownloadProps) {
  const handleDownload = async () => {
    setWizardState(prev => ({ ...prev, downloadState: { status: 'DOWNLOADING' } }))
    try {
      const blob = await downloadForm3Docx(serviceRequestId)
      const url = URL.createObjectURL(blob)
      const a = document.createElement('a')
      a.href = url
      a.download = `Form3-${wizardState.formData.partA?.companyName ?? 'draft'}.docx`
      a.click()
      URL.revokeObjectURL(url)
      setWizardState(prev => ({
        ...prev,
        downloadState: { status: 'DOWNLOADED', downloadedAt: new Date().toISOString() },
      }))
    } catch {
      setWizardState(prev => ({
        ...prev,
        downloadState: { status: 'NOT_DOWNLOADED' },
      }))
    }
  }

  const handleUpload = async (file: File) => {
    setWizardState(prev => ({ ...prev, downloadState: { status: 'UPLOADING', progress: 0 } }))
    try {
      const result = await uploadSignedForm3(serviceRequestId, taskId, file)
      setWizardState(prev => ({
        ...prev,
        downloadState: { status: 'UPLOAD_SUCCESS', documentId: result.document.id },
      }))
    } catch (error: any) {
      setWizardState(prev => ({
        ...prev,
        downloadState: { status: 'UPLOAD_ERROR', error: error.message },
      }))
    }
  }

  // ... render based on downloadState.status
}
```

### 5.9 React Query Hooks

File: `apps/app/lib/queries/form3/`

```typescript
// hooks/use-form3-draft.ts
import { useQuery } from '@tanstack/react-query'
import { useAuth } from '@repo/auth/client'
import { fetchForm3Draft } from '../fetchers/form3'

export const form3QueryKeys = {
  all: ['form3'] as const,
  draft: (serviceRequestId: string) =>
    [...form3QueryKeys.all, 'draft', serviceRequestId] as const,
}

export function useForm3Draft(serviceRequestId: string) {
  const { getToken } = useAuth()
  return useQuery({
    queryKey: form3QueryKeys.draft(serviceRequestId),
    queryFn: () => fetchForm3Draft(getToken, serviceRequestId),
    staleTime: 5 * 60 * 1000,
  })
}

// hooks/use-form3-save.ts
import { useMutation, useQueryClient } from '@tanstack/react-query'

export function useForm3Save(serviceRequestId: string) {
  const { getToken } = useAuth()
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (input: { data: Partial<Form3Data>; currentLastSavedAt: string | null }) =>
      saveForm3Draft(getToken, serviceRequestId, input),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: form3QueryKeys.draft(serviceRequestId) })
    },
  })
}

// fetchers/form3.ts
import { createAuthenticatedClient } from '@/lib/api-client'

export async function fetchForm3Draft(
  getToken: () => Promise<string | null>,
  serviceRequestId: string
) {
  const client = createAuthenticatedClient(getToken)
  const res = await client.get(`/service-requests/${serviceRequestId}/form3-draft`)
  if (!res.ok) throw new Error('Failed to fetch draft')
  return res.json() as Promise<{ form3Data: Partial<Form3Data> | null; lastSavedAt: string | null }>
}

export async function saveForm3Draft(
  getToken: () => Promise<string | null>,
  serviceRequestId: string,
  input: { data: Partial<Form3Data>; currentLastSavedAt: string | null }
) {
  const client = createAuthenticatedClient(getToken)
  const res = await client.patch(`/service-requests/${serviceRequestId}/form3-draft`, {
    json: input,
  })
  if (res.status === 409) {
    const error = new Error('Conflict') as Error & { status: number }
    error.status = 409
    throw error
  }
  if (!res.ok) throw new Error('Failed to save draft')
  return res.json() as Promise<{ form3Data: Partial<Form3Data>; lastSavedAt: string }>
}

export async function downloadForm3Docx(
  getToken: () => Promise<string | null>,
  serviceRequestId: string
): Promise<Blob> {
  const token = await getToken()
  const res = await fetch(
    `${process.env.NEXT_PUBLIC_API_SUBMISSIONS_URL}/service-requests/${serviceRequestId}/form3.docx`,
    { headers: { Authorization: `Bearer ${token}` } }
  )
  if (!res.ok) throw new Error('Failed to generate DOCX')
  return res.blob()
}

export async function uploadSignedForm3(
  getToken: () => Promise<string | null>,
  serviceRequestId: string,
  taskId: string,
  file: File
) {
  const token = await getToken()
  const formData = new FormData()
  formData.append('file', file)
  const res = await fetch(
    `${process.env.NEXT_PUBLIC_API_SUBMISSIONS_URL}/service-requests/${serviceRequestId}/tasks/${taskId}/upload`,
    {
      method: 'POST',
      headers: { Authorization: `Bearer ${token}` },
      body: formData,
    }
  )
  if (!res.ok) {
    const body = await res.json().catch(() => ({}))
    throw new Error(body.message ?? 'Upload failed')
  }
  return res.json()
}
```

---

## 6. Database Schema

### 6.1 service_requests Table — Additions

The existing `service_requests` table (see `packages/database/src/schema/compliance/service-requests.ts`) already has an `intakeData` JSONB column. For Form 3, we store the wizard data in this column.

**New columns to add via migration:**

```typescript
// Addition to packages/database/src/schema/compliance/service-requests.ts
import { varchar, timestamp } from 'drizzle-orm/pg-core'

// Add these columns to the existing serviceRequests table definition:
form3TemplateVersion: varchar("form3_template_version", { length: 20 }),
form3LastSavedAt: timestamp("form3_last_saved_at", { mode: "date" }),
```

The `intakeData` JSONB column stores `Form3Data` typed via `.$type<Form3Data>()`. The `status` column already uses `serviceRequestStatusEnum`.

### 6.2 tasks Table — Form 3 Upload Task

The existing `tasks` table (see `packages/database/src/schema/compliance/tasks.ts`) is used as-is. On Form 3 service request creation, insert this task:

```typescript
// Drizzle INSERT for the Form 3 signed upload task
await db.insert(tasks).values({
  organizationId: ctx.orgId,
  serviceRequestId: newServiceRequest.id,
  templateKey: 'form3_signed_upload',        // Unique per service request
  title: 'Upload Signed Form 3',
  description: 'Print the pre-filled Form 3, obtain all required signatures, and upload the signed copy.',
  taskType: 'upload_document',
  required: true,
  isBlocking: true,
  status: 'todo',
  actionKind: 'doc_upload',
  actionRef: { docRequirementGroup: 'form3_signed' },
  completionTrigger: 'doc:form3_signed',
  regulatorKey: 'pacra',
  sequence: 1,
})
```

### 6.3 documents Table

The existing `documents` table is used as-is. R2 storage key format:

```
documents/{organizationId}/{serviceRequestId}/{documentId}/{filename}
```

When a re-upload occurs, the old document is archived:

```typescript
await db.update(documents)
  .set({ status: 'archived', archivedAt: new Date() })
  .where(and(
    eq(documents.organizationId, ctx.orgId),
    eq(documents.id, oldDocumentId)
  ))
```

### 6.4 timeline_events Table

Used as-is. Events created by the Form 3 feature:

| eventType | title | metadata (safe fields only) |
|-----------|-------|-----------------------------|
| `status_change` | "Service request created" | `{ fromStatus: null, toStatus: 'pending_data' }` |
| `status_change` | "Form 3 draft saved" | `{ stepId: 'partA', byteCount: 1234 }` |
| `document_upload` | "Form 3 DOCX generated" | `{ templateVersion: 'v1' }` |
| `document_upload` | "Signed Form 3 uploaded" | `{ documentId: '...', mimeType: '...', sizeBytes: 12345 }` |
| `document_upload` | "Signed Form 3 re-uploaded" | `{ documentId: '...', supersededDocumentId: '...' }` |

**Important:** `metadata` MUST NOT contain `form3Data` or any PII fields. Only IDs, counts, and status values are permitted.

### 6.5 Indexes

Add these indexes for Form 3 query performance:

```sql
-- Support idempotency check: find active company_incorporation requests per org
CREATE INDEX idx_service_requests_org_name_status
  ON service_requests (organization_id, name)
  WHERE status NOT IN ('accepted', 'cancelled', 'waived');
```

### 6.6 Migration File

File: `drizzle/migrations/0012_form3_data.sql`

```sql
-- Migration: Add Form 3 columns to service_requests
ALTER TABLE service_requests
  ADD COLUMN form3_template_version VARCHAR(20),
  ADD COLUMN form3_last_saved_at TIMESTAMP;

-- Partial index for idempotency check
CREATE INDEX idx_service_requests_org_name_active
  ON service_requests (organization_id, name)
  WHERE status NOT IN ('accepted', 'cancelled', 'waived');

-- Index for backoffice queue queries
CREATE INDEX idx_service_requests_org_regulator_type_status
  ON service_requests (organization_id, regulator, status);
```

---

## 7. API Endpoints

All endpoints use the existing Hono route pattern: OpenAPI route definition + handler function.

### Endpoint 1: POST /api/v1/service-requests

**Auth:** Clerk org-scoped JWT (`requireAuth`, `requireOrg`)
**Idempotency:** Returns existing record if active `company_incorporation` exists for this org.

```typescript
// apps/api-submissions/src/routes/service-requests/index.ts

import { createRoute, z } from '@hono/zod-openapi'
import type { AppRouteHandler } from '@repo/backend/types'
import { buildServiceContext, buildServiceDependencies } from '@repo/backend/core/context'
import { serviceRequests, tasks, timelineEvents } from '@repo/database/schema'
import { and, eq, notInArray } from 'drizzle-orm'

const createServiceRequestRoute = createRoute({
  method: 'post',
  path: '/api/v1/service-requests',
  request: {
    body: {
      content: { 'application/json': {
        schema: z.object({
          name: z.literal('company_incorporation'),
          regulatorKey: z.literal('pacra'),
        })
      }}
    }
  },
  responses: {
    200: { description: 'Existing service request returned' },
    201: { description: 'New service request created' },
  },
})

export const createServiceRequestHandler: AppRouteHandler<typeof createServiceRequestRoute> = async (c) => {
  const ctx = buildServiceContext(c)
  const deps = buildServiceDependencies(c)
  const body = c.req.valid('json')

  // 1. Idempotency check (ADR-8)
  const existing = await deps.db.query.serviceRequests.findFirst({
    where: and(
      eq(serviceRequests.organizationId, ctx.orgId),
      eq(serviceRequests.name, 'company_incorporation'),
      notInArray(serviceRequests.status, ['accepted', 'cancelled', 'waived'])
    ),
    with: { tasks: true },
  })

  if (existing) {
    return c.json({ success: true, data: { serviceRequest: existing, tasks: existing.tasks, created: false } }, 200)
  }

  // 2. Create service request + task in transaction
  const result = await deps.db.transaction(async (tx) => {
    // INSERT service request
    const [sr] = await tx.insert(serviceRequests).values({
      organizationId: ctx.orgId,
      name: 'company_incorporation',
      regulator: 'pacra',
      status: 'pending_data',
      description: 'PACRA Company Incorporation — Form 3',
    }).returning()

    // INSERT upload task
    const [task] = await tx.insert(tasks).values({
      organizationId: ctx.orgId,
      serviceRequestId: sr.id,
      templateKey: 'form3_signed_upload',
      title: 'Upload Signed Form 3',
      description: 'Print, sign, and upload the completed Form 3.',
      taskType: 'upload_document',
      required: true,
      isBlocking: true,
      status: 'todo',
      actionKind: 'doc_upload',
      actionRef: { docRequirementGroup: 'form3_signed' },
      completionTrigger: 'doc:form3_signed',
      regulatorKey: 'pacra',
      sequence: 1,
    }).returning()

    // INSERT timeline event
    await tx.insert(timelineEvents).values({
      organizationId: ctx.orgId,
      serviceRequestId: sr.id,
      eventType: 'status_change',
      title: 'Service request created',
      actorType: 'tenant',
      metadata: { fromStatus: null, toStatus: 'pending_data', type: 'company_incorporation' },
    })

    return { serviceRequest: sr, tasks: [task] }
  })

  return c.json({ success: true, data: { ...result, created: true } }, 201)
}
```

### Endpoint 2: GET /api/v1/service-requests/:id/form3-draft

**Auth:** Clerk org-scoped JWT
**Returns:** `{ form3Data: Partial<Form3Data> | null, lastSavedAt: string | null }`

```typescript
export const getForm3DraftHandler: AppRouteHandler<typeof getForm3DraftRoute> = async (c) => {
  const ctx = buildServiceContext(c)
  const deps = buildServiceDependencies(c)
  const { id } = c.req.param()

  const sr = await deps.db.query.serviceRequests.findFirst({
    where: and(
      eq(serviceRequests.organizationId, ctx.orgId),
      eq(serviceRequests.id, id)
    ),
  })

  if (!sr) return c.json({ code: 'NOT_FOUND', message: 'Service request not found' }, 404)

  return c.json({
    success: true,
    data: {
      form3Data: sr.intakeData as Partial<Form3Data> | null,
      lastSavedAt: sr.form3LastSavedAt?.toISOString() ?? null,
    },
  }, 200)
}
```

### Endpoint 3: PATCH /api/v1/service-requests/:id/form3-draft

**Auth:** Clerk org-scoped JWT
**Body:** `{ data: Partial<Form3Data>, currentLastSavedAt: string | null }`
**Optimistic locking:** Returns 409 if `currentLastSavedAt` doesn't match DB value.

```typescript
export const patchForm3DraftHandler: AppRouteHandler<typeof patchForm3DraftRoute> = async (c) => {
  const ctx = buildServiceContext(c)
  const deps = buildServiceDependencies(c)
  const { id } = c.req.param()
  const body = c.req.valid('json')

  const sr = await deps.db.query.serviceRequests.findFirst({
    where: and(
      eq(serviceRequests.organizationId, ctx.orgId),
      eq(serviceRequests.id, id)
    ),
  })

  if (!sr) return c.json({ code: 'NOT_FOUND', message: 'Service request not found' }, 404)

  // Optimistic locking check
  if (body.currentLastSavedAt !== null) {
    const dbLastSaved = sr.form3LastSavedAt?.toISOString() ?? null
    if (dbLastSaved !== body.currentLastSavedAt) {
      return c.json({ code: 'CONFLICT', message: 'Draft was updated in another session' }, 409)
    }
  }

  // Deep merge incoming data into existing data
  const existingData = (sr.intakeData as Partial<Form3Data>) ?? {}
  const mergedData = mergeForm3Data(existingData, body.data)
  const now = new Date()

  await deps.db.update(serviceRequests)
    .set({
      intakeData: mergedData,
      form3LastSavedAt: now,
      updatedAt: now,
    })
    .where(and(
      eq(serviceRequests.organizationId, ctx.orgId),
      eq(serviceRequests.id, id)
    ))

  // Timeline event
  await deps.db.insert(timelineEvents).values({
    organizationId: ctx.orgId,
    serviceRequestId: id,
    eventType: 'status_change',
    title: 'Form 3 draft saved',
    actorType: 'tenant',
    metadata: { byteCount: JSON.stringify(mergedData).length },
  })

  return c.json({
    success: true,
    data: { form3Data: mergedData, lastSavedAt: now.toISOString() },
  }, 200)
}

/**
 * Deep merges incoming partial data into existing form data.
 * Arrays are REPLACED (not appended) — e.g. incoming partB with 1 director
 * replaces existing partB entirely.
 */
function mergeForm3Data(
  existing: Partial<Form3Data>,
  incoming: Partial<Form3Data>
): Partial<Form3Data> {
  const merged = { ...existing }
  for (const [key, value] of Object.entries(incoming)) {
    if (value === undefined) continue
    if (value !== null && typeof value === 'object' && !Array.isArray(value)) {
      // Deep merge objects (but replace arrays within them)
      merged[key as keyof Form3Data] = {
        ...(existing[key as keyof Form3Data] as any ?? {}),
        ...value,
      } as any
    } else {
      merged[key as keyof Form3Data] = value as any
    }
  }
  return merged
}
```

### Endpoint 4: GET /api/v1/service-requests/:id/form3.docx

**Auth:** Clerk org-scoped JWT
**Returns:** `ArrayBuffer` with Content-Disposition header.

See Section 4 for the full generation logic. Handler outline:

```typescript
export const getForm3DocxHandler: AppRouteHandler<typeof getForm3DocxRoute> = async (c) => {
  const ctx = buildServiceContext(c)
  const deps = buildServiceDependencies(c)
  const { id } = c.req.param()

  // 1. Fetch service request with tenant filter
  const sr = await deps.db.query.serviceRequests.findFirst({
    where: and(eq(serviceRequests.organizationId, ctx.orgId), eq(serviceRequests.id, id)),
  })
  if (!sr) return c.json({ code: 'NOT_FOUND', message: 'Service request not found' }, 404)

  const form3Data = sr.intakeData as Form3Data | null
  if (!form3Data) return c.json({ code: 'NO_DATA', message: 'Form 3 data not yet entered' }, 422)

  // 2. Template version check
  if (sr.form3TemplateVersion && sr.form3TemplateVersion !== c.env.FORM3_TEMPLATE_VERSION) {
    return c.json({
      code: 'TEMPLATE_VERSION_MISMATCH',
      message: 'Form data was saved with an older template version. Please contact support.',
    }, 409)
  }

  // 3. Validate completeness
  const validation = validateForm3Data(form3Data, c.env)
  if (!validation.isValid) {
    return c.json({ code: 'VALIDATION_ERROR', fieldErrors: validation.errors }, 422)
  }

  try {
    // 4. Load template and partials from R2
    const templateBuffer = await loadTemplate(c.env.DOCX_TEMPLATES, c.env.FORM3_TEMPLATE_OBJECT_KEY)
    const partials: Partials = {
      directorRow: await loadPartial('director', c.env.DOCX_TEMPLATES),
      shareholderRow: await loadPartial('shareholder', c.env.DOCX_TEMPLATES),
      beneficialOwnerRow: await loadPartial('beneficial-owner', c.env.DOCX_TEMPLATES),
      guarantorRow: await loadPartial('guarantor', c.env.DOCX_TEMPLATES),
    }

    // 5. Generate DOCX
    const arrayBuffer = await generateForm3Docx(form3Data, templateBuffer, partials)

    // 6. Update status if first generation
    if (sr.status === 'pending_data') {
      await deps.db.update(serviceRequests)
        .set({ status: 'in_progress', form3TemplateVersion: c.env.FORM3_TEMPLATE_VERSION })
        .where(and(eq(serviceRequests.organizationId, ctx.orgId), eq(serviceRequests.id, id)))
    }

    // 7. Timeline event
    await deps.db.insert(timelineEvents).values({
      organizationId: ctx.orgId,
      serviceRequestId: id,
      eventType: 'document_upload',
      title: 'Form 3 DOCX generated',
      actorType: 'tenant',
      metadata: { templateVersion: c.env.FORM3_TEMPLATE_VERSION },
    })

    // 8. Return DOCX with Content-Disposition
    const safeName = (form3Data.partA?.companyName ?? 'Form3').replace(/[^\w\s-]/g, '')
    return new Response(arrayBuffer, {
      headers: {
        'Content-Type': 'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
        'Content-Disposition': `attachment; filename="Form3-${safeName}.docx"`,
        'Cache-Control': 'no-store',
      },
    })
  } catch (error) {
    const errorId = crypto.randomUUID()
    console.error({ event: 'form3_generation_failed', errorId, serviceRequestId: id, error })
    return c.json({ errorId, message: 'Document generation failed' }, 500)
  }
}
```

### Endpoint 5: POST /api/v1/service-requests/:id/tasks/:taskId/upload

**Auth:** Clerk org-scoped JWT
**Body:** `multipart/form-data` with field `"file"`

```typescript
export const uploadSignedForm3Handler: AppRouteHandler<typeof uploadRoute> = async (c) => {
  const ctx = buildServiceContext(c)
  const deps = buildServiceDependencies(c)
  const { id, taskId } = c.req.param()

  // 1. Parse multipart (Constraint 5 — Web FormData, NOT multer)
  const formData = await c.req.formData()
  const file = formData.get('file') as File | null
  if (!file) return c.json({ code: 'NO_FILE', message: 'No file provided' }, 400)

  const arrayBuffer = await file.arrayBuffer()

  // 2. Validate file
  if (arrayBuffer.byteLength === 0) {
    return c.json({ code: 'EMPTY_FILE', message: 'File is empty' }, 422)
  }
  const maxSize = parseInt(c.env.FORM3_MAX_UPLOAD_SIZE_BYTES ?? '10485760')
  if (arrayBuffer.byteLength > maxSize) {
    return c.json({ code: 'FILE_TOO_LARGE', message: `File exceeds ${maxSize} bytes` }, 413)
  }

  // 3. Validate MIME type (accept PDF and DOCX only)
  const mimeType = file.type
  const allowedTypes = [
    'application/pdf',
    'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
  ]
  if (!allowedTypes.includes(mimeType)) {
    return c.json({ code: 'INVALID_FILE_TYPE', message: 'Only PDF and DOCX files accepted' }, 415)
  }

  // 4. Verify task belongs to this service request and org
  const task = await deps.db.query.tasks.findFirst({
    where: and(
      eq(tasks.organizationId, ctx.orgId),
      eq(tasks.serviceRequestId, id),
      eq(tasks.id, taskId)
    ),
  })
  if (!task) return c.json({ code: 'TASK_NOT_FOUND', message: 'Task not found' }, 404)

  // 5. Transaction: create document, update task, evaluate readiness
  const result = await deps.db.transaction(async (tx) => {
    // Archive old document if re-upload
    const existingDocs = await tx.select().from(documents).where(
      and(
        eq(documents.organizationId, ctx.orgId),
        eq(documents.serviceRequestId, id),
        eq(documents.requirementKey, 'form3_signed'),
        eq(documents.status, 'active')
      )
    )
    for (const doc of existingDocs) {
      await tx.update(documents)
        .set({ status: 'archived', archivedAt: new Date() })
        .where(eq(documents.id, doc.id))
    }

    // INSERT new document record
    const documentId = crypto.randomUUID()
    const storageKey = `documents/${ctx.orgId}/${id}/${documentId}/${file.name}`

    const [newDoc] = await tx.insert(documents).values({
      id: documentId,
      organizationId: ctx.orgId,
      serviceRequestId: id,
      requirementKey: 'form3_signed',
      kind: 'source',
      status: 'active',
      storageKey,
      filename: file.name,
      name: 'Signed Form 3',
      mimeType,
      sizeBytes: arrayBuffer.byteLength,
      uploadedAt: new Date(),
      createdBy: ctx.userId,
    }).returning()

    // Upload to R2
    await c.env.DOCUMENTS.put(storageKey, arrayBuffer)

    // UPDATE task to done
    const [updatedTask] = await tx.update(tasks)
      .set({ status: 'done', completedAt: new Date(), completedBy: ctx.userId })
      .where(eq(tasks.id, taskId))
      .returning()

    // Timeline event
    const isReupload = existingDocs.length > 0
    await tx.insert(timelineEvents).values({
      organizationId: ctx.orgId,
      serviceRequestId: id,
      eventType: 'document_upload',
      title: isReupload ? 'Signed Form 3 re-uploaded' : 'Signed Form 3 uploaded',
      actorType: 'tenant',
      metadata: {
        documentId,
        mimeType,
        sizeBytes: arrayBuffer.byteLength,
        ...(isReupload ? { supersededDocumentId: existingDocs[0]?.id } : {}),
      },
    })

    return { document: newDoc, task: updatedTask }
  })

  // 6. Evaluate readiness and update status
  const readiness = await evaluateServiceRequestReadiness(id, ctx.orgId, deps.db)
  if (readiness.newStatus) {
    await deps.db.update(serviceRequests)
      .set({ status: readiness.newStatus })
      .where(and(eq(serviceRequests.organizationId, ctx.orgId), eq(serviceRequests.id, id)))
  }

  return c.json({
    success: true,
    data: {
      document: result.document,
      task: result.task,
      serviceRequestStatus: readiness.newStatus,
    },
  }, 200)
}
```

### Endpoint 6: GET /api/v1/service-requests/:id

**Auth:** Clerk org-scoped JWT
**Returns:** Service request with tasks. Does NOT return raw `form3Data` — returns a summary to avoid PII over-exposure.

```typescript
export const getServiceRequestHandler: AppRouteHandler<typeof getServiceRequestRoute> = async (c) => {
  const ctx = buildServiceContext(c)
  const deps = buildServiceDependencies(c)
  const { id } = c.req.param()

  const sr = await deps.db.query.serviceRequests.findFirst({
    where: and(eq(serviceRequests.organizationId, ctx.orgId), eq(serviceRequests.id, id)),
    with: { tasks: true, documents: true, timelineEvents: { orderBy: (e, { desc }) => [desc(e.occurredAt)] } },
  })

  if (!sr) return c.json({ code: 'NOT_FOUND', message: 'Service request not found' }, 404)

  const form3Data = sr.intakeData as Partial<Form3Data> | null
  const form3DataSummary = form3Data ? {
    hasData: true,
    completedParts: Array.from(determineCompletedSteps(form3Data)),
    lastSavedAt: sr.form3LastSavedAt?.toISOString() ?? null,
  } : { hasData: false, completedParts: [], lastSavedAt: null }

  // Strip intakeData from response to avoid PII leakage
  const { intakeData, ...srWithoutData } = sr

  return c.json({
    success: true,
    data: { ...srWithoutData, form3DataSummary },
  }, 200)
}
```

---

## 8. Implementation Sequence

### Phase 1 — Data Layer (Days 1–2)

| Step | Task | Verify |
|------|------|--------|
| 1.1 | Create `form3.types.ts` with all interfaces | `npx tsc --noEmit` passes |
| 1.2 | Create `form3.errors.ts` with all error classes | `npx tsc --noEmit` passes |
| 1.3 | Write migration `0012_form3_data.sql` and run `drizzle-kit push` | `SELECT` on new columns succeeds |
| 1.4 | Update `service-requests.ts` schema with new columns | Types align with migration |
| 1.5 | Implement `form3.status.ts` with `evaluateServiceRequestReadiness` | Unit tests: incomplete task → `in_progress`; complete task + no payment → `awaiting_payment`; all complete → `ready_for_submission` |
| 1.6 | Implement `mergeForm3Data` utility | Unit test: merging partB with 1 director into existing partB with 2 directors replaces the array |
| 1.7 | Implement `form3.validate.ts` with all V1–V26 rules | Unit tests for each rule |

### Phase 2 — Template Preparation (Day 3)

| Step | Task | Verify |
|------|------|--------|
| 2.1 | Read Section 4.5 (placeholder table) to understand expected keys | N/A |
| 2.2 | Obtain official PACRA Form 3 DOCX | File opens in Word |
| 2.3 | Inject `{{PLACEHOLDER}}` tokens (Section 4.3 workflow) | All tokens visible in document |
| 2.4 | Extract partial XML snippets for directors/shareholders/beneficialOwners/guarantors | Files saved in `assets/templates/partials/` |
| 2.5 | Write `scripts/validate-placeholders.ts` | Script exits 0 on valid template |
| 2.6 | Run validation script and fix any split tokens | Script passes with 0 errors |
| 2.7 | Upload template to local R2 | `wrangler r2 object list` shows the file |

### Phase 3 — DOCX Generation Service (Days 4–5)

| Step | Task | Verify |
|------|------|--------|
| 3.1 | Implement `form3-generator.utils.ts` (escapeXml, formatDate, checkbox, escapeRegex, normalizePlaceholders, applyReplacements) | Unit tests with fixture data |
| 3.2 | Implement `buildPartAReplacements` | Snapshot test with fixture PartA |
| 3.3 | Implement `buildPartFGHReplacements` | Snapshot tests: individual secretary, firm secretary, Part G, Part H |
| 3.4 | Implement `buildDirectorSection`, `buildShareholderSection`, `buildBeneficialOwnerSection`, `buildGuarantorSection` | Snapshot tests per section |
| 3.5 | Implement `loadTemplate` and `loadPartial` | Integration test: loads from local R2 |
| 3.6 | Implement `generateForm3Docx` | End-to-end test: generates valid DOCX from fixture data; Word opens the file |
| 3.7 | Wire up `GET /form3.docx` route handler | `curl` returns downloadable DOCX |

### Phase 4 — Wizard UI (Days 6–8)

| Step | Task | Verify |
|------|------|--------|
| 4.1 | Create React Query hooks (`use-form3-draft`, `use-form3-save`) | Hooks compile |
| 4.2 | Create shared components (`PersonFields`, `AddressFields`, `PersonCard`) | Storybook renders |
| 4.3 | Implement `Form3Wizard.tsx` with hydration (`useEffect`, Section 5.3) | Fill Part A, refresh page, confirm data restores |
| 4.4 | Implement `beforeunload` warning (Section 5.4) | Modify field, navigate away, confirm browser warning |
| 4.5 | Implement `StepPartA` with react-hook-form + Zod | Fill form, advance step, data auto-saves |
| 4.6 | Implement `StepDirectors` (add/remove directors) | Add 2 directors, remove 1, verify array state |
| 4.7 | Implement `StepShareholders`, `StepBeneficialOwnership`, `StepGuarantors` | Conditional rendering based on companyType |
| 4.8 | Implement `StepSecretary` (individual/firm toggle) | Toggle between types, verify correct fields render |
| 4.9 | Implement `StepDeclaration`, `StepLodgingPerson` | V14: declarant matches director or secretary name |
| 4.10 | Implement `StepReview` (read-only summary) | All parts display correctly |
| 4.11 | Implement auto-save with 1500ms debounce (Section 5.6) | Network tab shows PATCH calls |

### Phase 5 — Download and Upload Loop (Days 9–10)

| Step | Task | Verify |
|------|------|--------|
| 5.1 | Implement `StepDownload` with all 6 download states | State transitions correct |
| 5.2 | Wire up DOCX download (calls `GET /form3.docx`) | File downloads with correct name |
| 5.3 | Wire up signed upload (`POST /tasks/:taskId/upload`) | File uploads; task transitions to `done` |
| 5.4 | Implement re-upload flow | Old document archived; new document active |
| 5.5 | Verify readiness evaluation fires after upload | Status transitions to `awaiting_payment` |

### Phase 6 — Backoffice and Hardening (Days 11–12)

| Step | Task | Verify |
|------|------|--------|
| 6.1 | Implement idempotency for `POST /service-requests` (ADR-8) | Double-click returns same record |
| 6.2 | Implement optimistic locking for `PATCH /form3-draft` | Two tabs saving concurrently → 409 for the slower one |
| 6.3 | Implement V26 validation (reserved PACRA suffixes) | "Acme (In Liquidation)" returns 422 |
| 6.4 | Add all timeline events | Timeline view shows complete audit trail |
| 6.5 | Add error boundaries to wizard components | Component crash shows fallback UI |
| 6.6 | Accessibility pass: labels, ARIA, keyboard navigation | Tab through all fields |

---

## 9. Error Handling & Edge Cases

### 9.1 Template Not Found in R2

**Condition:** `loadTemplate()` returns null from R2.
**Error:** `Form3TemplateNotFoundError`
**HTTP:** 500 `{ errorId, message: 'Document generation failed' }`
**Action:** Log error with `errorId`. DevOps must verify template upload to R2 bucket.

### 9.2 Template Corrupted (JSZip Failure)

**Condition:** `JSZip.loadAsync()` throws on the template buffer.
**Error:** `Form3GenerationError`
**HTTP:** 500 `{ errorId, message: 'Document generation failed' }`
**Action:** Re-upload a verified template. Check template DOCX opens in Word.

### 9.3 XML Special Characters in User Data

**Condition:** User enters `AT&T` as company name.
**Defence:** `escapeXml()` applied to all field values before insertion (ADR-5).
**Failure mode if missed:** Word refuses to open the DOCX ("file is corrupted").

### 9.4 Split Placeholders After Template Re-save

**Condition:** Someone opens the template in Word and re-saves it, splitting `{{COMPANY_NAME}}` across runs.
**Defence:** `normalizePlaceholders()` (ADR-2, Layer 2).
**Verification:** `validate-placeholders.ts` should be re-run after any template edit.

### 9.5 Duplicate Service Request Creation

**Condition:** Double-click on "Start Incorporation" button.
**Defence:** Idempotency check returns existing record (ADR-8).
**HTTP:** 200 `{ created: false }`

### 9.6 Draft Conflict (Two Tabs)

**Condition:** User opens wizard in two browser tabs and saves from both.
**Defence:** Optimistic locking via `currentLastSavedAt` field. Second save gets 409.
**Client handling:** Show conflict banner, reload draft from server.

### 9.7 File Upload — Empty File

**Condition:** User uploads a 0-byte file.
**HTTP:** 422 `{ code: 'EMPTY_FILE' }`

### 9.8 File Upload — Oversized File

**Condition:** File exceeds `FORM3_MAX_UPLOAD_SIZE_BYTES`.
**HTTP:** 413 `{ code: 'FILE_TOO_LARGE' }`

### 9.9 File Upload — Wrong MIME Type

**Condition:** User uploads a JPEG or text file instead of PDF/DOCX.
**HTTP:** 415 `{ code: 'INVALID_FILE_TYPE' }`

### 9.10 R2 Write Failure During Upload

**Condition:** `env.DOCUMENTS.put()` throws during file upload.
**Action:** Transaction rolls back (document record not committed). Return 500.
**Note:** The R2 put is inside the Drizzle transaction block, but R2 is not transactional. If DB commits but R2 fails, the document record will reference a missing object. Mitigate by putting R2 write BEFORE the DB commit and wrapping in try/catch.

### 9.11 Template Version Mismatch

**Condition:** `form3TemplateVersion` on saved data differs from `env.FORM3_TEMPLATE_VERSION`.
**HTTP:** 409 `{ code: 'TEMPLATE_VERSION_MISMATCH' }`
**User action:** Contact support to migrate draft to new template.

### 9.12 Company Type Change Mid-Wizard

**Condition:** User fills shareholders (Part C), then changes companyType to `private_guarantee`.
**Defence:** `handleCompanyTypeChange()` clears partC/partD and resets those completed steps (Section 5.7).

### 9.13 Reserved PACRA Suffix (V26)

**Condition:** Company name contains "(In Liquidation)" or "(In Receivership)".
**HTTP:** 422 with field error on `partA.companyName`.
**Reason:** These suffixes are reserved by PACRA for companies under formal proceedings.

### 9.14 Network Failure During Auto-save

**Condition:** PATCH /form3-draft fails due to network timeout.
**Client:** `saveStatus` transitions to `'error'`. Wizard remains usable. Auto-save retries on next change.
**Data:** `isDirty` remains `true`, preventing navigation without warning.

### 9.15 Unresolved Placeholders in Generated DOCX

**Condition:** A `{{PLACEHOLDER}}` key was not found in the replacements map.
**Defence:** `applyReplacements()` logs a structured warning per unresolved token.
**Action:** Developer must add the missing key to the appropriate builder function.

---

## 10. Testing Strategy

### 10.1 Unit Tests (Vitest)

**File:** `packages/api-services/src/domains/form3/__tests__/`

| Test Suite | What to Test |
|------------|-------------|
| `form3-generator.utils.test.ts` | `escapeXml` — all 5 special chars + null/undefined; `formatDate` — valid dates, invalid input; `checkbox` — true/false; `normalizePlaceholders` — split runs merged correctly; `applyReplacements` — all keys replaced, unresolved tokens logged |
| `form3.validate.test.ts` | V1–V26 individually. Each test: valid input passes, invalid input fails with correct error |
| `form3.status.test.ts` | `evaluateServiceRequestReadiness` — 3 scenarios: task incomplete, task done + no payment, all ready |
| `form3-generator.service.test.ts` | `buildPartAReplacements` — snapshot test; `buildPartFGHReplacements` — both secretary types; `buildDirectorSection` — snapshot with fixture director; `generateForm3Docx` — end-to-end with test fixture template |

**Test fixture data:**

```typescript
// packages/api-services/src/domains/form3/__tests__/fixtures/form3-data.fixture.ts

export const testForm3Data: Form3Data = {
  serviceRequestId: '00000000-0000-0000-0000-000000000001',
  organizationId: 'org_test_123',
  templateVersion: 'v1',
  partA: {
    companyName: 'Acme Technologies Limited',
    companyType: 'private_shares',
    principalBusinessActivity: 'Software development and consulting services',
    articlesType: 'Standard',
    articlesRestrictBusiness: false,
    nominalCapitalZmw: 100000,
    numberOfShares: 1000,
    shareClass: 'ordinary',
    parValue: 100,
    registeredOffice: {
      plot: '123', street: 'Independence Avenue', area: 'Longacres',
      town: 'Lusaka', province: 'Lusaka', country: 'Zambia',
    },
    postalAddress: { postBox: 'P.O. Box 12345', area: 'Longacres', town: 'Lusaka', province: 'Lusaka' },
    contactEmail: 'info@acme.co.zm',
    contactPhone: '+260971234567',
    firstFinancialYearEnd: '2027-03-31',
    recurringYearEndMonthDay: '03-31',
  },
  partB: {
    directors: [
      {
        firstName: 'John', surname: 'Banda', gender: 'male',
        dateOfBirth: '1985-06-15', nationality: 'Zambian',
        identityType: 'nrc', identityNumber: '123456/10/1',
        isResidentInZambia: true, email: 'john@acme.co.zm',
        mobile: '+260971111111', occupation: 'Engineer',
        position: 'Managing Director', appointmentDate: '2026-01-01',
        physicalAddress: { plot: '1', street: 'Main Rd', area: 'Kabulonga', town: 'Lusaka', province: 'Lusaka', country: 'Zambia' },
        postalAddress: { postBox: 'P.O. Box 100', area: 'Kabulonga', town: 'Lusaka', province: 'Lusaka' },
      },
      // ... second director
    ],
  },
  partF: {
    secretary: {
      type: 'individual', firstName: 'Grace', surname: 'Mwanza',
      gender: 'female', dateOfBirth: '1990-03-20', nationality: 'Zambian',
      identityType: 'nrc', identityNumber: '654321/10/1',
      isResidentInZambia: true, email: 'grace@acme.co.zm',
      mobile: '+260972222222',
      physicalAddress: { plot: '5', street: 'Cairo Rd', area: 'Town Centre', town: 'Lusaka', province: 'Lusaka', country: 'Zambia' },
      postalAddress: { postBox: 'P.O. Box 200', area: 'Town Centre', town: 'Lusaka', province: 'Lusaka' },
    },
  },
  partG: { declarantName: 'John Banda', declarantCapacity: 'first_director' },
  partH: {
    firstName: 'John', surname: 'Banda', gender: 'male',
    dateOfBirth: '1985-06-15', nationality: 'Zambian',
    identityType: 'nrc', identityNumber: '123456/10/1',
    isResidentInZambia: true, email: 'john@acme.co.zm',
    mobile: '+260971111111', capacity: 'First Director',
    physicalAddress: { plot: '1', street: 'Main Rd', area: 'Kabulonga', town: 'Lusaka', province: 'Lusaka', country: 'Zambia' },
    postalAddress: { postBox: 'P.O. Box 100', area: 'Kabulonga', town: 'Lusaka', province: 'Lusaka' },
  },
}
```

### 10.2 Integration Tests

| Test | Scope |
|------|-------|
| `POST /service-requests` idempotency | Create → returns 201. Call again → returns 200 with same ID. |
| `PATCH /form3-draft` optimistic locking | Save from session A → 200. Save from session B with stale `lastSavedAt` → 409. |
| `GET /form3.docx` generation | Full round-trip: create request → save draft → download DOCX → verify it's a valid ZIP. |
| `POST /tasks/:taskId/upload` | Upload PDF → task transitions to `done` → readiness evaluates correctly. |
| Tenant isolation | Create request for org A. Attempt to fetch from org B → 404. |

### 10.3 E2E Tests (Playwright)

| Test | Flow |
|------|------|
| Happy path | Start wizard → fill all parts → download DOCX → upload signed copy → verify task complete |
| Draft restoration | Fill Part A → close browser → reopen → verify Part A data restored |
| Company type switch | Select `private_shares` → fill shareholders → switch to `private_guarantee` → verify shareholders cleared, guarantors visible |

---

## 11. Security Considerations

### 11.1 Tenant Data Isolation

- Every query MUST include `eq(table.organizationId, ctx.orgId)`.
- The `buildServiceContext(c)` function extracts `orgId` from the authenticated Clerk session.
- No endpoint accepts `organizationId` as a request parameter — it is always derived from the JWT.

### 11.2 PII Protection

- `GET /service-requests/:id` does NOT return raw `form3Data`. It returns a summary (`hasData`, `completedParts`, `lastSavedAt`).
- `GET /form3-draft` returns the full data only to the authenticated tenant. Backoffice access uses a separate route.
- Timeline event `metadata` MUST NOT contain form data or PII fields. Only IDs, counts, and status values.

### 11.3 File Upload Security

- MIME type validated from the `file.type` property.
- File size capped at `FORM3_MAX_UPLOAD_SIZE_BYTES` (default 10MB).
- Storage keys include `organizationId` to prevent cross-tenant access.
- Downloaded files use `Content-Disposition: attachment` to prevent inline rendering.

### 11.4 XML Injection Prevention

- All user data passes through `escapeXml()` before insertion into OOXML (ADR-5).
- `applyReplacements()` does NOT escape — the boundary is the caller.
- Block-level OOXML values (section clones) are NEVER passed through `escapeXml()`.

### 11.5 Rate Limiting

- `GET /form3.docx` uses `FORM3_DOWNLOAD_RATE_LIMIT` to cap downloads per tenant per minute.
- Implement using the existing `strictRateLimit` middleware from `@repo/backend/middleware/rate-limit`.

---

## 12. Deployment & Configuration

### 12.1 Pre-deployment Checklist

- [ ] Migration `0012_form3_data.sql` applied to production database
- [ ] Form 3 template uploaded to R2 production bucket
- [ ] All partial XML files uploaded to R2 under `templates/partials/`
- [ ] `validate-placeholders.ts` passes against production template
- [ ] Environment variables set in `wrangler.toml` / worker secrets
- [ ] `FORM3_TEMPLATE_VERSION` matches the uploaded template

### 12.2 Environment Variables

| Variable | Where Set | Example |
|----------|-----------|---------|
| `FORM3_TEMPLATE_OBJECT_KEY` | `wrangler.toml [vars]` | `templates/form3-template-v1.docx` |
| `FORM3_TEMPLATE_VERSION` | `wrangler.toml [vars]` | `v1` |
| `FORM3_MIN_NOMINAL_CAPITAL_ZMW` | `wrangler.toml [vars]` | `5000` |
| `FORM3_MAX_UPLOAD_SIZE_BYTES` | `wrangler.toml [vars]` | `10485760` |
| `FORM3_DOWNLOAD_RATE_LIMIT` | `wrangler.toml [vars]` | `10` |

### 12.3 Template Version Upgrade Procedure

When PACRA revises Form 3:

1. Prepare the new template following Section 4.3.
2. Upload as `templates/form3-template-v2.docx`.
3. Update `FORM3_TEMPLATE_OBJECT_KEY` and `FORM3_TEMPLATE_VERSION` to `v2`.
4. Deploy the worker. New generations use v2.
5. Existing drafts saved with `form3TemplateVersion: 'v1'` will receive a 409 on download, prompting support migration.

### 12.4 Monitoring

Log events to watch:

| Event | Severity | Action |
|-------|----------|--------|
| `form3_generation_failed` | ERROR | Check R2 template availability, review errorId in logs |
| `unresolved_placeholder` | WARN | Add missing key to builder function |
| `TEMPLATE_NOT_FOUND` | ERROR | Re-upload template to R2 |
| `CONFLICT` (409 on draft save) | INFO | Normal — user has multiple tabs open |

---

## 13. Glossary

| Term | Definition |
|------|-----------|
| **Form 3** | PACRA "Application for Incorporation of a Company" — the physical form tenants must submit |
| **PACRA** | Patents and Companies Registration Agency — Zambian corporate regulator |
| **Service Request** | A one-off compliance action (not recurring). Company incorporation is a service request. |
| **Task** | A gating input that the tenant must complete before submission can proceed |
| **Wizard** | Multi-step frontend form that collects Form 3 data from the tenant |
| **Template Prep** | One-time process of injecting `{{PLACEHOLDER}}` tokens into the official PACRA DOCX |
| **Partial** | An XML snippet extracted from the template representing a repeatable row/section |
| **Run** | An OOXML `<w:r>` element — the smallest unit of text formatting in a Word document |
| **Run Split** | When Word fragments a placeholder like `{{NAME}}` across multiple `<w:r>` elements |
| **Block-level Placeholder** | A `{{SECTION}}` token that is replaced with cloned OOXML, not escaped text |
| **Readiness** | The computed state indicating whether all gates (tasks, documents, payment) are satisfied |

---

## 14. Known Limitations & Future Work

### 14.1 v1 Limitations

1. **Beneficial owner sub-tables:** Part D beneficial owners with body corporate entries have nested sub-tables (directors, shareholders, beneficial owners of the body corporate). v1 uses flat fields and does not generate nested sub-tables. Tracked for v2.

2. **No form3Data encryption at rest:** The `intakeData` JSONB column stores Form 3 data in plaintext. A future iteration should implement field-level encryption using `encryptField()`/`decryptField()` from `src/lib/crypto/encrypt-field.ts`. This requires adding an `DOCUMENT_ENCRYPTION_KEY` secret.

3. **Single share allocation per shareholder:** v1 supports one `ShareAllocation` per shareholder. Multiple share classes per shareholder deferred to v2.

4. **No concurrent editing support:** Two users in the same organisation editing the wizard simultaneously will encounter 409 conflicts. Real-time collaboration (e.g. via CRDT) is out of scope.

5. **Template version migration:** When upgrading from v1 to v2 template, existing drafts require manual migration by support. An automated migration script is deferred.

### 14.2 Future Work

- **PDF generation option** — Some tenants prefer PDF over DOCX. Add a `/form3.pdf` endpoint using a DOCX-to-PDF conversion service.
- **Digital signatures** — Replace physical print-and-sign with digital signatures (pending PACRA regulatory approval).
- **Bulk incorporation** — Enterprise tenants incorporating multiple subsidiaries simultaneously.
- **Form 3 amendment** — Handle post-incorporation corrections via a Form 3A amendment flow.
- **Audit log export** — Backoffice ability to export complete Form 3 timeline for regulatory audit purposes.
