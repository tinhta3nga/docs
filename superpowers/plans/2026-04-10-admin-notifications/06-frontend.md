# Phase 6: marketplace-admin — Frontend (Bell + Dropdown + Notification Center)

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` or `superpowers:executing-plans`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the hardcoded bell in the admin header with a live, real-time notification bell. Add a dropdown showing the latest 20 notifications, and a full notification center page at `/dashboard/notifications`.

**Tech Stack:** Next.js 15, React 19, TypeScript, TanStack Query v5, Radix UI (Popover), Tailwind CSS 4, shadcn/ui components, Sonner toast

**Rule files to read first:** `workspace-context/rules/05-nextjs-frontend-shared.md`, `workspace-context/rules/07-marketplace-admin.md`

**Key constraints from rules:**
- Use **React Query** for all server state (no Redux for notification data)
- Use **Radix UI Popover** for the dropdown (shadcn/ui wrapper)
- Use **shadcn/ui** + **Tailwind** for the notification center page (NOT MUI DataGrid — this is a list, not a table)
- Ant Design `notification` API for multi-line errors; `sonner` for quick toasts
- `"use client"` directive on all interactive components
- Token in `localStorage["userData"].token`
- API base URL from `window.RUNTIME_CONFIG?.NEXT_PUBLIC_API_ENDPOINT` (runtime config pattern)

**Reference files to read:**
- `marketplace-admin/components/dashboard/header.tsx` — the file to edit
- `marketplace-admin/app/api/instance.ts` — understand the lazy singleton + runtime config pattern
- `marketplace-admin/app/api/admin.ts` — example API module to copy pattern from
- `marketplace-admin/app/provider.tsx` — confirm QueryClientProvider is present
- `marketplace-admin/components/ui/popover.tsx` — confirm Popover is available

**Depends on:** Phase 4 (API endpoints must exist)

---

## File Map

| Action | File |
|---|---|
| Create | `marketplace-admin/app/api/notification.ts` |
| Create | `marketplace-admin/hooks/useNotificationStream.ts` |
| Edit | `marketplace-admin/components/dashboard/header.tsx` |
| Create | `marketplace-admin/components/dashboard/NotificationDropdown.tsx` |
| Create | `marketplace-admin/app/dashboard/(dashboard)/notifications/page.tsx` |

---

## Task 6.1 — Create notification API module

**File:** `marketplace-admin/app/api/notification.ts`

- [ ] **Read** `marketplace-admin/app/api/instance.ts` to understand the singleton + runtime config pattern
- [ ] **Read** one other API module (e.g., `admin.ts`) to see the full pattern
- [ ] **Create** the file — follow the lazy singleton pattern exactly:

```typescript
import axios, { AxiosInstance } from 'axios';

let instance: AxiosInstance | null = null;

function getApiEndpoint(): string {
  if (typeof window !== 'undefined' && window.RUNTIME_CONFIG?.NEXT_PUBLIC_API_ENDPOINT?.trim()) {
    let endpoint = window.RUNTIME_CONFIG.NEXT_PUBLIC_API_ENDPOINT.trim();
    if (!endpoint.replace(/\/$/, '').endsWith('/v1')) {
      endpoint = endpoint.replace(/\/$/, '') + '/v1';
    }
    return endpoint;
  }
  let endpoint = process.env.NEXT_PUBLIC_API_ENDPOINT || '';
  if (endpoint && !endpoint.replace(/\/$/, '').endsWith('/v1')) {
    endpoint = endpoint.replace(/\/$/, '') + '/v1';
  }
  return endpoint;
}

function getInstance(): AxiosInstance {
  if (!instance) {
    instance = axios.create({ baseURL: getApiEndpoint() });
    instance.interceptors.request.use((config) => {
      config.baseURL = getApiEndpoint(); // update on each request for runtime config
      const data = JSON.parse(localStorage.getItem('userData') || '{}');
      if (data?.token) {
        config.headers.Authorization = `Bearer ${data.token}`;
      }
      return config;
    });
    instance.interceptors.response.use(
      (res) => res,
      (error) => Promise.reject(error?.response?.data ?? error),
    );
  }
  return instance;
}

export interface AdminNotificationItem {
  id: string;
  event_id: string;
  admin_id: string;
  notification_type: string;
  category: string;
  title: string;
  body: string;
  action_url: string;
  entity_type?: string;
  entity_id?: string;
  read_at?: string | null;
  created_at: string;
  metadata?: Record<string, unknown>;
}

