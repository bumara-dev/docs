---
title: "Inventory Module - UI/UX Specification"
description: "Page structure, components, forms, and user experience patterns for the Inventory Module."
---

## Navigation Structure

### Primary Sidebar

The Inventory module appears in the "Apps" group of the primary sidebar:

```typescript
// apps/app/config/modules.ts (already registered)
{
  id: 'inventory',
  title: 'Inventory',
  url: '/inventory',
  icon: PackageIcon,
  group: 'Apps',
}
```

### Secondary Sidebar

When a user navigates to `/inventory/*`, the secondary sidebar shows:

```typescript
// apps/app/config/secondary-sidebar/inventory-sidebar.ts
import type { NavGroupType } from '@/types/navigation.types';
import {
  LayoutDashboard,
  Package,
  Warehouse,
  History,
  ClipboardList,
  ArrowLeftRight,
  ClipboardCheck,
  Settings,
} from 'lucide-react';

export const INVENTORY_MENU: NavGroupType[] = [
  {
    title: 'Overview',
    items: [
      { title: 'Dashboard', url: '/inventory', icon: LayoutDashboard },
    ],
  },
  {
    title: 'Inventory',
    items: [
      { title: 'Items', url: '/inventory/items', icon: Package },
      { title: 'Stock Levels', url: '/inventory/stock', icon: Warehouse },
      { title: 'Movements', url: '/inventory/movements', icon: History },
    ],
  },
  {
    title: 'Operations',
    items: [
      { title: 'Adjustments', url: '/inventory/adjustments', icon: ClipboardList },
      { title: 'Transfers', url: '/inventory/transfers', icon: ArrowLeftRight },
      { title: 'Stock Counts', url: '/inventory/counts', icon: ClipboardCheck },
    ],
  },
  {
    title: 'Configuration',
    items: [
      { 
        title: 'Settings', 
        url: '/inventory/settings', 
        icon: Settings,
        // Admin-only, shown but disabled for non-admins
      },
    ],
  },
];
```

---

## Page Structure

### Directory Layout

```
apps/app/app/(authenticated)/(modules)/inventory/
├── page.tsx                    # Dashboard
├── layout.tsx                  # Inventory layout with secondary sidebar
├── items/
│   ├── page.tsx                # Items list
│   ├── new/
│   │   └── page.tsx            # Create item
│   └── [id]/
│       ├── page.tsx            # Item detail
│       └── edit/
│           └── page.tsx        # Edit item
├── stock/
│   └── page.tsx                # Stock balances grid
├── movements/
│   └── page.tsx                # Movements ledger
├── adjustments/
│   ├── page.tsx                # Adjustments list
│   ├── new/
│   │   └── page.tsx            # Create adjustment
│   └── [id]/
│       └── page.tsx            # Adjustment detail
├── transfers/
│   ├── page.tsx                # Transfers list
│   ├── new/
│   │   └── page.tsx            # Create transfer
│   └── [id]/
│       └── page.tsx            # Transfer detail
├── counts/
│   ├── page.tsx                # Counts list
│   ├── new/
│   │   └── page.tsx            # Create count
│   └── [id]/
│       └── page.tsx            # Count detail (with entry grid)
└── settings/
    └── page.tsx                # Locations, UoM settings
```

---

## Screen Specifications

### 1. Inventory Dashboard (`/inventory`)

**Purpose:** Overview of inventory health and quick actions.

**Layout:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Inventory Dashboard                                    [+ Quick Action] │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐ │
│  │  Total Items │  │  Low Stock   │  │  Stock Value │  │  Locations  │ │
│  │     156      │  │      12      │  │  K 45,230    │  │      3      │ │
│  │  +5 this mo  │  │  ⚠️ Alert    │  │  (optional)  │  │             │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └─────────────┘ │
│                                                                          │
│  ┌────────────────────────────────┐  ┌────────────────────────────────┐ │
│  │  Low Stock Items               │  │  Recent Movements              │ │
│  │  ┌────────────────────────┐   │  │  ┌────────────────────────┐   │ │
│  │  │ Office Chair    5/10   │   │  │  │ +50 Paper Reams        │   │ │
│  │  │ Printer Paper   20/50  │   │  │  │ -5 Office Chairs       │   │ │
│  │  │ Stapler         3/10   │   │  │  │ +100 Pens              │   │ │
│  │  └────────────────────────┘   │  │  └────────────────────────┘   │ │
│  │  [View All Low Stock]          │  │  [View All Movements]          │ │
│  └────────────────────────────────┘  └────────────────────────────────┘ │
│                                                                          │
│  Quick Actions                                                           │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐         │
│  │ New Adjustment  │  │  New Transfer   │  │  New Count      │         │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Components:**

