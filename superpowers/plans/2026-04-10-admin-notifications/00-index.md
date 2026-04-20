# Admin Notification System — Implementation Plan Index

**Feature:** Real-time admin notification pipeline (user → admin direction only)  
**Date:** 2026-04-10  
**Status:** Approved — ready to implement

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan. Execute phases in order — each phase depends on the previous.

---

## What We're Building

Admins currently have no in-system way to know when users submit requests. This plan adds:

1. **Real-time bell badge** in the admin header (live unread count via SSE + 30s polling fallback)
2. **Notification dropdown** (latest 20 notifications, mark-as-read, "view all" link)
3. **Notification center page** at `/dashboard/notifications` (full paginated list + tabs)
4. **Fan-out pipeline** (RabbitMQ event → eligible admins determined by router key → one MongoDB doc per admin)
5. **Extensible registry** (adding a new notification type = 7 steps, no core changes)

**Scope: user → admin only.** Admin-to-admin system action notifications are out of scope.

---

## How to Register a New Notification Type (Summary)

When you need to add a future type (e.g., `violation_report`, `legal_advice_request`):

| Step | Where | What |
|---|---|---|
| 1 | `marketplace-admin-api/internal/notificationregistry/types.go` | Add `NotificationType` constant + `EventTypeToNotificationType` entry |
| 2 | `marketplace-admin-api/internal/notificationregistry/registry.go` | Add `TypeConfig` entry (RouterKeys, TTL, ActionURL, Category, Label) |
| 3 | `marketplace-backend/internal/rabbitmq/event.go` | Add event constant |
| 4 | `marketplace-backend/internal/rabbitmq/events/admin_notification_events.go` | Add payload struct |
| 5 | Relevant marketplace-backend business handler | Add fire-and-forget `PublishEvent` call |
| 6 | `marketplace-admin-api/internal/rabbitmq/consumer/admin_notification.go` | Add binding key to consumer |
| 7 | `marketplace-admin-api/internal/adminnotification/mapper.go` | Add `case` in `buildNotificationContent` |

**Zero changes to:** consumer handler, storage, API endpoints, or frontend.

---

## Phase Overview

| Phase | Repo | What | Doc |
|---|---|---|---|
| 1 | marketplace-backend | Add 6 RabbitMQ event types + publish calls in 4–6 handlers | [01-backend-publish-events.md](./01-backend-publish-events.md) |
| 2 | marketplace-admin-api | Notification type registry + MongoDB schema + indexes | [02-registry-schema.md](./02-registry-schema.md) |
| 3 | marketplace-admin-api | RabbitMQ consumer + in-memory SSE hub + event mapper + handler | [03-consumer-sse-hub.md](./03-consumer-sse-hub.md) |
| 4 | marketplace-admin-api | REST API (list, unread-count, mark-read, mark-all-read) + SSE stream endpoint | [04-rest-api-sse.md](./04-rest-api-sse.md) |
| 5 | marketplace-admin-api | TTL cleanup cronjob (daily, 90 days) | [05-ttl-cronjob.md](./05-ttl-cronjob.md) |
| 6 | marketplace-admin | Notification API module + SSE hook + bell + dropdown + center page | [06-frontend.md](./06-frontend.md) |
| 7 | rules | Update architecture + admin-api + admin rules files | [07-update-rules.md](./07-update-rules.md) |

---

## Initial Notification Types

| Type constant | Event | RouterKey gate | Action URL |
|---|---|---|---|
| `publication_registration_request` | `user.publication_request.submitted` | `product.copyright-requests.approve` | `/product/copyright-requests/:id` |
| `publication_registration_resubmit` | `user.publication_request.resubmitted` | `product.copyright-requests.approve` | `/product/copyright-requests/:id` |
| `exploitation_registration_request` | `user.exploitation_request.submitted` | `product.copyright-requests.approve` | `/product/exploitation-requests/:id` |
| `exploitation_registration_resubmit` | `user.exploitation_request.resubmitted` | `product.copyright-requests.approve` | `/product/exploitation-requests/:id` |
| `encoding_registration_request` | `user.encoding_request.submitted` | `product.encoding-requests.approve` | `/product/encoding-requests/:id` |
| `kyc_request` | `user.kyc.submitted` | `kyc.requests.approve` | `/kyc/:id` |