export interface ListNotificationsParams {
  limit?: number;
  offset?: number;
  unread_only?: boolean;
}

export interface ListNotificationsResponse {
  success: boolean;
  paging: { current: number; hasNext: boolean; itemNum: number };
  data: AdminNotificationItem[];
}

export interface UnreadCountResponse {
  success: boolean;
  data: { unread_count: number };
}

export const notificationApi = {
  list(params: ListNotificationsParams = {}) {
    return getInstance().get<ListNotificationsResponse>('/notifications', { params });
  },
  unreadCount() {
    return getInstance().get<UnreadCountResponse>('/notifications/unread-count');
  },
  markRead(id: string) {
    return getInstance().patch(`/notifications/${id}/read`);
  },
  markAllRead() {
    return getInstance().post('/notifications/read-all');
  },
};
```

> **Do NOT add an SSE call here** — SSE uses the native `EventSource` API, not axios.

- [ ] **Verify:** TypeScript compilation passes (`npm run build` or `tsc --noEmit`)

---

## Task 6.2 — Create SSE stream hook

**File:** `marketplace-admin/hooks/useNotificationStream.ts`

This hook opens a persistent SSE connection to `/notifications/stream?token=<JWT>`. On new notifications, it invalidates React Query caches so the bell and dropdown auto-refresh. It reconnects automatically on error.

- [ ] **Read** existing hooks in `marketplace-admin/hooks/` for patterns
- [ ] **Read** `marketplace-admin/lib/auth.ts` (or equivalent) to find the function that reads the token from localStorage
- [ ] **Create** the file:

```typescript
'use client';

import { useEffect, useRef, useCallback } from 'react';
import { useQueryClient } from '@tanstack/react-query';
import { toast } from 'sonner';

function getApiBase(): string {
  if (typeof window !== 'undefined' && window.RUNTIME_CONFIG?.NEXT_PUBLIC_API_ENDPOINT?.trim()) {
    let ep = window.RUNTIME_CONFIG.NEXT_PUBLIC_API_ENDPOINT.trim().replace(/\/$/, '');
    if (!ep.endsWith('/v1')) ep += '/v1';
    return ep;
  }
  let ep = (process.env.NEXT_PUBLIC_API_ENDPOINT || '').replace(/\/$/, '');
  if (ep && !ep.endsWith('/v1')) ep += '/v1';
  return ep;
}

function getToken(): string | null {
  if (typeof window === 'undefined') return null;
  try {
    const data = JSON.parse(localStorage.getItem('userData') || '{}');
    return data?.token ?? null;
  } catch {
    return null;
  }
}

/**
 * Opens a persistent SSE connection to /notifications/stream.
 * Invalidates React Query caches on new notifications.
 * Auto-reconnects with 10s delay on error.
 *
 * @param onNew optional callback called with the raw notification data on each new event
 */
