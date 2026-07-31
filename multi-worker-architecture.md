---
title: "Bumara Multi-Worker Architecture"
description: "This document provides comprehensive documentation on Bumara's multi-worker Cloudflare architecture, authentication system, Hono context patterns, and..."
---

This document provides comprehensive documentation on Bumara's multi-worker Cloudflare architecture, authentication system, Hono context patterns, and deployment procedures.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Worker Structure](#worker-structure)
3. [Authentication & Authorization](#authentication--authorization)
4. [Hono Context & Service Patterns](#hono-context--service-patterns)
5. [Inter-Worker Communication](#inter-worker-communication)
6. [Cloudflare DNS Configuration](#cloudflare-dns-configuration)
7. [Deployment Process](#deployment-process)
8. [Environment Variables](#environment-variables)
9. [Troubleshooting](#troubleshooting)

---

## Architecture Overview

### High-Level Architecture

```
                    Internet Traffic
                          │
    ┌─────────────────────┼─────────────────────────────────┐
    │                     │                                 │
    ▼                     ▼                                 ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ api-compliance  │ │  api-backoffice │ │  api-payroll    │ │   api-jobs      │
│ api.bumara.com  │ │ api-backoffice. │ │ api-payroll.    │ │ webhooks.bumara │
│                 │ │ bumara.com      │ │ bumara.com      │ │ .com            │
└────────┬────────┘ └────────┬────────┘ └────────┬────────┘ └────────┬────────┘
         │                   │                   │                   │
         │                   │   Service Bindings (Zero-Latency RPC) │
         └───────────────────┼───────────────────┼───────────────────┘
                             │                   │
                             ▼                   ▼
                    ┌─────────────────────────────────────┐
                    │        Shared Resources             │
                    │  - Neon PostgreSQL (DATABASE_URL)   │
                    │  - Clerk Auth (CLERK_SECRET_KEY)    │
                    │  - AWS S3 (Document Storage)        │
                    │  - Stripe (Payments)                │
                    └─────────────────────────────────────┘
```

### Workers Overview

| Worker | Domain | Purpose | Key Features |
|--------|--------|---------|--------------|
| **api-compliance** | `api.bumara.com` | Public tenant API | Organizations, regulators, filings, tasks, documents |
| **api-backoffice** | `api-backoffice.bumara.com` | Internal admin | User management, submissions, dashboard, catalog |
| **api-payroll** | `api-payroll.bumara.com` | Payroll processing | 82 endpoints for employees, runs, payslips, loans |
| **api-jobs** | `webhooks.bumara.com` | Background work | Webhooks, cron jobs, async processing |

### Frontend Applications

| App | Domain | API Worker |
|-----|--------|------------|
| **apps/app** | `app.bumara.com` | api-compliance |
| **apps/backoffice** | `backoffice.bumara.com` | api-backoffice |

---

## Worker Structure

Each worker follows a consistent file structure:

```
apps/api-{name}/
├── src/
│   ├── index.ts           # Worker entry point (fetch handler)
│   ├── app.ts             # Hono app with middleware chain
│   ├── env.ts             # Environment validation with Zod
│   ├── scheduled.ts       # Cron handlers (api-jobs only)
│   └── routes/
│       ├── index.ts       # Route aggregator + health check
│       └── {feature}/
│           ├── index.ts   # Hono sub-router export
│           ├── routes.ts  # OpenAPI route definitions
│           └── handlers.ts# Handler implementations
├── wrangler.toml          # Cloudflare Worker configuration
├── tsconfig.json
└── package.json
```

### Worker Entry Point (`index.ts`)

```typescript
// apps/api-compliance/src/index.ts
import app from './app';
import type { Env } from './env';

export default {
  fetch(request: Request, env: Env, ctx: ExecutionContext) {
    return app.fetch(request, env, ctx);
  },
};

// Re-export types for Hono RPC clients
export type { AppRouter } from './routes';
export type { Env } from './env';
```

### Environment Validation (`env.ts`)

```typescript
// apps/api-compliance/src/env.ts
import { z } from 'zod';
import type { ServiceBindings } from '@repo/worker-types';

export const envSchema = z.object({
  DATABASE_URL: z.string(),
  CLERK_SECRET_KEY: z.string().startsWith('sk_'),
  CLERK_PUBLISHABLE_KEY: z.string().startsWith('pk_'),
  WORKER_NAME: z.string().default('api-compliance'),
  NODE_ENV: z.enum(['development', 'staging', 'production']).default('development'),
});

export interface Env extends ServiceBindings {
  DATABASE_URL: string;
  CLERK_SECRET_KEY: string;
  CLERK_PUBLISHABLE_KEY: string;
  WORKER_NAME: string;
  NODE_ENV: 'development' | 'staging' | 'production';

  // Service bindings (from wrangler.toml)
  API_JOBS?: Fetcher;
  API_PAYROLL?: Fetcher;
}
```

---

## Authentication & Authorization

### Authentication Flow

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Frontend   │     │   Clerk      │     │   Worker     │
│   (React)    │     │   Auth       │     │   API        │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       │  1. User Login     │                    │
       │───────────────────>│                    │
       │                    │                    │
       │  2. JWT Token      │                    │
       │<───────────────────│                    │
       │                    │                    │
       │  3. API Request + Bearer Token          │
       │────────────────────────────────────────>│
       │                    │                    │
       │                    │  4. Verify JWT     │
       │                    │<───────────────────│
       │                    │                    │
       │                    │  5. Session Claims │
       │                    │───────────────────>│
       │                    │                    │
       │  6. Response       │                    │
       │<────────────────────────────────────────│
```

### Middleware Chain

The authentication middleware chain is applied in order:

```typescript
// 1. clerkAuth - Validates JWT and hydrates auth object
// 2. attachAuthToContext - Extracts claims and sets context variables
// 3. requireAuth - Verifies user exists in database
// 4. requireOrg - Verifies organization exists and user has access
// 5. requireRole/requirePermission - RBAC checks
```

### Context Variables Set by Auth Middleware

After authentication middleware runs, these variables are available in handlers:

| Variable | Type | Source | Description |
|----------|------|--------|-------------|
| `userId` | `string` | Clerk JWT | Clerk user ID (e.g., `user_xxx`) |
| `sessionId` | `string` | Clerk JWT | Clerk session ID |
| `orgId` | `string` | Clerk JWT | Current organization ID |
| `orgRole` | `string` | Clerk JWT | User's role (e.g., `org:admin`) |
| `orgSlug` | `string` | Clerk JWT | Organization slug |
| `orgPermissions` | `string[]` | Clerk JWT | Array of permissions |
| `authHas` | `function` | Clerk SDK | Permission check function |
| `staffAgentId` | `string` | Database | Backoffice agent ID (backoffice only) |
| `staffRole` | `string` | Database | Backoffice role (backoffice only) |
| `logger` | `PinoLogger` | Middleware | Request-scoped logger |

### Role Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│                    ORGANIZATION ROLES                       │
├─────────────────────────────────────────────────────────────┤
│  org:admin     │ Full organization access (owner/creator)   │
│  org:manager   │ Manages submissions and approvals          │
│  org:member    │ Default role with basic access             │
├─────────────────────────────────────────────────────────────┤
│                    BACKOFFICE ROLES                         │
├─────────────────────────────────────────────────────────────┤
│  org:backoffice_admin   │ Full backoffice access            │
│  org:backoffice_manager │ Manage/approve backoffice tasks   │
│  org:backoffice_member  │ Review/submit only                │
└─────────────────────────────────────────────────────────────┘
```

### Permission System

Permissions follow the format: `org:feature:action`

```typescript
// Example permissions
Permissions.COMPLIANCE_READ    // org:compliance:read
Permissions.COMPLIANCE_CREATE  // org:compliance:create
Permissions.COMPLIANCE_SUBMIT  // org:compliance:submit
Permissions.COMPLIANCE_APPROVE // org:compliance:approve
Permissions.PAYROLL_RUN        // org:payroll:run
Permissions.PAYMENTS_PROCESS   // org:payments:process
```

### Middleware Usage Examples

#### Tenant Routes (api-compliance)

```typescript
// routes/organizations/routes.ts
export const getOrganizationRoute = createRoute({
  method: 'get',
  path: '/organizations/{id}',
  middleware: [requireAuth, requireOrg], // Auth + Org verification
  // ...
});
```

#### Backoffice Routes (api-backoffice)

```typescript
// routes/users/routes.ts
const backofficeMiddleware = [
  requireAuth,           // User must be authenticated
  requireBackofficeOrg,  // Must be in backoffice organization
  requireActiveStaff,    // Must have active staff record
  requireBackoffice,     // Must have backoffice role
];

export const listUsersRoute = createRoute({
  method: 'get',
  path: '/backoffice/users',
  middleware: backofficeMiddleware,
  // ...
});
```

#### Permission-Based Routes

```typescript
// Require specific permission
export const approvePayrollRoute = createRoute({
  method: 'post',
  path: '/payroll/runs/{id}/approve',
  middleware: [
    requireAuth,
    requireOrg,
    requirePermission(Permissions.PAYROLL_APPROVE),
  ],
  // ...
});
```

---

## Hono Context & Service Patterns

### AppBindings Type

All workers share a common type definition for Hono context:

```typescript
// packages/backend/src/types/index.ts
export type AppBindings = {
  Variables: {
    logger: PinoLogger;
    userId?: string;
    sessionId?: string;
    orgId?: string;
    orgRole?: string;
    orgSlug?: string;
    orgPermissions?: string[];
    authHas?: (params: { role?: string; permission?: string }) => boolean;
    staffAgentId?: string;  // Backoffice only
    staffRole?: string;     // Backoffice only
  };
};
```

### Service Context Pattern

Handlers use a service context pattern to pass request-scoped data to business logic:

```typescript
// packages/backend/src/core/context/service-context.ts

export function buildServiceContext(c: Context<AppBindings>): ServiceContext {
  const forwardedFor = c.req.header('x-forwarded-for');
  const ipAddress = forwardedFor
    ? forwardedFor.split(',')[0]?.trim()
    : c.req.header('x-real-ip');

  return {
    userId: c.get('userId') ?? null,
    actor: deriveActor(c.get('userId') ?? null, c.get('staffAgentId')),
    orgId: c.get('orgId'),
    roles: c.get('orgRole') ? [c.get('orgRole')] : undefined,
    ipAddress: ipAddress || undefined,
    userAgent: c.req.header('user-agent') || undefined,
    requestId: c.get('requestId'),
    env: c.env,
  };
}

export function buildServiceDependencies(c: Context<AppBindings>): ServiceDependencies {
  return {
    db,              // Database client
    logger: c.get('logger'),
    now: () => new Date(),
  };
}
```

### Handler Pattern

Handlers follow a consistent pattern using the service context:

```typescript
// routes/filings/handlers.ts
import { buildServiceContext, buildServiceDependencies, handleServiceError } from '@repo/backend/context';
import { getFilings } from '@repo/api-services/domains/compliance';

export const listFilingsHandler: AppRouteHandler<typeof listFilingsRoute> = async (c) => {
  const ctx = buildServiceContext(c);
  const deps = buildServiceDependencies(c);

  try {
    const { data, total } = await getFilings(ctx, deps, {
      organizationId: ctx.orgId!,
      limit: 50,
      offset: 0,
    });

    return c.json({
      success: true,
      data,
      meta: { total, limit: 50, offset: 0 },
    }, 200);
  } catch (error) {
    return handleServiceError(c, error, 'Failed to fetch filings');
  }
};
```

### Error Handling

The `handleServiceError` function maps service errors to HTTP responses:

```typescript
// Service error codes map to HTTP status codes
const serviceErrorStatusMap = {
  FORBIDDEN: 403,
  NOT_FOUND: 404,
  UNAUTHORIZED: 401,
  VALIDATION_ERROR: 422,
  BAD_REQUEST: 400,
  CONFLICT: 409,
  INTERNAL: 500,
};
```

---

## Inter-Worker Communication

Workers communicate via Cloudflare Service Bindings for zero-latency RPC.

### Service Bindings Configuration

```toml
# apps/api-compliance/wrangler.toml
[[services]]
binding = "API_JOBS"
service = "bumara-api-jobs"

[[services]]
binding = "API_PAYROLL"
service = "bumara-api-payroll"
```

### Making Inter-Worker Calls

```typescript
// From api-compliance, call api-jobs
const response = await env.API_JOBS.fetch(
  new Request('http://internal/jobs/send-email', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ to: 'user@example.com', template: 'welcome' })
  })
);

const result = await response.json();
```

### Type-Safe RPC Clients

For external clients (frontends), use the `@repo/api-client` package:

```typescript
// apps/app (tenant frontend)
import { createComplianceClient } from '@repo/api-client/compliance';

const client = createComplianceClient('https://api.bumara.com');
const { data } = await client.organizations.$get();
```

---

## Cloudflare DNS Configuration

### Required DNS Records

Configure the following CNAME records in Cloudflare DNS:

| Type | Name | Target | Proxy |
|------|------|--------|-------|
| CNAME | `api` | `bumara-api-compliance.workers.dev` | Proxied (orange cloud) |
| CNAME | `api-backoffice` | `bumara-api-backoffice.workers.dev` | Proxied (orange cloud) |
| CNAME | `api-payroll` | `bumara-api-payroll.workers.dev` | Proxied (orange cloud) |
| CNAME | `webhooks` | `bumara-api-jobs.workers.dev` | Proxied (orange cloud) |

### Step-by-Step DNS Setup

1. **Login to Cloudflare Dashboard**
   - Go to https://dash.cloudflare.com
   - Select your domain (`bumara.com`)

2. **Navigate to DNS Settings**
   - Click "DNS" in the left sidebar
   - Click "Add record"

3. **Add CNAME for api-compliance**
   ```
   Type: CNAME
   Name: api
   Target: bumara-api-compliance.workers.dev
   Proxy status: Proxied (orange cloud ON)
   TTL: Auto
   ```

4. **Add CNAME for api-backoffice**
   ```
   Type: CNAME
   Name: api-backoffice
   Target: bumara-api-backoffice.workers.dev
   Proxy status: Proxied (orange cloud ON)
   TTL: Auto
   ```

5. **Add CNAME for api-payroll**
   ```
   Type: CNAME
   Name: api-payroll
   Target: bumara-api-payroll.workers.dev
   Proxy status: Proxied (orange cloud ON)
   TTL: Auto
   ```

6. **Add CNAME for api-jobs (webhooks)**
   ```
   Type: CNAME
   Name: webhooks
   Target: bumara-api-jobs.workers.dev
   Proxy status: Proxied (orange cloud ON)
   TTL: Auto
   ```

### SSL/TLS Configuration

1. Go to **SSL/TLS** in Cloudflare dashboard
2. Set encryption mode to **Full (strict)**
3. Enable **Always Use HTTPS**
4. Enable **Automatic HTTPS Rewrites**

### Custom Domain in Wrangler

The wrangler.toml files are already configured with routes:

```toml
# apps/api-compliance/wrangler.toml
[env.production]
routes = [
  { pattern = "api.bumara.com/*", zone_name = "bumara.com" }
]
```

---

## Deployment Process

### Prerequisites

1. **Cloudflare Account** with Workers paid plan ($5/month minimum)
2. **Wrangler CLI** authenticated: `wrangler login`
3. **Environment secrets** configured in Cloudflare dashboard

### Setting Environment Secrets

For each worker, set secrets via Cloudflare dashboard or CLI:

```bash
# Via CLI (for each worker)
cd apps/api-compliance

# Required secrets
wrangler secret put DATABASE_URL
wrangler secret put CLERK_SECRET_KEY
wrangler secret put CLERK_PUBLISHABLE_KEY
wrangler secret put CLERK_INTERNAL_ORG_ID  # For backoffice

# Optional secrets
wrangler secret put AWS_ACCESS_KEY_ID
wrangler secret put AWS_SECRET_ACCESS_KEY
wrangler secret put STRIPE_SECRET_KEY
```

Or via Cloudflare Dashboard:
1. Go to **Workers & Pages**
2. Select your worker
3. Go to **Settings** > **Variables**
4. Add each secret under **Environment Variables** > **Encrypt**

### Deployment Commands

#### Deploy All Workers to Production

```bash
# From repository root
pnpm --filter "api-*" deploy:production
```

#### Deploy Individual Workers

```bash
# Deploy api-compliance
pnpm --filter api-compliance deploy:production

# Deploy api-backoffice
pnpm --filter api-backoffice deploy:production

# Deploy api-payroll
pnpm --filter api-payroll deploy:production

# Deploy api-jobs
pnpm --filter api-jobs deploy:production
```

#### Deploy to Staging

```bash
# All workers
pnpm --filter "api-*" deploy:staging

# Individual worker
pnpm --filter api-compliance deploy:staging
```

### Deployment Order

For first-time deployment, deploy in this order to ensure service bindings work:

1. **api-jobs** (no dependencies)
2. **api-payroll** (depends on api-jobs)
3. **api-compliance** (depends on api-jobs, api-payroll)
4. **api-backoffice** (depends on api-compliance, api-jobs)

```bash
pnpm --filter api-jobs deploy:production
pnpm --filter api-payroll deploy:production
pnpm --filter api-compliance deploy:production
pnpm --filter api-backoffice deploy:production
```

### Verifying Deployment

After deployment, verify each worker:

```bash
# Health checks
curl https://api.bumara.com/api/v1/health
curl https://api-backoffice.bumara.com/api/v1/health
curl https://api-payroll.bumara.com/api/v1/health
curl https://webhooks.bumara.com/api/v1/health
```

Expected response:
```json
{
  "status": "ok",
  "worker": "api-compliance",
  "timestamp": "2024-01-15T12:00:00.000Z"
}
```

### Rollback

To rollback to a previous version:

```bash
# Via Cloudflare Dashboard
# 1. Go to Workers & Pages > Your Worker
# 2. Click "Deployments" tab
# 3. Find the previous deployment
# 4. Click "Rollback to this deployment"

# Or redeploy from a previous git commit
git checkout <previous-commit>
pnpm --filter api-compliance deploy:production
```

---

## Environment Variables

### Required for All Workers

| Variable | Description | Example |
|----------|-------------|---------|
| `DATABASE_URL` | Neon PostgreSQL connection string | `postgresql://user:pass@host/db?sslmode=require` |
| `CLERK_SECRET_KEY` | Clerk backend API key | `sk_live_xxx` or `sk_test_xxx` |
| `CLERK_PUBLISHABLE_KEY` | Clerk frontend key | `pk_live_xxx` or `pk_test_xxx` |

### Required for api-backoffice

| Variable | Description | Example |
|----------|-------------|---------|
| `CLERK_INTERNAL_ORG_ID` | Internal backoffice organization ID | `org_xxx` |

### Optional (Feature-Specific)

| Variable | Worker(s) | Description |
|----------|-----------|-------------|
| `AWS_REGION` | api-compliance | S3 region for documents |
| `AWS_ACCESS_KEY_ID` | api-compliance | S3 access key |
| `AWS_SECRET_ACCESS_KEY` | api-compliance | S3 secret key |
| `AWS_BUCKET_NAME` | api-compliance | S3 bucket name |
| `STRIPE_SECRET_KEY` | api-compliance, api-jobs | Stripe API key |
| `SVIX_TOKEN` | api-jobs | Webhook verification |

### Wrangler Variables (Non-Secret)

These are set in `wrangler.toml` under `[vars]`:

```toml
[vars]
WORKER_NAME = "api-compliance"

[env.staging.vars]
WORKER_NAME = "api-compliance-staging"

[env.production.vars]
WORKER_NAME = "api-compliance"
```

---

## Troubleshooting

### Common Issues

#### 1. "CLERK_INTERNAL_ORG_ID not configured"

**Cause**: Backoffice worker missing required environment variable.

**Solution**:
```bash
cd apps/api-backoffice
wrangler secret put CLERK_INTERNAL_ORG_ID
# Enter your backoffice organization ID from Clerk dashboard
```

#### 2. Service Binding Errors

**Cause**: Workers deployed out of order or service not found.

**Solution**:
1. Verify all workers are deployed
2. Check wrangler.toml service names match deployed worker names
3. Redeploy the worker that has the binding

#### 3. CORS Errors

**Cause**: Frontend domain not in allowed origins list.

**Solution**: Add domain to `ALLOWED_ORIGINS` in `app.ts`:
```typescript
const ALLOWED_ORIGINS = [
  'https://your-new-domain.com',
  // ... existing origins
];
```

#### 4. Database Connection Errors

**Cause**: Invalid DATABASE_URL or network issues.

**Solution**:
1. Verify DATABASE_URL is set correctly
2. Ensure Neon database allows connections from Cloudflare IPs
3. Check connection string includes `?sslmode=require`

### Viewing Logs

#### Real-time Logs

```bash
# Tail logs for a specific worker
wrangler tail --env production

# Filter by status
wrangler tail --env production --status error
```

#### Cloudflare Dashboard

1. Go to **Workers & Pages** > **Your Worker**
2. Click **Logs** tab
3. View real-time or historical logs

### Debugging Locally

```bash
# Run worker locally
cd apps/api-compliance
pnpm dev

# Access at http://localhost:8787
# Runs with wrangler dev server
```

---

## Quick Reference

### API Endpoints

| Worker | Base URL | OpenAPI Docs |
|--------|----------|--------------|
| api-compliance | `https://api.bumara.com/api/v1` | `/api/v1/reference` |
| api-backoffice | `https://api-backoffice.bumara.com/api/v1` | `/api/v1/reference` |
| api-payroll | `https://api-payroll.bumara.com/api/v1` | `/api/v1/reference` |
| api-jobs | `https://webhooks.bumara.com/api/v1` | `/api/v1/reference` |

### Useful Commands

```bash
# Install dependencies
pnpm install

# Run all workers locally (separate terminals)
pnpm --filter api-compliance dev  # Port 8787
pnpm --filter api-backoffice dev  # Port 8788
pnpm --filter api-payroll dev     # Port 8789
pnpm --filter api-jobs dev        # Port 8790

# Type check all workers
pnpm --filter "api-*" typecheck

# Deploy all to production
pnpm --filter "api-*" deploy:production

# View worker status
wrangler deployments list
```