```typescript
// apps/app/zones/inventory/modules/dashboard/
├── inventory-dashboard.tsx
├── stats-cards.tsx
├── low-stock-widget.tsx
├── recent-movements-widget.tsx
└── quick-actions.tsx
```

### 2. Items List (`/inventory/items`)

**Purpose:** Browse, search, and manage inventory items.

**Features:**
- Search by name, SKU, barcode
- Filter by category, status, low stock
- Sortable columns
- Bulk actions (archive)

**Layout:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Items                                              [+ Add Item]        │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 🔍 Search items...              [Category ▼] [Status ▼] [More ▼] │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │ □  Name              SKU        On Hand   Reorder   Status    ⋮    ││
│  ├─────────────────────────────────────────────────────────────────────┤│
│  │ □  Office Chair      CHAIR-001  45 EA     10        Active    ⋮    ││
│  │ □  Printer Paper     PAPER-A4   20 Reams  50        ⚠️ Low    ⋮    ││
│  │ □  Stapler           STPL-001   3 EA      10        ⚠️ Low    ⋮    ││
│  │ □  Desk Lamp         LAMP-001   25 EA     5         Active    ⋮    ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                          │
│  Showing 1-20 of 156 items                         [< 1 2 3 ... 8 >]   │
└─────────────────────────────────────────────────────────────────────────┘
```

**Row Actions (⋮ menu):**
- View details
- Edit (Manager+)
- View stock by location
- Archive (Admin)

### 3. Item Detail (`/inventory/items/[id]`)

**Purpose:** View item details, stock levels, and history.

**Tabs:**
1. **Overview** — Basic info, settings
2. **Stock** — Balance by location
3. **History** — Recent movements

**Layout:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│  ← Back    Office Chair                              [Edit] [Archive]   │
├─────────────────────────────────────────────────────────────────────────┤
│  SKU: CHAIR-001  •  Barcode: 1234567890123  •  Category: Furniture     │
├─────────────────────────────────────────────────────────────────────────┤
│  [Overview]  [Stock]  [History]                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Stock by Location                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Location          On Hand    Reserved   Available   Status     │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │  Main Warehouse    30 EA      0          30          ✓ OK       │   │
│  │  Retail Store      15 EA      2          13          ✓ OK       │   │
│  │  Delivery Van      0 EA       0          0           —          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  Settings                                                                │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Default UoM:     EA (Each)                                      │   │
│  │  Reorder Level:   10                                             │   │
│  │  Reorder Qty:     50                                             │   │
│  │  Track Inventory: Yes                                            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4. Stock Levels (`/inventory/stock`)

**Purpose:** View stock across all items and locations.

**Features:**
- Pivot view: Items × Locations
- Filter by location, category
- Low stock highlighting
- Export to CSV

**Layout:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Stock Levels                                          [Export CSV]     │
├─────────────────────────────────────────────────────────────────────────┤
│  [Location: All ▼]  [Category: All ▼]  [□ Low stock only]              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │ Item               Main Warehouse   Retail Store   Delivery Van     ││
│  ├─────────────────────────────────────────────────────────────────────┤│
│  │ Office Chair       30 EA            15 EA          0                ││
│  │ Printer Paper      100 Reams        ⚠️ 20 Reams   0                ││
│  │ Stapler            10 EA            ⚠️ 3 EA       0                ││
│  │ Desk Lamp          20 EA            5 EA           0                ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5. Movements Ledger (`/inventory/movements`)

**Purpose:** View complete audit trail of stock changes.

**Features:**
- Filter by date range, item, location, type
- Click to view source document
- Infinite scroll or pagination

**Layout:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Stock Movements                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│  Date Range: [Last 30 days ▼]  Item: [All ▼]  Location: [All ▼]        │
│  Type: [All ▼]                                                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │ Date        Item           Location       Type         Qty    By   ││
│  ├─────────────────────────────────────────────────────────────────────┤│
│  │ Jan 20      Office Chair   Main WH        Adjustment   +50    JD   ││
│  │ Jan 19      Printer Paper  Retail Store   Transfer In  +30    MK   ││
│  │ Jan 19      Printer Paper  Main WH        Transfer Out -30    MK   ││
│  │ Jan 18      Stapler        Main WH        Count Var    -2     JD   ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                          │
│  [Load more...]                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6. Adjustments (`/inventory/adjustments`)

**List View:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Adjustments                                       [+ New Adjustment]   │
├─────────────────────────────────────────────────────────────────────────┤
│  Status: [All ▼]  Location: [All ▼]  Reason: [All ▼]                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │ Date        Location       Reason         Lines   Status     ⋮     ││
│  ├─────────────────────────────────────────────────────────────────────┤│
│  │ Jan 20      Main WH        Opening Bal    15      ✓ Posted   ⋮     ││
│  │ Jan 18      Retail Store   Damage         3       ✓ Posted   ⋮     ││
│  │ Jan 15      Main WH        Correction     1       📝 Draft   ⋮     ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Create/Edit View:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│  New Adjustment                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Location *         [Main Warehouse ▼]                                   │
│  Reason *           [Damage ▼]                                           │
│  Notes              [Water damage from roof leak                    ]    │
│                                                                          │
│  ─────────────────────────────────────────────────────────────────────  │
│  Items                                                     [+ Add Item]  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Item *                    Qty *      UoM         Cost     ✕     │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │ [Office Chair ▼]          [-5    ]   [EA ▼]      [     ]  ✕     │   │
│  │ [Desk Lamp ▼]             [-2    ]   [EA ▼]      [     ]  ✕     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  📎 Attachments (0)                                  [Upload Evidence]   │
│                                                                          │
│  ─────────────────────────────────────────────────────────────────────  │
│                                        [Cancel]  [Save Draft]  [Post]   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7. Transfers (`/inventory/transfers`)

**Create View:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│  New Transfer                                                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  From Location *    [Main Warehouse ▼]                                   │
│  To Location *      [Retail Store ▼]                                     │
│  Notes              [Weekly store replenishment                     ]    │
│                                                                          │
│  ─────────────────────────────────────────────────────────────────────  │
│  Items                                                     [+ Add Item]  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Item *              Available    Qty *      UoM         ✕        │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │ [Office Chair ▼]    30 EA        [10    ]   [EA ▼]      ✕        │   │
│  │ [Printer Paper ▼]   100 Reams    [20    ]   [Reams ▼]   ✕        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  📎 Attachments (0)                              [Upload Delivery Note]  │
│                                                                          │
│  ─────────────────────────────────────────────────────────────────────  │
│                                   [Cancel]  [Save Draft]  [Ship & Receive]│
└─────────────────────────────────────────────────────────────────────────┘
```