---

## Key Architecture Decisions

| Decision | Choice | Reason |
|---|---|---|
| Fan-out model | Fan-out on write (one doc per admin per event) | 30–40 admins → simple queries, simple read state |
| Real-time delivery | SSE (Server-Sent Events) | One-way push sufficient; simpler than WebSocket |
| SSE auth | Token in query param (`?token=`) | Browser EventSource cannot send custom headers |
| RBAC enforcement | At fan-out time (consumer), NOT at API read time | Every admin reads their own feed; gating is who gets what |
| Aggregation | None — individual documents, sorted newest-first | "Bubble to top" = natural from `created_at DESC` sort |
| TTL | 90 days, daily cronjob | `adminnotificationcol.DeleteOlderThan` |
| Idempotency | `processed_admin_notification_events` collection | Mirrors marketplace-backend's proven pattern |
| New consumer in admin-api | Yes — first consumer in admin-api | Required for this feature; rules updated in Phase 7 |
| SSE hub | In-memory (single instance) | 30–40 admins, single instance deployment |
| State management | React Query (server state) | Per rules/07-marketplace-admin.md — no Redux for server data |

---

## New Files Summary

### marketplace-backend (6 files touched)
- Edit: `internal/rabbitmq/event.go`
- Create: `internal/rabbitmq/events/admin_notification_events.go`
- Edit: 4–6 business handler files (publication, exploitation, encoding, KYC submit/resubmit)

### marketplace-admin-api (21 files)
- Create: `internal/notificationregistry/types.go`
- Create: `internal/notificationregistry/registry.go`
- Create: `internal/ssehub/hub.go`
- Create: `internal/adminnotification/mapper.go`
- Create: `internal/adminnotification/handler.go`
- Create: `internal/rabbitmq/consumer/admin_notification.go`
- Create: `schema/adminnotificationcol/model.go`
- Create: `schema/adminnotificationcol/query.go`
- Create: `schema/processedadminnotificationeventcol/model.go`
- Create: `schema/processedadminnotificationeventcol/query.go`
- Edit: `migrate/create_index.go`
- Edit: `business/notification/router.go`
- Create: `business/notification/list.go`
- Create: `business/notification/unread_count.go`
- Create: `business/notification/mark_read.go`
- Create: `business/notification/mark_all_read.go`
- Create: `business/notification/stream.go`
- Edit: `routers/ver1.go`
- Edit: `business/role/keys.go`
- Create: `services/cronjob/admin_notification_ttl.go`
- Edit: `main.go`

### marketplace-admin (5 files)
- Create: `app/api/notification.ts`
- Create: `hooks/useNotificationStream.ts`
- Edit: `components/dashboard/header.tsx`
- Create: `components/dashboard/NotificationDropdown.tsx`
- Create: `app/dashboard/(dashboard)/notifications/page.tsx`

### Rules (3 files)
- Edit: `workspace-context/rules/01-architecture.md`
- Edit: `workspace-context/rules/04-marketplace-admin-api.md`
- Edit: `workspace-context/rules/07-marketplace-admin.md`

---

## End-to-End Verification Flow

After all phases are complete, verify with this sequence:

1. Start both backends (`make dev` in marketplace-backend and marketplace-admin-api)
2. Start marketplace-admin frontend (`npm run dev`)
3. Submit a publication copyright request as a marketplace user
4. Confirm in RabbitMQ management: event in `marketplace.admin_api.notifications` queue
5. Confirm in MongoDB `admin_api` database: documents in `admin_notifications` (one per eligible admin)
6. Log in to marketplace-admin as an eligible admin (has `product.copyright-requests.approve`)
7. Bell shows unread count ≥ 1
8. DevTools → Network → EventStream: `/notifications/stream` open, event received
9. Click bell → dropdown shows notification with correct Vietnamese title
10. Click notification → navigates to correct page, notification marked read, count decrements
11. Log in as an admin WITHOUT the router key → bell shows 0 (fan-out was role-gated)
12. Resubmit a rejected publication → new notification with "Nộp lại" context appears
13. Verify TTL cronjob compiles and runs without error (adjust interval temporarily for testing)
14. Verify `make build` + `make test` pass in both backends
15. Verify `npm run build` passes in marketplace-admin
