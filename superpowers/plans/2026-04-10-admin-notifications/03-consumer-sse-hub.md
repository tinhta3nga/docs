# Phase 3: marketplace-admin-api — RabbitMQ Consumer + In-Memory SSE Hub

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` or `superpowers:executing-plans`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the in-memory SSE hub (for real-time push to connected admin browsers), the admin notification consumer (reads events from RabbitMQ), the event mapper (fan-out logic), and the handler (idempotency + orchestration). Wire everything in `main.go`.

**Tech Stack:** Go 1.24, RabbitMQ (amqp), sync.RWMutex for hub concurrency

**Rule files to read first:** `workspace-context/rules/02-go-backend-shared.md`, `workspace-context/rules/04-marketplace-admin-api.md`

**Reference files to read:**
- `marketplace-backend/internal/notification/handler.go` — mirror this pattern
- `marketplace-backend/internal/notification/mapper.go` — mirror this pattern
- `marketplace-backend/internal/rabbitmq/consumer/notification.go` — mirror consumer constructor
- `marketplace-admin-api/internal/rabbitmq/consumer/` — see existing consumers if any
- `marketplace-admin-api/schema/admincol/model.go` — understand `Admin.HasRouterKey(ctx, key)`
- `marketplace-admin-api/main.go` — understand service wiring before editing

**Depends on:** Phase 2 (notificationregistry, adminnotificationcol, processedadminnotificationeventcol must exist)

---

## File Map

| Action | File |
|---|---|
| Create | `marketplace-admin-api/internal/ssehub/hub.go` |
| Create | `marketplace-admin-api/internal/adminnotification/mapper.go` |
| Create | `marketplace-admin-api/internal/adminnotification/handler.go` |
| Create | `marketplace-admin-api/internal/rabbitmq/consumer/admin_notification.go` |
| Edit | `marketplace-admin-api/main.go` |

---

## Task 3.1 — Create the in-memory SSE hub

**File:** `marketplace-admin-api/internal/ssehub/hub.go`

The hub manages a map of `adminID → []chan SSEEvent`. When the consumer creates a new notification, it publishes to the relevant admin's channels. The SSE stream handler in Phase 4 subscribes/unsubscribes.

- [ ] **Create** the file:

```go
package ssehub

import "sync"

// SSEEvent is the payload sent over the SSE stream to the frontend.
type SSEEvent struct {
	Type string `json:"type"` // e.g. "new_notification"
	Data any    `json:"data"`
}

// Hub manages per-admin SSE subscriptions.
// Safe for concurrent use. Designed for single-instance deployment
// (30–40 admins). For multi-instance, replace with Redis pub/sub.
type Hub struct {
	mu          sync.RWMutex
	subscribers map[string][]chan SSEEvent // adminID → active channels
}

// New creates a new Hub.
func New() *Hub {
	return &Hub{
		subscribers: make(map[string][]chan SSEEvent),
	}
}

// Subscribe registers a new channel for an admin.
// The caller MUST call Unsubscribe when done (e.g., via defer).
func (h *Hub) Subscribe(adminID string) chan SSEEvent {
	ch := make(chan SSEEvent, 8) // buffered to avoid blocking Publish
	h.mu.Lock()
	h.subscribers[adminID] = append(h.subscribers[adminID], ch)
	h.mu.Unlock()
	return ch
}

// Unsubscribe removes a channel from the admin's subscriber list and closes it.
func (h *Hub) Unsubscribe(adminID string, ch chan SSEEvent) {
	h.mu.Lock()
	defer h.mu.Unlock()
	chans := h.subscribers[adminID]
	for i, c := range chans {
		if c == ch {
			h.subscribers[adminID] = append(chans[:i], chans[i+1:]...)
			break
		}
	}
	if len(h.subscribers[adminID]) == 0 {
		delete(h.subscribers, adminID)
	}
	close(ch)
}

// Publish sends an event to all active channels for one admin.
// Non-blocking: channels with full buffers are skipped (client too slow).
func (h *Hub) Publish(adminID string, event SSEEvent) {
	h.mu.RLock()
	chans := h.subscribers[adminID]
	h.mu.RUnlock()
	for _, ch := range chans {
		select {
		case ch <- event:
		default:
			// Buffer full — skip to avoid blocking the consumer goroutine
		}
	}
}

