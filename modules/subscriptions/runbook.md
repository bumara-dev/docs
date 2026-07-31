---
title: "Subscriptions Runbook"
description: "If a customer needs to be upgraded outside the normal flow:"
---

## Common Operations

### Manually Upgrade a Customer

If a customer needs to be upgraded outside the normal flow:

1. **Update in Stripe Dashboard**:
   - Find the customer's subscription
   - Change to the new plan

2. **Verify webhook processed**:
   ```sql
   SELECT * FROM subscriptions
   WHERE organization_id = 'org_xxxxx'
   ORDER BY updated_at DESC
   LIMIT 1;
   ```

3. **If webhook failed, manually update**:
   ```sql
   UPDATE subscriptions
   SET plan_tier = 'pro',
       updated_at = NOW()
   WHERE organization_id = 'org_xxxxx';
   ```

---

### Reset Monthly Usage

If usage needs to be reset mid-cycle:

```sql
UPDATE subscription_usage
SET service_requests_used = 0,
    ai_credits_used = 0,
    updated_at = NOW()
WHERE organization_id = 'org_xxxxx'
  AND period_key = '2026-01';
```

**Note:** Storage is cumulative and typically shouldn't be reset.

---

### Grant Enterprise Features

For custom enterprise deals:

```sql
UPDATE subscriptions
SET plan_tier = 'enterprise',
    status = 'active',
    updated_at = NOW()
WHERE organization_id = 'org_xxxxx';
```

---

### Check Customer's Current Limits

```sql
SELECT
  s.organization_id,
  s.plan_tier,
  s.status,
  u.period_key,
  u.service_requests_used,
  u.storage_used_mb,
  u.ai_credits_used
FROM subscriptions s
LEFT JOIN subscription_usage u
  ON s.organization_id = u.organization_id
  AND u.period_key = TO_CHAR(NOW(), 'YYYY-MM')
WHERE s.organization_id = 'org_xxxxx';
```

---

### View Audit Log

```sql
SELECT
  action,
  before_state,
  after_state,
  changed_by_user_id,
  created_at
FROM subscription_audit_log
WHERE organization_id = 'org_xxxxx'
ORDER BY created_at DESC
LIMIT 20;
```

---

## Troubleshooting

### "Subscription not found" Error

**Cause:** Organization doesn't have a subscription record.

**Fix:** Create a default subscription:
```sql
INSERT INTO subscriptions (organization_id, plan_tier, status)
VALUES ('org_xxxxx', 'start', 'active')
ON CONFLICT (organization_id) DO NOTHING;
```

---

### Webhook Not Processing

**Symptoms:**
- Subscription status not updating after payment
- Plan tier not changing after checkout

**Checks:**

1. **Verify webhook secret**:
   ```bash
   echo $STRIPE_WEBHOOK_SECRET
   ```

2. **Check webhook logs in Stripe Dashboard**:
   - Go to Developers > Webhooks
   - Check for failed deliveries

3. **Check backend logs**:
   ```bash
   # Look for webhook processing errors
   grep "webhook" logs/backend.log | tail -50
   ```

4. **Manually replay webhook**:
   - In Stripe Dashboard, find the event
   - Click "Resend"

---

### Usage Not Tracking

**Symptoms:**
- Service requests not counting
- Storage not updating

**Checks:**

1. **Verify usage record exists**:
   ```sql
   SELECT * FROM subscription_usage
   WHERE organization_id = 'org_xxxxx'
     AND period_key = TO_CHAR(NOW(), 'YYYY-MM');
   ```

2. **Create if missing**:
   ```sql
   INSERT INTO subscription_usage (
     organization_id,
     period_key,
     period_start,
     period_end,
     service_requests_used,
     storage_used_mb,
     ai_credits_used
   ) VALUES (
     'org_xxxxx',
     TO_CHAR(NOW(), 'YYYY-MM'),
     DATE_TRUNC('month', NOW()),
     DATE_TRUNC('month', NOW()) + INTERVAL '1 month',
     0, 0, 0
   ) ON CONFLICT DO NOTHING;
   ```

---

### Limit Not Enforcing

**Symptoms:**
- Users can create more than their limit