### 8. Stock Counts (`/inventory/counts/[id]`)

**Count Entry View (Optimized for Speed):**

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Stock Count - Main Warehouse                          Status: Counting │
├─────────────────────────────────────────────────────────────────────────┤
│  Progress: 45/60 items counted                         [Complete Count] │
├─────────────────────────────────────────────────────────────────────────┤
│  🔍 Search items...                                    [□ Show variances]│
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │ Item               SKU          System    Counted   Variance        ││
│  ├─────────────────────────────────────────────────────────────────────┤│
│  │ Office Chair       CHAIR-001    30 EA     [30    ]  0               ││
│  │ Printer Paper      PAPER-A4     100       [98    ]  ⚠️ -2           ││
│  │ Stapler            STPL-001     10 EA     [      ]  —               ││
│  │ Desk Lamp          LAMP-001     20 EA     [20    ]  0               ││
│  │ Filing Cabinet     CAB-001      5 EA      [6     ]  ⚠️ +1           ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                          │
│  Keyboard navigation: Tab to move, Enter to confirm                      │
└─────────────────────────────────────────────────────────────────────────┘
```

**UX Optimizations:**
- Tab key moves to next item
- Enter confirms and moves to next
- Auto-focus on count input
- Visual highlight on variance
- Sticky header during scroll

### 9. Settings (`/inventory/settings`)

**Admin-only page for configuration:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Inventory Settings                                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Locations                                            [+ Add Location]   │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Name              Type        Default   Items    ⋮                │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │ Main Warehouse    Warehouse   ✓         120      ⋮                │   │
│  │ Retail Store      Store                 45       ⋮                │   │
│  │ Delivery Van      Van                   5        ⋮                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│  Units of Measure                                          [+ Add Unit]  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Code    Name           Precision    Source     ⋮                  │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │ EA      Each           0            System     —                  │   │
│  │ KG      Kilogram       3            System     —                  │   │
│  │ BOX     Box            0            Custom     ⋮                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                          │
│  Unit Conversions                                    [+ Add Conversion]  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ From      To        Multiplier    Bidirectional   ⋮              │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │ BOX       EA        12            Yes             ⋮              │   │
│  │ KG        G         1000          Yes             ⋮              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Component Library

### Reusable Components

```
apps/app/zones/inventory/modules/
├── shared/
│   ├── item-select.tsx           # Searchable item picker
│   ├── location-select.tsx       # Location dropdown
│   ├── uom-select.tsx            # Unit of measure picker
│   ├── quantity-input.tsx        # Numeric input with UoM
│   ├── stock-badge.tsx           # Low stock / OK badge
│   ├── status-badge.tsx          # Draft/Posted/Void badges
│   └── movement-type-badge.tsx   # Movement type indicator
├── dashboard/
│   ├── inventory-dashboard.tsx
│   ├── stats-cards.tsx
│   ├── low-stock-widget.tsx
│   └── recent-movements-widget.tsx
├── items/
│   ├── items-list.tsx
│   ├── item-form.tsx
│   ├── item-detail.tsx
│   └── item-stock-table.tsx
├── stock/
│   ├── stock-grid.tsx
│   └── movements-table.tsx
├── adjustments/
│   ├── adjustments-list.tsx
│   ├── adjustment-form.tsx
│   ├── adjustment-lines-editor.tsx
│   └── adjustment-detail.tsx
├── transfers/
│   ├── transfers-list.tsx
│   ├── transfer-form.tsx
│   ├── transfer-lines-editor.tsx
│   └── transfer-detail.tsx
├── counts/
│   ├── counts-list.tsx
│   ├── count-form.tsx
│   ├── count-entry-grid.tsx      # Keyboard-optimized entry
│   └── count-detail.tsx
└── settings/
    ├── locations-settings.tsx
    ├── units-settings.tsx
    └── conversions-settings.tsx