// --- Singleton ---

var defaultHub *Hub

// SetDefault sets the package-level default hub (called in main.go).
func SetDefault(h *Hub) { defaultHub = h }

// Default returns the package-level default hub.
func Default() *Hub { return defaultHub }
```

- [ ] **Verify:** `make build` passes

---

## Task 3.2 — Create the event mapper (fan-out logic)

**File:** `marketplace-admin-api/internal/adminnotification/mapper.go`

The mapper converts a raw RabbitMQ event into a slice of `AdminNotification` documents — one per eligible admin. It uses the registry to determine who should receive the notification and what it should say.

- [ ] **Read** `marketplace-backend/internal/notification/mapper.go` first
- [ ] **Read** `marketplace-admin-api/schema/admincol/model.go` to confirm `HasRouterKey(ctx, key)` signature
- [ ] **Find** the `admincol` query function for listing all active admins (search for `FindAll`, `ListAll`, or `Active` in `marketplace-admin-api/schema/admincol/query.go`). If it does not exist, create `FindAllActive(ctx) ([]*Admin, error)` in that file.
- [ ] **Create** the mapper file:

```go
package adminnotification

import (
	"context"
	"encoding/json"
	"fmt"
	"strings"
	"time"

	"marketplace-admin-api/internal/notificationregistry"
	"marketplace-admin-api/internal/plog"
	"marketplace-admin-api/internal/rabbitmq"
	"marketplace-admin-api/schema/adminnotificationcol"
	"marketplace-admin-api/schema/admincol"
)

// EventToAdminNotifications converts a RabbitMQ event into fan-out notification
// documents — one per admin who has the required router key.
// Returns nil (no notifications) if the event type is not in the registry.
func EventToAdminNotifications(ctx context.Context, event rabbitmq.Event) ([]*adminnotificationcol.AdminNotification, error) {
	logger := plog.NewBizLogger("[internal][adminnotification][mapper]")

	// 1. Resolve notification type from registry
	notifType, ok := notificationregistry.EventTypeToNotificationType[event.EventType]
	if !ok {
		logger.Info().Str("event_type", event.EventType).Msg("unknown event type, skipping")
		return nil, nil
	}
	cfg, ok := notificationregistry.Registry[notifType]
	if !ok {
		return nil, nil
	}

	// 2. Build notification content from payload
	title, body, entityID, entityType, err := buildNotificationContent(notifType, event.Payload)
	if err != nil {
		return nil, fmt.Errorf("build notification content: %w", err)
	}
	actionURL := buildActionURL(cfg.ActionURLPattern, entityID)

	// 3. Get all active admins
	admins, err := admincol.FindAllActive(ctx)
	if err != nil {
		return nil, fmt.Errorf("find all active admins: %w", err)
	}

	// 4. Fan out: one notification per eligible admin
	var notifications []*adminnotificationcol.AdminNotification
	for _, admin := range admins {
		eligible := false
		for _, key := range cfg.RouterKeys {
			has, err := admin.HasRouterKey(ctx, key)
			if err != nil {
				logger.Warn().Err(err).Str("admin_id", admin.GetIDString()).Msg("router key check failed")
				continue
			}
			if has {
				eligible = true
				break
			}
		}
		if !eligible {
			continue
		}

		notifications = append(notifications, &adminnotificationcol.AdminNotification{
			CreatedAt:        time.Now().UTC(),
			EventID:          event.EventID,
			AdminID:          admin.GetIDString(),
			NotificationType: string(notifType),
			Category:         cfg.Category,
			Title:            title,
			Body:             body,
			ActionURL:        actionURL,
			EntityType:       entityType,
			EntityID:         entityID,
		})
	}

	return notifications, nil
}

// buildActionURL replaces :entity_id placeholder in the URL pattern.
func buildActionURL(pattern, entityID string) string {
	if entityID == "" {
		return strings.ReplaceAll(pattern, "/:entity_id", "")
	}
	return strings.ReplaceAll(pattern, ":entity_id", entityID)
}

// --- Payload types (must match marketplace-backend/internal/rabbitmq/events/admin_notification_events.go) ---

