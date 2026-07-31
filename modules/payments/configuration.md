---
title: "Configuration"
description: "Configure price IDs for each plan and billing period:"
---

## Environment Variables

### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `PAYMENT_PROVIDER` | Active payment provider | `stripe`, `lenco`, `mock` |
| `PAYMENT_API_KEY` | Provider API key | `sk_live_...` |

### Optional Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `PAYMENT_WEBHOOK_SECRET` | Webhook signing secret | - |
| `STRIPE_SECRET_KEY` | Stripe-specific key (fallback) | - |
| `STRIPE_WEBHOOK_SECRET` | Stripe-specific webhook secret | - |
| `LENCO_API_KEY` | Lenco-specific key (fallback) | - |
| `LENCO_WEBHOOK_SECRET` | Lenco-specific webhook secret | - |

## Provider Configuration

### Stripe

```env
PAYMENT_PROVIDER=stripe
PAYMENT_API_KEY=sk_live_xxxxxxxxxxxxxxxxxxxxx
PAYMENT_WEBHOOK_SECRET=whsec_xxxxxxxxxxxxxxxxxxxxx

# Optional: Stripe-specific fallbacks
STRIPE_SECRET_KEY=sk_live_xxxxxxxxxxxxxxxxxxxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxxxxxxxxxxxxxxxxxxx
```

#### Stripe Price IDs

Configure price IDs for each plan and billing period:

```env
# Monthly prices
STRIPE_PRICE_START_MONTHLY=price_xxxxxx
STRIPE_PRICE_PLUS_MONTHLY=price_xxxxxx
STRIPE_PRICE_PRO_MONTHLY=price_xxxxxx
STRIPE_PRICE_ENTERPRISE_MONTHLY=price_xxxxxx

# Yearly prices
STRIPE_PRICE_START_YEARLY=price_xxxxxx
STRIPE_PRICE_PLUS_YEARLY=price_xxxxxx
STRIPE_PRICE_PRO_YEARLY=price_xxxxxx
STRIPE_PRICE_ENTERPRISE_YEARLY=price_xxxxxx
```

### Lenco

```env
PAYMENT_PROVIDER=lenco
PAYMENT_API_KEY=lenco_live_xxxxxxxxxxxxxxxxxxxxx
PAYMENT_WEBHOOK_SECRET=lenco_whsec_xxxxxxxxxxxxxxxxxxxxx

# Optional: Lenco-specific fallbacks
LENCO_API_KEY=lenco_live_xxxxxxxxxxxxxxxxxxxxx
LENCO_WEBHOOK_SECRET=lenco_whsec_xxxxxxxxxxxxxxxxxxxxx
```

### Mock (Testing)

```env
PAYMENT_PROVIDER=mock
# No API key needed for mock provider
```

## Configuration Module

The configuration is validated using Zod:

```typescript
// packages/payments/config.ts

import { createEnv } from "@t3-oss/env-nextjs";
import { z } from "zod";

export const config = () =>
  createEnv({
    server: {
      PAYMENT_PROVIDER: z.enum(["stripe", "lenco", "mock"]).default("stripe"),
      PAYMENT_API_KEY: z.string().min(1).optional(),
      PAYMENT_WEBHOOK_SECRET: z.string().optional(),

      // Provider-specific (fallbacks)
      STRIPE_SECRET_KEY: z.string().startsWith("sk_").optional(),
      STRIPE_WEBHOOK_SECRET: z.string().startsWith("whsec_").optional(),
      LENCO_API_KEY: z.string().optional(),
      LENCO_WEBHOOK_SECRET: z.string().optional(),
    },
    runtimeEnv: {
      PAYMENT_PROVIDER: process.env.PAYMENT_PROVIDER,
      PAYMENT_API_KEY: process.env.PAYMENT_API_KEY,
      PAYMENT_WEBHOOK_SECRET: process.env.PAYMENT_WEBHOOK_SECRET,
      STRIPE_SECRET_KEY: process.env.STRIPE_SECRET_KEY,
      STRIPE_WEBHOOK_SECRET: process.env.STRIPE_WEBHOOK_SECRET,
      LENCO_API_KEY: process.env.LENCO_API_KEY,
      LENCO_WEBHOOK_SECRET: process.env.LENCO_WEBHOOK_SECRET,
    },
  });
```

## Helper Functions

### getApiKey

Returns the appropriate API key for the current provider:

```typescript
import { getApiKey } from "@repo/payments";

const apiKey = getApiKey();
// Returns PAYMENT_API_KEY, or falls back to provider-specific key
```

### getWebhookSecret

Returns the webhook secret for the current provider:

```typescript
import { getWebhookSecret } from "@repo/payments";

const secret = getWebhookSecret();
// Returns PAYMENT_WEBHOOK_SECRET, or falls back to provider-specific secret
```

## Environment Setup

### Development

Create `.env.local` for local development:

```env
# Use test/sandbox credentials
PAYMENT_PROVIDER=stripe
PAYMENT_API_KEY=sk_test_xxxxxxxxxxxxxxxxxxxxx
PAYMENT_WEBHOOK_SECRET=whsec_xxxxxxxxxxxxxxxxxxxxx
```

For testing without external calls:

```env
PAYMENT_PROVIDER=mock
```

### Staging

```env
PAYMENT_PROVIDER=stripe
PAYMENT_API_KEY=sk_test_xxxxxxxxxxxxxxxxxxxxx  # Still test mode
PAYMENT_WEBHOOK_SECRET=whsec_xxxxxxxxxxxxxxxxxxxxx
```

### Production

```env
PAYMENT_PROVIDER=stripe  # or 'lenco'
PAYMENT_API_KEY=sk_live_xxxxxxxxxxxxxxxxxxxxx
PAYMENT_WEBHOOK_SECRET=whsec_xxxxxxxxxxxxxxxxxxxxx
```

## Switching Providers

To switch from Stripe to Lenco:

1. **Configure Lenco credentials:**
```env
LENCO_API_KEY=lenco_live_xxxxx
LENCO_WEBHOOK_SECRET=lenco_whsec_xxxxx
```

2. **Update provider selection:**
```env
PAYMENT_PROVIDER=lenco
PAYMENT_API_KEY=lenco_live_xxxxx
PAYMENT_WEBHOOK_SECRET=lenco_whsec_xxxxx
```

3. **Configure Lenco webhook endpoint:**
   - Add `https://api.bumara.com/webhooks/payments` in Lenco dashboard
   - Enable required events

4. **Deploy:**
   - No code changes required
   - Gateway factory automatically loads Lenco adapter

## Stripe Dashboard Configuration

### Products and Prices

Create products and prices in Stripe Dashboard:

1. Go to Products
2. Create product for each plan:
   - Name: "Bumara Start", "Bumara Plus", etc.
   - Add monthly and yearly prices
3. Note price IDs for environment variables

### Webhook Endpoint

1. Go to Developers → Webhooks
2. Add endpoint: `https://api.bumara.com/webhooks/payments`
3. Select events:
   - `checkout.session.completed`
   - `checkout.session.expired`
   - `customer.subscription.*`
   - `payment_intent.*`
   - `charge.refunded`
   - `invoice.*`
4. Save and copy signing secret

### Customer Portal (Optional)

1. Go to Settings → Billing → Customer portal
2. Configure allowed actions:
   - Update payment method
   - Cancel subscription
   - View invoices
3. Save portal link for integration

## Validation

### Startup Validation

Configuration is validated at module load time:

```typescript
// Throws if required variables are missing
const settings = config();

// Check specific values
if (settings.PAYMENT_PROVIDER === 'stripe' && !settings.PAYMENT_API_KEY) {
  throw new Error('PAYMENT_API_KEY required for Stripe provider');
}
```

### Runtime Checks

```typescript
import { getPaymentGateway } from "@repo/payments";

try {
  const gateway = await getPaymentGateway();
  // Configuration is valid, gateway is ready
} catch (error) {
  // Configuration error (missing API key, etc.)
  console.error('Payment gateway configuration error:', error);
}
```

## Security Best Practices

### API Keys

- Never commit API keys to version control
- Use environment variables or secrets manager
- Rotate keys periodically
- Use test keys in non-production environments

### Webhook Secrets

- Always verify webhook signatures
- Use separate secrets for each environment
- Rotate secrets when compromised

### Access Control

```typescript
// Webhook endpoint should be public (no auth)
// but signature verification provides security

// Admin endpoints should require authentication
export async function POST(request: Request) {
  // Verify admin access
  const session = await getServerSession();
  if (!session?.user?.isAdmin) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // Safe to proceed
}
```

## Troubleshooting

### Common Issues

#### "PAYMENT_API_KEY is required"

```
Error: PAYMENT_API_KEY is required for stripe provider
```

**Solution:** Set the `PAYMENT_API_KEY` environment variable with your Stripe secret key.

#### "Invalid webhook signature"

```
Error: Webhook signature verification failed
```

**Solution:**
- Verify `PAYMENT_WEBHOOK_SECRET` matches the signing secret from Stripe Dashboard
- Ensure raw request body is used for verification (not parsed JSON)
- Check that the correct header is being read (`stripe-signature`)

#### "Unknown payment provider"

```
Error: Unknown payment provider: xyz
```

**Solution:** Set `PAYMENT_PROVIDER` to one of: `stripe`, `lenco`, `mock`

### Debug Mode

Enable debug logging:

```typescript
// In development, add detailed logging
if (process.env.NODE_ENV === 'development') {
  console.log('Payment config:', {
    provider: config().PAYMENT_PROVIDER,
    hasApiKey: !!config().PAYMENT_API_KEY,
    hasWebhookSecret: !!config().PAYMENT_WEBHOOK_SECRET,
  });
}
```
