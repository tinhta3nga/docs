# Phase 7: Update Architecture and Rule Files

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` or `superpowers:executing-plans`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Update the living rulebook to reflect the changes introduced by this feature. Rules that drift from the code are worse than no rules (per CLAUDE.md). This phase must be completed before the implementation is considered done.

**Files to update:**
- `workspace-context/rules/01-architecture.md` — add new events + new consumer to the RabbitMQ catalog
- `workspace-context/rules/04-marketplace-admin-api.md` — update "Publish-Only" rule, document notification consumer + registry pattern
- `workspace-context/rules/07-marketplace-admin.md` — add SSE hook pattern + notification center page

---

## File Map

| Action | File |
|---|---|
| Edit | `workspace-context/rules/01-architecture.md` |
| Edit | `workspace-context/rules/04-marketplace-admin-api.md` |
| Edit | `workspace-context/rules/07-marketplace-admin.md` |

---

## Task 7.1 — Update architecture rules

**File:** `workspace-context/rules/01-architecture.md`

- [ ] **Read** the full file first
- [ ] **In the RabbitMQ Event Catalog table**, add these rows under "marketplace-backend (publishes)":

| Routing Key | Trigger | Consumer |
|---|---|---|
| `user.publication_request.submitted` | User submits publication copyright request | marketplace-admin-api |
| `user.publication_request.resubmitted` | User resubmits rejected publication request | marketplace-admin-api |
| `user.exploitation_request.submitted` | User submits exploitation (protection) request | marketplace-admin-api |
| `user.exploitation_request.resubmitted` | User resubmits rejected exploitation request | marketplace-admin-api |
| `user.encoding_request.submitted` | User submits encoding request | marketplace-admin-api |
| `user.kyc.submitted` | User submits KYC identity verification | marketplace-admin-api |

- [ ] **Add a new section or row** under "marketplace-admin-api (consumes)" (create this section if it doesn't exist):

```markdown
**marketplace-admin-api (also consumes — new in 2026-04):**

| Queue | Binding Keys | Handler |
|---|---|---|
| `marketplace.admin_api.notifications` | `user.publication_request.*`, `user.exploitation_request.*`, `user.encoding_request.submitted`, `user.kyc.submitted` | `internal/adminnotification/handler.go` |
```

- [ ] **Save** the changes

---

## Task 7.2 — Update marketplace-admin-api rules

**File:** `workspace-context/rules/04-marketplace-admin-api.md`

- [ ] **Read** the full file first — especially the "RabbitMQ: Publish-Only" section
- [ ] **Update** the "RabbitMQ: Publish-Only" section heading and content to reflect the new reality:

**Replace** the heading and description. Change from:
> **RabbitMQ: Publish-Only**
> marketplace-admin-api **never consumes** RabbitMQ messages — it only publishes.

**To:**
> **RabbitMQ: Publish + One Consumer**
> marketplace-admin-api **publishes** events to CRM and marketplace-backend (unchanged).
> It also has **one consumer**: the admin notification consumer that receives user submission events.

- [ ] **Add** a new subsection after the existing RabbitMQ section:

```markdown
### Admin Notification Consumer

Added 2026-04. Consumes user-submission events from marketplace-backend and fans them out to eligible admins.

- Queue: `marketplace.admin_api.notifications`
- Consumer constructor: `internal/rabbitmq/consumer/admin_notification.go`
- Handler + mapper: `internal/adminnotification/`
- Wired in `main.go` as goroutine (optional, graceful if RabbitMQ unavailable)

### Notification Type Registry

All admin notification types are defined in `internal/notificationregistry/`:
- `types.go` — `NotificationType` constants + `EventTypeToNotificationType` mapping
- `registry.go` — `Registry` map: type → {RouterKeys, TTLDays, ActionURLPattern, Category, Label}

**To add a new notification type:**
1. Add constant to `types.go` + entry to `registry.go`
2. Add event + payload + publish in marketplace-backend (Phase 1 doc)
3. Add binding key to `admin_notification.go` consumer
4. Add case in `internal/adminnotification/mapper.go`
No changes to core pipeline (consumer, handler, storage, API).