type publicationPayload struct {
	RequestID      string    `json:"request_id"`
	UserID         string    `json:"user_id"`
	UserName       string    `json:"user_name"`
	IsResubmission bool      `json:"is_resubmission"`
	Round          int       `json:"round"`
	CreatedAt      time.Time `json:"created_at"`
}

type encodingPayload struct {
	RequestID string    `json:"request_id"`
	UserID    string    `json:"user_id"`
	UserName  string    `json:"user_name"`
	CreatedAt time.Time `json:"created_at"`
}

type kycPayload struct {
	UserID    string    `json:"user_id"`
	UserName  string    `json:"user_name"`
	CreatedAt time.Time `json:"created_at"`
}

// buildNotificationContent parses the event payload and returns display strings.
func buildNotificationContent(notifType notificationregistry.NotificationType, raw json.RawMessage) (title, body, entityID, entityType string, err error) {
	switch notifType {
	case notificationregistry.TypePublicationRequest:
		var p publicationPayload
		if err = json.Unmarshal(raw, &p); err != nil {
			return
		}
		title = fmt.Sprintf("Yêu cầu đăng ký công bố mới")
		body = fmt.Sprintf("User %s vừa nộp yêu cầu đăng ký công bố", p.UserName)
		entityID = p.RequestID
		entityType = "publication_request"

	case notificationregistry.TypePublicationResubmit:
		var p publicationPayload
		if err = json.Unmarshal(raw, &p); err != nil {
			return
		}
		roundStr := ""
		if p.Round > 0 {
			roundStr = fmt.Sprintf(" (Lần %d)", p.Round)
		}
		title = fmt.Sprintf("Nộp lại yêu cầu đăng ký công bố%s", roundStr)
		body = fmt.Sprintf("User %s đã nộp lại yêu cầu đăng ký công bố%s", p.UserName, roundStr)
		entityID = p.RequestID
		entityType = "publication_request"

	case notificationregistry.TypeExploitationRequest:
		var p publicationPayload
		if err = json.Unmarshal(raw, &p); err != nil {
			return
		}
		title = "Yêu cầu đăng ký khai thác mới"
		body = fmt.Sprintf("User %s vừa nộp yêu cầu đăng ký khai thác", p.UserName)
		entityID = p.RequestID
		entityType = "exploitation_request"

	case notificationregistry.TypeExploitationResubmit:
		var p publicationPayload
		if err = json.Unmarshal(raw, &p); err != nil {
			return
		}
		roundStr := ""
		if p.Round > 0 {
			roundStr = fmt.Sprintf(" (Lần %d)", p.Round)
		}
		title = fmt.Sprintf("Nộp lại yêu cầu đăng ký khai thác%s", roundStr)
		body = fmt.Sprintf("User %s đã nộp lại yêu cầu đăng ký khai thác%s", p.UserName, roundStr)
		entityID = p.RequestID
		entityType = "exploitation_request"

	case notificationregistry.TypeEncodingRequest:
		var p encodingPayload
		if err = json.Unmarshal(raw, &p); err != nil {
			return
		}
		title = "Yêu cầu mã hóa blockchain mới"
		body = fmt.Sprintf("User %s vừa gửi yêu cầu mã hóa blockchain", p.UserName)
		entityID = p.RequestID
		entityType = "encoding_request"

	case notificationregistry.TypeKYCRequest:
		var p kycPayload
		if err = json.Unmarshal(raw, &p); err != nil {
			return
		}
		title = "Yêu cầu xác minh danh tính mới"
		body = fmt.Sprintf("User %s vừa nộp hồ sơ xác minh danh tính (KYC)", p.UserName)
		entityID = p.UserID // KYC links to user profile, not a separate entity
		entityType = "user"

	default:
		err = fmt.Errorf("unhandled notification type: %s", notifType)
	}
	return
}
```

- [ ] **Check if `admincol.FindAllActive` exists.** If not, add it to `marketplace-admin-api/schema/admincol/query.go`:

```go
// FindAllActive returns all admin accounts with Active=true.
// Used by the notification fan-out mapper.
func FindAllActive(ctx context.Context) ([]*Admin, error) {
    coll := mgm.Coll(&Admin{})
    filter := bson.D{{Key: "active", Value: true}}
    cursor, err := coll.Find(ctx, filter)
    if err != nil {
        return nil, err
    }
    defer cursor.Close(ctx)
    var results []*Admin
    if err := cursor.All(ctx, &results); err != nil {
        return nil, err
    }
    return results, nil
}
```

- [ ] **Verify:** `make build` passes

---

## Task 3.3 — Create the consumer handler

**File:** `marketplace-admin-api/internal/adminnotification/handler.go`

The handler is invoked by the RabbitMQ consumer for each incoming message. It orchestrates idempotency check → mapper → DB insert → SSE push → mark processed.

- [ ] **Read** `marketplace-backend/internal/notification/handler.go` fully before writing
- [ ] **Create** the file:

```go
package adminnotification

