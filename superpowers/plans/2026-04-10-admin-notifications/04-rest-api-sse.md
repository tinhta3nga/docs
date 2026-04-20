# Phase 4: marketplace-admin-api — REST API + SSE Endpoint

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` or `superpowers:executing-plans`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement 5 HTTP handlers for the admin notification API (list, unread-count, mark-read, mark-all-read, SSE stream), update the router, and mount the notification group in `ver1.go`. Every authenticated admin can access their own notifications — no `RequireRouterKey` on read endpoints. RBAC is enforced at fan-out time (consumer), not at read time.

**Tech Stack:** Go 1.24, Gin, JWT (for SSE auth via query param)

**Rule files to read first:** `workspace-context/rules/02-go-backend-shared.md`, `workspace-context/rules/04-marketplace-admin-api.md`

**Reference files to read:**
- `marketplace-backend/business/notification/list.go`, `unread_count.go`, `mark_read.go` — mirror these patterns
- `marketplace-admin-api/business/notification/router.go` — existing file (update it)
- `marketplace-admin-api/business/notification/send.go` — keep this, do not remove it
- `marketplace-admin-api/routers/ver1.go` — add mount point
- `marketplace-admin-api/internal/response/` — SuccessResponse, SuccessResponseWithPaging, ErrorResponse
- `marketplace-admin-api/internal/jwt/jwt.go` — Verify function signature
- `marketplace-admin-api/middleware/auth.go` — understand how current_admin is set

**Depends on:** Phase 2 (adminnotificationcol) + Phase 3 (ssehub)

---

## File Map

| Action | File |
|---|---|
| Edit | `marketplace-admin-api/business/notification/router.go` |
| Create | `marketplace-admin-api/business/notification/list.go` |
| Create | `marketplace-admin-api/business/notification/unread_count.go` |
| Create | `marketplace-admin-api/business/notification/mark_read.go` |
| Create | `marketplace-admin-api/business/notification/mark_all_read.go` |
| Create | `marketplace-admin-api/business/notification/stream.go` |
| Edit | `marketplace-admin-api/routers/ver1.go` |
| Edit | `marketplace-admin-api/business/role/keys.go` |

---

## Task 4.1 — Update the notification router

**File:** `marketplace-admin-api/business/notification/router.go`

- [ ] **Read** the existing file (`send.go` must be preserved)
- [ ] **Replace** the router content — keep `Send()` route, add new routes:

```go
package notification

import (
	"github.com/gin-gonic/gin"

	"marketplace-admin-api/middleware"
)

func Router(r *gin.RouterGroup) {
	// Existing: admin sends notification to marketplace users (keep)
	r.POST("/send", middleware.AuthMiddleware(), Send())

	// New: admin reads their own notifications (no RequireRouterKey — every admin gets their own feed)
	r.GET("", middleware.AuthMiddleware(), List())
	r.GET("/unread-count", middleware.AuthMiddleware(), UnreadCount())
	r.PATCH("/:id/read", middleware.AuthMiddleware(), MarkRead())
	r.POST("/read-all", middleware.AuthMiddleware(), MarkAllRead())

	// SSE stream — auth via query param (browser EventSource cannot send headers)
	r.GET("/stream", Stream())
}
```

- [ ] **Verify:** `make build` passes

---

## Task 4.2 — Implement List handler

**File:** `marketplace-admin-api/business/notification/list.go`

- [ ] **Read** `marketplace-backend/business/notification/list.go` for the pattern
- [ ] **Create** the file following the handler factory pattern:

```go
package notification

import (
	"net/http"

	"github.com/gin-gonic/gin"

	"marketplace-admin-api/internal/plog"
	"marketplace-admin-api/internal/response"
	"marketplace-admin-api/schema/adminnotificationcol"
	"marketplace-admin-api/schema/admincol"
)

