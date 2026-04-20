# Phase 5: marketplace-admin-api — TTL Cleanup Cronjob

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` or `superpowers:executing-plans`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a daily cronjob that deletes `admin_notifications` documents older than 90 days. Prevents unbounded collection growth. Follows the exact cronjob pattern already used in the codebase.

**Tech Stack:** Go 1.24, ticker-based goroutine (same as EncodingChainProcessor)

**Rule files to read first:** `workspace-context/rules/02-go-backend-shared.md` (Cronjob Pattern section), `workspace-context/rules/04-marketplace-admin-api.md`

**Reference files to read:**
- `marketplace-admin-api/services/cronjob/` — read any existing cronjob file to understand the exact pattern
- `marketplace-admin-api/main.go` — understand where cronjobs are wired

**Depends on:** Phase 2 (`adminnotificationcol.DeleteOlderThan` must exist)

---

## File Map

| Action | File |
|---|---|
| Create | `marketplace-admin-api/services/cronjob/admin_notification_ttl.go` |
| Edit | `marketplace-admin-api/main.go` |

---

## Task 5.1 — Create the TTL cronjob

**File:** `marketplace-admin-api/services/cronjob/admin_notification_ttl.go`

- [ ] **Read** one existing cronjob file (e.g., `encoding_chain_processor.go`) to confirm the exact struct/method/ticker pattern used
- [ ] **Create** the file:

```go
package cronjob

import (
	"context"
	"time"

	"marketplace-admin-api/internal/plog"
	"marketplace-admin-api/schema/adminnotificationcol"
)

// AdminNotificationTTLJob deletes admin_notifications documents older than ttlDays.
// Runs on a configurable interval (default: daily).
type AdminNotificationTTLJob struct {
	interval time.Duration
	ttlDays  int
}

// NewAdminNotificationTTLJob creates a new TTL cleanup job.
// interval: how often to run (e.g. 24*time.Hour for daily)
// ttlDays: notifications older than this many days will be deleted
func NewAdminNotificationTTLJob(interval time.Duration, ttlDays int) *AdminNotificationTTLJob {
	return &AdminNotificationTTLJob{
		interval: interval,
		ttlDays:  ttlDays,
	}
}

// Start runs the TTL job immediately, then repeats on the configured interval.
// Stops cleanly when ctx is cancelled.
// Call as: go job.Start(ctx)
func (j *AdminNotificationTTLJob) Start(ctx context.Context) {
	logger := plog.NewBizLogger("[services][cronjob][admin_notification_ttl]")
	logger.Info().Int("ttl_days", j.ttlDays).Msg("admin notification TTL job starting")

	ticker := time.NewTicker(j.interval)
	defer ticker.Stop()

	// Run immediately on start (don't wait for first tick)
	j.run(ctx, logger)

	for {
		select {
		case <-ctx.Done():
			logger.Info().Msg("admin notification TTL job stopped")
			return
		case <-ticker.C:
			j.run(ctx, logger)
		}
	}
}

func (j *AdminNotificationTTLJob) run(ctx context.Context, logger plog.Logger) {
	before := time.Now().UTC().AddDate(0, 0, -j.ttlDays)
	deleted, err := adminnotificationcol.DeleteOlderThan(ctx, before)
	if err != nil {
		logger.Error().Err(err).Msg("admin notification TTL cleanup failed")
		return
	}
	if deleted > 0 {
		logger.Info().Int64("deleted", deleted).Msg("admin notification TTL cleanup complete")
	}
}
```

> **Note:** Check the `plog.Logger` interface in `marketplace-admin-api/internal/plog/` — if it is an interface type, use it; if `NewBizLogger` returns a concrete type, use that. Adjust the `run` method signature accordingly to match the existing codebase pattern.

- [ ] **Verify:** `make build` passes

---

## Task 5.2 — Wire in main.go

**File:** `marketplace-admin-api/main.go`

- [ ] **Read** main.go to find where existing cronjobs are started (look for `go encodingChainProcessor.Start(ctx)` or similar)
- [ ] **Add** the TTL job after the existing cronjob wires:

```go
// Admin notification TTL cleanup (runs daily, removes notifications older than 90 days)
ttlJob := cronjob.NewAdminNotificationTTLJob(24*time.Hour, 90)
go ttlJob.Start(ctx)
```

- [ ] **Verify:** `make build` passes, `make test` shows no regressions

---

## Verification Checklist

- [ ] `make build`: clean compile
- [ ] `make test`: no regressions
- [ ] Cronjob struct follows existing codebase pattern exactly (ticker, immediate first run, ctx.Done stop)
- [ ] Logger name format: `[services][cronjob][admin_notification_ttl]`
- [ ] `DeleteOlderThan` is called with UTC timestamp, 90 days back
- [ ] Cronjob launched as goroutine in `main.go` after service singletons
- [ ] Non-zero delete count is logged at Info level
- [ ] Zero delete count is silently skipped (no unnecessary log spam)
- [ ] Error is logged at Error level but does not crash the process
