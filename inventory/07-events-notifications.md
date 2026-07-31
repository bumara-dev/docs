---
title: "Inventory Module - Events & Notifications"
description: "Event catalog, notification routing, and alert policies for the Inventory Module."
---

## Overview

The Inventory Module emits events for:

1. **Stock alerts** — Low stock, negative stock attempts
2. **Operation completion** — Adjustments posted, transfers received, counts completed
3. **Configuration changes** — Items created/archived, locations added

Events integrate with Bumara's existing notification infrastructure:
- **Knock Labs** for in-app notifications and email
- **Tasks module** for auto-created tasks (e.g., reorder reminders)

---

## Event Catalog

### Stock Alert Events

#### `inventory.low_stock`

Emitted when stock balance drops to or below the item's reorder level.

| Field | Type | Description |
|-------|------|-------------|
| `eventType` | string | `inventory.low_stock` |
| `organizationId` | string | Organization ID |
| `itemId` | string | Item that triggered alert |
| `itemName` | string | Item name for display |
| `itemSku` | string | Item SKU (nullable) |
| `locationId` | string | Location where stock is low |
| `locationName` | string | Location name for display |
| `currentQty` | string | Current on-hand quantity |
| `reorderLevel` | string | Configured reorder threshold |
| `reorderQty` | string | Suggested reorder quantity (nullable) |
| `uomCode` | string | Unit of measure code |
| `triggeredBy` | string | User who posted the operation |
| `sourceType` | string | What caused it: `adjustment`, `transfer`, `count` |
| `sourceId` | string | Reference ID of source operation |
| `occurredAt` | string | ISO timestamp |

**Example payload:**

```json
{
  "eventType": "inventory.low_stock",
  "organizationId": "org_123",
  "itemId": "item_456",
  "itemName": "Office Chair",
  "itemSku": "CHAIR-001",
  "locationId": "loc_789",
  "locationName": "Main Warehouse",
  "currentQty": "8.0000",
  "reorderLevel": "10.0000",
  "reorderQty": "50.0000",
  "uomCode": "EA",
  "triggeredBy": "user_abc",
  "sourceType": "adjustment",
  "sourceId": "adj_xyz",
  "occurredAt": "2026-01-20T14:30:00Z"
}
```

#### `inventory.negative_stock_attempted`

Emitted when an operation is blocked due to negative stock policy.

| Field | Type | Description |
|-------|------|-------------|
| `eventType` | string | `inventory.negative_stock_attempted` |
| `organizationId` | string | Organization ID |
| `itemId` | string | Item that would go negative |
| `itemName` | string | Item name |
| `locationId` | string | Location |
| `locationName` | string | Location name |
| `currentQty` | string | Current balance |
| `requestedQty` | string | Quantity that was requested |
| `wouldResultIn` | string | What balance would have been |
| `operationType` | string | `adjustment`, `transfer`, `count` |
| `operationId` | string | Operation ID |
| `attemptedBy` | string | User who attempted |
| `occurredAt` | string | ISO timestamp |

### Operation Events

#### `inventory.adjustment_posted`

Emitted when an adjustment is successfully posted.

| Field | Type | Description |
|-------|------|-------------|
| `eventType` | string | `inventory.adjustment_posted` |
| `organizationId` | string | Organization ID |
| `adjustmentId` | string | Adjustment ID |
| `locationId` | string | Location ID |
| `locationName` | string | Location name |
| `reason` | string | Adjustment reason code |
| `lineCount` | number | Number of items adjusted |
| `totalQtyChange` | string | Net quantity change (sum) |
| `postedBy` | string | User who posted |
| `postedAt` | string | ISO timestamp |

#### `inventory.adjustment_voided`

Emitted when an adjustment is voided.