// List returns paginated notifications for the currently authenticated admin.
// GET /notifications?limit=20&offset=0&unread_only=false
func List() gin.HandlerFunc {
	logger := plog.NewBizLogger("[business][notification][list]")
	return func(c *gin.Context) {
		admin, ok := currentAdmin(c)
		if !ok {
			return
		}

		var req listRequest
		if err := c.ShouldBindQuery(&req); err != nil {
			logger.Warn().Err(err).Msg("bind query failed")
			code := response.ErrorResponse(err.Error())
			c.JSON(code.Code, code)
			c.Abort()
			return
		}
		req.setDefaults()

		items, total, err := adminnotificationcol.ListByAdminID(
			c.Request.Context(),
			admin.GetIDString(),
			req.Limit,
			req.Offset,
			req.UnreadOnly,
		)
		if err != nil {
			logger.Error().Err(err).Msg("list notifications failed")
			code := response.ErrorResponse(err.Error())
			c.JSON(code.Code, code)
			c.Abort()
			return
		}

		page := req.Offset/req.Limit + 1
		hasNext := int64(req.Offset+req.Limit) < total
		c.JSON(http.StatusOK, response.SuccessResponseWithPaging(items, response.Paging{
			Current: page,
			HasNext: hasNext,
			ItemNum: int(total),
		}))
	}
}

type listRequest struct {
	Limit      int  `form:"limit"`
	Offset     int  `form:"offset"`
	UnreadOnly bool `form:"unread_only"`
}

func (r *listRequest) setDefaults() {
	if r.Limit <= 0 {
		r.Limit = 20
	}
	if r.Limit > 100 {
		r.Limit = 100
	}
	if r.Offset < 0 {
		r.Offset = 0
	}
}

// currentAdmin extracts the authenticated admin from gin context.
// Returns (nil, false) and writes a 401 response if not found.
func currentAdmin(c *gin.Context) (*admincol.Admin, bool) {
	val, exists := c.Get("current_admin")
	if !exists {
		code := response.ErrorResponse("unauthorized")
		c.JSON(http.StatusUnauthorized, code)
		c.Abort()
		return nil, false
	}
	admin, ok := val.(*admincol.Admin)
	if !ok {
		code := response.ErrorResponse("unauthorized")
		c.JSON(http.StatusUnauthorized, code)
		c.Abort()
		return nil, false
	}
	return admin, true
}
```

- [ ] **Verify:** `make build` passes

---

## Task 4.3 — Implement UnreadCount handler

**File:** `marketplace-admin-api/business/notification/unread_count.go`

```go
package notification

import (
	"net/http"

	"github.com/gin-gonic/gin"

	"marketplace-admin-api/internal/plog"
	"marketplace-admin-api/internal/response"
	"marketplace-admin-api/schema/adminnotificationcol"
)

// UnreadCount returns the number of unread notifications for the current admin.
// GET /notifications/unread-count
func UnreadCount() gin.HandlerFunc {
	logger := plog.NewBizLogger("[business][notification][unread_count]")
	return func(c *gin.Context) {
		admin, ok := currentAdmin(c)
		if !ok {
			return
		}

		count, err := adminnotificationcol.CountUnread(c.Request.Context(), admin.GetIDString())
		if err != nil {
			logger.Error().Err(err).Msg("count unread failed")
			code := response.ErrorResponse(err.Error())
			c.JSON(code.Code, code)
			c.Abort()
			return
		}

		c.JSON(http.StatusOK, response.SuccessResponse(map[string]int64{
			"unread_count": count,
		}))
	}
}
```

- [ ] **Verify:** `make build` passes

---

## Task 4.4 — Implement MarkRead handler

**File:** `marketplace-admin-api/business/notification/mark_read.go`

```go
package notification

import (
	"net/http"

	"github.com/gin-gonic/gin"

	"marketplace-admin-api/internal/plog"
	"marketplace-admin-api/internal/response"
	"marketplace-admin-api/schema/adminnotificationcol"
)

