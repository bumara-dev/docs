---
title: "Bumara Architecture Documentation"
description: "Technical architecture documentation for the Bumara platform."
---

## Overview

Bumara is a **Zambian business compliance operations platform** with multiple integrated apps: Compliance, Documents, Invoicing, and Payroll.

### Tech Stack

| Layer | Technology |
|-------|------------|
| Frontend | Next.js 16+ (App Router), React Query, TailwindCSS, ShadCN |
| Backend | Hono.js (RPC architecture) on Cloudflare Workers |
| Database | PostgreSQL (Neon) + Drizzle ORM |
| Auth | Clerk (multi-tenant) |
| Storage | AWS S3 (presigned URLs) |
| Deploy | Vercel (Frontend) + Cloudflare (Backend) |

---

## Documentation Index

### Core Architecture

| Document | Description |
|----------|-------------|
| [DATABASE_SCHEMA.md](/ARCHITECTURE/DATABASE_SCHEMA) | Complete database schema with tables, relations, and patterns |
| [NAPSA_INTEGRATION.md](/ARCHITECTURE/NAPSA_INTEGRATION) | NAPSA regulator integration details |

### Module Documentation

| Module | Description | Documentation |
|--------|-------------|---------------|
| **Documents** | File storage with AWS S3, presigned URLs, audit trail | [📁 modules/documents/](/modules/documents) |
| **Tasks** | Compliance task management for filings and service requests | [📁 modules/tasks/](/modules/tasks) |
| **Notifications** | Event-driven notifications with Knock Labs | [📁 modules/notifications/](/modules/notifications) |
| **Subscriptions** | Subscription plans and billing management | [📁 modules/subscriptions/](/modules/subscriptions) |
| **Payments** | Payment processing with Stripe | [📁 modules/payments/](/modules/payments) |
| **Inventory** | Stock tracking, ledger, adjustments, transfers, counts | [📁 inventory/](/inventory) |

### Architecture Decision Records (ADRs)

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-0001](/adr/0001-documents-s3-presigned-urls) | Documents Storage with S3 Presigned URLs | Accepted |

### Setup & Operations

| Document | Description |
|----------|-------------|
| [API-SETUP.md](/API-SETUP) | Hono backend setup with Neon, ngrok, Clerk |
| [SETUP.md](/SETUP) | Local development environment setup |
| [ONBOARDING.md](/ONBOARDING) | Developer onboarding guide |

### Product

| Document | Description |
|----------|-------------|
| [PRD](/PRODUCT/prd) | Product Requirements Document |
| [Personas](/PRODUCT/personas) | User personas |

---

## Key Principles

### 1. Multi-Tenant Isolation

Every database query filters by `organization_id` derived from the authenticated Clerk session. Never accept `org_id` from client request body.

### 2. Compliance-First Design

- Audit everything (state transitions, document access)
- Immutability for accepted filings
- Evidence documents are locked automatically

### 3. Money Must Be Exact

- Store amounts as integer cents or `NUMERIC(18,2)`
- Never use JavaScript floats for money math
- Round to 2 decimal places

### 4. Mobile-First Performance

- Paginate all lists by default
- Prefer server components + streaming
- Keep bundle size small

---

## Repository Structure

```
bumara/
├── apps/
│   ├── app/           # Tenant-facing Next.js app
│   ├── backoffice/    # Backoffice console
│   ├── api/           # API documentation site
│   └── web/           # Marketing website
├── packages/
│   ├── backend/       # Hono RPC routes (Cloudflare Worker)
│   ├── api-services/  # Business logic services
│   ├── database/      # Drizzle schema + migrations
│   ├── design-system/ # ShadCN-based UI components
│   ├── auth/          # Clerk auth utilities
│   └── storage/       # Storage abstraction
└── docs/
    ├── ARCHITECTURE/  # This folder
    ├── PRODUCT/       # Product documentation
    ├── modules/       # Per-module documentation
    └── adr/           # Architecture Decision Records
```

---

## Related Resources

- [Cursor Rules](https://github.com/bumara-dev/bumara/tree/main/.cursor/rules) - AI-assisted development guidelines
- Development Workflow - Git workflow and PR process
- Quick Reference - Common commands and patterns

