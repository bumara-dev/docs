---
title: "Notifications UI"
description: "Frontend components for the notification system."
---

## Table of Contents

1. [Overview](#overview)
2. [Component Architecture](#component-architecture)
3. [NotificationBell](#notificationbell)
4. [NotificationCenter](#notificationcenter)
5. [NotificationItem](#notificationitem)
6. [Preferences UI](#preferences-ui)
7. [Hooks](#hooks)
8. [Integration](#integration)

---

## Overview

The notification UI replaces the Knock-based implementation with custom components:

| Component | Purpose |
|-----------|---------|
| `NotificationBell` | Header icon with unread badge |
| `NotificationCenter` | Dropdown/panel with notification list |
| `NotificationItem` | Individual notification row |
| `NotificationPreferences` | Settings page for preferences |

---

## Component Architecture

```
packages/notifications/
├── components/
│   ├── notification-bell.tsx
│   ├── notification-center.tsx
│   ├── notification-item.tsx
│   └── notification-preferences.tsx
├── hooks/
│   ├── use-notifications.ts
│   ├── use-unread-count.ts
│   └── use-notification-preferences.ts
└── index.ts
```

---

## NotificationBell

Header button with unread count badge.

```typescript
// packages/notifications/components/notification-bell.tsx

'use client';

import { useState, useRef } from 'react';
import { Bell } from 'lucide-react';
import { Button } from '@repo/design-system/components/ui/button';
import {
  Popover,
  PopoverContent,
  PopoverTrigger,
} from '@repo/design-system/components/ui/popover';
import { useUnreadCount } from '../hooks/use-unread-count';
import { NotificationCenter } from './notification-center';

export function NotificationBell() {
  const [open, setOpen] = useState(false);
  const { data: unreadCount } = useUnreadCount();
  
  return (
    <Popover open={open} onOpenChange={setOpen}>
      <PopoverTrigger asChild>
        <Button
          variant="ghost"
          size="icon"
          className="relative"
          aria-label={`Notifications${unreadCount?.count ? ` (${unreadCount.count} unread)` : ''}`}
        >
          <Bell className="h-5 w-5" />
          {unreadCount?.count > 0 && (
            <span className="absolute -top-1 -right-1 flex h-5 w-5 items-center justify-center rounded-full bg-red-500 text-xs font-medium text-white">
              {unreadCount.count > 99 ? '99+' : unreadCount.count}
            </span>
          )}
        </Button>
      </PopoverTrigger>
      <PopoverContent 
        className="w-96 p-0" 
        align="end"
        sideOffset={8}
      >
        <NotificationCenter onClose={() => setOpen(false)} />
      </PopoverContent>
    </Popover>
  );
}
```

---

## NotificationCenter

Main notification list with filters and actions.

```typescript
// packages/notifications/components/notification-center.tsx

'use client';

import { useState } from 'react';
import { Check, Filter, Settings } from 'lucide-react';
import { Button } from '@repo/design-system/components/ui/button';
import { Tabs, TabsList, TabsTrigger } from '@repo/design-system/components/ui/tabs';
import { ScrollArea } from '@repo/design-system/components/ui/scroll-area';
import { Skeleton } from '@repo/design-system/components/ui/skeleton';
import { useNotifications } from '../hooks/use-notifications';
import { useMarkAllRead } from '../hooks/use-notifications';
import { NotificationItem } from './notification-item';
import Link from 'next/link';

interface NotificationCenterProps {
  onClose?: () => void;
}

export function NotificationCenter({ onClose }: NotificationCenterProps) {
  const [filter, setFilter] = useState<'all' | 'unread'>('all');
  const [category, setCategory] = useState<string | undefined>();
  
  const { 
    data, 
    isLoading, 
    hasNextPage, 
    fetchNextPage, 
    isFetchingNextPage 
  } = useNotifications({
    unreadOnly: filter === 'unread',
    category,
  });
  
  const { mutate: markAllRead, isPending: isMarkingAllRead } = useMarkAllRead();
  
  const notifications = data?.pages.flatMap((page) => page.data) ?? [];
  const isEmpty = !isLoading && notifications.length === 0;
  
  return (
    <div className="flex flex-col">
      {/* Header */}
      <div className="flex items-center justify-between border-b px-4 py-3">
        <h3 className="font-semibold">Notifications</h3>
        <div className="flex items-center gap-2">
          <Button
            variant="ghost"
            size="sm"
            onClick={() => markAllRead({ category })}
            disabled={isMarkingAllRead}
          >
            <Check className="mr-1 h-4 w-4" />
            Mark all read
          </Button>
          <Button
            variant="ghost"
            size="icon"
            asChild
          >
            <Link href="/settings/notifications" onClick={onClose}>
              <Settings className="h-4 w-4" />
            </Link>
          </Button>
        </div>
      </div>
      
      {/* Filters */}
      <div className="border-b px-4 py-2">
        <Tabs value={filter} onValueChange={(v) => setFilter(v as 'all' | 'unread')}>
          <TabsList className="grid w-full grid-cols-2">
            <TabsTrigger value="all">All</TabsTrigger>
            <TabsTrigger value="unread">Unread</TabsTrigger>
          </TabsList>
        </Tabs>
      </div>
      
      {/* Category filter */}
      <div className="flex gap-1 overflow-x-auto border-b px-4 py-2">
        {['All', 'TASK', 'FILING', 'PAYMENT', 'SUBMISSION'].map((cat) => (
          <Button
            key={cat}
            variant={category === (cat === 'All' ? undefined : cat) ? 'secondary' : 'ghost'}
            size="sm"
            onClick={() => setCategory(cat === 'All' ? undefined : cat)}
          >
            {cat.charAt(0) + cat.slice(1).toLowerCase()}
          </Button>
        ))}
      </div>
      
      {/* Notification list */}
      <ScrollArea className="h-[400px]">
        {isLoading ? (
          <div className="space-y-2 p-4">
            {Array.from({ length: 5 }).map((_, i) => (
              <NotificationSkeleton key={i} />
            ))}
          </div>
        ) : isEmpty ? (
          <EmptyState filter={filter} />
        ) : (
          <div className="divide-y">
            {notifications.map((notification) => (
              <NotificationItem
                key={notification.id}
                notification={notification}
                onClick={onClose}
              />
            ))}
            {hasNextPage && (
              <div className="p-4">
                <Button
                  variant="outline"
                  className="w-full"
                  onClick={() => fetchNextPage()}
                  disabled={isFetchingNextPage}
                >
                  {isFetchingNextPage ? 'Loading...' : 'Load more'}
                </Button>
              </div>
            )}
          </div>
        )}
      </ScrollArea>
    </div>
  );
}

function NotificationSkeleton() {
  return (
    <div className="flex gap-3 p-3">
      <Skeleton className="h-10 w-10 rounded-full" />
      <div className="flex-1 space-y-2">
        <Skeleton className="h-4 w-3/4" />
        <Skeleton className="h-3 w-1/2" />
      </div>
    </div>
  );
}

function EmptyState({ filter }: { filter: 'all' | 'unread' }) {
  return (
    <div className="flex flex-col items-center justify-center py-12 text-center">
      <Bell className="h-12 w-12 text-muted-foreground/50" />
      <p className="mt-4 text-sm text-muted-foreground">
        {filter === 'unread' 
          ? "You're all caught up!"
          : 'No notifications yet'}
      </p>
    </div>
  );
}
```

---

## NotificationItem

Individual notification row with click handling.

```typescript
// packages/notifications/components/notification-item.tsx

'use client';

import { formatDistanceToNow } from 'date-fns';
import { 
  FileText, 
  CheckSquare, 
  CreditCard, 
  Send, 
  Bell,
  Circle 
} from 'lucide-react';
import { cn } from '@repo/design-system/lib/utils';
import { useMarkRead } from '../hooks/use-notifications';
import { useRouter } from 'next/navigation';

interface Notification {
  id: string;
  category: string;
  severity: string;
  title: string;
  body: string;
  deepLink: string | null;
  readAt: string | null;
  createdAt: string;
}

interface NotificationItemProps {
  notification: Notification;
  onClick?: () => void;
}

export function NotificationItem({ notification, onClick }: NotificationItemProps) {
  const router = useRouter();
  const { mutate: markRead } = useMarkRead();
  
  const isUnread = !notification.readAt;
  const Icon = getCategoryIcon(notification.category);
  const severityColor = getSeverityColor(notification.severity);
  
  const handleClick = () => {
    // Mark as read
    if (isUnread) {
      markRead({ id: notification.id });
    }
    
    // Navigate if deep link exists
    if (notification.deepLink) {
      router.push(notification.deepLink);
    }
    
    onClick?.();
  };
  
  return (
    <button
      type="button"
      onClick={handleClick}
      className={cn(
        'flex w-full gap-3 p-4 text-left transition-colors hover:bg-muted/50',
        isUnread && 'bg-muted/30'
      )}
    >
      {/* Icon */}
      <div className={cn(
        'flex h-10 w-10 shrink-0 items-center justify-center rounded-full',
        severityColor.bg
      )}>
        <Icon className={cn('h-5 w-5', severityColor.icon)} />
      </div>
      
      {/* Content */}
      <div className="flex-1 min-w-0">
        <div className="flex items-start justify-between gap-2">
          <p className={cn(
            'text-sm',
            isUnread ? 'font-semibold' : 'font-medium'
          )}>
            {notification.title}
          </p>
          {isUnread && (
            <Circle className="h-2 w-2 shrink-0 fill-blue-500 text-blue-500" />
          )}
        </div>
        <p className="mt-1 text-sm text-muted-foreground line-clamp-2">
          {notification.body}
        </p>
        <p className="mt-1 text-xs text-muted-foreground">
          {formatDistanceToNow(new Date(notification.createdAt), { addSuffix: true })}
        </p>
      </div>
    </button>
  );
}

function getCategoryIcon(category: string) {
  switch (category) {
    case 'TASK':
      return CheckSquare;
    case 'FILING':
      return FileText;
    case 'PAYMENT':
      return CreditCard;
    case 'SUBMISSION':
      return Send;
    default:
      return Bell;
  }
}

function getSeverityColor(severity: string) {
  switch (severity) {
    case 'URGENT':
      return { bg: 'bg-red-100', icon: 'text-red-600' };
    case 'IMPORTANT':
      return { bg: 'bg-amber-100', icon: 'text-amber-600' };
    default:
      return { bg: 'bg-blue-100', icon: 'text-blue-600' };
  }
}
```

---

## Preferences UI

Settings page for notification preferences.

```typescript
// packages/notifications/components/notification-preferences.tsx

'use client';

import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { Button } from '@repo/design-system/components/ui/button';
import { Switch } from '@repo/design-system/components/ui/switch';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@repo/design-system/components/ui/card';
import { Form, FormControl, FormDescription, FormField, FormItem, FormLabel } from '@repo/design-system/components/ui/form';
import { useNotificationPreferences, useUpdatePreferences } from '../hooks/use-notification-preferences';
import { toast } from 'sonner';

const preferencesSchema = z.object({
  emailEnabled: z.boolean(),
  whatsAppEnabled: z.boolean(),
  smsEnabled: z.boolean(),
  quietHoursEnabled: z.boolean(),
});

type PreferencesForm = z.infer<typeof preferencesSchema>;

export function NotificationPreferences() {
  const { data: preferences, isLoading } = useNotificationPreferences();
  const { mutate: updatePreferences, isPending } = useUpdatePreferences();
  
  const form = useForm<PreferencesForm>({
    resolver: zodResolver(preferencesSchema),
    defaultValues: {
      emailEnabled: true,
      whatsAppEnabled: true,
      smsEnabled: false,
      quietHoursEnabled: false,
    },
    values: preferences ? {
      emailEnabled: preferences.emailEnabled,
      whatsAppEnabled: preferences.whatsAppEnabled,
      smsEnabled: preferences.smsEnabled,
      quietHoursEnabled: preferences.quietHoursEnabled,
    } : undefined,
  });
  
  const onSubmit = (data: PreferencesForm) => {
    updatePreferences(data, {
      onSuccess: () => {
        toast.success('Preferences saved');
      },
      onError: () => {
        toast.error('Failed to save preferences');
      },
    });
  };
  
  if (isLoading) {
    return <PreferencesSkeleton />;
  }
  
  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
        {/* Channel Preferences */}
        <Card>
          <CardHeader>
            <CardTitle>Notification Channels</CardTitle>
            <CardDescription>
              Choose how you want to receive notifications
            </CardDescription>
          </CardHeader>
          <CardContent className="space-y-4">
            <FormField
              control={form.control}
              name="emailEnabled"
              render={({ field }) => (
                <FormItem className="flex items-center justify-between rounded-lg border p-4">
                  <div className="space-y-0.5">
                    <FormLabel className="text-base">Email Notifications</FormLabel>
                    <FormDescription>
                      Receive notifications via email
                    </FormDescription>
                  </div>
                  <FormControl>
                    <Switch
                      checked={field.value}
                      onCheckedChange={field.onChange}
                    />
                  </FormControl>
                </FormItem>
              )}
            />
            
            <FormField
              control={form.control}
              name="whatsAppEnabled"
              render={({ field }) => (
                <FormItem className="flex items-center justify-between rounded-lg border p-4">
                  <div className="space-y-0.5">
                    <FormLabel className="text-base">WhatsApp Notifications</FormLabel>
                    <FormDescription>
                      Receive notifications via WhatsApp
                    </FormDescription>
                  </div>
                  <FormControl>
                    <Switch
                      checked={field.value}
                      onCheckedChange={field.onChange}
                    />
                  </FormControl>
                </FormItem>
              )}
            />
            
            <FormField
              control={form.control}
              name="smsEnabled"
              render={({ field }) => (
                <FormItem className="flex items-center justify-between rounded-lg border p-4">
                  <div className="space-y-0.5">
                    <FormLabel className="text-base">SMS Notifications</FormLabel>
                    <FormDescription>
                      Receive urgent notifications via SMS
                    </FormDescription>
                  </div>
                  <FormControl>
                    <Switch
                      checked={field.value}
                      onCheckedChange={field.onChange}
                    />
                  </FormControl>
                </FormItem>
              )}
            />
          </CardContent>
        </Card>
        
        {/* Quiet Hours */}
        <Card>
          <CardHeader>
            <CardTitle>Quiet Hours</CardTitle>
            <CardDescription>
              Pause non-urgent notifications during certain hours
            </CardDescription>
          </CardHeader>
          <CardContent>
            <FormField
              control={form.control}
              name="quietHoursEnabled"
              render={({ field }) => (
                <FormItem className="flex items-center justify-between rounded-lg border p-4">
                  <div className="space-y-0.5">
                    <FormLabel className="text-base">Enable Quiet Hours</FormLabel>
                    <FormDescription>
                      22:00 - 07:00 (Africa/Lusaka)
                    </FormDescription>
                  </div>
                  <FormControl>
                    <Switch
                      checked={field.value}
                      onCheckedChange={field.onChange}
                    />
                  </FormControl>
                </FormItem>
              )}
            />
          </CardContent>
        </Card>
        
        <Button type="submit" disabled={isPending}>
          {isPending ? 'Saving...' : 'Save Preferences'}
        </Button>
      </form>
    </Form>
  );
}
```

---

## Hooks

### useNotifications

```typescript
// packages/notifications/hooks/use-notifications.ts

import { useInfiniteQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { apiClient } from '@repo/api-client';

interface NotificationsParams {
  unreadOnly?: boolean;
  category?: string;
}

export function useNotifications(params: NotificationsParams = {}) {
  return useInfiniteQuery({
    queryKey: ['notifications', params],
    queryFn: async ({ pageParam = 1 }) => {
      const response = await apiClient.notifications.$get({
        query: {
          page: pageParam,
          limit: 20,
          unreadOnly: params.unreadOnly ?? false,
          category: params.category,
        },
      });
      return response.json();
    },
    getNextPageParam: (lastPage) => 
      lastPage.pagination.hasMore ? lastPage.pagination.page + 1 : undefined,
    initialPageParam: 1,
  });
}

export function useMarkRead() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: async ({ id }: { id: string }) => {
      const response = await apiClient.notifications[':id'].read.$post({
        param: { id },
      });
      return response.json();
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['notifications'] });
      queryClient.invalidateQueries({ queryKey: ['unread-count'] });
    },
  });
}

export function useMarkAllRead() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: async ({ category }: { category?: string }) => {
      const response = await apiClient.notifications['read-all'].$post({
        json: { category },
      });
      return response.json();
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['notifications'] });
      queryClient.invalidateQueries({ queryKey: ['unread-count'] });
    },
  });
}
```

### useUnreadCount

```typescript
// packages/notifications/hooks/use-unread-count.ts

import { useQuery } from '@tanstack/react-query';
import { apiClient } from '@repo/api-client';

export function useUnreadCount() {
  return useQuery({
    queryKey: ['unread-count'],
    queryFn: async () => {
      const response = await apiClient.notifications['unread-count'].$get();
      return response.json();
    },
    refetchInterval: 30000, // Poll every 30 seconds
  });
}
```

---

## Integration

### Replace Knock Provider

```typescript
// apps/app/app/(authenticated)/components/notifications-provider.tsx

'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import type { ReactNode } from 'react';

const queryClient = new QueryClient();

interface NotificationsProviderProps {
  children: ReactNode;
}

export function NotificationsProvider({ children }: NotificationsProviderProps) {
  // React Query provider is likely already in the app
  // This component can be simplified or removed
  return children;
}
```

### Header Integration

```typescript
// apps/app/components/layout/header.tsx

import { NotificationBell } from '@repo/notifications';

export function Header() {
  return (
    <header className="...">
      {/* ... other header content ... */}
      <NotificationBell />
    </header>
  );
}
```

### Settings Page

```typescript
// apps/app/app/(authenticated)/(home)/(general)/settings/notifications/page.tsx

import { NotificationPreferences } from '@repo/notifications';

export default function NotificationsSettingsPage() {
  return (
    <div className="container max-w-2xl py-8">
      <h1 className="text-2xl font-bold mb-6">Notification Settings</h1>
      <NotificationPreferences />
    </div>
  );
}
```

---

## Package Export

```typescript
// packages/notifications/index.ts

// Components
export { NotificationBell } from './components/notification-bell';
export { NotificationCenter } from './components/notification-center';
export { NotificationItem } from './components/notification-item';
export { NotificationPreferences } from './components/notification-preferences';

// Hooks
export { useNotifications, useMarkRead, useMarkAllRead } from './hooks/use-notifications';
export { useUnreadCount } from './hooks/use-unread-count';
export { useNotificationPreferences, useUpdatePreferences } from './hooks/use-notification-preferences';
```