// MarkRead marks a single notification as read (owner-scoped).
// PATCH /notifications/:id/read
func MarkRead() gin.HandlerFunc {
	logger := plog.NewBizLogger("[business][notification][mark_read]")
	return func(c *gin.Context) {
		admin, ok := currentAdmin(c)
		if !ok {
			return
		}

		id := c.Param("id")
		if id == "" {
			code := response.ErrorResponse("notification id required")
			c.JSON(code.Code, code)
			c.Abort()
			return
		}

		updated, err := adminnotificationcol.MarkRead(c.Request.Context(), id, admin.GetIDString())
		if err != nil {
			logger.Error().Err(err).Str("id", id).Msg("mark read failed")
			code := response.ErrorResponse(err.Error())
			c.JSON(code.Code, code)
			c.Abort()
			return
		}
		if !updated {
			// Not found or not owner — return 404
			c.JSON(http.StatusNotFound, response.ErrorResponse("notification not found"))
			c.Abort()
			return
		}

		c.JSON(http.StatusOK, response.SuccessResponse(nil))
	}
}
```

- [ ] **Verify:** `make build` passes

---

## Task 4.5 — Implement MarkAllRead handler

**File:** `marketplace-admin-api/business/notification/mark_all_read.go`

```go
package notification

import (
	"net/http"

	"github.com/gin-gonic/gin"

	"marketplace-admin-api/internal/plog"
	"marketplace-admin-api/internal/response"
	"marketplace-admin-api/schema/adminnotificationcol"
)

// MarkAllRead marks all unread notifications as read for the current admin.
// POST /notifications/read-all
func MarkAllRead() gin.HandlerFunc {
	logger := plog.NewBizLogger("[business][notification][mark_all_read]")
	return func(c *gin.Context) {
		admin, ok := currentAdmin(c)
		if !ok {
			return
		}

		if err := adminnotificationcol.MarkAllRead(c.Request.Context(), admin.GetIDString()); err != nil {
			logger.Error().Err(err).Msg("mark all read failed")
			code := response.ErrorResponse(err.Error())
			c.JSON(code.Code, code)
			c.Abort()
			return
		}

		c.JSON(http.StatusOK, response.SuccessResponse(nil))
	}
}
```

- [ ] **Verify:** `make build` passes

---

## Task 4.6 — Implement SSE Stream handler

**File:** `marketplace-admin-api/business/notification/stream.go`

> **Why auth via query param?** Browser `EventSource` API does not support custom request headers. The token must be passed as a URL query parameter. The SSE handler does its own JWT validation instead of using `AuthMiddleware()`.

> **Why `X-Accel-Buffering: no`?** Nginx buffers responses by default, which breaks SSE. This header disables buffering for this endpoint.

- [ ] **Read** `marketplace-admin-api/internal/jwt/jwt.go` to confirm the `Verify(token, key)` function signature
- [ ] **Read** `marketplace-admin-api/middleware/auth.go` to understand how email is extracted from claims and how the admin is loaded
- [ ] **Create** the file:

```go
package notification

import (
	"io"
	"net/http"
	"os"

	"github.com/gin-gonic/gin"

	"marketplace-admin-api/internal/jwt"
	"marketplace-admin-api/internal/plog"
	"marketplace-admin-api/internal/response"
	"marketplace-admin-api/internal/ssehub"
	"marketplace-admin-api/schema/admincol"
)

