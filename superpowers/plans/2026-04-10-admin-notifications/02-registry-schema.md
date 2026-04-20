# Phase 2: marketplace-admin-api — Notification Type Registry + MongoDB Schema

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` or `superpowers:executing-plans`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the notification type registry (the extensibility backbone), the `admin_notifications` MongoDB collection schema, and the `processed_admin_notification_events` idempotency collection. This phase is pure data layer — no HTTP handlers, no consumers yet.

**Tech Stack:** Go 1.24, MongoDB (mgm ORM), kamva/mgm DefaultModel

**Rule files to read first:** `workspace-context/rules/02-go-backend-shared.md`, `workspace-context/rules/04-marketplace-admin-api.md`

**Reference files to read:**
- `marketplace-backend/schema/notificationcol/model.go` — mirror this structure
- `marketplace-backend/schema/processednotificationcol/model.go` — mirror for idempotency
- `marketplace-admin-api/schema/admincol/model.go` — understand Admin model + GetRouterKeys

---

## File Map

| Action | File |
|---|---|
| Create | `marketplace-admin-api/internal/notificationregistry/types.go` |
| Create | `marketplace-admin-api/internal/notificationregistry/registry.go` |
| Create | `marketplace-admin-api/schema/adminnotificationcol/model.go` |
| Create | `marketplace-admin-api/schema/adminnotificationcol/query.go` |
| Create | `marketplace-admin-api/schema/processedadminnotificationeventcol/model.go` |
| Create | `marketplace-admin-api/schema/processedadminnotificationeventcol/query.go` |
| Edit | `marketplace-admin-api/migrate/create_index.go` |

---

## The Registry Pattern — Why It Exists

The registry is the single source of truth for all admin notification types. When a new notification type needs to be added in the future (e.g., `violation_report`), you add **one constant + one registry entry**. The consumer, fan-out logic, storage, and API are all type-agnostic — they read from the registry.

**To register a new type in the future:**
1. Add constant to `types.go`
2. Add entry to `Registry` in `registry.go`
3. (In Phase 1 doc) Add event + payload + publish in marketplace-backend
4. (In Phase 3 doc) Add binding key to consumer + case in mapper

---

## Task 2.1 — Create notification type registry

### `marketplace-admin-api/internal/notificationregistry/types.go`

- [ ] **Create** the file:

```go
package notificationregistry

// NotificationType identifies a category of admin notification.
// Each type maps to one entry in the Registry.
type NotificationType string

const (
	// TypePublicationRequest: user submits a new publication copyright registration request.
	TypePublicationRequest NotificationType = "publication_registration_request"

	// TypePublicationResubmit: user resubmits a previously rejected publication request.
	TypePublicationResubmit NotificationType = "publication_registration_resubmit"

	// TypeExploitationRequest: user submits a new exploitation (protection) registration request.
	// Note: "exploitation" = "protection" in the old schema.
	TypeExploitationRequest NotificationType = "exploitation_registration_request"

	// TypeExploitationResubmit: user resubmits a previously rejected exploitation request.
	TypeExploitationResubmit NotificationType = "exploitation_registration_resubmit"

	// TypeEncodingRequest: user submits a blockchain encoding request.
	TypeEncodingRequest NotificationType = "encoding_registration_request"

	// TypeKYCRequest: user submits KYC identity verification documents.
	TypeKYCRequest NotificationType = "kyc_request"
)

// RabbitMQ routing key → NotificationType mapping.
// Used by the consumer mapper to look up type from event.EventType.
var EventTypeToNotificationType = map[string]NotificationType{
	"user.publication_request.submitted":    TypePublicationRequest,
	"user.publication_request.resubmitted":  TypePublicationResubmit,
	"user.exploitation_request.submitted":   TypeExploitationRequest,
	"user.exploitation_request.resubmitted": TypeExploitationResubmit,
	"user.encoding_request.submitted":       TypeEncodingRequest,
	"user.kyc.submitted":                    TypeKYCRequest,
}
```

### `marketplace-admin-api/internal/notificationregistry/registry.go`

- [ ] **Create** the file:

```go
package notificationregistry

// TypeConfig holds all configuration for one notification type.
// The consumer reads this to determine fan-out routing and behavior.
type TypeConfig struct {
	// RouterKeys: admins with ANY of these router keys receive this notification.
	// Uses the same wildcard rules as middleware.RequireRouterKey.
	RouterKeys []string

	// TTLDays: how long to keep notifications of this type in MongoDB.
	TTLDays int

	// ActionURLPattern: relative admin frontend URL, with :entity_id as placeholder.
	// Example: "/product/copyright-requests/:entity_id"
	ActionURLPattern string

	// Category: grouping category for frontend display and filtering.
	Category string

	// Label: Vietnamese human-readable label for frontend display.
	Label string
}