import (
	"context"
	"encoding/json"
	"fmt"

	amqp "github.com/rabbitmq/amqp091-go"

	"marketplace-admin-api/internal/plog"
	"marketplace-admin-api/internal/rabbitmq"
	"marketplace-admin-api/internal/ssehub"
	"marketplace-admin-api/schema/adminnotificationcol"
	"marketplace-admin-api/schema/processedadminnotificationeventcol"
)

// HandleAdminNotificationEvent processes one RabbitMQ delivery.
// Mirrors marketplace-backend/internal/notification/handler.go exactly.
func HandleAdminNotificationEvent(ctx context.Context, d amqp.Delivery) error {
	logger := plog.NewBizLogger("[internal][adminnotification][handler]")

	// 1. Unmarshal envelope
	var event rabbitmq.Event
	if err := json.Unmarshal(d.Body, &event); err != nil {
		logger.Error().Err(err).Msg("failed to unmarshal event envelope")
		return err
	}

	// 2. Validate required fields
	if event.EventID == "" || event.EventType == "" {
		logger.Warn().Msg("event missing event_id or event_type, skipping")
		return nil
	}

	// 3. Idempotency check
	exists, err := processedadminnotificationeventcol.ExistsByEventID(ctx, event.EventID)
	if err != nil {
		logger.Error().Err(err).Str("event_id", event.EventID).Msg("idempotency check failed")
		return err
	}
	if exists {
		logger.Info().Str("event_id", event.EventID).Msg("event already processed, skipping")
		return nil
	}

	// 4. Map event to fan-out notifications
	notifications, err := EventToAdminNotifications(ctx, event)
	if err != nil {
		logger.Error().Err(err).Str("event_type", event.EventType).Msg("mapper failed")
		return err
	}

	// 5. Unknown event type — mark processed and skip (don't block queue)
	if notifications == nil {
		if markErr := processedadminnotificationeventcol.Insert(ctx, event.EventID); markErr != nil {
			logger.Error().Err(markErr).Msg("failed to mark unknown event as processed")
		}
		return nil
	}

	// 6. Persist fan-out notifications
	if len(notifications) > 0 {
		if err := adminnotificationcol.InsertMany(ctx, notifications); err != nil {
			logger.Error().Err(err).Msg("failed to insert notifications")
			return fmt.Errorf("insert notifications: %w", err)
		}

		// 7. Push SSE events to connected admin browsers
		hub := ssehub.Default()
		if hub != nil {
			for _, n := range notifications {
				hub.Publish(n.AdminID, ssehub.SSEEvent{
					Type: "new_notification",
					Data: n,
				})
			}
		}
	}

	// 8. Mark event as processed (after successful insert)
	if err := processedadminnotificationeventcol.Insert(ctx, event.EventID); err != nil {
		logger.Error().Err(err).Str("event_id", event.EventID).Msg("failed to mark event processed")
		return err
	}

	logger.Info().
		Str("event_id", event.EventID).
		Str("event_type", event.EventType).
		Int("fan_out_count", len(notifications)).
		Msg("admin notification event processed")

	return nil
}
```

- [ ] **Verify:** `make build` passes

---

## Task 3.4 — Create the RabbitMQ consumer constructor

**File:** `marketplace-admin-api/internal/rabbitmq/consumer/admin_notification.go`

- [ ] **Read** `marketplace-backend/internal/rabbitmq/consumer/notification.go` for the constructor pattern
- [ ] **Read** `marketplace-admin-api/internal/rabbitmq/` to understand the `Consumer`, `Config`, and `NewConsumer` types/functions available
- [ ] **Create** the file:

```go
package consumer

