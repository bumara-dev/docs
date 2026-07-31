---
title: "Environment Setup"
description: "Configuration for backoffice authentication and authorization."
---

This guide covers all environment variables and Clerk dashboard configuration required for the backoffice.

---

## Environment Variables

### Backoffice App (`apps/backoffice`)

Create or update `.env.local`:

```bash
# ============================================
# Clerk Authentication
# ============================================

# Clerk secret key (from Clerk Dashboard > API Keys)
CLERK_SECRET_KEY=sk_test_xxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Clerk publishable key (from Clerk Dashboard > API Keys)
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_xxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# ============================================
# Backoffice Access Control
# ============================================

# Internal organization ID for backoffice staff
# Get this from Clerk Dashboard > Organizations > Your backoffice org
CLERK_INTERNAL_ORG_ID=org_xxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Allowed email domains (comma-separated)
# Only users with these email domains can access backoffice
CLERK_ALLOWED_BACKOFFICE_DOMAINS=@bumara.com,@bumara.co.zm

# ============================================
# MFA Configuration
# ============================================

# Set to "true" to require MFA for all backoffice users (default: true)
# Set to "false" to disable mandatory MFA (not recommended for production)
REQUIRE_MFA=true

# ============================================
# API Connection
# ============================================

# Backend API URL
NEXT_PUBLIC_API_URL=http://localhost:8787

# ============================================
# Security
# ============================================

# Arcjet protection
ARCJET_KEY=ajkey_xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ARCJET_ENV=development

# Feature flags secret
FLAGS_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### Backend API (`packages/backend`)

Required in Cloudflare Workers environment:

```bash
# Clerk Authentication
CLERK_SECRET_KEY=sk_test_xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
CLERK_PUBLISHABLE_KEY=pk_test_xxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Backoffice organization ID (must match app config)
CLERK_INTERNAL_ORG_ID=org_xxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Webhook signing secret (for Clerk webhooks)
CLERK_WEBHOOK_SIGNING_SECRET=whsec_xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

---

## Clerk Dashboard Configuration

### Step 1: Create Backoffice Organization

1. Go to **Clerk Dashboard** → **Organizations**
2. Click **Create Organization**
3. Configure:
   - **Name:** `Bumara Internal` (or similar)
   - **Slug:** `bumara-internal`
4. Copy the **Organization ID** (starts with `org_`)
5. Set as `CLERK_INTERNAL_ORG_ID` in environment

### Step 2: Configure Roles

Clerk provides default roles. You can use these directly:

**Using Default Roles (Recommended):**

| Role | Clerk Key | Use For |
|------|-----------|---------|
| Admin | `org:admin` | Full backoffice access |
| Member | `org:member` | Basic staff access |

**Optional: Add Manager Role:**

1. Go to **Organizations** → **Roles & Permissions**
2. Click **Create Role**
3. Configure:
   - **Name:** Manager
   - **Key:** `org:manager`
4. Assign appropriate permissions

**Optional: Create Backoffice-Specific Roles:**

If you need to distinguish backoffice from tenant roles:

1. Create roles:
   - `org:backoffice_admin`
   - `org:backoffice_manager`
   - `org:backoffice_member`
2. Assign permissions to each

### Step 3: Add Staff Members

Since there's no public sign-up, add users directly:

1. Go to **Users** in Clerk Dashboard
2. Click **Create User** or use **Invite**
3. Enter staff email (must match `CLERK_ALLOWED_BACKOFFICE_DOMAINS`)
4. Set password
5. Go to **Organizations** → Select backoffice org
6. Click **Add Member** → Select user → Assign role

### Step 4: Configure MFA Settings

1. Go to **User & Authentication** → **Multi-factor**
2. Enable **Authenticator application (TOTP)**
3. Optionally enable **Backup codes**
4. **Do NOT** enable "Require MFA" globally in Clerk
   - The app enforces MFA via `MfaGuard` after sign-in
   - Global enforcement blocks MFA setup flow

### Step 5: Configure Session Settings

1. Go to **Sessions** in Clerk Dashboard
2. Recommended settings:
   - **Session lifetime:** 7 days
   - **Inactivity timeout:** 1 hour
   - **Single session mode:** Optional (enable for extra security)