| Field | Type | Description |
|-------|------|-------------|
| `eventType` | string | `inventory.adjustment_voided` |
| `organizationId` | string | Organization ID |
| `adjustmentId` | string | Adjustment ID |
| `voidReason` | string | Reason for voiding |
| `voidedBy` | string | User who voided |
| `voidedAt` | string | ISO timestamp |

#### `inventory.transfer_shipped`

Emitted when a transfer is shipped (stock leaves source).

| Field | Type | Description |
|-------|------|-------------|
| `eventType` | string | `inventory.transfer_shipped` |
| `organizationId` | string | Organization ID |
| `transferId` | string | Transfer ID |
| `fromLocationId` | string | Source location ID |
| `fromLocationName` | string | Source location name |
| `toLocationId` | string | Destination location ID |
| `toLocationName` | string | Destination location name |
| `lineCount` | number | Number of items |
| `shippedBy` | string | User who shipped |
| `shippedAt` | string | ISO timestamp |

#### `inventory.transfer_received`

Emitted when a transfer is received (stock arrives at destination).

| Field | Type | Description |
|-------|------|-------------|
| `eventType` | string | `inventory.transfer_received` |
| `organizationId` | string | Organization ID |
| `transferId` | string | Transfer ID |
| `fromLocationId` | string | Source location ID |
| `fromLocationName` | string | Source location name |
| `toLocationId` | string | Destination location ID |
| `toLocationName` | string | Destination location name |
| `lineCount` | number | Number of items |
| `receivedBy` | string | User who received |
| `receivedAt` | string | ISO timestamp |

#### `inventory.count_posted`

Emitted when a stock count is posted with variances.

| Field | Type | Description |
|-------|------|-------------|
| `eventType` | string | `inventory.count_posted` |
| `organizationId` | string | Organization ID |
| `countId` | string | Count ID |
| `countType` | string | `cycle` or `full` |
| `locationId` | string | Location ID |
| `locationName` | string | Location name |
| `totalLines` | number | Total items counted |
| `varianceCount` | number | Lines with variance |
| `totalVarianceQty` | string | Sum of absolute variances |
| `postedBy` | string | User who posted |
| `postedAt` | string | ISO timestamp |

### Configuration Events

#### `inventory.item_created`

| Field | Type | Description |
|-------|------|-------------|
| `eventType` | string | `inventory.item_created` |
| `organizationId` | string | Organization ID |
| `itemId` | string | New item ID |
| `itemName` | string | Item name |
| `itemSku` | string | SKU (nullable) |
| `createdBy` | string | User who created |
| `createdAt` | string | ISO timestamp |

#### `inventory.item_archived`

| Field | Type | Description |
|-------|------|-------------|
| `eventType` | string | `inventory.item_archived` |
| `organizationId` | string | Organization ID |
| `itemId` | string | Archived item ID |
| `itemName` | string | Item name |
| `archivedBy` | string | User who archived |
| `archivedAt` | string | ISO timestamp |

---

## Notification Routing

### Knock Labs Integration