// Registry maps each NotificationType to its configuration.
// To add a new notification type:
//  1. Add a constant in types.go
//  2. Add an entry here
//  3. See Phase 1 doc for marketplace-backend changes
//  4. See Phase 3 doc for consumer mapper changes
var Registry = map[NotificationType]TypeConfig{
	TypePublicationRequest: {
		RouterKeys:       []string{"product.copyright-requests.approve"},
		TTLDays:          90,
		ActionURLPattern: "/product/copyright-requests/:entity_id",
		Category:         "product",
		Label:            "Yêu cầu đăng ký công bố",
	},
	TypePublicationResubmit: {
		RouterKeys:       []string{"product.copyright-requests.approve"},
		TTLDays:          90,
		ActionURLPattern: "/product/copyright-requests/:entity_id",
		Category:         "product",
		Label:            "Nộp lại yêu cầu đăng ký công bố",
	},
	TypeExploitationRequest: {
		RouterKeys:       []string{"product.copyright-requests.approve"},
		TTLDays:          90,
		ActionURLPattern: "/product/exploitation-requests/:entity_id",
		Category:         "product",
		Label:            "Yêu cầu đăng ký khai thác",
	},
	TypeExploitationResubmit: {
		RouterKeys:       []string{"product.copyright-requests.approve"},
		TTLDays:          90,
		ActionURLPattern: "/product/exploitation-requests/:entity_id",
		Category:         "product",
		Label:            "Nộp lại yêu cầu đăng ký khai thác",
	},
	TypeEncodingRequest: {
		RouterKeys:       []string{"product.encoding-requests.approve"},
		TTLDays:          90,
		ActionURLPattern: "/product/encoding-requests/:entity_id",
		Category:         "product",
		Label:            "Yêu cầu mã hóa blockchain",
	},
	TypeKYCRequest: {
		RouterKeys:       []string{"kyc.requests.approve"},
		TTLDays:          90,
		ActionURLPattern: "/kyc/:entity_id",
		Category:         "kyc",
		Label:            "Yêu cầu xác minh danh tính (KYC)",
	},
}
```

- [ ] **Verify:** `make build` in marketplace-admin-api passes

---

## Task 2.2 — Create admin_notifications schema

**Read first:** `marketplace-backend/schema/notificationcol/model.go` and `query.go`

### `marketplace-admin-api/schema/adminnotificationcol/model.go`

- [ ] **Create** the file:

```go
package adminnotificationcol

import (
	"time"

	"github.com/kamva/mgm/v3"
)

// AdminNotification is one notification document scoped to a single admin.
// Fan-out on write: one document is created per eligible admin per event.
// Collection: "admin_notifications"
type AdminNotification struct {
	mgm.DefaultModel `json:",inline" bson:",inline,omitnested"`

	CreatedAt time.Time  `json:"created_at"              bson:"created_at"`
	UpdatedAt time.Time  `json:"updated_at,omitempty"     bson:"updated_at,omitempty"`

	// EventID is the UUID from the originating RabbitMQ event.
	// Combined with AdminID, forms the uniqueness key (compound index).
	EventID string `json:"event_id" bson:"event_id"`

	// AdminID is the recipient admin's ObjectID hex string.
	AdminID string `json:"admin_id" bson:"admin_id"`

	// NotificationType matches a key in notificationregistry.Registry.
	NotificationType string `json:"notification_type" bson:"notification_type"`

	// Category is the display group (e.g. "product", "kyc").
	Category string `json:"category" bson:"category"`

	// Title and Body are the display strings shown in the bell dropdown.
	Title string `json:"title" bson:"title"`
	Body  string `json:"body"  bson:"body"`

	// ActionURL is the relative admin frontend path to navigate to on click.
	ActionURL string `json:"action_url" bson:"action_url"`

	// EntityType and EntityID link the notification to its source document.
	EntityType string `json:"entity_type,omitempty" bson:"entity_type,omitempty"`
	EntityID   string `json:"entity_id,omitempty"   bson:"entity_id,omitempty"`

	// ReadAt is nil when unread. Set to now when admin opens the notification.
	ReadAt *time.Time `json:"read_at,omitempty" bson:"read_at,omitempty"`

	// Metadata holds optional extra data for future extensibility.
	Metadata map[string]any `json:"metadata,omitempty" bson:"metadata,omitempty"`
}

func (*AdminNotification) CollectionName() string {
	return "admin_notifications"
}
```

### `marketplace-admin-api/schema/adminnotificationcol/query.go`

- [ ] **Read** `marketplace-backend/schema/notificationcol/query.go` for the BSON query patterns
- [ ] **Create** the file with these functions:

```go
package adminnotificationcol

