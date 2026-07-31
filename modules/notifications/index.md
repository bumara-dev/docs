---
title: "Notifications Module"
description: "Production-grade in-house notification system for Bumara with multi-channel delivery, outbox pattern, and full tenant isolation."
---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Module Structure](#module-structure)
4. [Quick Start](#quick-start)
5. [Related Documentation](#related-documentation)

---

## Overview

The Notifications module provides a reliable, scalable notification system that supports:

- **In-App Notifications**: Real-time feed per user/tenant with read/unread tracking
- **Email**: Transactional emails via Resend with bounce/complaint handling
- **WhatsApp**: Business-initiated template messages via Meta WhatsApp Cloud API
- **SMS**: (Deferred) Gateway integration for urgent fallback

### Key Principles

| Principle | Implementation |
|-----------|----------------|
| **Outbox Pattern** | All notifications go through `event_outbox` table first |
| **Idempotency** | Unique constraints prevent duplicate deliveries |
| **Multi-Tenant** | Every query scoped by `tenant_id` from auth |
| **Audit Trail** | All state changes logged |
| **Retry with Backoff** | Failed deliveries retry with exponential backoff |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              EVENT SOURCES                                   │
├─────────────────┬─────────────────┬─────────────────┬───────────────────────┤
│   Tenant App    │   Backoffice    │   Scheduler     │   Domain Services     │
│   (mutations)   │   (actions)     │   (cron jobs)   │   (state changes)     │
└────────┬────────┴────────┬────────┴────────┬────────┴───────────┬───────────┘
         │                 │                 │                    │
         └─────────────────┴─────────────────┴────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           EVENT OUTBOX TABLE                                 │
│   (event_outbox: id, tenant_id, event_type, payload, status, created_at)    │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      CLOUDFLARE QUEUE: notification-outbox                   │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           OUTBOX PROCESSOR WORKER                            │
│   1. Resolve recipients (routing rules)                                      │
│   2. Create notifications rows (in-app feed)                                 │
│   3. Create notification_deliveries rows (email/whatsapp)                    │
│   4. Enqueue to delivery queue                                               │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      CLOUDFLARE QUEUE: notification-delivery                 │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DELIVERY SENDER WORKER                              │
│   1. Fetch delivery row                                                      │
│   2. Call channel adapter (Email/WhatsApp)                                   │
│   3. Update status to SENT                                                   │
│   4. Store provider message ID                                               │
└──────────────┬──────────────────────────────────────────────┬───────────────┘
               │                                              │
               ▼                                              ▼
┌──────────────────────────────┐              ┌──────────────────────────────┐
│       EMAIL ADAPTER          │              │      WHATSAPP ADAPTER        │
│       (Resend API)           │              │   (Meta Cloud API)           │
└──────────────┬───────────────┘              └──────────────┬───────────────┘
               │                                              │
               ▼                                              ▼
┌──────────────────────────────┐              ┌──────────────────────────────┐
│   /webhooks/email            │              │   /webhooks/whatsapp         │
│   (bounces, complaints)      │              │   (delivered, read, failed)  │
└──────────────┬───────────────┘              └──────────────┬───────────────┘
               │                                              │
               └──────────────────────┬───────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       notification_deliveries TABLE                          │
│   (status updated: SENT → DELIVERED | FAILED | BOUNCED)                     │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Module Structure

```
packages/
├── notifications/           # Frontend components + hooks
│   ├── components/
│   │   ├── notification-bell.tsx
│   │   ├── notification-center.tsx
│   │   └── notification-item.tsx
│   ├── hooks/
│   │   └── use-notifications.ts
│   ├── adapters/           # Channel adapters
│   │   ├── email.adapter.ts
│   │   ├── whatsapp.adapter.ts
│   │   └── types.ts
│   └── templates/
│       └── whatsapp-templates.ts
│
├── api-services/src/domains/notifications/
│   ├── events.ts           # Event catalog + Zod schemas
│   ├── routing.ts          # Recipient + channel resolution
│   ├── notifications.service.ts
│   └── preferences.service.ts
│
├── backend/src/
│   ├── modules/notifications/
│   │   ├── notifications.routes.ts
│   │   ├── notifications.handlers.ts
│   │   └── preferences.routes.ts
│   ├── queues/
│   │   ├── outbox-processor.ts
│   │   └── delivery-sender.ts
│   ├── jobs/
│   │   ├── scheduler.ts
│   │   ├── filing-reminders.ts
│   │   └── task-reminders.ts
│   └── modules/webhooks/
│       ├── whatsapp-webhook.handlers.ts
│       └── email-webhook.handlers.ts
│
└── database/src/schema/notifications/
    ├── event-outbox.ts
    ├── notifications.ts
    ├── notification-deliveries.ts
    ├── notification-preferences.ts
    └── enums.ts

docs/modules/notifications/
├── README.md               # This file
├── data-model.md           # Database schema details
├── events-catalog.md       # Event types and payloads
├── routing.md              # Recipient/channel routing rules
├── workers.md              # Outbox processor + delivery sender
├── adapters.md             # Email + WhatsApp adapters
├── webhooks.md             # Delivery receipt handlers
├── scheduler.md            # Cron jobs for reminders
├── api.md                  # API endpoints
├── ui.md                   # Frontend components
├── security-permissions.md # Security considerations
├── provider-setup.md       # Resend + Meta setup
└── runbook.md              # Operations + troubleshooting
```

---

## Quick Start

### 1. Database Migration

```bash
cd packages/database
pnpm drizzle-kit generate
pnpm drizzle-kit migrate
```

### 2. Environment Variables

```bash
# Email (Resend)
RESEND_TOKEN=re_xxxxx
RESEND_FROM=notifications@bumara.com

# WhatsApp (Meta)
WHATSAPP_PHONE_NUMBER_ID=xxxxx
WHATSAPP_ACCESS_TOKEN=xxxxx
WHATSAPP_VERIFY_TOKEN=xxxxx
WHATSAPP_WEBHOOK_SECRET=xxxxx

# Cloudflare Queues (auto-configured via wrangler)
```

### 3. Deploy Workers

```bash
cd packages/backend
pnpm wrangler deploy
```

---

## Related Documentation

| Document | Description |
|----------|-------------|
| [Data Model](/modules/notifications/data-model) | Database schema and migrations |
| [Events Catalog](/modules/notifications/events-catalog) | Event types and payload schemas |
| [Routing](/modules/notifications/routing) | Recipient and channel resolution |
| [Workers](/modules/notifications/workers) | Outbox processor and delivery sender |
| [Adapters](/modules/notifications/adapters) | Email and WhatsApp adapters |
| [Webhooks](/modules/notifications/webhooks) | Delivery receipt handlers |
| [Scheduler](/modules/notifications/scheduler) | Cron jobs for reminders |
| [API](/modules/notifications/api) | REST endpoints |
| [UI](/modules/notifications/ui) | Frontend components |
| [Security](/modules/notifications/security-permissions) | Security and permissions |
| [Provider Setup](/modules/notifications/provider-setup) | Resend and Meta configuration |
| [Runbook](/modules/notifications/runbook) | Operations and troubleshooting |