```

### Form Patterns

Following existing patterns from `apps/app/components/ui/form/`:

```typescript
// Example: Item Form
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { Form, FormField } from '@repo/design-system/components/ui/form';
import { TextField, SelectField } from '@/components/ui/form';

const itemFormSchema = z.object({
  name: z.string().min(1, 'Name is required').max(200),
  sku: z.string().max(50).optional(),
  barcode: z.string().max(50).optional(),
  categoryId: z.string().uuid().optional(),
  defaultUomId: z.string().uuid({ message: 'Unit of measure is required' }),
  trackInventory: z.boolean().default(true),
  reorderLevel: z.string().regex(/^\d+(\.\d{1,4})?$/).optional(),
  reorderQty: z.string().regex(/^\d+(\.\d{1,4})?$/).optional(),
});

export function ItemForm({ defaultValues, onSubmit }) {
  const form = useForm({
    resolver: zodResolver(itemFormSchema),
    defaultValues: {
      trackInventory: true,
      ...defaultValues,
    },
  });

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)}>
        <TextField
          control={form.control}
          name="name"
          label="Item Name"
          required
        />
        <TextField
          control={form.control}
          name="sku"
          label="SKU"
        />
        {/* ... more fields */}
      </form>
    </Form>
  );
}
```

---

## UI States

### Loading States

```typescript
// Skeleton for lists
<ItemsListSkeleton count={10} />

// Skeleton for detail pages
<ItemDetailSkeleton />

// Loading spinner for actions
<Button disabled={isPending}>
  {isPending ? <Spinner className="mr-2" /> : null}
  Post Adjustment
