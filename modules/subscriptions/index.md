---
title: "Subscriptions & Pricing Module"
description: "This module implements comprehensive pricing plans, feature gates, and usage tracking for Bumara. It enables subscription-based access control with..."
---

This module implements comprehensive pricing plans, feature gates, and usage tracking for Bumara. It enables subscription-based access control with per-feature limits and Stripe integration for billing.

## Overview

The subscription system provides:
- **Plan-based feature gating** - Control access to features based on subscription tier
- **Usage tracking** - Monitor service requests, storage, AI credits, and obligations
- **Server-side enforcement** - Validate limits at API boundaries
- **Stripe integration** - Handle checkouts, plan changes, and webhooks
- **Audit logging** - Track all subscription changes for compliance

## Plan Tiers

| Tier | Monthly Price | Obligations | Service Requests | Users | Storage | Regulators |
|------|---------------|-------------|------------------|-------|---------|------------|
| Start | ZMW 1,500 | 3 | 2/month | 1 | 500 MB | ZRA, NAPSA, PACRA |
| Plus | ZMW 3,500 | 4 | 3/month | 2 | 1 GB | + NHIMA, ZPPA |
| Pro | ZMW 7,500 | 7 | 5/month | 5 | 5 GB | + WCF, Local Councils |
| Enterprise | Custom | Unlimited | Unlimited | Unlimited | Custom | All |

## Add-ons

| Add-on | Price | Availability |
|--------|-------|--------------|
| Payroll | ZMW 85/employee | Plus, Pro |
| Extra Storage | ZMW 150/mo per 10 GB | Start, Plus, Pro |
| SMS Notifications | ZMW 100/mo | All tiers |
| Smart Invoicing | Coming soon | Plus, Pro |
| Inventory | Coming soon | Pro |
| AI Assistance | Coming soon | Pro |

## Quick Links

- [Architecture](/modules/subscriptions/architecture)
- [Data Model](/modules/subscriptions/data-model)
- [API Reference](/modules/subscriptions/api)
- [Enforcement](/modules/subscriptions/enforcement)
- [Webhooks](/modules/subscriptions/webhooks)
- [Frontend Integration](/modules/subscriptions/frontend)
- [Configuration](/modules/subscriptions/configuration)
- [Runbook](/modules/subscriptions/runbook)
