---
title: "Backoffice Authentication & Authorization"
description: "Comprehensive security documentation for the Bumara backoffice console."
---

The backoffice is an internal application used by Bumara staff to manage tenant compliance operations, payouts, and submissions. It requires robust authentication and authorization to protect sensitive business data.

---

## Quick Navigation

| Document | Description |
|----------|-------------|
| [Architecture](/backoffice/backoffice-auth/architecture) | Security layers, defense in depth, component hierarchy |
| [Authentication Flow](/backoffice/backoffice-auth/authentication-flow) | Sign-in process, MFA setup, session management |
| [Authorization](/backoffice/backoffice-auth/authorization) | Roles, permissions, middleware reference |
| [Environment Setup](/backoffice/backoffice-auth/environment-setup) | Environment variables, Clerk dashboard config |
| [Development Guide](/backoffice/backoffice-auth/development-guide) | Code examples for protecting routes and components |
| [Troubleshooting](/backoffice/backoffice-auth/troubleshooting) | Common issues and solutions |

---

## Overview

### Two Realms

The Bumara platform operates with two distinct realms, each with separate security contexts:

| Realm | Description | Application | Users |
|-------|-------------|-------------|-------|
| **Tenant** | Customer organizations managing their compliance | `apps/app` | Business users |
| **Backoffice** | Internal Bumara staff console | `apps/backoffice` | Bumara employees |

### Key Security Principles

1. **Defense in Depth** - Multiple security layers (middleware, guards, API checks)
2. **Zero Trust** - Every request is verified at each layer
3. **Invite-Only Access** - No public sign-up; admins create users in Clerk
4. **Mandatory MFA** - Two-factor authentication required for all staff
5. **Instant Revocation** - Staff can be deactivated without Clerk changes
6. **Audit Everything** - All sensitive actions logged with actor type

---

## Getting Started Checklist

For new developers joining the project:

- [ ] **Understand the architecture** - Read [Architecture](/backoffice/backoffice-auth/architecture)
- [ ] **Set up environment** - Follow [Environment Setup](/backoffice/backoffice-auth/environment-setup)
- [ ] **Get backoffice access** - Request invitation from admin
- [ ] **Configure MFA** - Complete 2FA setup on first login
- [ ] **Learn the patterns** - Review [Development Guide](/backoffice/backoffice-auth/development-guide)

For adding new backoffice features:

- [ ] Apply correct middleware chain to API routes
- [ ] Use server component guards in layouts
- [ ] Add audit logging for state changes
- [ ] Test with different role levels

---

## Technology Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| Identity Provider | [Clerk](https://clerk.com) | Authentication, MFA, organizations |
| Frontend | Next.js 16+ (App Router) | Server components, middleware |
| Backend | Hono.js | API routes with middleware chain |
| Database | PostgreSQL + Drizzle | Staff records, audit logs |

---

## Key Implementation Files

| Component | Location |
|-----------|----------|
| Next.js Middleware | `apps/backoffice/proxy.ts` |
| Auth Helpers | `packages/auth/helpers.ts` |
| API Middleware | `packages/backend/src/core/middleware/auth.ts` |
| Sign-In Form | `apps/backoffice/components/auth/sign-in-form.tsx` |
| MFA Guard | `apps/backoffice/components/shared/auth/mfa-guard.tsx` |
| MFA Setup Page | `apps/backoffice/app/(mfa-setup)/setup-mfa/page.tsx` |
| Auth Guard | `apps/backoffice/components/shared/auth/auth-guard.tsx` |
| Backoffice Guard | `apps/backoffice/components/shared/organization/backoffice-org-guard.tsx` |
| Authenticated Layout | `apps/backoffice/app/(authenticated)/layout.tsx` |
| Environment Config | `apps/backoffice/env.ts` |

---

## Related Documentation

- Security Audit Report - Historical security audit
- [Clerk Documentation](https://clerk.com/docs) - Official Clerk docs
- [Hono Clerk Integration](https://clerk.com/docs/references/hono/overview) - Backend auth

---

## Support

Having issues with backoffice access?

1. Check [Troubleshooting](/backoffice/backoffice-auth/troubleshooting) for common solutions
2. Contact your admin if you need role changes
3. Reach out to engineering for technical issues