export function useNotificationStream(onNew?: (data: unknown) => void) {
  const esRef = useRef<EventSource | null>(null);
  const reconnectTimer = useRef<ReturnType<typeof setTimeout> | null>(null);
  const queryClient = useQueryClient();

  const connect = useCallback(() => {
    const token = getToken();
    if (!token) return;

    const url = `${getApiBase()}/notifications/stream?token=${encodeURIComponent(token)}`;
    const es = new EventSource(url);
    esRef.current = es;

    es.addEventListener('notification', (e: MessageEvent) => {
      try {
        const data = JSON.parse(e.data);
        // Invalidate both query keys so bell count and dropdown list refresh
        queryClient.invalidateQueries({ queryKey: ['notification-unread-count'] });
        queryClient.invalidateQueries({ queryKey: ['notifications'] });
        onNew?.(data?.data ?? data);
      } catch {
        // Malformed event — ignore
      }
    });

    es.onerror = () => {
      es.close();
      esRef.current = null;
      // Reconnect after 10s
      reconnectTimer.current = setTimeout(connect, 10_000);
    };
  }, [queryClient, onNew]);

  useEffect(() => {
    connect();
    return () => {
      esRef.current?.close();
      if (reconnectTimer.current) clearTimeout(reconnectTimer.current);
    };
  }, [connect]);
}
```

- [ ] **Verify:** TypeScript compilation passes

---

## Task 6.3 — Update header: replace hardcoded bell

**File:** `marketplace-admin/components/dashboard/header.tsx`

- [ ] **Read the full file** before making any changes
- [ ] **Locate** the hardcoded bell block:
  ```tsx
  <button className='relative h-6 w-6'>
    <Image src='/icons/Bell.svg' alt='notification' fill />
    <span ...>35</span>
  </button>
  ```
- [ ] **Add imports** at the top of the file:
  ```typescript
  import { useQuery } from '@tanstack/react-query';
  import { notificationApi } from '@/app/api/notification';
  import { useNotificationStream } from '@/hooks/useNotificationStream';
  import { NotificationDropdown } from '@/components/dashboard/NotificationDropdown';
  ```
- [ ] **Add** these hooks inside the component (after existing hooks):
  ```typescript
  // Live unread count — refetches every 30s as polling fallback
  const { data: unreadData } = useQuery({
    queryKey: ['notification-unread-count'],
    queryFn: () => notificationApi.unreadCount().then((r) => r.data),
    refetchInterval: 30_000,
    staleTime: 10_000,
  });

  // SSE real-time connection — invalidates cache on new events
  useNotificationStream();

  const unreadCount = unreadData?.data?.unread_count ?? 0;
  const displayCount = unreadCount > 99 ? '99+' : unreadCount > 0 ? String(unreadCount) : null;
  ```
- [ ] **Replace** the hardcoded bell button block with `<NotificationDropdown>`:
  ```tsx
  <NotificationDropdown unreadCount={unreadCount} displayCount={displayCount} />
  ```
- [ ] **Verify:** `npm run build` passes (or `npm run lint` at minimum)

---

## Task 6.4 — Create NotificationDropdown component

**File:** `marketplace-admin/components/dashboard/NotificationDropdown.tsx`

- [ ] **Read** `marketplace-admin/components/ui/popover.tsx` to confirm `Popover`, `PopoverTrigger`, `PopoverContent` are available
- [ ] **Read** `marketplace-admin/components/ui/scroll-area.tsx` if it exists — use it for the scrollable list
- [ ] **Create** the file:

```tsx
'use client';

import { useRouter } from 'next/navigation';
import Image from 'next/image';
import { formatDistanceToNow } from 'date-fns';
import { vi } from 'date-fns/locale';
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query';

import { Popover, PopoverContent, PopoverTrigger } from '@/components/ui/popover';
import { notificationApi, AdminNotificationItem } from '@/app/api/notification';

interface NotificationDropdownProps {
  unreadCount: number;
  displayCount: string | null;
}