---

## Environment Validation

The app validates environment variables at startup:

**File:** `apps/backoffice/env.ts`

```typescript
import { createEnv } from "@t3-oss/env-nextjs";
import { z } from "zod";

export const env = createEnv({
  server: {
    CLERK_SECRET_KEY: z.string(),
    CLERK_INTERNAL_ORG_ID: z.string(),
    CLERK_ALLOWED_BACKOFFICE_DOMAINS: z.string(),
    FLAGS_SECRET: z.string(),
    ARCJET_KEY: z.string(),
    ARCJET_ENV: z.enum(["development", "production"]),
    REQUIRE_MFA: z
      .string()
      .default("true")
      .transform((val) => val === "true"),
  },
  client: {
    NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY: z.string(),
    NEXT_PUBLIC_API_URL: z.string(),
  },
  runtimeEnv: {
    CLERK_SECRET_KEY: process.env.CLERK_SECRET_KEY,
    CLERK_INTERNAL_ORG_ID: process.env.CLERK_INTERNAL_ORG_ID,
    CLERK_ALLOWED_BACKOFFICE_DOMAINS: process.env.CLERK_ALLOWED_BACKOFFICE_DOMAINS,
    ARCJET_KEY: process.env.ARCJET_KEY,
    FLAGS_SECRET: process.env.FLAGS_SECRET,
    ARCJET_ENV: process.env.ARCJET_ENV,
    REQUIRE_MFA: process.env.REQUIRE_MFA,
    NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY: process.env.NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY,
    NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL,
  },
});
```

If environment variables are missing or invalid, the app will fail to start with clear error messages.

---

## CORS Configuration

The backend API only accepts requests from allowed origins:

**File:** `packages/backend/src/core/http/create-app.ts`

```typescript
const ALLOWED_ORIGINS = [
  // Production
  "https://app.bumara.com",
  "https://backoffice.bumara.com",
  "https://bumara.com",
  // Vercel previews
  /^https:\/\/bumara-.*\.vercel\.app$/,
  // Development
  "http://localhost:3000",
  "http://localhost:3001",
  "http://localhost:3003",
];
```

Add your development/preview URLs as needed.

---

## Webhook Configuration

For syncing Clerk data to the database:

1. Go to **Clerk Dashboard** → **Webhooks**
2. Create new webhook endpoint
3. URL: `https://your-api-domain/webhooks/clerk`
4. Select events:
   - `user.created`
   - `user.updated`
   - `user.deleted`
   - `organization.created`
   - `organization.updated`
   - `organizationMembership.created`
   - `organizationMembership.updated`
   - `organizationMembership.deleted`
5. Copy **Signing Secret** → Set as `CLERK_WEBHOOK_SIGNING_SECRET`

---

## Local Development

### Quick Start

1. Copy environment template:
   ```bash
   cd apps/backoffice
   cp .env.example .env.local
   ```

2. Fill in Clerk credentials from dashboard

3. Get organization ID:
   - Create org in Clerk Dashboard
   - Copy org ID to `CLERK_INTERNAL_ORG_ID`

4. Add yourself as staff member in Clerk

5. Start development server:
   ```bash
   pnpm dev
   ```

### Testing MFA Flow

1. Set `REQUIRE_MFA=true`
2. Sign in with a user that has no MFA
3. Should redirect to `/setup-mfa`
4. Complete setup wizard
5. Should redirect to dashboard

### Disabling MFA for Development

```bash
REQUIRE_MFA=false
```

> **Warning:** Never disable MFA in production.

---

## Production Checklist

Before deploying to production:

- [ ] Use production Clerk keys (not test keys)
- [ ] Set `REQUIRE_MFA=true`
- [ ] Verify `CLERK_INTERNAL_ORG_ID` is correct
- [ ] Configure allowed domains correctly
- [ ] Set up webhook endpoint
- [ ] Verify CORS origins include production domains
- [ ] Enable Arcjet protection
- [ ] Review session settings in Clerk

---

## Next Steps

- [Development Guide](/backoffice/backoffice-auth/development-guide) - Protect new routes
- [Troubleshooting](/backoffice/backoffice-auth/troubleshooting) - Common issues