// Stream opens a Server-Sent Events connection for real-time notification push.
// GET /notifications/stream?token=<JWT>
//
// Auth is via query param because browser EventSource cannot send custom headers.
// The token is validated as a standard admin JWT (same signing key as AuthMiddleware).
func Stream() gin.HandlerFunc {
	logger := plog.NewBizLogger("[business][notification][stream]")
	return func(c *gin.Context) {
		// 1. Validate JWT from query param
		token := c.Query("token")
		if token == "" {
			c.JSON(http.StatusUnauthorized, response.ErrorResponse("token required"))
			c.Abort()
			return
		}

		claims, err := jwt.Verify(token, os.Getenv("KEY_API_KEY"))
		if err != nil {
			logger.Warn().Err(err).Msg("invalid SSE token")
			c.JSON(http.StatusUnauthorized, response.ErrorResponse("unauthorized"))
			c.Abort()
			return
		}

		// 2. Load admin by email from claims
		admin, err := admincol.FindByEmail(c.Request.Context(), claims.Email)
		if err != nil || !admin.Active {
			logger.Warn().Str("email", claims.Email).Msg("admin not found or inactive for SSE")
			c.JSON(http.StatusUnauthorized, response.ErrorResponse("unauthorized"))
			c.Abort()
			return
		}

		// 3. Set SSE headers
		c.Header("Content-Type", "text/event-stream")
		c.Header("Cache-Control", "no-cache")
		c.Header("Connection", "keep-alive")
		c.Header("X-Accel-Buffering", "no") // disable nginx buffering

		// 4. Subscribe to hub
		ch := ssehub.Default().Subscribe(admin.GetIDString())
		defer ssehub.Default().Unsubscribe(admin.GetIDString(), ch)

		// 5. Send initial ping so frontend knows connection is alive
		c.SSEvent("ping", "connected")
		c.Writer.Flush()

		logger.Info().Str("admin_id", admin.GetIDString()).Msg("SSE stream connected")

		// 6. Stream events until client disconnects
		c.Stream(func(w io.Writer) bool {
			select {
			case event, ok := <-ch:
				if !ok {
					return false // channel closed
				}
				c.SSEvent("notification", event)
				return true
			case <-c.Request.Context().Done():
				logger.Info().Str("admin_id", admin.GetIDString()).Msg("SSE stream disconnected")
				return false
			}
		})
	}
}
```

- [ ] **Check** `jwt.Verify` signature — adjust call if the function name or parameter order differs
- [ ] **Check** `admincol.FindByEmail` exists — if not, find the equivalent function in admincol queries
- [ ] **Verify:** `make build` passes

---

## Task 4.7 — Mount in ver1.go

**File:** `marketplace-admin-api/routers/ver1.go`

- [ ] **Read** the file to find the existing notification block (it may be commented out or missing)
- [ ] **Add** the mount point in the appropriate alphabetical/logical position:

```go
notificationGroup := v1.Group("notifications")
notification.Router(notificationGroup)
```

- [ ] **Verify:** `make build` passes

---

## Task 4.8 — Add router key constants (future-proofing)

**File:** `marketplace-admin-api/business/role/keys.go`

- [ ] **Read** the existing constants to follow naming convention
- [ ] **Add** at the end of the notification-related section (or create one):

```go
// Notification management (reserved for future admin-only controls)
KeyNotificationSend = "notification.send"
KeyNotificationList = "notification.list"
```

> These keys are not used by middleware on any current route — they are reserved for future admin-only notification management features. The read endpoints are deliberately open to all authenticated admins.

- [ ] **Verify:** `make build` passes

---

## Verification Checklist

- [ ] `make build`: clean compile
- [ ] `make test`: no regressions
- [ ] `GET /notifications` returns paginated list, newest first, scoped to current admin
- [ ] `GET /notifications/unread-count` returns `{"unread_count": N}`
- [ ] `PATCH /notifications/:id/read` returns 200 if updated, 404 if not found/not owner
- [ ] `POST /notifications/read-all` updates all unread for current admin
- [ ] `GET /notifications/stream?token=<valid>` opens SSE stream, sends `ping` on connect
- [ ] `GET /notifications/stream?token=<invalid>` returns 401
- [ ] SSE stream closes cleanly when client disconnects (context done)
- [ ] `notification.Router` is mounted in ver1.go (confirmed by checking route list at startup)
- [ ] Existing `POST /notifications/send` still works (not removed)

---

## API Response Reference

```
GET /notifications
→ { success: true, paging: { current, hasNext, itemNum }, data: [...] }

GET /notifications/unread-count
→ { success: true, data: { unread_count: 42 } }

PATCH /notifications/:id/read
→ 200: { success: true, data: null }
→ 404: { success: false, message: "notification not found" }

POST /notifications/read-all
→ { success: true, data: null }

GET /notifications/stream?token=xxx
→ Content-Type: text/event-stream
→ event: ping\ndata: "connected"\n\n
→ event: notification\ndata: {...SSEEvent JSON...}\n\n
```
