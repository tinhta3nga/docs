# Phase 1: marketplace-backend — Publish New Admin Notification Events

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` or `superpowers:executing-plans` to implement task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add 6 new RabbitMQ event types that marketplace-backend publishes whenever users submit publication, exploitation, encoding, or KYC requests. These events flow to marketplace-admin-api which fans them out to eligible admins.

**Tech Stack:** Go 1.24, Gin, RabbitMQ (fire-and-forget `PublishEvent` pattern)

**Rule files to read first:** `workspace-context/rules/02-go-backend-shared.md`, `workspace-context/rules/03-marketplace-backend.md`

---

## File Map

| Action | File |
|---|---|
| Edit | `marketplace-backend/internal/rabbitmq/event.go` |
| Create | `marketplace-backend/internal/rabbitmq/events/admin_notification_events.go` |
| Edit | `marketplace-backend/business/product/original/<submit_handler>.go` — find the handler that creates/submits a publication copyright request |
| Edit | `marketplace-backend/business/product/original/<resubmit_handler>.go` — find the handler for resubmission after rejection |
| Edit | `marketplace-backend/business/product/` — find where exploitation/protection requests are submitted |
| Edit | `marketplace-backend/business/document/` — find the encoding request submission handler |
| Edit | `marketplace-backend/business/account/` — find the KYC submission handler |

> **Before editing any handler:** Read the full file first. Find the exact point after successful DB write. Follow the fire-and-forget pattern already used in the codebase (e.g., see `business/account/legal_advice/create.go`).

---

## Task 1.1 — Add event constants

**File:** `marketplace-backend/internal/rabbitmq/event.go`

- [ ] **Read the file** to understand the existing constant block
- [ ] **Add 6 new constants** after the existing block (do not reorder existing ones):

```go
// Admin notification events — user-initiated submissions that notify admins
EventUserPublicationRequestSubmitted    = "user.publication_request.submitted"
EventUserPublicationRequestResubmitted  = "user.publication_request.resubmitted"
EventUserExploitationRequestSubmitted   = "user.exploitation_request.submitted"
EventUserExploitationRequestResubmitted = "user.exploitation_request.resubmitted"
EventUserEncodingRequestSubmitted       = "user.encoding_request.submitted"
EventUserKYCSubmitted                   = "user.kyc.submitted"
```

- [ ] **Verify:** `make build` in marketplace-backend passes

---

## Task 1.2 — Add payload structs

**New file:** `marketplace-backend/internal/rabbitmq/events/admin_notification_events.go`

- [ ] **Read** `marketplace-backend/internal/rabbitmq/events/notification_events.go` to understand the existing payload struct pattern
- [ ] **Create** the new file:

```go
package events

import "time"

// PublicationRequestSubmittedPayload is published when a user submits or resubmits
// a publication copyright registration request.
type PublicationRequestSubmittedPayload struct {
	RequestID      string    `json:"request_id"`
	UserID         string    `json:"user_id"`
	UserName       string    `json:"user_name"`
	IsResubmission bool      `json:"is_resubmission"`
	Round          int       `json:"round,omitempty"` // 0 for first submission, N for resubmission
	CreatedAt      time.Time `json:"created_at"`
}

// ExploitationRequestSubmittedPayload is published when a user submits or resubmits
// an exploitation (formerly protection) registration request.
type ExploitationRequestSubmittedPayload struct {
	RequestID      string    `json:"request_id"`
	UserID         string    `json:"user_id"`
	UserName       string    `json:"user_name"`
	IsResubmission bool      `json:"is_resubmission"`
	Round          int       `json:"round,omitempty"`
	CreatedAt      time.Time `json:"created_at"`
}

// EncodingRequestSubmittedPayload is published when a user submits an encoding request.
type EncodingRequestSubmittedPayload struct {
	RequestID string    `json:"request_id"`
	UserID    string    `json:"user_id"`
	UserName  string    `json:"user_name"`
	CreatedAt time.Time `json:"created_at"`
}