export function NotificationDropdown({ unreadCount, displayCount }: NotificationDropdownProps) {
  const router = useRouter();
  const queryClient = useQueryClient();

  // Fetch latest 20 notifications for the dropdown
  const { data, isLoading } = useQuery({
    queryKey: ['notifications', { limit: 20, offset: 0 }],
    queryFn: () => notificationApi.list({ limit: 20, offset: 0 }).then((r) => r.data),
    staleTime: 10_000,
  });

  const notifications: AdminNotificationItem[] = data?.data ?? [];

  // Mark one as read
  const markReadMutation = useMutation({
    mutationFn: (id: string) => notificationApi.markRead(id),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['notification-unread-count'] });
      queryClient.invalidateQueries({ queryKey: ['notifications'] });
    },
  });

  // Mark all as read
  const markAllReadMutation = useMutation({
    mutationFn: () => notificationApi.markAllRead(),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['notification-unread-count'] });
      queryClient.invalidateQueries({ queryKey: ['notifications'] });
    },
  });

  const handleNotificationClick = (item: AdminNotificationItem) => {
    if (!item.read_at) {
      markReadMutation.mutate(item.id);
    }
    if (item.action_url) {
      router.push(`/dashboard${item.action_url}`);
    }
  };

  return (
    <Popover>
      <PopoverTrigger asChild>
        <button className="relative h-6 w-6" aria-label={`${unreadCount} thông báo chưa đọc`}>
          <Image src="/icons/Bell.svg" alt="notification" fill />
          {displayCount && (
            <span className="absolute -top-2 -right-3.5 flex h-4 min-w-[20px] items-center justify-center rounded-full bg-[#EE3B2B] px-2 py-1 text-[0.6875rem] leading-none font-semibold text-white shadow-sm">
              {displayCount}
            </span>
          )}
        </button>
      </PopoverTrigger>

      <PopoverContent className="w-96 p-0" align="end">
        {/* Header */}
        <div className="flex items-center justify-between border-b px-4 py-3">
          <span className="font-semibold text-sm">Thông báo</span>
          {unreadCount > 0 && (
            <button
              className="text-xs text-blue-600 hover:underline"
              onClick={() => markAllReadMutation.mutate()}
              disabled={markAllReadMutation.isPending}
            >
              Đánh dấu tất cả đã đọc
            </button>
          )}
        </div>

        {/* Notification list */}
        <div className="max-h-[400px] overflow-y-auto">
          {isLoading && (
            <div className="px-4 py-8 text-center text-sm text-gray-400">Đang tải...</div>
          )}
          {!isLoading && notifications.length === 0 && (
            <div className="px-4 py-8 text-center text-sm text-gray-400">
              Không có thông báo mới
            </div>
          )}
          {notifications.map((item) => (
            <button
              key={item.id}
              className={`w-full text-left px-4 py-3 border-b last:border-0 hover:bg-gray-50 transition-colors ${
                !item.read_at ? 'bg-blue-50/40' : ''
              }`}
              onClick={() => handleNotificationClick(item)}
            >
              <div className="flex items-start gap-3">
                {/* Unread indicator */}
                <div className="mt-1.5 flex-shrink-0">
                  {!item.read_at ? (
                    <div className="h-2 w-2 rounded-full bg-blue-500" />
                  ) : (
                    <div className="h-2 w-2 rounded-full bg-transparent" />
                  )}
                </div>
                <div className="flex-1 min-w-0">
                  <p className={`text-sm truncate ${!item.read_at ? 'font-semibold' : 'font-normal'}`}>
                    {item.title}
                  </p>
                  <p className="text-xs text-gray-500 truncate mt-0.5">{item.body}</p>
                  <p className="text-xs text-gray-400 mt-1">
                    {formatDistanceToNow(new Date(item.created_at), { addSuffix: true, locale: vi })}
                  </p>
                </div>
              </div>
            </button>
          ))}
        </div>

        {/* Footer */}
        <div className="border-t px-4 py-3 text-center">
          <button
            className="text-xs text-blue-600 hover:underline"
            onClick={() => router.push('/dashboard/notifications')}
          >
            Xem tất cả thông báo
          </button>
        </div>
      </PopoverContent>
    </Popover>
  );
}
```

> **date-fns dependency:** Check if `date-fns` is already in `package.json`. If not, install it: `npm install date-fns`. If the project has a different time formatting utility, use that instead.

- [ ] **Verify:** `npm run build` passes

---

## Task 6.5 — Create notification center page

**File:** `marketplace-admin/app/dashboard/(dashboard)/notifications/page.tsx`

- [ ] **Read** a similar list page (e.g., the KYC list page) to understand the page structure pattern
- [ ] **Create** the file:

```tsx
'use client';

import { useState } from 'react';
import { useRouter } from 'next/navigation';
import { formatDistanceToNow } from 'date-fns';
import { vi } from 'date-fns/locale';
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query';

import { notificationApi, AdminNotificationItem } from '@/app/api/notification';

const PAGE_SIZE = 20;

