---
title: "Tasks Module - Runbook"
description: "Operations, troubleshooting, testing, and definition of done for the Tasks module."
---

## Table of Contents

1. [Scheduled Jobs](#scheduled-jobs)
2. [Monitoring](#monitoring)
3. [Troubleshooting](#troubleshooting)
4. [Common Operations](#common-operations)
5. [Testing Checklist](#testing-checklist)
6. [Definition of Done](#definition-of-done)

---

## Scheduled Jobs

### Job Overview

| Job | Schedule | Purpose | SLA |
|-----|----------|---------|-----|
| Due Soon Check | Daily 08:00 UTC | Emit `task.due_soon` events | &lt; 60s |
| Overdue Check | Daily 09:00 UTC | Emit `task.overdue` events | &lt; 60s |

### Monitoring Job Health

```bash
# Check recent job executions (via logs)
wrangler tail --filter "job:task_due_soon"
wrangler tail --filter "job:task_overdue"
```

### Manual Job Trigger

For testing or catch-up:

```bash
# Trigger via scheduled event simulation
curl -X POST "https://api.bumara.com/internal/jobs/task-due-soon" \
  -H "Authorization: Bearer $INTERNAL_JOB_TOKEN"

curl -X POST "https://api.bumara.com/internal/jobs/task-overdue" \
  -H "Authorization: Bearer $INTERNAL_JOB_TOKEN"
```

---

## Monitoring

### Key Metrics

| Metric | Alert Threshold | Description |
|--------|-----------------|-------------|
| `tasks.overdue.total` | > 50 per org | Many overdue tasks |
| `tasks.blocked.total` | > 10 per org | Many blocked tasks |
| `tasks.api.error_rate` | > 5% | API error rate |
| `tasks.api.latency_p99` | > 500ms | Slow API responses |
| `tasks.job.failure` | any | Scheduled job failed |

### Dashboard Queries

```sql
-- Current task status distribution by org
SELECT
  organization_id,
  status,
  COUNT(*) as count
FROM tasks
WHERE deleted_at IS NULL
GROUP BY organization_id, status
ORDER BY organization_id, status;

-- Overdue tasks by org (active only)
SELECT
  organization_id,
  COUNT(*) as overdue_count
FROM tasks
WHERE due_on < NOW()
  AND status NOT IN ('done', 'skipped')
  AND deleted_at IS NULL
GROUP BY organization_id
ORDER BY overdue_count DESC;

-- Task completion rate by org (last 30 days)
SELECT
  organization_id,
  COUNT(*) FILTER (WHERE status = 'done') as completed,
  COUNT(*) as total,
  ROUND(100.0 * COUNT(*) FILTER (WHERE status = 'done') / NULLIF(COUNT(*), 0), 1) as completion_rate
FROM tasks
WHERE created_at > NOW() - INTERVAL '30 days'
  AND deleted_at IS NULL
GROUP BY organization_id
ORDER BY completion_rate DESC;
```

### Alerts Configuration

```yaml
# Example alerting rules
alerts:
  - name: high_overdue_tasks
    condition: tasks.overdue.total > 50
    severity: warning
    notify: [slack-compliance-channel]

  - name: api_error_spike
    condition: tasks.api.error_rate > 0.05
    severity: critical
    notify: [pagerduty-on-call]

  - name: job_failure
    condition: tasks.job.failure > 0
    severity: warning
    notify: [slack-engineering]
```

---

## Troubleshooting

### Common Issues

#### 1. "Task not found" errors

**Symptoms:** User gets 404 when accessing a task they should have access to.

**Possible causes:**
- Task belongs to different organization
- Task was deleted (soft delete)
- Task ID is incorrect

**Diagnosis:**

```sql
-- Check if task exists at all
SELECT id, organization_id, deleted_at FROM tasks WHERE id = '<task_id>';

-- Compare with user's org
-- Check audit logs for deletion
SELECT * FROM audit_logs
WHERE entity_type = 'task' AND entity_id = '<task_id>'
ORDER BY timestamp DESC LIMIT 10;
```

#### 2. Status transition rejected

**Symptoms:** User gets 409 CONFLICT when changing status.

**Possible causes:**
- Invalid transition (e.g., done → doing)
- Missing required reason for blocked/skipped
- Task is required but user trying to skip

**Diagnosis:**

```sql
-- Check current status
SELECT id, status, required FROM tasks WHERE id = '<task_id>';
```

**Solution:** Refer to [transition matrix](/modules/tasks/security-permissions#status-transition-guards).

#### 3. Filing not becoming ready

**Symptoms:** Filing can't be submitted despite tasks appearing complete.

**Possible causes:**
- Required tasks not all done
- Hidden blocked tasks
- Tasks in `doing` status (not `done`)

**Diagnosis:**

```sql
-- Check task completion for filing
SELECT
  id,
  title,
  status,
  required
FROM tasks
WHERE filing_id = '<filing_id>'
  AND deleted_at IS NULL
ORDER BY required DESC, sequence ASC;

-- Check readiness
SELECT
  COUNT(*) FILTER (WHERE required AND status != 'done') as incomplete_required,
  COUNT(*) FILTER (WHERE status = 'blocked') as blocked
FROM tasks
WHERE filing_id = '<filing_id>' AND deleted_at IS NULL;
```

#### 4. Duplicate notifications

**Symptoms:** Users receiving multiple notifications for same event.

**Possible causes:**
- Job running multiple times
- Duplicate event emission
- Notification idempotency failure

**Diagnosis:**

```sql
-- Check for duplicate events in logs
SELECT * FROM audit_logs
WHERE entity_type = 'task'
  AND entity_id = '<task_id>'
  AND timestamp > NOW() - INTERVAL '1 hour'
ORDER BY timestamp DESC;
```

---

## Common Operations

### Bulk Update Tasks

**Caution:** Always include `organization_id` filter.

```sql
-- Mark all tasks for a filing as done (emergency override)
UPDATE tasks
SET
  status = 'done',
  completed_at = NOW(),
  completed_by = '<admin_user_id>',
  updated_at = NOW()
WHERE filing_id = '<filing_id>'
  AND organization_id = '<org_id>'
  AND status NOT IN ('done', 'skipped');

-- Record in audit logs (run for each updated task)
INSERT INTO audit_logs (
  organization_id, user_id, actor_type, action,
  entity_type, entity_id, changes, metadata, timestamp
)
SELECT
  organization_id,
  '<admin_user_id>',
  'STAFF',
  'update',
  'task',
  id,
  jsonb_build_object(
    'before', jsonb_build_object('status', 'todo'),
    'after', jsonb_build_object('status', 'done')
  ),
  jsonb_build_object('reason', 'Bulk override by admin'),
  NOW()
FROM tasks WHERE filing_id = '<filing_id>' AND organization_id = '<org_id>';
```

### Reassign Tasks Bulk

```sql
-- Reassign all tasks from one member to another
UPDATE tasks
SET
  assigned_to_member_id = '<new_member_id>',
  updated_at = NOW()
WHERE assigned_to_member_id = '<old_member_id>'
  AND organization_id = '<org_id>'
  AND status NOT IN ('done', 'skipped');
```

### Clean Up Orphaned Tasks

Tasks without a parent (filing or service request):

```sql
-- Find orphaned tasks
SELECT * FROM tasks
WHERE filing_id IS NULL
  AND service_request_id IS NULL
  AND deleted_at IS NULL;

-- Soft delete orphaned tasks
UPDATE tasks
SET deleted_at = NOW(), updated_at = NOW()
WHERE filing_id IS NULL
  AND service_request_id IS NULL
  AND deleted_at IS NULL;
```

### Export Task Data

```sql
-- Export all tasks for an org (CSV-friendly)
COPY (
  SELECT
    t.id,
    t.title,
    t.status,
    t.task_type,
    t.required,
    t.due_on,
    t.completed_at,
    f.id as filing_id,
    sr.id as service_request_id,
    om.email as assignee_email
  FROM tasks t
  LEFT JOIN filings f ON t.filing_id = f.id
  LEFT JOIN service_requests sr ON t.service_request_id = sr.id
  LEFT JOIN organization_members om ON t.assigned_to_member_id = om.id
  WHERE t.organization_id = '<org_id>'
    AND t.deleted_at IS NULL
  ORDER BY t.created_at DESC
) TO '/tmp/tasks_export.csv' WITH CSV HEADER;
```

---

## Testing Checklist

### Unit Tests

Located in `packages/api-services/src/domains/tasks/__tests__/`

- [ ] **Status transitions**
  - [ ] Valid transitions succeed
  - [ ] Invalid transitions throw ConflictError
  - [ ] Blocked requires reason
  - [ ] Skip requires reason + optional task
  - [ ] Reopen requires permission

- [ ] **Permission checks**
  - [ ] Admin can do everything
  - [ ] Manager can't skip required
  - [ ] Member can only update own/unassigned tasks
  - [ ] Member can't assign tasks

- [ ] **Filing readiness**
  - [ ] All required done → ready
  - [ ] Some required pending → not ready
  - [ ] Blocked tasks → not ready
  - [ ] Optional tasks don't affect readiness

### Integration Tests

Located in `packages/api-services/src/domains/tasks/__tests__/integration/`

- [ ] **Create filing → tasks flow**
  ```
  1. Create filing
  2. Verify tasks generated (if template)
  3. Complete required tasks
  4. Verify filing becomes ready
  5. Request submission succeeds
  ```

- [ ] **Info request flow**
  ```
  1. Backoffice creates info_request task
  2. Tenant sees task in list
  3. Tenant completes task
  4. Backoffice sees completion
  ```

- [ ] **Multi-tenant isolation**
  ```
  1. Create task in org A
  2. Try to access from org B → 404
  3. Try to update from org B → 404
  4. List from org B → empty
  ```

- [ ] **Audit logging**
  ```
  1. Create task
  2. Update status
  3. Skip task
  4. Verify audit entries exist with correct data
  ```

### E2E Tests

Located in `apps/app/__tests__/e2e/tasks/`

- [ ] **Tenant app flows**
  - [ ] View task list
  - [ ] Filter by status
  - [ ] Open task drawer
  - [ ] Mark task in progress
  - [ ] Mark task done
  - [ ] Add comment
  - [ ] View filing tasks

- [ ] **Backoffice flows**
  - [ ] View org task queue
  - [ ] Create info request
  - [ ] Assign task
  - [ ] Skip task with reason

### Performance Tests

- [ ] List 1000 tasks &lt; 500ms
- [ ] Readiness check with 50 tasks &lt; 100ms
- [ ] Due soon job with 10k tasks &lt; 30s

---

## Definition of Done

### For a Task (the feature task, not compliance tasks)

- [ ] **Documentation**
  - [ ] All 7 doc files created
  - [ ] Docs reference actual code paths
  - [ ] Mermaid diagrams render correctly
  - [ ] Links in index files updated

- [ ] **Database**
  - [ ] Schema changes applied
  - [ ] Migration runs without error
  - [ ] Indexes verified
  - [ ] Existing data migrated

- [ ] **Backend**
  - [ ] All endpoints implemented
  - [ ] Zod schemas validate correctly
  - [ ] Org isolation enforced
  - [ ] Transitions validated server-side
  - [ ] Audit logs recorded
  - [ ] Error responses follow standard format

- [ ] **Frontend**
  - [ ] Types aligned with backend
  - [ ] API calls replace mock data
  - [ ] Loading/error states handled
  - [ ] Mobile responsive

- [ ] **Testing**
  - [ ] Unit test coverage > 80%
  - [ ] Integration tests pass
  - [ ] Manual testing complete
  - [ ] No regressions

- [ ] **Quality**
  - [ ] No TypeScript errors
  - [ ] Lint passes
  - [ ] No console errors in browser
  - [ ] Accessibility checked

### Acceptance Criteria Summary

1. Tenant can view and manage their tasks
2. Tasks gated by Filing/Service Request parent
3. Required tasks block submission readiness
4. Backoffice can create info requests
5. Status changes follow transition rules
6. All mutations are audit logged
7. No cross-org data access possible
8. Notifications work for key events