// KYCSubmittedPayload is published when a user submits their KYC identity verification.
type KYCSubmittedPayload struct {
	UserID    string    `json:"user_id"`
	UserName  string    `json:"user_name"`
	CreatedAt time.Time `json:"created_at"`
}
```

- [ ] **Verify:** `make build` passes

---

## Task 1.3 — Add publish call: Publication request (first submission)

**Goal:** Find and edit the handler that creates a new publication/copyright registration request.

- [ ] **Explore** `marketplace-backend/business/product/original/` — list all files, find the create/submit handler (likely `create.go`, `submit.go`, or a file that inserts into a `copyright_registration_requests`-like collection)
- [ ] **Read** the handler fully before editing
- [ ] **Add** fire-and-forget publish after the successful DB insert:

```go
// After the DB insert succeeds
_ = rabbitmq.PublishEvent(c.Request.Context(),
    rabbitmq.EventUserPublicationRequestSubmitted,
    events.PublicationRequestSubmittedPayload{
        RequestID:      doc.GetIDString(),
        UserID:         currentUser.GetIDString(),
        UserName:       currentUser.FullName,
        IsResubmission: false,
        Round:          0,
        CreatedAt:      doc.CreatedAt,
    },
)
```

- [ ] **Verify:** `make build` passes

---

## Task 1.4 — Add publish call: Publication request resubmission

**Goal:** Find the handler where a user resubmits a rejected publication request.

- [ ] **Explore** `marketplace-backend/business/product/original/` — find the update/resubmit handler (look for status transitions like `rejected → pending` or `adjustment → pending`)
- [ ] **Read** the handler fully
- [ ] **Add** publish call after successful update. Determine the resubmission round from the document if available; default to 1 if not tracked:

```go
_ = rabbitmq.PublishEvent(c.Request.Context(),
    rabbitmq.EventUserPublicationRequestResubmitted,
    events.PublicationRequestSubmittedPayload{
        RequestID:      doc.GetIDString(),
        UserID:         currentUser.GetIDString(),
        UserName:       currentUser.FullName,
        IsResubmission: true,
        Round:          1, // increment if the schema tracks resubmission count
        CreatedAt:      time.Now().UTC(),
    },
)
```

- [ ] **Verify:** `make build` passes

---

## Task 1.5 — Add publish call: Exploitation (protection) request submission

> **Note:** "exploitation" = "protection" in the old schema. The collection is `protection_registration_requests`. Use the new event name `EventUserExploitationRequestSubmitted`.

- [ ] **Explore** `marketplace-backend/business/product/` — find exploitation/protection request create handler (check `business/product/derivative/` or any `protection` directory)
- [ ] **Read** the handler fully
- [ ] **Add** publish call (first submission):

```go
_ = rabbitmq.PublishEvent(c.Request.Context(),
    rabbitmq.EventUserExploitationRequestSubmitted,
    events.ExploitationRequestSubmittedPayload{
        RequestID:      doc.GetIDString(),
        UserID:         currentUser.GetIDString(),
        UserName:       currentUser.FullName,
        IsResubmission: false,
        Round:          0,
        CreatedAt:      doc.CreatedAt,
    },
)
```

- [ ] **Also find** the resubmission handler and add `EventUserExploitationRequestResubmitted` publish
- [ ] **Verify:** `make build` passes

---

## Task 1.6 — Add publish call: Encoding request submission

- [ ] **Explore** `marketplace-backend/business/document/` — find the handler that creates an encoding request (look for writes to an encoding or document encoding collection)
- [ ] **Read** the handler fully
- [ ] **Add** publish call:

```go
_ = rabbitmq.PublishEvent(c.Request.Context(),
    rabbitmq.EventUserEncodingRequestSubmitted,
    events.EncodingRequestSubmittedPayload{
        RequestID: doc.GetIDString(),
        UserID:    currentUser.GetIDString(),
        UserName:  currentUser.FullName,
        CreatedAt: doc.CreatedAt,
    },
)
```

- [ ] **Verify:** `make build` passes

---

## Task 1.7 — Add publish call: KYC submission

- [ ] **Explore** `marketplace-backend/business/account/` — find the KYC submission handler (look for handlers that submit identity verification documents to the eKYC provider or write KYC status)
- [ ] **Read** the handler fully
- [ ] **Add** publish call:

```go
_ = rabbitmq.PublishEvent(c.Request.Context(),
    rabbitmq.EventUserKYCSubmitted,
    events.KYCSubmittedPayload{
        UserID:    currentUser.GetIDString(),
        UserName:  currentUser.FullName,
        CreatedAt: time.Now().UTC(),
    },
)
```

- [ ] **Verify:** `make build` passes, `make test` shows no regressions

---

## Verification Checklist

- [ ] `make build` in marketplace-backend: clean compile
- [ ] `make test` in marketplace-backend: no regressions
- [ ] 6 new event constants in `event.go`
- [ ] 4 payload structs in `admin_notification_events.go`
- [ ] 6 publish calls total across the 4–6 handler files (publication submit, publication resubmit, exploitation submit, exploitation resubmit, encoding submit, KYC submit)
- [ ] All publish calls follow fire-and-forget pattern: wrapped in `_ = rabbitmq.PublishEvent(...)`
- [ ] No publish calls block on error or return the error to the HTTP caller

---

## Fire-and-Forget Pattern Reference

```go
// CORRECT — fire and forget
_ = rabbitmq.PublishEvent(c.Request.Context(), routingKey, payload)

// WRONG — do not return the error
if err := rabbitmq.PublishEvent(...); err != nil {
    return err  // ❌ this blocks the user's request
}
```

Reference implementation: `marketplace-backend/business/account/legal_advice/create.go`