import (
	"context"
	"time"

	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/bson/primitive"
	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"

	"github.com/kamva/mgm/v3"
)

// InsertMany bulk-inserts notifications. Uses unordered insert so duplicate
// (event_id, admin_id) pairs are silently skipped (idempotent re-delivery).
func InsertMany(ctx context.Context, items []*AdminNotification) error {
	if len(items) == 0 {
		return nil
	}
	docs := make([]interface{}, len(items))
	for i, item := range items {
		docs[i] = item
	}
	coll := mgm.Coll(&AdminNotification{})
	_, err := coll.InsertMany(ctx, docs, options.InsertMany().SetOrdered(false))
	if err != nil {
		// Ignore duplicate key errors (E11000) — expected for idempotent re-delivery
		if mongo.IsDuplicateKeyError(err) {
			return nil
		}
		return err
	}
	return nil
}

// ListByAdminID returns paginated notifications for one admin, newest first.
func ListByAdminID(ctx context.Context, adminID string, limit, offset int, unreadOnly bool) ([]*AdminNotification, int64, error) {
	coll := mgm.Coll(&AdminNotification{})

	filter := bson.D{{Key: "admin_id", Value: adminID}}
	if unreadOnly {
		filter = append(filter, bson.E{Key: "read_at", Value: nil})
	}

	total, err := coll.CountDocuments(ctx, filter)
	if err != nil {
		return nil, 0, err
	}

	opts := options.Find().
		SetSort(bson.D{{Key: "created_at", Value: -1}}).
		SetLimit(int64(limit)).
		SetSkip(int64(offset))

	cursor, err := coll.Find(ctx, filter, opts)
	if err != nil {
		return nil, 0, err
	}
	defer cursor.Close(ctx)

	var results []*AdminNotification
	if err := cursor.All(ctx, &results); err != nil {
		return nil, 0, err
	}
	return results, total, nil
}

// GetByID returns a single notification by its ObjectID hex string.
func GetByID(ctx context.Context, id string) (*AdminNotification, error) {
	oid, err := primitive.ObjectIDFromHex(id)
	if err != nil {
		return nil, err
	}
	coll := mgm.Coll(&AdminNotification{})
	result := &AdminNotification{}
	if err := coll.FindOne(ctx, bson.D{{Key: "_id", Value: oid}}).Decode(result); err != nil {
		return nil, err
	}
	return result, nil
}

// MarkRead sets read_at to now for the given notification, scoped to adminID (owner check).
// Returns (true, nil) if updated, (false, nil) if not found or not owner.
func MarkRead(ctx context.Context, id, adminID string) (bool, error) {
	oid, err := primitive.ObjectIDFromHex(id)
	if err != nil {
		return false, nil
	}
	now := time.Now().UTC()
	coll := mgm.Coll(&AdminNotification{})
	res, err := coll.UpdateOne(ctx,
		bson.D{
			{Key: "_id", Value: oid},
			{Key: "admin_id", Value: adminID},
			{Key: "read_at", Value: nil}, // only update if currently unread
		},
		bson.D{{Key: "$set", Value: bson.D{{Key: "read_at", Value: now}}}},
	)
	if err != nil {
		return false, err
	}
	return res.ModifiedCount > 0, nil
}

// MarkAllRead sets read_at = now for all unread notifications of one admin.
func MarkAllRead(ctx context.Context, adminID string) error {
	now := time.Now().UTC()
	coll := mgm.Coll(&AdminNotification{})
	_, err := coll.UpdateMany(ctx,
		bson.D{
			{Key: "admin_id", Value: adminID},
			{Key: "read_at", Value: nil},
		},
		bson.D{{Key: "$set", Value: bson.D{{Key: "read_at", Value: now}}}},
	)
	return err
}

// CountUnread returns the number of unread notifications for one admin.
func CountUnread(ctx context.Context, adminID string) (int64, error) {
	coll := mgm.Coll(&AdminNotification{})
	return coll.CountDocuments(ctx, bson.D{
		{Key: "admin_id", Value: adminID},
		{Key: "read_at", Value: nil},
	})
}

// DeleteOlderThan removes notifications older than the given time.
// Used by the TTL cleanup cronjob. Returns count of deleted documents.
func DeleteOlderThan(ctx context.Context, before time.Time) (int64, error) {
	coll := mgm.Coll(&AdminNotification{})
	res, err := coll.DeleteMany(ctx, bson.D{
		{Key: "created_at", Value: bson.D{{Key: "$lt", Value: before}}},
	})
	if err != nil {
		return 0, err
	}
	return res.DeletedCount, nil
}
```

- [ ] **Verify:** `make build` passes

---

## Task 2.3 — Create processed_admin_notification_events schema

**Read first:** `marketplace-backend/schema/processednotificationcol/model.go` and `query.go`

### `marketplace-admin-api/schema/processedadminnotificationeventcol/model.go`

- [ ] **Create** the file:

```go
package processedadminnotificationeventcol