**Checks:**

1. **Verify subscription caps**:
   ```sql
   SELECT plan_tier FROM subscriptions
   WHERE organization_id = 'org_xxxxx';
   ```

2. **Check enforcement is called**:
   - Search for `enforceServiceRequestLimit` in service code
   - Ensure it's called before creation

3. **Check current usage**:
   ```sql
   SELECT service_requests_used FROM subscription_usage
   WHERE organization_id = 'org_xxxxx'
     AND period_key = TO_CHAR(NOW(), 'YYYY-MM');
   ```

---

### Plan Change Not Working

**Symptoms:**
- Checkout completes but plan doesn't change

**Checks:**

1. **Verify Stripe metadata includes organizationId**:
   - Check checkout session in Stripe Dashboard
   - `metadata.organizationId` should be set

2. **Check webhook received**:
   - Stripe Dashboard > Webhooks > Recent events
   - Look for `checkout.session.completed`

3. **Manual fix if needed**:
   ```sql
   UPDATE subscriptions
   SET plan_tier = 'plus',
       external_subscription_id = 'sub_xxxxx',
       status = 'active',
       updated_at = NOW()
   WHERE organization_id = 'org_xxxxx';
   ```

---

## Monitoring

### Key Metrics

1. **Active subscriptions by plan**:
   ```sql
   SELECT plan_tier, COUNT(*) as count
   FROM subscriptions
   WHERE status = 'active'
   GROUP BY plan_tier;
   ```

2. **Monthly recurring revenue (MRR)**:
   ```sql
   SELECT
     plan_tier,
     COUNT(*) as subscriptions,
     CASE plan_tier
       WHEN 'start' THEN COUNT(*) * 1500
       WHEN 'plus' THEN COUNT(*) * 3500
       WHEN 'pro' THEN COUNT(*) * 7500
     END as mrr_zmw
   FROM subscriptions
   WHERE status = 'active'
   GROUP BY plan_tier;
   ```

3. **Usage approaching limits**:
   ```sql
   SELECT
     s.organization_id,
     s.plan_tier,
     u.service_requests_used,
     CASE s.plan_tier
       WHEN 'start' THEN 2
       WHEN 'plus' THEN 3
       WHEN 'pro' THEN 5
     END as limit,
     ROUND(u.service_requests_used::numeric /
       CASE s.plan_tier
         WHEN 'start' THEN 2
         WHEN 'plus' THEN 3
         WHEN 'pro' THEN 5
       END * 100, 1) as usage_percent
   FROM subscriptions s
   JOIN subscription_usage u
     ON s.organization_id = u.organization_id
     AND u.period_key = TO_CHAR(NOW(), 'YYYY-MM')
   WHERE s.status = 'active'
     AND s.plan_tier != 'enterprise'
   HAVING ROUND(u.service_requests_used::numeric /
       CASE s.plan_tier
         WHEN 'start' THEN 2
         WHEN 'plus' THEN 3
         WHEN 'pro' THEN 5
       END * 100, 1) >= 80
   ORDER BY usage_percent DESC;
   ```

---

## Emergency Procedures

### Disable Enforcement (Emergency Only)

If enforcement is causing issues and needs to be disabled temporarily:

1. **Comment out enforcement calls** in service files
2. **Deploy hotfix**
3. **Fix underlying issue**
4. **Re-enable enforcement**
5. **Reconcile any over-usage**

### Bulk Grant Access

If a regulator needs to be added to all plans:

```sql
-- This would require updating the plans package
-- and redeploying, then:
UPDATE subscriptions
SET updated_at = NOW()
WHERE status = 'active';
```

### Data Recovery

If subscription data is corrupted:

1. **Check audit log for last good state**:
   ```sql
   SELECT * FROM subscription_audit_log
   WHERE organization_id = 'org_xxxxx'
   ORDER BY created_at DESC;
   ```

2. **Restore from audit log**:
   ```sql
   UPDATE subscriptions
   SET plan_tier = (audit_log.before_state->>'plan')::text
   -- etc.
   ```

3. **Verify with Stripe**:
   - Cross-reference with Stripe Dashboard
   - Ensure database matches Stripe state
