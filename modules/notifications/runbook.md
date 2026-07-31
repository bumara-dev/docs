---
title: "Notifications Runbook"
description: "Operations guide for monitoring, troubleshooting, and maintaining the notification system."
---

## Table of Contents

1. [System Health](#system-health)
2. [Monitoring](#monitoring)
3. [Common Issues](#common-issues)
4. [Troubleshooting](#troubleshooting)
5. [Maintenance Tasks](#maintenance-tasks)
6. [Emergency Procedures](#emergency-procedures)

---

## System Health

### Health Check Endpoints

```bash
# Check API health
curl https://api.bumara.com/health

# Check queue status (via Cloudflare dashboard)
# Workers > Queues > notification-outbox
# Workers > Queues > notification-delivery
```

### Key Metrics to Monitor

| Metric | Warning | Critical | Action |
|--------|---------|----------|--------|
| Outbox queue depth | > 100 | > 1000 | Scale workers, check for failures |
| Delivery queue depth | > 500 | > 5000 | Check provider rate limits |
| Failed deliveries/hour | > 10 | > 100 | Check provider status, review errors |
| Webhook failures/hour | > 5 | > 50 | Verify signatures, check endpoints |
| Scheduler job duration | > 30s | > 60s | Optimize queries, check DB |

### Quick Status Check

```sql
-- Outbox status summary
SELECT 
  status, 
  COUNT(*) as count,
  MIN(created_at) as oldest
FROM event_outbox
WHERE created_at > NOW() - INTERVAL '24 hours'
GROUP BY status;

-- Delivery status summary
SELECT 
  channel,
  status,
  COUNT(*) as count
FROM notification_deliveries
WHERE created_at > NOW() - INTERVAL '24 hours'
GROUP BY channel, status
ORDER BY channel, status;

-- Failed deliveries
SELECT 
  id,
  channel,
  last_error,
  attempts,
  created_at
FROM notification_deliveries
WHERE status = 'failed'
  AND created_at > NOW() - INTERVAL '24 hours'
ORDER BY created_at DESC
LIMIT 20;
```

---

## Monitoring

### Structured Logs

The notification system outputs structured JSON logs:

```json
{
  "level": "info",
  "timestamp": "2025-01-01T08:00:00.000Z",
  "job": "filing_due_soon",
  "eventsCreated": 45,
  "durationMs": 1234
}

{
  "level": "error",
  "timestamp": "2025-01-01T08:01:00.000Z",
  "component": "delivery_sender",
  "deliveryId": "uuid-123",
  "channel": "WHATSAPP",
  "error": "Rate limit exceeded",
  "attempt": 3
}
```

### Log Queries (Cloudflare)

```bash
# View recent errors
wrangler tail --filter "level:error"

# View outbox processing
wrangler tail --filter "job:*"

# View delivery attempts
wrangler tail --filter "component:delivery_sender"
```

### Alerting Rules

Set up alerts in your monitoring system:

```yaml
# Example Datadog/PagerDuty alerts
alerts:
  - name: High outbox queue depth
    query: avg:notification.outbox.depth > 500
    severity: warning
    
  - name: Critical outbox queue depth
    query: avg:notification.outbox.depth > 2000
    severity: critical
    
  - name: Delivery failure spike
    query: sum:notification.delivery.failed.count > 50
    window: 5m
    severity: critical
    
  - name: Scheduler job failure
    query: count:notification.job.failed > 0
    severity: warning
```

---

## Common Issues

### Issue: Outbox Events Not Processing

**Symptoms:**
- Events stuck in `pending` status
- No new notifications appearing
- Queue depth increasing

**Diagnosis:**
```sql
-- Check pending events
SELECT COUNT(*) FROM event_outbox WHERE status = 'pending';

-- Check for stuck processing events
SELECT * FROM event_outbox 
WHERE status = 'processing' 
  AND updated_at < NOW() - INTERVAL '10 minutes';
```

**Resolution:**
1. Check Cloudflare Queue status in dashboard
2. Verify worker is deployed and running
3. Reset stuck events:

```sql
-- Reset stuck processing events
UPDATE event_outbox
SET status = 'pending', updated_at = NOW()
WHERE status = 'processing'
  AND updated_at < NOW() - INTERVAL '10 minutes';
```

### Issue: Deliveries Failing

**Symptoms:**
- High failure rate in deliveries
- `failed` status in notification_deliveries
- Error messages in logs

**Diagnosis:**
```sql
-- Check failure reasons
SELECT 
  channel,
  last_error,
  COUNT(*) as count
FROM notification_deliveries
WHERE status = 'failed'
  AND created_at > NOW() - INTERVAL '24 hours'
GROUP BY channel, last_error
ORDER BY count DESC;
```

**Common Errors and Fixes:**

| Error | Cause | Fix |
|-------|-------|-----|
| Rate limit exceeded | Too many requests | Implement backoff, contact provider |
| Invalid recipient | Bad email/phone format | Fix validation, mark user invalid |
| Template not found | Template deleted/renamed | Update template mapping |
| Authentication failed | Token expired | Rotate credentials |

### Issue: Webhooks Not Updating Status

**Symptoms:**
- Deliveries stuck in `sent` status
- No `delivered` status updates

**Diagnosis:**
1. Check webhook logs in provider dashboard
2. Verify webhook URL is correct
3. Check for signature verification failures

**Resolution:**
1. Verify webhook endpoint is accessible:
```bash
curl -X POST https://api.bumara.com/webhooks/email \
  -H "Content-Type: application/json" \
  -d '{"test": true}'
# Should return 400 (missing headers) or 401 (invalid signature)
```

2. Re-register webhook in provider dashboard
3. Update webhook secret if compromised

### Issue: Duplicate Notifications

**Symptoms:**
- Users receiving same notification multiple times
- Multiple rows in notifications table for same event

**Diagnosis:**
```sql
-- Check for duplicates
SELECT 
  tenant_id,
  user_id,
  outbox_event_id,
  COUNT(*) as count
FROM notifications
GROUP BY tenant_id, user_id, outbox_event_id
HAVING COUNT(*) > 1;
```

**Resolution:**
1. Add unique constraint if missing
2. Clean up duplicates:

```sql
-- Delete duplicates keeping the first
DELETE FROM notifications n1
USING notifications n2
WHERE n1.id > n2.id
  AND n1.tenant_id = n2.tenant_id
  AND n1.user_id = n2.user_id
  AND n1.outbox_event_id = n2.outbox_event_id;
```

---

## Troubleshooting

### Debug Workflow

1. **Identify the event**
   ```sql
   SELECT * FROM event_outbox WHERE id = 'uuid';
   ```

2. **Check notifications created**
   ```sql
   SELECT * FROM notifications WHERE outbox_event_id = 'uuid';
   ```

3. **Check deliveries**
   ```sql
   SELECT * FROM notification_deliveries 
   WHERE notification_id IN (
     SELECT id FROM notifications WHERE outbox_event_id = 'uuid'
   );
   ```

4. **Check provider logs**
   - Resend: Dashboard > Emails > Search by message ID
   - Meta: Graph API Explorer > Check message status

### Manual Retry

```sql
-- Retry a specific failed delivery
UPDATE notification_deliveries
SET 
  status = 'queued',
  attempts = 0,
  next_retry_at = NOW(),
  last_error = NULL,
  updated_at = NOW()
WHERE id = 'delivery-uuid';

-- Then trigger queue processing
-- (Or wait for next outbox poll cycle)
```

### Force Reprocess Event

```sql
-- Reprocess a failed outbox event
UPDATE event_outbox
SET 
  status = 'pending',
  attempts = 0,
  next_retry_at = NOW(),
  last_error = NULL,
  updated_at = NOW()
WHERE id = 'outbox-uuid';
```

---

## Maintenance Tasks

### Daily

- [ ] Review failed deliveries (should be &lt; 1%)
- [ ] Check queue depths
- [ ] Verify scheduler jobs ran successfully

### Weekly

- [ ] Review delivery success rates by channel
- [ ] Check for users with consistently failing deliveries
- [ ] Review and clear dead letter queue

### Monthly

- [ ] Rotate API keys (if required by policy)
- [ ] Review notification volume trends
- [ ] Clean up old processed events

### Cleanup Old Data

```sql
-- Archive processed outbox events older than 30 days
DELETE FROM event_outbox
WHERE status = 'processed'
  AND processed_at < NOW() - INTERVAL '30 days';

-- Archive old deliveries (keep 90 days)
DELETE FROM notification_deliveries
WHERE created_at < NOW() - INTERVAL '90 days'
  AND status IN ('delivered', 'read');
```

---

## Emergency Procedures

### Pause All Notifications

**When to use:** Critical bug, security issue, or provider outage.

```sql
-- Stop outbox processing
UPDATE event_outbox
SET status = 'paused'
WHERE status = 'pending';

-- Note: Requires adding 'paused' to outbox_status enum
-- Alternative: Scale down workers in Cloudflare dashboard
```

**Cloudflare Dashboard:**
1. Go to Workers > Queues
2. Click on `notification-outbox`
3. Pause the consumer

### Resume Notifications

```sql
-- Resume paused events
UPDATE event_outbox
SET 
  status = 'pending',
  updated_at = NOW()
WHERE status = 'paused';
```

**Cloudflare Dashboard:**
1. Go to Workers > Queues
2. Click on `notification-outbox`
3. Resume the consumer

### Provider Outage

**Email (Resend) outage:**
1. Monitor Resend status page
2. Deliveries will retry automatically
3. If prolonged, consider temporary provider switch

**WhatsApp (Meta) outage:**
1. Monitor Meta status page
2. Deliveries will retry automatically
3. Consider pausing WhatsApp notifications

### Data Breach Response

If notification content or recipient data is compromised:

1. **Immediately:**
   - Rotate all provider API keys
   - Rotate webhook secrets
   - Pause notifications

2. **Investigation:**
   - Review audit logs
   - Identify affected data
   - Determine scope

3. **Recovery:**
   - Generate new credentials
   - Update all configurations
   - Resume with new credentials
   - Notify affected parties per policy

---

## Contact Information

| Issue | Contact |
|-------|---------|
| Resend issues | support@resend.com |
| Meta WhatsApp issues | [Meta Business Help Center](https://www.facebook.com/business/help) |
| Cloudflare Workers | [Cloudflare Support](https://support.cloudflare.com) |
| Internal escalation | #platform-oncall Slack channel |