Events are sent to Knock Labs for notification delivery using the existing integration in [`packages/notifications/`](https://github.com/bumara-dev/bumara/tree/main/packages/notifications).

**Workflow Trigger Pattern:**

```typescript
// packages/inventory/src/notifications.ts
import { notifications } from '@repo/notifications';

export async function emitLowStockNotification(
  ctx: ServiceContext,
  payload: LowStockEventPayload
) {
  await notifications.workflows.trigger('inventory-low-stock', {
    recipients: [
      // Organization-level: notify managers and admins
      { 
        id: ctx.orgId,
        collection: 'organizations',
      },
    ],
    data: {
      itemName: payload.itemName,
      itemSku: payload.itemSku,
      locationName: payload.locationName,
      currentQty: payload.currentQty,
      reorderLevel: payload.reorderLevel,
      reorderQty: payload.reorderQty,
      uomCode: payload.uomCode,
      itemUrl: `${process.env.APP_URL}/inventory/items/${payload.itemId}`,
    },
    tenant: ctx.orgId,
  });
}
```

### Recipient Routing

| Event | Recipients | Channels |
|-------|------------|----------|
| `inventory.low_stock` | Managers, Admins | In-app, Email (digest) |
| `inventory.negative_stock_attempted` | Admins | In-app, Email (immediate) |
| `inventory.adjustment_posted` | Managers, Admins | In-app |
| `inventory.transfer_received` | Transfer creator, Managers | In-app |
| `inventory.count_posted` | Managers, Admins | In-app |
| `inventory.item_created` | — | Audit log only |
| `inventory.item_archived` | Managers, Admins | In-app |

### Knock Workflow Templates

**`inventory-low-stock`:**

```yaml
# Knock workflow definition
name: Inventory Low Stock Alert
trigger: inventory-low-stock

steps:
  - channel: in_app_feed
    template:
      title: "Low Stock Alert: {{itemName}}"
      body: "{{itemName}} at {{locationName}} is low ({{currentQty}} {{uomCode}}). Reorder level: {{reorderLevel}}."
      action_url: "{{itemUrl}}"
      
  - delay: { duration: "1 hour" }
  
  - channel: email
    template:
      subject: "Low Stock: {{itemName}} needs reordering"
      # Email template with more details
```

**`inventory-count-posted`:**

```yaml
name: Stock Count Completed
trigger: inventory-count-posted

steps:
  - channel: in_app_feed
    template:
      title: "Stock Count Posted"
      body: "Count at {{locationName}} completed. {{varianceCount}} items had variances."
      action_url: "{{countUrl}}"
```

---

## Low Stock Alert Policy

### Deduplication

To prevent alert spam, implement rate limiting:

```typescript
// packages/inventory/src/low-stock-alerts.ts

const LOW_STOCK_ALERT_COOLDOWN_HOURS = 24;

export async function shouldSendLowStockAlert(
  ctx: ServiceContext,
  deps: ServiceDeps,
  itemId: string,
  locationId: string
): Promise<boolean> {
  const { db } = deps;
  
  // Check if alert was sent recently
  const recentAlert = await db
    .select()
    .from(inventoryAlertLog)
    .where(
      and(
        eq(inventoryAlertLog.organizationId, ctx.orgId),
        eq(inventoryAlertLog.itemId, itemId),
        eq(inventoryAlertLog.locationId, locationId),
        eq(inventoryAlertLog.alertType, 'low_stock'),
        gt(inventoryAlertLog.sentAt, subHours(new Date(), LOW_STOCK_ALERT_COOLDOWN_HOURS))
      )
    )
    .limit(1);
  
  return recentAlert.length === 0;
}

export async function recordLowStockAlert(
  ctx: ServiceContext,
  deps: ServiceDeps,
  itemId: string,
  locationId: string
): Promise<void> {
  await deps.db.insert(inventoryAlertLog).values({
    organizationId: ctx.orgId,
    itemId,
    locationId,
    alertType: 'low_stock',
    sentAt: new Date(),
  });
}
```

### Alert Log Table (Optional)

```typescript
// packages/database/src/schema/inventory/alert-log.ts
export const inventoryAlertLog = pgTable(
  'inventory_alert_log',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    organizationId: text('organization_id')
      .notNull()
      .references(() => organizations.id, { onDelete: 'cascade' }),
    itemId: uuid('item_id').references(() => inventoryItems.id, { onDelete: 'cascade' }),
    locationId: uuid('location_id').references(() => inventoryLocations.id, { onDelete: 'cascade' }),
    alertType: text('alert_type').notNull(), // 'low_stock', 'negative_stock_attempted'
    sentAt: timestamp('sent_at', { mode: 'date' }).notNull(),
  },
  (table) => [
    index('idx_inventory_alert_log_lookup').on(
      table.organizationId, 
      table.itemId, 
      table.locationId, 
      table.alertType, 
      table.sentAt
    ),
  ]
);
```

### Alert Frequency Settings

Future enhancement: Allow organizations to configure alert frequency.

```typescript
interface InventoryNotificationSettings {
  lowStockAlertFrequencyHours: number; // default: 24
  lowStockEmailDigest: boolean; // default: true (batch emails)
  lowStockEmailImmediate: boolean; // default: false
  negativeStockAlertEmail: boolean; // default: true (always immediate)
}
```

---

## Task Auto-Creation

### Low Stock → Reorder Task

When low stock is detected, optionally create a task for reordering:

```typescript
// In stock-ledger.service.ts
async function checkAndAlertLowStock(
  ctx: ServiceContext,
  deps: ServiceDeps,
  itemId: string,
  locationId: string
): Promise<void> {
  const balance = await getBalance(ctx, deps, itemId, locationId);
  const item = await getItem(ctx, deps, itemId);
  const location = await getLocation(ctx, deps, locationId);
  
  if (!item.reorderLevel || balance.onHandQty > parseFloat(item.reorderLevel)) {
    return; // Not low stock
  }
  
  // Check dedup
  if (!(await shouldSendLowStockAlert(ctx, deps, itemId, locationId))) {
    return; // Already alerted recently
  }
  
  // Send notification
  await emitLowStockNotification(ctx, {
    itemId,
    itemName: item.name,
    itemSku: item.sku,
    locationId,
    locationName: location.name,
    currentQty: balance.onHandQty,
    reorderLevel: item.reorderLevel,
    reorderQty: item.reorderQty,
    uomCode: item.defaultUom.code,
    triggeredBy: ctx.userId,
    sourceType: 'adjustment', // or transfer, count
    sourceId: ctx.sourceId,
  });
  
  // Record alert
  await recordLowStockAlert(ctx, deps, itemId, locationId);
  
  // Optional: Create reorder task
  if (ctx.orgSettings?.autoCreateReorderTasks) {
    await createReorderTask(ctx, deps, {
      itemId,
      itemName: item.name,
      locationId,
      locationName: location.name,
      suggestedQty: item.reorderQty,
    });
  }
}

async function createReorderTask(
  ctx: ServiceContext,
  deps: ServiceDeps,
  input: ReorderTaskInput
): Promise<void> {
  // Check if reorder task already exists for this item
  const existingTask = await deps.db
    .select()
    .from(tasks)
    .where(
      and(
        eq(tasks.organizationId, ctx.orgId),
        eq(tasks.templateKey, `inventory:reorder:${input.itemId}`),
        inArray(tasks.status, ['todo', 'doing'])
      )
    )
    .limit(1);
  
  if (existingTask.length > 0) {
    return; // Task already exists
  }
  
  await deps.db.insert(tasks).values({
    organizationId: ctx.orgId,
    title: `Reorder: ${input.itemName}`,
    description: `Stock at ${input.locationName} is low. Consider ordering ${input.suggestedQty || 'more'} units.`,
    taskType: 'custom',
    required: false,
    templateKey: `inventory:reorder:${input.itemId}`,
    actionKind: 'navigate',
    actionRef: { href: `/inventory/items/${input.itemId}` },
    status: 'todo',
  });
}
```

### Scheduled Count Tasks (Future)

Pattern for recurring count reminders:

```typescript
// Future: Scheduled job creates count tasks
// e.g., Monthly cycle count for location X
await createTask(ctx, deps, {
  title: `Monthly Cycle Count: ${location.name}`,
  description: `Scheduled cycle count for ${location.name}. Due by end of month.`,
  taskType: 'custom',
  templateKey: `inventory:count:${locationId}:${monthKey}`,
  dueOn: endOfMonth,
  actionKind: 'navigate',
  actionRef: { href: `/inventory/counts/new?locationId=${locationId}` },
});
```

---

## Event Emission Implementation

### Service Layer Integration

```typescript
// packages/inventory/src/adjustments.service.ts

export async function postAdjustment(
  ctx: ServiceContext,
  deps: ServiceDeps,
  id: string,
  idempotencyKey: string
): Promise<Adjustment> {
  // ... posting logic ...
  
  await db.transaction(async (tx) => {
    // Create movements and update balances
    for (const line of adjustment.lines) {
      await createMovement(tx, ctx, { ... });
      await updateBalance(tx, ctx, { ... });
    }
    
    // Update adjustment status
    await tx.update(inventoryAdjustments)
      .set({ status: 'posted', postedAt: new Date(), postedBy: ctx.userId })
      .where(eq(inventoryAdjustments.id, id));
  });
  
  // Events emitted AFTER transaction commits
  
  // 1. Emit posted event
  await emitAdjustmentPostedEvent(ctx, {
    adjustmentId: id,
    locationId: adjustment.locationId,
    reason: adjustment.reason,
    lineCount: adjustment.lines.length,
    postedBy: ctx.userId,
  });
  
  // 2. Check low stock for affected items
  for (const line of adjustment.lines) {
    await checkAndAlertLowStock(ctx, deps, line.itemId, adjustment.locationId);
  }
  
  return getAdjustment(ctx, deps, id);
}
```

### Audit Log Integration

All events should also be recorded in the audit log:

```typescript
// packages/database/src/primitives/audit.ts

await recordAuditLog(ctx, deps, {
  entityType: 'inventory_adjustment',
  entityId: adjustmentId,
  action: 'posted',
  details: {
    locationId,
    reason,
    lineCount,
    totalQtyChange,
  },
  userId: ctx.userId,
});
```

---

## Notification Templates

### In-App Notification Examples

**Low Stock:**
```
🔴 Low Stock Alert: Office Chair
Office Chair at Main Warehouse is low (8 EA). Reorder level: 10.
[View Item]
```

**Transfer Received:**
```
📦 Transfer Received
Transfer from Main Warehouse arrived at Retail Store. 5 items received.
[View Transfer]
```

**Count Posted:**
```
📋 Stock Count Posted
Count at Main Warehouse completed. 3 items had variances totaling 15 units.
[View Count]
```

### Email Templates

**Low Stock Digest (Daily):**

```
Subject: Bumara Inventory: 5 items need reordering

Hi {{recipientName}},

The following items at {{organizationName}} are at or below reorder levels:

| Item          | Location       | On Hand | Reorder Level |
|---------------|----------------|---------|---------------|
| Office Chair  | Main Warehouse | 8 EA    | 10 EA         |
| Printer Paper | Retail Store   | 20 Reams| 50 Reams      |
| ...           | ...            | ...     | ...           |

[View Inventory Dashboard]

---
Bumara Compliance Platform
```

**Negative Stock Attempt (Immediate):**

```
Subject: ⚠️ Negative Stock Blocked: Office Chair

Hi {{recipientName}},

An operation was blocked because it would have caused negative stock:

Item: Office Chair
Location: Main Warehouse
Current Stock: 5 EA
Attempted Change: -10 EA
Would Result In: -5 EA

User: John Doe
Operation: Stock Adjustment (ADJ-001)
Time: January 20, 2026 at 2:30 PM

[View Adjustment]

---
Bumara Compliance Platform
```

---

## Related Documentation

- [Workflows](/inventory/05-workflows) — When events are emitted
- [API Spec](/inventory/04-api-spec) — Event triggers
- [Notifications Module](/modules/notifications) — Platform notification system

---

## Open Questions

1. **Email frequency**: Daily digest vs. immediate per alert?
2. **Mobile push**: Add push notifications for critical alerts?
3. **Webhook events**: Expose inventory events to external systems via Svix?
4. **Alert thresholds**: Support multiple thresholds (warning at 50%, critical at reorder level)?
