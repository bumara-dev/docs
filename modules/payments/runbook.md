---
title: "Runbook"
description: "This runbook covers common operational tasks and troubleshooting for the payments module."
---

## Operations Guide

This runbook covers common operational tasks and troubleshooting for the payments module.

## Monitoring

### Key Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| Payment success rate | % of successful payments | &lt; 95% |
| Webhook delivery rate | % of webhooks received | &lt; 99% |
| Average processing time | Webhook handler latency | > 5s |
| Failed payment count | Failed payments per hour | > 10/hr |

### Health Checks

```bash
# Check payment service status
curl https://api.bumara.com/health/payments

# Expected response
{
  "status": "healthy",
  "provider": "stripe",
  "lastWebhook": "2024-01-15T10:30:00Z"
}
```

### Dashboard Queries

#### Recent Payment Activity

```sql
-- Payment requests in last 24 hours
SELECT
  status,
  COUNT(*) as count,
  SUM(total_amount) as total_amount
FROM payment_requests
WHERE created_at > NOW() - INTERVAL '24 hours'
GROUP BY status;
```

#### Failed Payments

```sql
-- Failed payments requiring attention
SELECT
  pr.id,
  pr.organization_id,
  pr.total_amount,
  pr.currency,
  pr.created_at,
  o.name as org_name
FROM payment_requests pr
JOIN organizations o ON o.id = pr.organization_id
WHERE pr.status = 'required_pending'
  AND pr.created_at < NOW() - INTERVAL '48 hours'
ORDER BY pr.created_at DESC;
```

#### Webhook Processing Issues

```sql
-- Payments with external ID but not verified
SELECT
  id,
  organization_id,
  status,
  external_payment_id,
  created_at
FROM payment_requests
WHERE external_payment_id IS NOT NULL
  AND status NOT IN ('paid_platform_verified', 'completed')
ORDER BY created_at DESC;
```

## Common Tasks

### View Payment Request Details

```sql
SELECT
  pr.*,
  o.name as org_name,
  f.id as filing_id,
  sr.id as service_request_id
FROM payment_requests pr
JOIN organizations o ON o.id = pr.organization_id
LEFT JOIN filings f ON f.id = pr.filing_id
LEFT JOIN service_requests sr ON sr.id = pr.service_request_id
WHERE pr.id = 'pr_xxx';
```

### View Subscription Details

```sql
SELECT
  s.*,
  o.name as org_name
FROM subscriptions s
JOIN organizations o ON o.id = s.organization_id
WHERE s.id = 'sub_xxx';
```

### Check Provider Sync

```bash
# Stripe: View payment intent
stripe payment_intents retrieve pi_xxx

# Stripe: View subscription
stripe subscriptions retrieve sub_xxx

# Stripe: View recent events
stripe events list --limit 10
```

## Troubleshooting

### Payment Not Verified After Checkout

**Symptoms:**
- Customer completed checkout
- Payment shows in provider dashboard
- `payment_requests.status` still `pending_gateway`

**Diagnosis:**

1. Check webhook delivery:
```bash
# Stripe Dashboard: Developers → Webhooks → Recent Events
# Look for payment_intent.succeeded event
```

2. Check webhook logs:
```sql
-- Application logs
SELECT * FROM logs
WHERE message LIKE '%webhook%'
  AND created_at > NOW() - INTERVAL '1 hour'
ORDER BY created_at DESC;
```

3. Check for missing metadata:
```bash
# View payment intent metadata
stripe payment_intents retrieve pi_xxx --expand metadata
```

**Resolution:**

If webhook was missed, manually update:

```sql
-- Only if confirmed payment in provider dashboard
UPDATE payment_requests
SET
  status = 'paid_platform_verified',
  paid_at = NOW(),
  verified_at = NOW(),
  external_payment_id = 'pi_xxx',
  payment_provider = 'stripe',
  updated_at = NOW()
WHERE id = 'pr_xxx';

-- Close associated ticket
UPDATE tickets
SET
  status = 'resolved',
  resolved_at = NOW(),
  updated_at = NOW()
WHERE payment_request_id = 'pr_xxx';
```

### Subscription Not Activated

**Symptoms:**
- Customer completed checkout
- Subscription shows active in Stripe
- `subscriptions.status` still `trial` or `incomplete`

**Diagnosis:**

1. Check checkout session:
```bash
stripe checkout sessions retrieve cs_xxx
```

2. Verify metadata:
```bash
# Check organizationId in metadata
stripe checkout sessions retrieve cs_xxx --expand metadata
```

**Resolution:**

```sql
-- Update subscription record
UPDATE subscriptions
SET
  status = 'active',
  external_subscription_id = 'sub_xxx',
  external_customer_id = 'cus_xxx',
  payment_provider = 'stripe',
  updated_at = NOW()
WHERE organization_id = 'org_xxx';
```

### Duplicate Payment

**Symptoms:**
- Multiple charges for same payment request
- Customer charged twice

**Diagnosis:**

1. Check payment request history:
```sql
SELECT * FROM payment_requests
WHERE id = 'pr_xxx';
```

2. Check provider for multiple payments:
```bash
stripe payment_intents list \
  --metadata.paymentRequestId=pr_xxx
```

**Resolution:**