import (
	"marketplace-admin-api/internal/rabbitmq"
)

// Routing keys for admin notification events (must match marketplace-backend constants).
const (
	eventPublicationSubmitted    = "user.publication_request.submitted"
	eventPublicationResubmitted  = "user.publication_request.resubmitted"
	eventExploitationSubmitted   = "user.exploitation_request.submitted"
	eventExploitationResubmitted = "user.exploitation_request.resubmitted"
	eventEncodingSubmitted       = "user.encoding_request.submitted"
	eventKYCSubmitted            = "user.kyc.submitted"
)

// NewAdminNotificationConsumer creates the consumer for user-submission events.
// Queue: "marketplace.admin_api.notifications"
// Binding keys: all user.*.submitted and user.*.resubmitted events.
func NewAdminNotificationConsumer(cfg *rabbitmq.Config) (*rabbitmq.Consumer, error) {
	return rabbitmq.NewConsumer(cfg,
		"marketplace.admin_api.notifications",
		[]string{
			eventPublicationSubmitted,
			eventPublicationResubmitted,
			eventExploitationSubmitted,
			eventExploitationResubmitted,
			eventEncodingSubmitted,
			eventKYCSubmitted,
		},
		10, // prefetch count — process up to 10 messages concurrently
	)
}
```

> **Important:** Check the exact `rabbitmq.NewConsumer` signature in marketplace-admin-api's `internal/rabbitmq/` — the parameter order or type may differ from marketplace-backend. Adapt accordingly.

- [ ] **Verify:** `make build` passes

---

## Task 3.5 — Wire consumer + SSE hub in main.go

**File:** `marketplace-admin-api/main.go`

- [ ] **Read** `main.go` fully to understand where services are initialized and where goroutines are launched
- [ ] **Add imports** for the new packages
- [ ] **Add** SSE hub initialization (before any route registration):

```go
// SSE hub — must be initialized before starting HTTP server
hub := ssehub.New()
ssehub.SetDefault(hub)
```

- [ ] **Add** admin notification consumer (after RabbitMQ producer is set up, following the same optional/graceful pattern):

```go
// Admin notification consumer (optional — only if RabbitMQ is configured)
if rabbitmqCfg := rabbitmq.FromEnv(os.Getenv); rabbitmqCfg != nil {
    adminNotifConsumer, err := consumer.NewAdminNotificationConsumer(rabbitmqCfg)
    if err != nil {
        logger.Warn().Err(err).Msg("failed to init admin notification consumer — notifications disabled")
    } else {
        defer adminNotifConsumer.Close()
        go adminNotifConsumer.Consume(ctx, adminnotification.HandleAdminNotificationEvent)
        logger.Info().Msg("admin notification consumer started")
    }
}
```

- [ ] **Verify:** `make build` passes, `make test` shows no regressions

---

## Verification Checklist

- [ ] `make build` in marketplace-admin-api: clean compile
- [ ] `make test`: no regressions
- [ ] `ssehub` package: Hub with Subscribe/Unsubscribe/Publish + singleton
- [ ] `adminnotification` package: mapper + handler both compile
- [ ] Consumer handles all 6 binding keys
- [ ] `main.go` initializes hub before routes, consumer in goroutine after RabbitMQ producer
- [ ] Handler correctly: validates → idempotency → maps → inserts → SSE push → marks processed
- [ ] Unknown event types are marked processed and silently skipped (queue never blocked)
- [ ] SSE publish is non-blocking (buffer + select default)

---

## Fan-Out Logic Notes

- `HasRouterKey(ctx, key)` already handles wildcard matching (`"product.*"` etc.) — no custom wildcard logic needed
- Super-admin accounts (IsSuperAdmin=true) return `["*"]` from GetRouterKeys — they will always pass `HasRouterKey` for any key, so they receive ALL notification types
- If `FindAllActive` returns 0 admins with the required key, `notifications` is empty — `InsertMany` is a no-op, no error
- SSE hub `Default()` may return nil before `main.go` runs `SetDefault` — the handler guards with `if hub != nil`