import (
	"time"

	"github.com/kamva/mgm/v3"
)

// ProcessedAdminNotificationEvent tracks which RabbitMQ events have already been
// processed by the admin notification consumer. The EventID (UUID) is used as the
// document _id to guarantee uniqueness and fast lookup.
// Collection: "processed_admin_notification_events"
type ProcessedAdminNotificationEvent struct {
	// ID stores the EventID string as the MongoDB _id (not an ObjectID).
	ID          string    `json:"id"           bson:"_id"`
	ProcessedAt time.Time `json:"processed_at" bson:"processed_at"`
}

func (*ProcessedAdminNotificationEvent) CollectionName() string {
	return "processed_admin_notification_events"
}
```

> **Note:** This struct does NOT embed `mgm.DefaultModel` because the `_id` is a string (EventID), not an ObjectID. Use `mgm.Coll(&ProcessedAdminNotificationEvent{})` directly.

### `marketplace-admin-api/schema/processedadminnotificationeventcol/query.go`

- [ ] **Create** the file:

```go
package processedadminnotificationeventcol

import (
	"context"
	"time"

	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/mongo"

	"github.com/kamva/mgm/v3"
)

// ExistsByEventID returns true if an event with this ID has already been processed.
func ExistsByEventID(ctx context.Context, eventID string) (bool, error) {
	coll := mgm.Coll(&ProcessedAdminNotificationEvent{})
	err := coll.FindOne(ctx, bson.D{{Key: "_id", Value: eventID}}).Err()
	if err == mongo.ErrNoDocuments {
		return false, nil
	}
	if err != nil {
		return false, err
	}
	return true, nil
}

// Insert marks an event as processed. Duplicate inserts are silently ignored
// (idempotent — safe to call even if event was already processed).
func Insert(ctx context.Context, eventID string) error {
	coll := mgm.Coll(&ProcessedAdminNotificationEvent{})
	doc := &ProcessedAdminNotificationEvent{
		ID:          eventID,
		ProcessedAt: time.Now().UTC(),
	}
	_, err := coll.InsertOne(ctx, doc)
	if err != nil && mongo.IsDuplicateKeyError(err) {
		return nil // already processed, not an error
	}
	return err
}
```

- [ ] **Verify:** `make build` passes

---

## Task 2.4 — Add MongoDB indexes

**File:** `marketplace-admin-api/migrate/create_index.go`

- [ ] **Read** the file to understand the existing index creation pattern
- [ ] **Add** indexes for both new collections at the end of the index creation function:

```go
// admin_notifications indexes
adminNotifColl := mgm.Coll(&adminnotificationcol.AdminNotification{})

// List queries: admin_id + created_at DESC
adminNotifColl.Indexes().CreateOne(ctx, mongo.IndexModel{
    Keys: bson.D{
        {Key: "admin_id", Value: 1},
        {Key: "created_at", Value: -1},
    },
})

// Unread count queries: admin_id + read_at
adminNotifColl.Indexes().CreateOne(ctx, mongo.IndexModel{
    Keys: bson.D{
        {Key: "admin_id", Value: 1},
        {Key: "read_at", Value: 1},
    },
})

// Deduplication: compound unique index on (event_id, admin_id)
adminNotifColl.Indexes().CreateOne(ctx, mongo.IndexModel{
    Keys: bson.D{
        {Key: "event_id", Value: 1},
        {Key: "admin_id", Value: 1},
    },
    Options: options.Index().SetUnique(true),
})

// TTL cleanup: created_at for range deletes
adminNotifColl.Indexes().CreateOne(ctx, mongo.IndexModel{
    Keys: bson.D{{Key: "created_at", Value: 1}},
})
```

- [ ] **Verify:** `make build` passes

---

## Verification Checklist

- [ ] `make build` in marketplace-admin-api: clean compile
- [ ] `notificationregistry` package compiles with 6 type constants + Registry map
- [ ] `adminnotificationcol` package compiles with model + all 6 query functions
- [ ] `processedadminnotificationeventcol` package compiles with model + 2 query functions
- [ ] Indexes added to `migrate/create_index.go`
- [ ] No import cycles introduced
- [ ] All JSON and BSON tags use snake_case
- [ ] Optional fields have `omitempty` on both json and bson tags