1. Refund duplicate payment:
```bash
stripe refunds create --payment-intent pi_xxx
```

2. Or via gateway:
```typescript
const gateway = await getPaymentGateway();
await gateway.refundPayment('pi_xxx', {
  reason: 'duplicate',
});
```

### Webhook Signature Failures

**Symptoms:**
- 401 responses on webhook endpoint
- Events not being processed

**Diagnosis:**

1. Check webhook secret matches:
```bash
echo $PAYMENT_WEBHOOK_SECRET
# Compare with Stripe Dashboard signing secret
```

2. Check header being read:
```typescript
// Ensure correct header name
const signature = headers.get("stripe-signature");
console.log("Signature header:", signature);
```

**Resolution:**

1. Update webhook secret:
```bash
# Get new secret from Stripe Dashboard
# Update environment variable
PAYMENT_WEBHOOK_SECRET=whsec_new_xxx
```

2. Restart application to pick up new secret

### Provider API Errors

**Symptoms:**
- Payment creation fails
- Error messages like "Your card was declined"

**Diagnosis:**

Check error details:
```typescript
try {
  await gateway.createPaymentSession({ ... });
} catch (error) {
  if (error instanceof PaymentError) {
    console.log('Code:', error.code);
    console.log('Details:', error.details);
  }
}
```

**Common Stripe Errors:**

| Error Code | Meaning | Action |
|------------|---------|--------|
| `card_declined` | Card declined | Ask customer to use different card |
| `insufficient_funds` | Not enough balance | Ask customer to use different card |
| `expired_card` | Card expired | Ask customer to update card |
| `incorrect_cvc` | Wrong CVC | Ask customer to re-enter |
| `rate_limit` | Too many requests | Implement backoff |

## Emergency Procedures

### Disable Payment Processing

If critical issues detected:

1. **Immediate:** Switch to mock provider:
```bash
# Update environment
PAYMENT_PROVIDER=mock
```

2. **Or:** Disable webhook endpoint:
```bash
# In Stripe Dashboard: Webhooks → Disable endpoint
```

3. **Notify team:**
   - Post in #payments-alerts channel
   - Create incident ticket

### Recover From Missed Webhooks

If webhooks were missed for a period:

1. **List missed events:**
```bash
stripe events list \
  --type payment_intent.succeeded \
  --created.gte=$(date -v-24H +%s) \
  --limit 100
```

2. **Process each event:**
```bash
# For each event ID
stripe events retrieve evt_xxx
# Check if payment_request was updated
# Manually update if needed
```

3. **Resend webhooks (Stripe Dashboard):**
   - Developers → Webhooks → Select endpoint
   - View events → Resend

### Data Recovery

If payment records are corrupted:

1. **Get source of truth from provider:**
```bash
# Export payments
stripe payment_intents list \
  --created.gte=2024-01-01 \
  --limit 100 \
  --output json > payments.json
```

2. **Reconcile with database:**
```sql
-- Find mismatches
SELECT
  pr.id,
  pr.external_payment_id,
  pr.status
FROM payment_requests pr
WHERE pr.external_payment_id IS NOT NULL
  AND pr.status != 'paid_platform_verified';
```

## Provider Migration

### Stripe to Lenco Migration

1. **Preparation:**
   - Configure Lenco credentials
   - Test in staging environment
   - Verify webhook endpoint works

2. **Migration steps:**
```bash
# 1. Set up Lenco webhook first (catches new events)
# In Lenco Dashboard: Add webhook endpoint

# 2. Update environment
PAYMENT_PROVIDER=lenco
PAYMENT_API_KEY=lenco_live_xxx
PAYMENT_WEBHOOK_SECRET=lenco_whsec_xxx

# 3. Deploy
# 4. Verify first transaction
# 5. Disable Stripe webhook endpoint
```

3. **Rollback if needed:**
```bash
PAYMENT_PROVIDER=stripe
PAYMENT_API_KEY=sk_live_xxx
```

## Maintenance

### Cleanup Old Sessions

```sql
-- Archive old payment sessions
UPDATE payment_requests
SET status = 'expired'
WHERE status = 'pending_gateway'
  AND created_at < NOW() - INTERVAL '24 hours';
```

### Audit Payment Records

Monthly audit script:

```sql
-- Verify all paid payments have external IDs
SELECT COUNT(*)
FROM payment_requests
WHERE status = 'paid_platform_verified'
  AND external_payment_id IS NULL;

-- Should return 0

-- Verify totals match
SELECT
  currency,
  SUM(total_amount) as db_total
FROM payment_requests
WHERE status = 'paid_platform_verified'
  AND paid_at >= DATE_TRUNC('month', NOW())
GROUP BY currency;

-- Compare with provider dashboard
```

## Contact & Escalation

### Internal Contacts

| Role | Contact | Responsibility |
|------|---------|---------------|
| Payments Owner | @payments-team | Feature decisions |
| On-call Engineer | PagerDuty | Incident response |
| Finance | @finance | Reconciliation |

### Provider Support

**Stripe:**
- Dashboard: https://dashboard.stripe.com
- Support: support@stripe.com
- Status: https://status.stripe.com

**Lenco:**
- Dashboard: https://dashboard.lenco.co
- Support: support@lenco.co
- Status: https://status.lenco.co