</Button>
```

### Empty States

```typescript
// No items yet
<EmptyState
  icon={Package}
  title="No items yet"
  description="Get started by adding your first inventory item."
  action={
    <Button onClick={() => router.push('/inventory/items/new')}>
      <Plus className="mr-2 h-4 w-4" />
      Add Item
    </Button>
  }
/>

// No search results
<EmptyState
  icon={Search}
  title="No items found"
  description="Try adjusting your search or filters."
/>

// No movements
<EmptyState
  icon={History}
  title="No movements yet"
  description="Stock movements will appear here after posting operations."
/>
```

### Error States

```typescript
// API error
<ErrorState
  title="Failed to load items"
  description={error.message}
  action={
    <Button variant="outline" onClick={() => refetch()}>
      Try Again
    </Button>
  }
/>

// Form validation error
<Alert variant="destructive">
  <AlertCircle className="h-4 w-4" />
  <AlertTitle>Validation Error</AlertTitle>
  <AlertDescription>{error.message}</AlertDescription>
</Alert>

// Business rule error (negative stock)
<Alert variant="warning">
  <AlertTriangle className="h-4 w-4" />
  <AlertTitle>Cannot Post Adjustment</AlertTitle>
  <AlertDescription>
    This adjustment would result in negative stock for "Office Chair" at Main Warehouse.
    Current: 5, Adjustment: -10.
  </AlertDescription>
</Alert>
```

---

## Permissions-Driven UI

### Role-Based Visibility

```typescript
// In component
import { usePermissions } from '@/lib/hooks/use-permissions';

export function ItemActions({ item }) {
  const { hasRole } = usePermissions();
  const canEdit = hasRole(['manager', 'admin']);
  const canArchive = hasRole(['admin']);

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="ghost" size="icon">
          <MoreHorizontal />
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent>
        <DropdownMenuItem>View Details</DropdownMenuItem>
        
        {canEdit && (
          <DropdownMenuItem>Edit</DropdownMenuItem>
        )}
        
        {canArchive && (
          <DropdownMenuSeparator />
          <DropdownMenuItem className="text-destructive">
            Archive
          </DropdownMenuItem>
        )}
      </DropdownMenuContent>
    </DropdownMenu>
  );
}
```

### Disabled States with Tooltips

```typescript
// Settings link in sidebar (disabled for non-admins)
{
  title: 'Settings',
  url: '/inventory/settings',
  icon: Settings,
  disabled: !hasRole(['admin']),
  disabledReason: 'Admin access required',
}

// Render with tooltip
<SidebarMenuItem>
  {item.disabled ? (
    <Tooltip>
      <TooltipTrigger asChild>
        <span className="flex items-center gap-2 opacity-50 cursor-not-allowed">
          <item.icon className="h-4 w-4" />
          {item.title}
        </span>
      </TooltipTrigger>
      <TooltipContent>{item.disabledReason}</TooltipContent>
    </Tooltip>
  ) : (
    <Link href={item.url}>...</Link>
  )}
</SidebarMenuItem>
```

---

## Mobile Responsiveness

### Responsive Table Pattern

```typescript
// Desktop: Full table
// Mobile: Card layout

export function ItemsList({ items }) {
  return (
    <>
      {/* Desktop table */}
      <div className="hidden md:block">
        <Table>
          <TableHeader>...</TableHeader>
          <TableBody>
            {items.map(item => <ItemRow key={item.id} item={item} />)}
          </TableBody>
        </Table>
      </div>
      
      {/* Mobile cards */}
      <div className="md:hidden space-y-4">
        {items.map(item => <ItemCard key={item.id} item={item} />)}
      </div>
    </>
  );
}
```

---

## Related Documentation

- [Architecture](/inventory/02-architecture) — Component locations
- [API Spec](/inventory/04-api-spec) — Data fetching endpoints
- [Workflows](/inventory/05-workflows) — Operation flows

---

## Open Questions

1. **Barcode scanning**: Support mobile camera scanning in MVP?
2. **Offline mode**: Cache data for offline viewing?
3. **Bulk import**: CSV import UI for items?
4. **Print**: Print count sheets, movement reports?