### SSE Hub

`internal/ssehub/` — in-memory pub/sub for admin SSE connections.
- Initialized in `main.go` as `ssehub.SetDefault(ssehub.New())`
- Used by `business/notification/stream.go` (subscribe/unsubscribe)
- Used by `internal/adminnotification/handler.go` (publish on new notification)
- Single-instance safe. For multi-instance: replace with Redis pub/sub.
```

- [ ] **Add** the new collections to the schema section if one exists:

```markdown
- `admin_notifications` → `schema/adminnotificationcol/` (fan-out per admin)
- `processed_admin_notification_events` → `schema/processedadminnotificationeventcol/` (idempotency)
```

- [ ] **Save** the changes

---

## Task 7.3 — Update marketplace-admin rules

**File:** `workspace-context/rules/07-marketplace-admin.md`

- [ ] **Read** the full file first
- [ ] **Add** a new section for the notification system:

```markdown
## Notification System

### Data Fetching

Notification data is **server state** — use React Query, not Redux:

```typescript
// Unread count (badge)
const { data } = useQuery({
  queryKey: ['notification-unread-count'],
  queryFn: () => notificationApi.unreadCount().then((r) => r.data),
  refetchInterval: 30_000, // 30s polling fallback
  staleTime: 10_000,
});

// Notification list
const { data } = useQuery({
  queryKey: ['notifications', params],
  queryFn: () => notificationApi.list(params).then((r) => r.data),
});
```

Query key conventions:
- `['notification-unread-count']` — bell badge
- `['notifications', { page, unreadOnly, limit }]` — list queries

### SSE Real-Time Connection

Use `useNotificationStream` hook (hooks/useNotificationStream.ts) in components that need live updates.

The hook:
- Opens `EventSource` to `/notifications/stream?token=<JWT>`
- On `notification` event: invalidates `notification-unread-count` and `notifications` query caches
- Auto-reconnects after 10s on error
- Cleans up EventSource on unmount

```typescript
// In a client component
useNotificationStream(); // basic — just keeps cache fresh
useNotificationStream((data) => toast.success('New notification!')); // with callback
```

### API Module

`app/api/notification.ts` — follows the lazy singleton + runtime config pattern (same as all other API modules).

Key types: `AdminNotificationItem`, `ListNotificationsParams`, `ListNotificationsResponse`, `UnreadCountResponse`.

### Notification Center Page

Route: `/dashboard/notifications`  
File: `app/dashboard/(dashboard)/notifications/page.tsx`

- Two tabs: "Tất cả" / "Chưa đọc" (toggles `unread_only` param)
- Pagination via `paging.hasNext` + `paging.current`
- Built with shadcn/ui + Tailwind (NOT MUI DataGrid — this is a list, not a data table)

### Bell Dropdown

`components/dashboard/NotificationDropdown.tsx` — Radix UI Popover with:
- Latest 20 notifications (sorted newest first by backend)
- Mark as read on click + navigate to `action_url`
- "Đánh dấu tất cả đã đọc" button
- Footer link → notification center

Mounted in `components/dashboard/header.tsx`.
```

- [ ] **Save** the changes

---

## Verification Checklist

- [ ] `01-architecture.md`: 6 new rows in the RabbitMQ event catalog under marketplace-backend (publishes)
- [ ] `01-architecture.md`: new consumer section for marketplace-admin-api
- [ ] `04-marketplace-admin-api.md`: "Publish-Only" heading updated to "Publish + One Consumer"
- [ ] `04-marketplace-admin-api.md`: registry pattern documented (7-step process for adding new types)
- [ ] `04-marketplace-admin-api.md`: SSE hub documented
- [ ] `04-marketplace-admin-api.md`: new collections listed
- [ ] `07-marketplace-admin.md`: React Query patterns for notifications documented
- [ ] `07-marketplace-admin.md`: SSE hook usage documented
- [ ] `07-marketplace-admin.md`: notification center page documented
- [ ] All rule updates are factual and match the actual implementation