export default function NotificationsPage() {
  const router = useRouter();
  const queryClient = useQueryClient();
  const [unreadOnly, setUnreadOnly] = useState(false);
  const [page, setPage] = useState(1);

  const offset = (page - 1) * PAGE_SIZE;

  const { data, isLoading } = useQuery({
    queryKey: ['notifications', { page, unreadOnly, limit: PAGE_SIZE }],
    queryFn: () =>
      notificationApi
        .list({ limit: PAGE_SIZE, offset, unread_only: unreadOnly })
        .then((r) => r.data),
  });

  const notifications: AdminNotificationItem[] = data?.data ?? [];
  const paging = data?.paging;

  const markReadMutation = useMutation({
    mutationFn: (id: string) => notificationApi.markRead(id),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['notifications'] });
      queryClient.invalidateQueries({ queryKey: ['notification-unread-count'] });
    },
  });

  const markAllReadMutation = useMutation({
    mutationFn: () => notificationApi.markAllRead(),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['notifications'] });
      queryClient.invalidateQueries({ queryKey: ['notification-unread-count'] });
    },
  });

  const handleClick = (item: AdminNotificationItem) => {
    if (!item.read_at) markReadMutation.mutate(item.id);
    if (item.action_url) router.push(`/dashboard${item.action_url}`);
  };

  const handleTabChange = (newUnreadOnly: boolean) => {
    setUnreadOnly(newUnreadOnly);
    setPage(1);
  };

  return (
    <div className="max-w-3xl mx-auto py-8 px-4">
      {/* Page header */}
      <div className="flex items-center justify-between mb-6">
        <h1 className="text-xl font-semibold">Trung tâm thông báo</h1>
        <button
          className="text-sm text-blue-600 hover:underline"
          onClick={() => markAllReadMutation.mutate()}
          disabled={markAllReadMutation.isPending}
        >
          Đánh dấu tất cả đã đọc
        </button>
      </div>

      {/* Tabs */}
      <div className="flex gap-4 border-b mb-4">
        <button
          className={`pb-2 text-sm font-medium border-b-2 transition-colors ${
            !unreadOnly ? 'border-blue-600 text-blue-600' : 'border-transparent text-gray-500'
          }`}
          onClick={() => handleTabChange(false)}
        >
          Tất cả
        </button>
        <button
          className={`pb-2 text-sm font-medium border-b-2 transition-colors ${
            unreadOnly ? 'border-blue-600 text-blue-600' : 'border-transparent text-gray-500'
          }`}
          onClick={() => handleTabChange(true)}
        >
          Chưa đọc
        </button>
      </div>

      {/* List */}
      <div className="divide-y rounded-lg border bg-white">
        {isLoading && (
          <div className="px-6 py-12 text-center text-sm text-gray-400">Đang tải...</div>
        )}
        {!isLoading && notifications.length === 0 && (
          <div className="px-6 py-12 text-center text-sm text-gray-400">
            Không có thông báo
          </div>
        )}
        {notifications.map((item) => (
          <button
            key={item.id}
            className={`w-full text-left px-6 py-4 hover:bg-gray-50 transition-colors ${
              !item.read_at ? 'bg-blue-50/30' : ''
            }`}
            onClick={() => handleClick(item)}
          >
            <div className="flex items-start gap-4">
              <div className="mt-1.5 flex-shrink-0">
                {!item.read_at ? (
                  <div className="h-2.5 w-2.5 rounded-full bg-blue-500" />
                ) : (
                  <div className="h-2.5 w-2.5 rounded-full bg-gray-200" />
                )}
              </div>
              <div className="flex-1 min-w-0">
                <p className={`text-sm ${!item.read_at ? 'font-semibold' : 'font-normal'}`}>
                  {item.title}
                </p>
                <p className="text-sm text-gray-500 mt-0.5">{item.body}</p>
                <p className="text-xs text-gray-400 mt-1">
                  {formatDistanceToNow(new Date(item.created_at), { addSuffix: true, locale: vi })}
                </p>
              </div>
            </div>
          </button>
        ))}
      </div>

      {/* Pagination */}
      {paging && (
        <div className="flex items-center justify-between mt-4">
          <button
            className="text-sm text-gray-600 hover:text-gray-900 disabled:opacity-40"
            onClick={() => setPage((p) => Math.max(1, p - 1))}
            disabled={page === 1}
          >
            ← Trang trước
          </button>
          <span className="text-sm text-gray-500">Trang {page}</span>
          <button
            className="text-sm text-gray-600 hover:text-gray-900 disabled:opacity-40"
            onClick={() => setPage((p) => p + 1)}
            disabled={!paging.hasNext}
          >
            Trang sau →
          </button>
        </div>
      )}
    </div>
  );
}
```

- [ ] **Verify:** `npm run build` passes, page is accessible at `/dashboard/notifications`

---

## Verification Checklist

- [ ] `npm run build`: clean build
- [ ] `npm run lint`: no lint errors
- [ ] `notificationApi` module follows lazy singleton + runtime config pattern
- [ ] `useNotificationStream` hook opens SSE connection, invalidates React Query on new events
- [ ] Bell badge shows dynamic count from API (not hardcoded)
- [ ] Bell badge hidden (no badge) when `unreadCount === 0`
- [ ] Bell badge shows `99+` when `unreadCount > 99`
- [ ] Clicking a notification: marks as read + navigates to action URL
- [ ] "Đánh dấu tất cả đã đọc" button calls markAllRead mutation, updates cache
- [ ] "Xem tất cả thông báo" navigates to `/dashboard/notifications`
- [ ] Notification center shows all notifications with pagination
- [ ] Tabs toggle between "Tất cả" and "Chưa đọc"
- [ ] Unread items visually distinguished (bold title, blue dot, blue background tint)
- [ ] No TypeScript errors (`strict: false` in tsconfig, but avoid `any` where possible)
- [ ] `date-fns` installed or alternative time formatting used
