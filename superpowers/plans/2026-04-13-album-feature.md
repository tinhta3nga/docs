# Album Feature Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the full-stack Album feature — users can group their own products into curated albums, with private/public visibility and a Spotify-style detail page showing edit controls only to the owner.

**Architecture:** Go backend adds an `album` MongoDB collection with 8 REST endpoints registered under `/v1/albums`. Next.js frontend adds a My Albums list page at `/profile/my-albums` and an Album Detail page at `/albums/[id]` that serves both the owner (with edit controls) and the public (read-only) via the same URL.

**Tech Stack:** Go 1.24, Gin, MongoDB (mgm), AWS S3; Next.js 15 App Router, Ant Design 5, TanStack Query, TypeScript

---

## File Structure

### Backend — new files
```
marketplace-backend/
  schema/albumcol/
    model.go          ← Album struct + CollectionName()
    query.go          ← Collection(), CRUD helpers, AddProducts, RemoveProducts
  business/album/
    router.go         ← AddRouter() — 7 routes
    create.go         ← POST /albums
    list_my.go        ← GET /albums/my
    get_detail.go     ← GET /albums/:id
    update.go         ← PUT /albums/:id
    delete.go         ← DELETE /albums/:id
    add_products.go   ← POST /albums/:id/products
    remove_products.go← DELETE /albums/:id/products
```

### Backend — modified files
```
marketplace-backend/routers/ver1.go  ← import album package + register router
```

### Frontend — new files
```
marketplace-frontend/
  app/api/album.ts
  app/[locale]/(profile)/profile/my-albums/page.tsx
  app/[locale]/(profile)/profile/my-albums/components/
    AlbumCard.tsx
    CreateAlbumModal.tsx
    EditAlbumModal.tsx
  app/[locale]/(album)/albums/[id]/page.tsx
  common/pages/Album/
    AlbumDetailPage.tsx
    AddProductsDrawer.tsx
```

### Frontend — modified files
```
marketplace-frontend/config/NavigationPath.ts
marketplace-frontend/common/components/Header/data.tsx
marketplace-frontend/common/components/ProfileLayout/SettingsLayout.tsx
```

---

## Task 1: Backend — Album Schema

**Files:**
- Create: `marketplace-backend/schema/albumcol/model.go`
- Create: `marketplace-backend/schema/albumcol/query.go`

- [ ] **Step 1: Create `model.go`**

```go
// marketplace-backend/schema/albumcol/model.go
package albumcol

import (
	"time"

	"api/internal/mongodb"

	"go.mongodb.org/mongo-driver/bson/primitive"
)

type Album struct {
	mongodb.DefaultModel `json:",inline" bson:",inline,omitnested"`
	CreatedAt            time.Time            `json:"created_at" bson:"created_at"`
	UpdatedAt            time.Time            `json:"updated_at" bson:"updated_at"`

	UserID      primitive.ObjectID   `json:"user_id" bson:"user_id"`
	Name        string               `json:"name" bson:"name"`
	Description string               `json:"description" bson:"description"`
	Category    string               `json:"category" bson:"category"` // category_code e.g. "music"
	CoverURL    string               `json:"cover_url" bson:"cover_url"`
	IsPublic    bool                 `json:"is_public" bson:"is_public"`
	ProductIDs  []primitive.ObjectID `json:"product_ids" bson:"product_ids"`
	IsDeleted   bool                 `json:"is_deleted" bson:"is_deleted"`
}

func (Album) CollectionName() string { return "album" }
```

- [ ] **Step 2: Create `query.go`**

```go
// marketplace-backend/schema/albumcol/query.go
package albumcol

import (
	"api/internal/mongodb"
	"context"

	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/bson/primitive"
	"go.mongodb.org/mongo-driver/mongo/options"
)

func Collection() *mongodb.Collection {
	return mongodb.CollectionByName(mongodb.GetDatabaseName(), "album")
}

func Create(ctx context.Context, album *Album) (interface{}, error) {
	return Collection().CreateWithCtx(ctx, album)
}

func FindByID(ctx context.Context, id string) (*Album, error) {
	oid, err := primitive.ObjectIDFromHex(id)
	if err != nil {
		return nil, err
	}
	var album Album
	err = Collection().FirstWithCtx(ctx, bson.M{"_id": oid, "is_deleted": false}, &album)
	if err != nil {
		return nil, err
	}
	return &album, nil
}

func FindByUserID(ctx context.Context, userID string, page, size int) ([]*Album, int64, error) {
	oid, err := primitive.ObjectIDFromHex(userID)
	if err != nil {
		return nil, 0, err
	}
	filter := bson.M{"user_id": oid, "is_deleted": false}

	total, err := Collection().CountWithCtx(ctx, filter)
	if err != nil {
		return nil, 0, err
	}

	var albums []*Album
	opts := options.Find().
		SetSkip(int64((page - 1) * size)).
		SetLimit(int64(size)).
		SetSort(bson.M{"created_at": -1})
	err = Collection().SimpleFindWithCtx(ctx, &albums, filter, opts)
	if err != nil {
		return nil, 0, err
	}
	return albums, total, nil
}

func FindPublicByID(ctx context.Context, id string) (*Album, error) {
	oid, err := primitive.ObjectIDFromHex(id)
	if err != nil {
		return nil, err
	}
	var album Album
	err = Collection().FirstWithCtx(ctx, bson.M{"_id": oid, "is_deleted": false, "is_public": true}, &album)
	if err != nil {
		return nil, err
	}
	return &album, nil
}

func Update(ctx context.Context, album *Album) error {
	return Collection().UpdateWithCtx(ctx, album)
}

func SoftDelete(ctx context.Context, id string, userID primitive.ObjectID) error {
	oid, err := primitive.ObjectIDFromHex(id)
	if err != nil {
		return err
	}
	_, err = Collection().UpdateOne(ctx,
		bson.M{"_id": oid, "user_id": userID},
		bson.M{"$set": bson.M{"is_deleted": true}},
	)
	return err
}

func AddProducts(ctx context.Context, id string, userID primitive.ObjectID, productOIDs []primitive.ObjectID) error {
	oid, err := primitive.ObjectIDFromHex(id)
	if err != nil {
		return err
	}
	_, err = Collection().UpdateOne(ctx,
		bson.M{"_id": oid, "user_id": userID},
		bson.M{"$addToSet": bson.M{"product_ids": bson.M{"$each": productOIDs}}},
	)
	return err
}

func RemoveProducts(ctx context.Context, id string, userID primitive.ObjectID, productOIDs []primitive.ObjectID) error {
	oid, err := primitive.ObjectIDFromHex(id)
	if err != nil {
		return err
	}
	_, err = Collection().UpdateOne(ctx,
		bson.M{"_id": oid, "user_id": userID},
		bson.M{"$pull": bson.M{"product_ids": bson.M{"$in": productOIDs}}},
	)
	return err
}
```

- [ ] **Step 3: Verify it compiles**

```bash
cd /Users/macos/workspace-user/marketplace-backend
go build ./schema/albumcol/...
```

Expected: no output (success).

- [ ] **Step 4: Commit**

```bash
cd /Users/macos/workspace-user/marketplace-backend
git add schema/albumcol/model.go schema/albumcol/query.go
git commit -m "$(cat <<'EOF'
feat(marketplace-backend): add albumcol schema and CRUD query helpers

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 2: Backend — Router + Create Handler

**Files:**
- Create: `marketplace-backend/business/album/router.go`
- Create: `marketplace-backend/business/album/create.go`

- [ ] **Step 1: Create `router.go`**

```go
// marketplace-backend/business/album/router.go
package album

import (
	"api/middleware"

	"github.com/gin-gonic/gin"
)

func AddRouter(r *gin.RouterGroup) {
	r.POST("", middleware.AuthMiddleware(), Create())
	r.GET("/my", middleware.AuthMiddleware(), ListMy())
	r.GET("/:id", GetDetail())
	r.PUT("/:id", middleware.AuthMiddleware(), Update())
	r.DELETE("/:id", middleware.AuthMiddleware(), Delete())
	r.POST("/:id/products", middleware.AuthMiddleware(), AddProducts())
	r.DELETE("/:id/products", middleware.AuthMiddleware(), RemoveProducts())
}
```

- [ ] **Step 2: Create `create.go`**

```go
// marketplace-backend/business/album/create.go
package album

import (
	"api/business/document"
	"api/internal/plog"
	"api/internal/response"
	"api/schema/albumcol"
	"api/schema/usercol"
	"net/http"
	"strings"
	"time"

	"github.com/gin-gonic/gin"
	"go.mongodb.org/mongo-driver/bson/primitive"
)

type CreateRequest struct {
	Name        string `form:"name" binding:"required"`
	Category    string `form:"category" binding:"required"`
	Description string `form:"description"`
	IsPublic    bool   `form:"is_public"`
}

func Create() gin.HandlerFunc {
	logger := plog.NewBizLogger("[album][create]")

	return func(c *gin.Context) {
		ctx := c.Request.Context()

		userVal, exist := c.Get("current_user")
		if !exist {
			code := response.ErrorResponse("Unauthorized")
			c.JSON(code.Code, code)
			c.Abort()
			return
		}
		u := userVal.(*usercol.User)
		userOID := u.ID.(primitive.ObjectID)

		var req CreateRequest
		if err := c.ShouldBind(&req); err != nil {
			code := response.ErrorResponse(err.Error())
			c.JSON(code.Code, code)
			c.Abort()
			return
		}

		req.Name = strings.TrimSpace(req.Name)
		req.Category = strings.ToLower(strings.TrimSpace(req.Category))

		var coverURL string
		fileHeader, err := c.FormFile("cover")
		if err == nil {
			obj, uploadErr := document.ProcessAndUploadFile(fileHeader, u.GetIDString(), true)
			if uploadErr != nil {
				logger.Err(uploadErr).Msg("cover upload failed")
				code := response.ErrorResponse("COVER_UPLOAD_FAILED")
				c.JSON(code.Code, code)
				c.Abort()
				return
			}
			coverURL = document.GetCDNURL(obj.Location, true)
		}

		album := &albumcol.Album{
			UserID:      userOID,
			Name:        req.Name,
			Category:    req.Category,
			Description: req.Description,
			CoverURL:    coverURL,
			IsPublic:    req.IsPublic,
			ProductIDs:  []primitive.ObjectID{},
			IsDeleted:   false,
			CreatedAt:   time.Now(),
			UpdatedAt:   time.Now(),
		}

		id, err := albumcol.Create(ctx, album)
		if err != nil {
			logger.Err(err).Msg("failed to create album")
			code := response.ErrorResponse("FAILED_TO_CREATE_ALBUM")
			c.JSON(code.Code, code)
			c.Abort()
			return
		}

		album.DefaultModel.IDField.ID = id.(primitive.ObjectID)
		c.JSON(http.StatusOK, response.SuccessResponse(album))
	}
}
```

- [ ] **Step 3: Verify it compiles**

```bash
cd /Users/macos/workspace-user/marketplace-backend
go build ./business/album/...
```

Expected: no output (success).

- [ ] **Step 4: Commit**

```bash
cd /Users/macos/workspace-user/marketplace-backend
git add business/album/router.go business/album/create.go
git commit -m "$(cat <<'EOF'
feat(marketplace-backend): add album router and create handler

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: Backend — ListMy + GetDetail Handlers

**Files:**
- Create: `marketplace-backend/business/album/list_my.go`
- Create: `marketplace-backend/business/album/get_detail.go`

- [ ] **Step 1: Create `list_my.go`**

```go
// marketplace-backend/business/album/list_my.go
package album

import (
	"api/internal/response"
	"api/schema/albumcol"
	"api/schema/usercol"
	"net/http"
	"strconv"

	"github.com/gin-gonic/gin"
	"go.mongodb.org/mongo-driver/bson/primitive"
)

type AlbumSummary struct {
	ID           string `json:"id"`
	Name         string `json:"name"`
	Description  string `json:"description"`
	Category     string `json:"category"`
	CoverURL     string `json:"cover_url"`
	IsPublic     bool   `json:"is_public"`
	ProductCount int    `json:"product_count"`
	CreatedAt    string `json:"created_at"`
}

type ListMyResponse struct {
	Data  []AlbumSummary `json:"data"`
	Total int64          `json:"total"`
	Page  int            `json:"page"`
	Size  int            `json:"size"`
}

func ListMy() gin.HandlerFunc {
	return func(c *gin.Context) {
		ctx := c.Request.Context()

		userVal, exist := c.Get("current_user")
		if !exist {
			code := response.ErrorResponse("Unauthorized")
			c.JSON(code.Code, code)
			c.Abort()
			return
		}
		u := userVal.(*usercol.User)

		page, _ := strconv.Atoi(c.DefaultQuery("page", "1"))
		size, _ := strconv.Atoi(c.DefaultQuery("size", "12"))
		if page < 1 {
			page = 1
		}
		if size < 1 || size > 100 {
			size = 12
		}

		albums, total, err := albumcol.FindByUserID(ctx, u.GetIDString(), page, size)
		if err != nil {
			code := response.ErrorResponse(err.Error())
			c.JSON(code.Code, code)
			c.Abort()
			return
		}

		summaries := make([]AlbumSummary, 0, len(albums))
		for _, a := range albums {
			summaries = append(summaries, AlbumSummary{
				ID:           a.GetIDString(),
				Name:         a.Name,
				Description:  a.Description,
				Category:     a.Category,
				CoverURL:     a.CoverURL,
				IsPublic:     a.IsPublic,
				ProductCount: len(a.ProductIDs),
				CreatedAt:    a.CreatedAt.Format("2006-01-02T15:04:05Z07:00"),
			})
		}

		c.JSON(http.StatusOK, response.SuccessResponse(ListMyResponse{
			Data:  summaries,
			Total: total,
			Page:  page,
			Size:  size,
		}))
	}
}
```

- [ ] **Step 2: Create `get_detail.go`**

```go
// marketplace-backend/business/album/get_detail.go
package album

import (
	"api/internal/response"
	"api/middleware"
	"api/schema/albumcol"
	"api/schema/originalproductcol"
	"api/schema/usercol"
	"net/http"

	"github.com/gin-gonic/gin"
	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/bson/primitive"
	"go.mongodb.org/mongo-driver/mongo"
)

type ProductInAlbum struct {
	ID              string      `json:"id"`
	Title           string      `json:"title"`
	ThumbnailURL    string      `json:"thumbnail_url"`
	Category        interface{} `json:"category"`
	CopyrightStatus bool        `json:"copyright_status"`
}

type AlbumDetail struct {
	ID          string           `json:"id"`
	Name        string           `json:"name"`
	Description string           `json:"description"`
	Category    string           `json:"category"`
	CoverURL    string           `json:"cover_url"`
	IsPublic    bool             `json:"is_public"`
	UserID      string           `json:"user_id"`
	UserName    string           `json:"user_name"`
	UserAvatar  string           `json:"user_avatar"`
	Products    []ProductInAlbum `json:"products"`
	CreatedAt   string           `json:"created_at"`
}

func GetDetail() gin.HandlerFunc {
	return func(c *gin.Context) {
		ctx := c.Request.Context()
		id := c.Param("id")

		// Resolve caller (optional auth)
		caller, _ := middleware.CurrentUser(c)

		// Fetch album (not deleted)
		album, err := albumcol.FindByID(ctx, id)
		if err != nil {
			if err == mongo.ErrNoDocuments {
				code := response.ErrorResponse("ALBUM_NOT_FOUND")
				c.JSON(http.StatusNotFound, code)
				return
			}
			code := response.ErrorResponse(err.Error())
			c.JSON(code.Code, code)
			return
		}

		// Access control: private album only visible to owner
		isOwner := caller != nil && caller.ID.(primitive.ObjectID) == album.UserID
		if !album.IsPublic && !isOwner {
			code := response.ErrorResponse("ALBUM_IS_PRIVATE")
			c.JSON(http.StatusForbidden, code)
			return
		}

		// Fetch owner info
		owner, _ := usercol.FindWithID(ctx, album.UserID.Hex())
		userName := ""
		userAvatar := ""
		if owner != nil {
			userName = owner.FullName
			userAvatar = owner.Avatar
		}

		// Enrich products
		products := make([]ProductInAlbum, 0)
		if len(album.ProductIDs) > 0 {
			var originals []*originalproductcol.Product
			filter := bson.M{
				"_id":        bson.M{"$in": album.ProductIDs},
				"is_deleted": false,
			}
			_ = originalproductcol.Collection().SimpleFindWithCtx(ctx, &originals, filter)
			for _, p := range originals {
				products = append(products, ProductInAlbum{
					ID:              p.GetIDString(),
					Title:           p.Title,
					ThumbnailURL:    p.ThumbnailURL,
					Category:        p.Category,
					CopyrightStatus: p.IsCopyrightRegistration,
				})
			}
		}

		detail := AlbumDetail{
			ID:          album.GetIDString(),
			Name:        album.Name,
			Description: album.Description,
			Category:    album.Category,
			CoverURL:    album.CoverURL,
			IsPublic:    album.IsPublic,
			UserID:      album.UserID.Hex(),
			UserName:    userName,
			UserAvatar:  userAvatar,
			Products:    products,
			CreatedAt:   album.CreatedAt.Format("2006-01-02T15:04:05Z07:00"),
		}

		c.JSON(http.StatusOK, response.SuccessResponse(detail))
	}
}
```

- [ ] **Step 3: Verify it compiles**

```bash
cd /Users/macos/workspace-user/marketplace-backend
go build ./business/album/...
```

Expected: no output (success). If `originalproductcol.Collection()` is not exported, check the file and use the correct exported function or query helper.

- [ ] **Step 4: Commit**

```bash
cd /Users/macos/workspace-user/marketplace-backend
git add business/album/list_my.go business/album/get_detail.go
git commit -m "$(cat <<'EOF'
feat(marketplace-backend): add album list_my and get_detail handlers

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: Backend — Update + Delete Handlers

**Files:**
- Create: `marketplace-backend/business/album/update.go`
- Create: `marketplace-backend/business/album/delete.go`

- [ ] **Step 1: Create `update.go`**

```go
// marketplace-backend/business/album/update.go
package album

import (
	"api/business/document"
	"api/internal/plog"
	"api/internal/response"
	"api/schema/albumcol"
	"api/schema/usercol"
	"net/http"
	"strings"
	"time"

	"github.com/gin-gonic/gin"
	"go.mongodb.org/mongo-driver/bson/primitive"
	"go.mongodb.org/mongo-driver/mongo"
)

type UpdateRequest struct {
	Name        string `form:"name"`
	Description string `form:"description"`
	IsPublic    *bool  `form:"is_public"`
}

func Update() gin.HandlerFunc {
	logger := plog.NewBizLogger("[album][update]")

	return func(c *gin.Context) {
		ctx := c.Request.Context()
		id := c.Param("id")

		userVal, exist := c.Get("current_user")
		if !exist {
			code := response.ErrorResponse("Unauthorized")
			c.JSON(code.Code, code)
			c.Abort()
			return
		}
		u := userVal.(*usercol.User)
		userOID := u.ID.(primitive.ObjectID)

		album, err := albumcol.FindByID(ctx, id)
		if err != nil {
			if err == mongo.ErrNoDocuments {
				code := response.ErrorResponse("ALBUM_NOT_FOUND")
				c.JSON(http.StatusNotFound, code)
				return
			}
			code := response.ErrorResponse(err.Error())
			c.JSON(code.Code, code)
			return
		}

		if album.UserID != userOID {
			code := response.ErrorResponse("FORBIDDEN")
			c.JSON(http.StatusForbidden, code)
			return
		}

		var req UpdateRequest
		if err := c.ShouldBind(&req); err != nil {
			code := response.ErrorResponse(err.Error())
			c.JSON(code.Code, code)
			c.Abort()
			return
		}

		if strings.TrimSpace(req.Name) != "" {
			album.Name = strings.TrimSpace(req.Name)
		}
		album.Description = req.Description
		if req.IsPublic != nil {
			album.IsPublic = *req.IsPublic
		}

		fileHeader, err := c.FormFile("cover")
		if err == nil {
			obj, uploadErr := document.ProcessAndUploadFile(fileHeader, u.GetIDString(), true)
			if uploadErr != nil {
				logger.Err(uploadErr).Msg("cover upload failed")
				code := response.ErrorResponse("COVER_UPLOAD_FAILED")
				c.JSON(code.Code, code)
				c.Abort()
				return
			}
			album.CoverURL = document.GetCDNURL(obj.Location, true)
		}

		album.UpdatedAt = time.Now()
		if err := albumcol.Update(ctx, album); err != nil {
			logger.Err(err).Msg("failed to update album")
			code := response.ErrorResponse("FAILED_TO_UPDATE_ALBUM")
			c.JSON(code.Code, code)
			c.Abort()
			return
		}

		c.JSON(http.StatusOK, response.SuccessResponse(album))
	}
}
```

- [ ] **Step 2: Create `delete.go`**

```go
// marketplace-backend/business/album/delete.go
package album

import (
	"api/internal/response"
	"api/schema/albumcol"
	"api/schema/usercol"
	"net/http"

	"github.com/gin-gonic/gin"
	"go.mongodb.org/mongo-driver/bson/primitive"
	"go.mongodb.org/mongo-driver/mongo"
)

func Delete() gin.HandlerFunc {
	return func(c *gin.Context) {
		ctx := c.Request.Context()
		id := c.Param("id")

		userVal, exist := c.Get("current_user")
		if !exist {
			code := response.ErrorResponse("Unauthorized")
			c.JSON(code.Code, code)
			c.Abort()
			return
		}
		u := userVal.(*usercol.User)
		userOID := u.ID.(primitive.ObjectID)

		// Verify ownership before deleting
		album, err := albumcol.FindByID(ctx, id)
		if err != nil {
			if err == mongo.ErrNoDocuments {
				code := response.ErrorResponse("ALBUM_NOT_FOUND")
				c.JSON(http.StatusNotFound, code)
				return
			}
			code := response.ErrorResponse(err.Error())
			c.JSON(code.Code, code)
			return
		}

		if album.UserID != userOID {
			code := response.ErrorResponse("FORBIDDEN")
			c.JSON(http.StatusForbidden, code)
			return
		}

		if err := albumcol.SoftDelete(ctx, id, userOID); err != nil {
			code := response.ErrorResponse("FAILED_TO_DELETE_ALBUM")
			c.JSON(code.Code, code)
			c.Abort()
			return
		}

		c.JSON(http.StatusOK, response.SuccessResponse(gin.H{"message": "Album deleted"}))
	}
}
```

- [ ] **Step 3: Verify it compiles**

```bash
cd /Users/macos/workspace-user/marketplace-backend
go build ./business/album/...
```

Expected: no output (success).

- [ ] **Step 4: Commit**

```bash
cd /Users/macos/workspace-user/marketplace-backend
git add business/album/update.go business/album/delete.go
git commit -m "$(cat <<'EOF'
feat(marketplace-backend): add album update and delete handlers

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: Backend — AddProducts + RemoveProducts Handlers

**Files:**
- Create: `marketplace-backend/business/album/add_products.go`
- Create: `marketplace-backend/business/album/remove_products.go`

- [ ] **Step 1: Create `add_products.go`**

```go
// marketplace-backend/business/album/add_products.go
package album

import (
	"api/internal/response"
	"api/schema/albumcol"
	"api/schema/originalproductcol"
	"api/schema/usercol"
	"net/http"

	"github.com/gin-gonic/gin"
	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/bson/primitive"
	"go.mongodb.org/mongo-driver/mongo"
)

type AddProductsRequest struct {
	ProductIDs []string `json:"product_ids" binding:"required,min=1"`
}

func AddProducts() gin.HandlerFunc {
	return func(c *gin.Context) {
		ctx := c.Request.Context()
		id := c.Param("id")

		userVal, exist := c.Get("current_user")
		if !exist {
			code := response.ErrorResponse("Unauthorized")
			c.JSON(code.Code, code)
			c.Abort()
			return
		}
		u := userVal.(*usercol.User)
		userOID := u.ID.(primitive.ObjectID)

		album, err := albumcol.FindByID(ctx, id)
		if err != nil {
			if err == mongo.ErrNoDocuments {
				code := response.ErrorResponse("ALBUM_NOT_FOUND")
				c.JSON(http.StatusNotFound, code)
				return
			}
			code := response.ErrorResponse(err.Error())
			c.JSON(code.Code, code)
			return
		}

		if album.UserID != userOID {
			code := response.ErrorResponse("FORBIDDEN")
			c.JSON(http.StatusForbidden, code)
			return
		}

		var req AddProductsRequest
		if err := c.ShouldBindJSON(&req); err != nil {
			code := response.ErrorResponse(err.Error())
			c.JSON(code.Code, code)
			c.Abort()
			return
		}

		// Convert to ObjectIDs, collect invalid hex strings
		productOIDs := make([]primitive.ObjectID, 0, len(req.ProductIDs))
		invalidIDs := make([]string, 0)
		for _, pid := range req.ProductIDs {
			oid, err := primitive.ObjectIDFromHex(pid)
			if err != nil {
				invalidIDs = append(invalidIDs, pid)
				continue
			}
			productOIDs = append(productOIDs, oid)
		}
		if len(invalidIDs) > 0 {
			c.JSON(http.StatusBadRequest, response.ErrorResponse("INVALID_PRODUCT_IDS"))
			return
		}

		// Verify each product is owned by the caller
		filter := bson.M{
			"_id": bson.M{"$in": productOIDs},
			"$or": bson.A{
				bson.M{"participants.id": userOID},
				bson.M{"creator_id": userOID},
			},
			"is_deleted": false,
		}
		var owned []*originalproductcol.Product
		_ = originalproductcol.Collection().SimpleFindWithCtx(ctx, &owned, filter)

		if len(owned) != len(productOIDs) {
			// Build set of owned IDs to find the invalid ones
			ownedSet := make(map[string]bool)
			for _, p := range owned {
				ownedSet[p.GetIDString()] = true
			}
			for _, oid := range productOIDs {
				if !ownedSet[oid.Hex()] {
					invalidIDs = append(invalidIDs, oid.Hex())
				}
			}
			c.JSON(http.StatusBadRequest, gin.H{
				"success":     false,
				"message":     "PRODUCTS_NOT_OWNED",
				"invalid_ids": invalidIDs,
			})
			return
		}

		if err := albumcol.AddProducts(ctx, id, userOID, productOIDs); err != nil {
			code := response.ErrorResponse("FAILED_TO_ADD_PRODUCTS")
			c.JSON(code.Code, code)
			c.Abort()
			return
		}

		c.JSON(http.StatusOK, response.SuccessResponse(gin.H{"message": "Products added"}))
	}
}
```

- [ ] **Step 2: Create `remove_products.go`**

```go
// marketplace-backend/business/album/remove_products.go
package album

import (
	"api/internal/response"
	"api/schema/albumcol"
	"api/schema/usercol"
	"net/http"

	"github.com/gin-gonic/gin"
	"go.mongodb.org/mongo-driver/bson/primitive"
	"go.mongodb.org/mongo-driver/mongo"
)

type RemoveProductsRequest struct {
	ProductIDs []string `json:"product_ids" binding:"required,min=1"`
}

func RemoveProducts() gin.HandlerFunc {
	return func(c *gin.Context) {
		ctx := c.Request.Context()
		id := c.Param("id")

		userVal, exist := c.Get("current_user")
		if !exist {
			code := response.ErrorResponse("Unauthorized")
			c.JSON(code.Code, code)
			c.Abort()
			return
		}
		u := userVal.(*usercol.User)
		userOID := u.ID.(primitive.ObjectID)

		album, err := albumcol.FindByID(ctx, id)
		if err != nil {
			if err == mongo.ErrNoDocuments {
				code := response.ErrorResponse("ALBUM_NOT_FOUND")
				c.JSON(http.StatusNotFound, code)
				return
			}
			code := response.ErrorResponse(err.Error())
			c.JSON(code.Code, code)
			return
		}

		if album.UserID != userOID {
			code := response.ErrorResponse("FORBIDDEN")
			c.JSON(http.StatusForbidden, code)
			return
		}

		var req RemoveProductsRequest
		if err := c.ShouldBindJSON(&req); err != nil {
			code := response.ErrorResponse(err.Error())
			c.JSON(code.Code, code)
			c.Abort()
			return
		}

		productOIDs := make([]primitive.ObjectID, 0, len(req.ProductIDs))
		for _, pid := range req.ProductIDs {
			oid, err := primitive.ObjectIDFromHex(pid)
			if err != nil {
				code := response.ErrorResponse("INVALID_PRODUCT_ID: " + pid)
				c.JSON(http.StatusBadRequest, code)
				return
			}
			productOIDs = append(productOIDs, oid)
		}

		if err := albumcol.RemoveProducts(ctx, id, userOID, productOIDs); err != nil {
			code := response.ErrorResponse("FAILED_TO_REMOVE_PRODUCTS")
			c.JSON(code.Code, code)
			c.Abort()
			return
		}

		c.JSON(http.StatusOK, response.SuccessResponse(gin.H{"message": "Products removed"}))
	}
}
```

- [ ] **Step 3: Verify it compiles**

```bash
cd /Users/macos/workspace-user/marketplace-backend
go build ./business/album/...
```

Expected: no output (success).

- [ ] **Step 4: Commit**

```bash
cd /Users/macos/workspace-user/marketplace-backend
git add business/album/add_products.go business/album/remove_products.go
git commit -m "$(cat <<'EOF'
feat(marketplace-backend): add album add_products and remove_products handlers

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 6: Backend — Register Router

**Files:**
- Modify: `marketplace-backend/routers/ver1.go`

- [ ] **Step 1: Add import and router registration**

In `routers/ver1.go`, add the `album` import alongside existing imports:

```go
import (
    // ... existing imports ...
    "api/business/album"
)
```

Then add after the `valuationRouter` block (near line 93):

```go
// Album routes
albumRouter := r.Group("albums")
album.AddRouter(albumRouter)
```

- [ ] **Step 2: Verify the full backend builds**

```bash
cd /Users/macos/workspace-user/marketplace-backend
go build ./...
```

Expected: no output (success).

- [ ] **Step 3: Commit**

```bash
cd /Users/macos/workspace-user/marketplace-backend
git add routers/ver1.go
git commit -m "$(cat <<'EOF'
feat(marketplace-backend): register album router in ver1.go

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 7: Frontend — Album API Module

**Files:**
- Create: `marketplace-frontend/app/api/album.ts`

- [ ] **Step 1: Create `album.ts`**

```ts
// marketplace-frontend/app/api/album.ts
import { getInstance } from "./instance";

const url = "/albums";

export const albumApi = {
  create(data: FormData) {
    return getInstance().post(url, data, {
      headers: { "Content-Type": "multipart/form-data" },
    });
  },

  getMyAlbums(params?: { page?: number; size?: number }) {
    return getInstance().get(`${url}/my`, { params });
  },

  getById(id: string) {
    return getInstance().get(`${url}/${id}`);
  },

  update(id: string, data: FormData) {
    return getInstance().put(`${url}/${id}`, data, {
      headers: { "Content-Type": "multipart/form-data" },
    });
  },

  remove(id: string) {
    return getInstance().delete(`${url}/${id}`);
  },

  addProducts(id: string, productIds: string[]) {
    return getInstance().post(`${url}/${id}/products`, {
      product_ids: productIds,
    });
  },

  removeProducts(id: string, productIds: string[]) {
    return getInstance().delete(`${url}/${id}/products`, {
      data: { product_ids: productIds },
    });
  },
};
```

- [ ] **Step 2: Verify TypeScript compiles**

```bash
cd /Users/macos/workspace-user/marketplace-frontend
npx tsc --noEmit 2>&1 | grep "app/api/album"
```

Expected: no output (no errors in `album.ts`).

- [ ] **Step 3: Commit**

```bash
cd /Users/macos/workspace-user/marketplace-frontend
git add app/api/album.ts
git commit -m "$(cat <<'EOF'
feat(marketplace-frontend): add albumApi module

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 8: Frontend — Navigation Wiring

**Files:**
- Modify: `marketplace-frontend/config/NavigationPath.ts`
- Modify: `marketplace-frontend/common/components/Header/data.tsx`
- Modify: `marketplace-frontend/common/components/ProfileLayout/SettingsLayout.tsx`

- [ ] **Step 1: Add `MY_ALBUMS` to `NavigationPath.ts`**

In `config/NavigationPath.ts`, add after the `AFFILIATE` line:

```ts
MY_ALBUMS: "/profile/my-albums",
```

- [ ] **Step 2: Add `my-albums` item to `data.tsx`**

In `common/components/Header/data.tsx`, in `getOtherMenuItems`, add after the `my-works` entry:

```ts
{
  key: "my-albums",
  href: NavigationPath.MY_ALBUMS,
  label: "Album của tôi",
  iconDefault: "/icons/menu/icon-pencil.svg",
},
```

- [ ] **Step 3: Add "Album của tôi" to `SettingsLayout.tsx` menu**

In `common/components/ProfileLayout/SettingsLayout.tsx`, the `menuItems` array currently starts with `settings`. Add the `my-albums` item before `settings`:

```ts
{
  key: "my-albums",
  label: "Album của tôi",
  path: NavigationPath.MY_ALBUMS,
},
```

Also add the import at the top if `NavigationPath` is not already imported (it already is — the file has `import { NavigationPath } from "@/config/NavigationPath"`).

- [ ] **Step 4: Verify TypeScript compiles**

```bash
cd /Users/macos/workspace-user/marketplace-frontend
npx tsc --noEmit 2>&1 | grep -E "(NavigationPath|data\.tsx|SettingsLayout)"
```

Expected: no output (no new errors).

- [ ] **Step 5: Commit**

```bash
cd /Users/macos/workspace-user/marketplace-frontend
git add config/NavigationPath.ts \
        common/components/Header/data.tsx \
        common/components/ProfileLayout/SettingsLayout.tsx
git commit -m "$(cat <<'EOF'
feat(marketplace-frontend): add MY_ALBUMS navigation path and menu entries

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 9: Frontend — My Albums List Page

**Files:**
- Create: `marketplace-frontend/app/[locale]/(profile)/profile/my-albums/page.tsx`
- Create: `marketplace-frontend/app/[locale]/(profile)/profile/my-albums/components/AlbumCard.tsx`
- Create: `marketplace-frontend/app/[locale]/(profile)/profile/my-albums/components/CreateAlbumModal.tsx`
- Create: `marketplace-frontend/app/[locale]/(profile)/profile/my-albums/components/EditAlbumModal.tsx`

- [ ] **Step 1: Create `AlbumCard.tsx`**

```tsx
// app/[locale]/(profile)/profile/my-albums/components/AlbumCard.tsx
"use client";
import React from "react";
import { Popconfirm, Tag } from "antd";
import { LockOutlined, GlobalOutlined, EditOutlined, DeleteOutlined } from "@ant-design/icons";
import { useRouter } from "next/navigation";

interface AlbumSummary {
  id: string;
  name: string;
  description: string;
  category: string;
  cover_url: string;
  is_public: boolean;
  product_count: number;
  created_at: string;
}

interface AlbumCardProps {
  album: AlbumSummary;
  onEdit: (album: AlbumSummary) => void;
  onDelete: (id: string) => void;
}

export default function AlbumCard({ album, onEdit, onDelete }: AlbumCardProps) {
  const router = useRouter();

  const categoryLabels: Record<string, string> = {
    music: "Âm nhạc",
    literature: "Văn học",
    film: "Phim",
    art: "Mỹ thuật",
    computer: "Phần mềm",
    photography: "Nhiếp ảnh",
    stage: "Sân khấu",
    architecture: "Kiến trúc",
    journalism: "Báo chí",
    coursebook: "Giáo trình",
  };

  return (
    <div className="rounded-xl overflow-hidden border border-gray-200 dark:border-gray-700 hover:shadow-md transition-shadow bg-white dark:bg-gray-800">
      {/* Cover */}
      <div
        className="relative aspect-video cursor-pointer bg-gray-100 dark:bg-gray-700"
        onClick={() => router.push(`/albums/${album.id}`)}
      >
        {album.cover_url ? (
          <img
            src={album.cover_url}
            alt={album.name}
            className="w-full h-full object-cover"
          />
        ) : (
          <div className="w-full h-full flex items-center justify-center text-gray-400 text-4xl">🎵</div>
        )}
        <div className="absolute top-2 left-2">
          {album.is_public ? (
            <Tag icon={<GlobalOutlined />} color="green">Công khai</Tag>
          ) : (
            <Tag icon={<LockOutlined />} color="default">Riêng tư</Tag>
          )}
        </div>
      </div>

      {/* Info */}
      <div className="p-3">
        <div
          className="font-semibold text-sm truncate cursor-pointer hover:text-blue-600 mb-1"
          onClick={() => router.push(`/albums/${album.id}`)}
        >
          {album.name}
        </div>
        <div className="text-xs text-gray-500 mb-3">
          {categoryLabels[album.category] ?? album.category} · {album.product_count} tác phẩm
        </div>
        <div className="flex gap-2">
          <button
            className="flex-1 flex items-center justify-center gap-1 py-1.5 text-xs border border-gray-200 dark:border-gray-600 rounded-lg hover:bg-gray-50 dark:hover:bg-gray-700 transition-colors"
            onClick={() => onEdit(album)}
          >
            <EditOutlined /> Sửa
          </button>
          <Popconfirm
            title="Xoá album này?"
            description="Album sẽ bị xoá và không thể khôi phục."
            okText="Xoá"
            cancelText="Huỷ"
            okButtonProps={{ danger: true }}
            onConfirm={() => onDelete(album.id)}
          >
            <button className="flex-1 flex items-center justify-center gap-1 py-1.5 text-xs border border-red-200 text-red-500 rounded-lg hover:bg-red-50 transition-colors">
              <DeleteOutlined /> Xoá
            </button>
          </Popconfirm>
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Create `CreateAlbumModal.tsx`**

```tsx
// app/[locale]/(profile)/profile/my-albums/components/CreateAlbumModal.tsx
"use client";
import React, { useState, useEffect } from "react";
import { Modal, Form, Input, Select, Upload, message, Checkbox, Empty } from "antd";
import { InboxOutlined } from "@ant-design/icons";
import { albumApi } from "@/app/api/album";
import { getInstance } from "@/app/api/instance";

const { TextArea } = Input;
const { Dragger } = Upload;

interface CreateAlbumModalProps {
  open: boolean;
  onClose: () => void;
  onCreated: () => void;
}

const CATEGORY_OPTIONS = [
  { value: "music", label: "Âm nhạc" },
  { value: "literature", label: "Văn học" },
  { value: "film", label: "Phim" },
  { value: "art", label: "Mỹ thuật" },
  { value: "computer", label: "Phần mềm" },
  { value: "photography", label: "Nhiếp ảnh" },
  { value: "stage", label: "Sân khấu" },
  { value: "architecture", label: "Kiến trúc" },
  { value: "journalism", label: "Báo chí" },
  { value: "coursebook", label: "Giáo trình" },
];

export default function CreateAlbumModal({ open, onClose, onCreated }: CreateAlbumModalProps) {
  const [form] = Form.useForm();
  const [loading, setLoading] = useState(false);
  const [coverFile, setCoverFile] = useState<File | null>(null);
  const [coverPreview, setCoverPreview] = useState<string | null>(null);
  const [selectedCategory, setSelectedCategory] = useState<string | null>(null);
  const [products, setProducts] = useState<any[]>([]);
  const [productsLoading, setProductsLoading] = useState(false);
  const [selectedProductIds, setSelectedProductIds] = useState<string[]>([]);
  const [productSearch, setProductSearch] = useState("");

  useEffect(() => {
    if (!selectedCategory) {
      setProducts([]);
      return;
    }
    setProductsLoading(true);
    getInstance()
      .get("/products/original/my", { params: { category: selectedCategory, size: 100 } })
      .then((res) => {
        if (res?.data?.success) {
          setProducts(res.data.data?.data ?? []);
        }
      })
      .catch(() => {})
      .finally(() => setProductsLoading(false));
  }, [selectedCategory]);

  const handleSubmit = async () => {
    try {
      await form.validateFields();
    } catch {
      return;
    }

    if (!coverFile) {
      message.error("Vui lòng chọn ảnh bìa");
      return;
    }

    const values = form.getFieldsValue();
    const formData = new FormData();
    formData.append("name", values.name);
    formData.append("category", values.category);
    formData.append("description", values.description ?? "");
    formData.append("is_public", "false");
    formData.append("cover", coverFile);

    setLoading(true);
    try {
      const res = await albumApi.create(formData);
      if (res?.data?.success) {
        const albumId = res.data.data?.id ?? res.data.data?._id;
        // Add selected products if any
        if (selectedProductIds.length > 0 && albumId) {
          await albumApi.addProducts(albumId, selectedProductIds);
        }
        message.success("Tạo album thành công");
        onCreated();
        handleClose();
      }
    } catch (err: any) {
      message.error(err?.response?.data?.message ?? "Tạo album thất bại");
    } finally {
      setLoading(false);
    }
  };

  const handleClose = () => {
    form.resetFields();
    setCoverFile(null);
    setCoverPreview(null);
    setSelectedCategory(null);
    setSelectedProductIds([]);
    setProductSearch("");
    onClose();
  };

  const filteredProducts = products.filter((p) =>
    p.title?.toLowerCase().includes(productSearch.toLowerCase())
  );

  return (
    <Modal
      title="Tạo Album mới"
      open={open}
      onCancel={handleClose}
      onOk={handleSubmit}
      okText="Tạo Album"
      cancelText="Huỷ"
      confirmLoading={loading}
      width={600}
      styles={{ body: { maxHeight: "70vh", overflowY: "auto" } }}
    >
      <Form form={form} layout="vertical">
        <Form.Item
          name="name"
          label="Tên album"
          rules={[{ required: true, message: "Vui lòng nhập tên album" }, { max: 100 }]}
        >
          <Input placeholder="Nhập tên album" maxLength={100} showCount />
        </Form.Item>

        <Form.Item
          name="category"
          label="Thể loại"
          rules={[{ required: true, message: "Vui lòng chọn thể loại" }]}
        >
          <Select
            options={CATEGORY_OPTIONS}
            placeholder="Chọn thể loại"
            onChange={(val) => {
              setSelectedCategory(val);
              setSelectedProductIds([]);
            }}
          />
        </Form.Item>

        <Form.Item name="description" label="Mô tả">
          <TextArea placeholder="Mô tả album (tuỳ chọn)" maxLength={500} showCount rows={3} />
        </Form.Item>

        <Form.Item label="Ảnh bìa" required>
          {coverPreview ? (
            <div className="relative">
              <img src={coverPreview} alt="cover" className="w-full h-48 object-cover rounded-lg" />
              <button
                className="absolute top-2 right-2 bg-black/50 text-white rounded px-2 py-1 text-xs"
                onClick={() => { setCoverFile(null); setCoverPreview(null); }}
              >
                Đổi ảnh
              </button>
            </div>
          ) : (
            <Dragger
              accept="image/*"
              showUploadList={false}
              beforeUpload={(file) => {
                setCoverFile(file);
                setCoverPreview(URL.createObjectURL(file));
                return false;
              }}
            >
              <p className="ant-upload-drag-icon"><InboxOutlined /></p>
              <p className="ant-upload-text">Kéo thả hoặc click để chọn ảnh bìa</p>
              <p className="ant-upload-hint">Hỗ trợ JPG, PNG, WEBP</p>
            </Dragger>
          )}
        </Form.Item>

        {selectedCategory && (
          <Form.Item label="Chọn tác phẩm (tuỳ chọn)">
            <Input
              placeholder="Tìm tác phẩm..."
              value={productSearch}
              onChange={(e) => setProductSearch(e.target.value)}
              className="mb-2"
            />
            {productsLoading ? (
              <div className="text-center py-4 text-gray-400">Đang tải...</div>
            ) : filteredProducts.length === 0 ? (
              <Empty description="Bạn chưa có tác phẩm nào trong danh mục này" />
            ) : (
              <div className="max-h-48 overflow-y-auto border rounded-lg p-2 space-y-1">
                {filteredProducts.map((p) => (
                  <div
                    key={p.id}
                    className="flex items-center gap-2 p-2 hover:bg-gray-50 rounded cursor-pointer"
                    onClick={() => {
                      setSelectedProductIds((prev) =>
                        prev.includes(p.id) ? prev.filter((x) => x !== p.id) : [...prev, p.id]
                      );
                    }}
                  >
                    <Checkbox checked={selectedProductIds.includes(p.id)} />
                    {p.thumbnail_url && (
                      <img src={p.thumbnail_url} alt={p.title} className="w-10 h-10 object-cover rounded" />
                    )}
                    <span className="text-sm truncate flex-1">{p.title}</span>
                  </div>
                ))}
              </div>
            )}
          </Form.Item>
        )}
      </Form>
    </Modal>
  );
}
```

- [ ] **Step 3: Create `EditAlbumModal.tsx`**

```tsx
// app/[locale]/(profile)/profile/my-albums/components/EditAlbumModal.tsx
"use client";
import React, { useState, useEffect } from "react";
import { Modal, Form, Input, Select, Upload, message, Switch } from "antd";
import { InboxOutlined } from "@ant-design/icons";
import { albumApi } from "@/app/api/album";

const { TextArea } = Input;
const { Dragger } = Upload;

const CATEGORY_OPTIONS = [
  { value: "music", label: "Âm nhạc" },
  { value: "literature", label: "Văn học" },
  { value: "film", label: "Phim" },
  { value: "art", label: "Mỹ thuật" },
  { value: "computer", label: "Phần mềm" },
  { value: "photography", label: "Nhiếp ảnh" },
  { value: "stage", label: "Sân khấu" },
  { value: "architecture", label: "Kiến trúc" },
  { value: "journalism", label: "Báo chí" },
  { value: "coursebook", label: "Giáo trình" },
];

interface Album {
  id: string;
  name: string;
  description: string;
  category: string;
  cover_url: string;
  is_public: boolean;
  product_count: number;
  created_at: string;
}

interface EditAlbumModalProps {
  open: boolean;
  album: Album | null;
  onClose: () => void;
  onUpdated: () => void;
}

export default function EditAlbumModal({ open, album, onClose, onUpdated }: EditAlbumModalProps) {
  const [form] = Form.useForm();
  const [loading, setLoading] = useState(false);
  const [newCoverFile, setNewCoverFile] = useState<File | null>(null);
  const [coverPreview, setCoverPreview] = useState<string | null>(null);

  useEffect(() => {
    if (album && open) {
      form.setFieldsValue({
        name: album.name,
        category: album.category,
        description: album.description,
        is_public: album.is_public,
      });
      setCoverPreview(album.cover_url || null);
      setNewCoverFile(null);
    }
  }, [album, open, form]);

  const handleSubmit = async () => {
    try {
      await form.validateFields();
    } catch {
      return;
    }
    if (!album) return;

    const values = form.getFieldsValue();
    const formData = new FormData();
    formData.append("name", values.name);
    formData.append("description", values.description ?? "");
    formData.append("is_public", values.is_public ? "true" : "false");
    if (newCoverFile) {
      formData.append("cover", newCoverFile);
    }

    setLoading(true);
    try {
      const res = await albumApi.update(album.id, formData);
      if (res?.data?.success) {
        message.success("Cập nhật album thành công");
        onUpdated();
        onClose();
      }
    } catch (err: any) {
      message.error(err?.response?.data?.message ?? "Cập nhật thất bại");
    } finally {
      setLoading(false);
    }
  };

  return (
    <Modal
      title="Sửa thông tin Album"
      open={open}
      onCancel={onClose}
      onOk={handleSubmit}
      okText="Lưu thay đổi"
      cancelText="Huỷ"
      confirmLoading={loading}
      width={560}
    >
      <Form form={form} layout="vertical">
        <Form.Item name="name" label="Tên album" rules={[{ required: true }, { max: 100 }]}>
          <Input maxLength={100} showCount />
        </Form.Item>

        <Form.Item name="category" label="Thể loại">
          <Select options={CATEGORY_OPTIONS} disabled />
        </Form.Item>

        <Form.Item name="description" label="Mô tả">
          <TextArea maxLength={500} showCount rows={3} />
        </Form.Item>

        <Form.Item name="is_public" label="Chế độ hiển thị" valuePropName="checked">
          <Switch checkedChildren="Công khai" unCheckedChildren="Riêng tư" />
        </Form.Item>

        <Form.Item label="Ảnh bìa">
          {coverPreview && !newCoverFile ? (
            <div className="relative">
              <img src={coverPreview} alt="cover" className="w-full h-40 object-cover rounded-lg" />
              <button
                className="absolute top-2 right-2 bg-black/50 text-white rounded px-2 py-1 text-xs"
                onClick={() => { setCoverPreview(null); }}
              >
                Đổi ảnh
              </button>
            </div>
          ) : (
            <Dragger
              accept="image/*"
              showUploadList={false}
              beforeUpload={(file) => {
                setNewCoverFile(file);
                setCoverPreview(URL.createObjectURL(file));
                return false;
              }}
            >
              <p className="ant-upload-drag-icon"><InboxOutlined /></p>
              <p className="ant-upload-text">Kéo thả hoặc click để thay ảnh bìa</p>
            </Dragger>
          )}
        </Form.Item>
      </Form>
    </Modal>
  );
}
```

- [ ] **Step 4: Create `page.tsx` (My Albums list)**

```tsx
// app/[locale]/(profile)/profile/my-albums/page.tsx
"use client";
import React, { useState } from "react";
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { Button, Pagination, Empty, message } from "antd";
import { PlusOutlined } from "@ant-design/icons";
import MainLayout from "@/common/components/Layouts/MainLayout";
import SettingsLayout from "@/common/components/ProfileLayout/SettingsLayout";
import RouteProtection from "@/common/components/Auth/RouteProtection";
import AlbumCard from "./components/AlbumCard";
import CreateAlbumModal from "./components/CreateAlbumModal";
import EditAlbumModal from "./components/EditAlbumModal";
import { albumApi } from "@/app/api/album";
import { useRouter } from "next/navigation";

interface AlbumSummary {
  id: string;
  name: string;
  description: string;
  category: string;
  cover_url: string;
  is_public: boolean;
  product_count: number;
  created_at: string;
}

export default function MyAlbumsPage() {
  const router = useRouter();
  const queryClient = useQueryClient();
  const [page, setPage] = useState(1);
  const [createOpen, setCreateOpen] = useState(false);
  const [editAlbum, setEditAlbum] = useState<AlbumSummary | null>(null);

  const { data, isLoading } = useQuery({
    queryKey: ["my-albums", page],
    queryFn: () => albumApi.getMyAlbums({ page, size: 12 }).then((r) => r.data.data),
  });

  const deleteMutation = useMutation({
    mutationFn: (id: string) => albumApi.remove(id),
    onSuccess: () => {
      message.success("Đã xoá album");
      queryClient.invalidateQueries({ queryKey: ["my-albums"] });
    },
    onError: () => message.error("Xoá album thất bại"),
  });

  const albums: AlbumSummary[] = data?.data ?? [];
  const total: number = data?.total ?? 0;

  return (
    <RouteProtection>
      <MainLayout>
        <SettingsLayout>
          <div className="space-y-6">
            {/* Header */}
            <div className="flex items-center justify-between">
              <h1 className="text-2xl font-bold">Album của tôi</h1>
              <Button
                type="primary"
                icon={<PlusOutlined />}
                onClick={() => setCreateOpen(true)}
              >
                Tạo Album
              </Button>
            </div>

            {/* Grid */}
            {isLoading ? (
              <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
                {Array.from({ length: 6 }).map((_, i) => (
                  <div key={i} className="rounded-xl bg-gray-100 dark:bg-gray-800 aspect-video animate-pulse" />
                ))}
              </div>
            ) : albums.length === 0 ? (
              <div className="py-20 text-center">
                <Empty
                  description={
                    <div>
                      <p className="text-base font-medium mb-1">Bạn chưa có album nào</p>
                      <p className="text-gray-500 text-sm">Hãy tạo album đầu tiên để sắp xếp tác phẩm của bạn</p>
                    </div>
                  }
                >
                  <Button type="primary" icon={<PlusOutlined />} onClick={() => setCreateOpen(true)}>
                    Tạo album đầu tiên
                  </Button>
                </Empty>
              </div>
            ) : (
              <>
                <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
                  {albums.map((album) => (
                    <AlbumCard
                      key={album.id}
                      album={album}
                      onEdit={(a) => setEditAlbum(a)}
                      onDelete={(id) => deleteMutation.mutate(id)}
                    />
                  ))}
                </div>
                {total > 12 && (
                  <div className="flex justify-center pt-4">
                    <Pagination
                      current={page}
                      total={total}
                      pageSize={12}
                      onChange={setPage}
                      showSizeChanger={false}
                    />
                  </div>
                )}
              </>
            )}
          </div>

          <CreateAlbumModal
            open={createOpen}
            onClose={() => setCreateOpen(false)}
            onCreated={() => {
              queryClient.invalidateQueries({ queryKey: ["my-albums"] });
              setCreateOpen(false);
            }}
          />

          <EditAlbumModal
            open={!!editAlbum}
            album={editAlbum}
            onClose={() => setEditAlbum(null)}
            onUpdated={() => {
              queryClient.invalidateQueries({ queryKey: ["my-albums"] });
              setEditAlbum(null);
            }}
          />
        </SettingsLayout>
      </MainLayout>
    </RouteProtection>
  );
}
```

- [ ] **Step 5: Verify TypeScript compiles**

```bash
cd /Users/macos/workspace-user/marketplace-frontend
npx tsc --noEmit 2>&1 | grep "my-albums"
```

Expected: no output (no errors in these files).

- [ ] **Step 6: Commit**

```bash
cd /Users/macos/workspace-user/marketplace-frontend
git add \
  app/[locale]/\(profile\)/profile/my-albums/page.tsx \
  "app/[locale]/(profile)/profile/my-albums/components/AlbumCard.tsx" \
  "app/[locale]/(profile)/profile/my-albums/components/CreateAlbumModal.tsx" \
  "app/[locale]/(profile)/profile/my-albums/components/EditAlbumModal.tsx"
git commit -m "$(cat <<'EOF'
feat(marketplace-frontend): add My Albums list page with create/edit/delete

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 10: Frontend — Album Detail Page + Add Products Drawer

**Files:**
- Create: `marketplace-frontend/common/pages/Album/AlbumDetailPage.tsx`
- Create: `marketplace-frontend/common/pages/Album/AddProductsDrawer.tsx`
- Create: `marketplace-frontend/app/[locale]/(album)/albums/[id]/page.tsx`

- [ ] **Step 1: Create `AddProductsDrawer.tsx`**

```tsx
// common/pages/Album/AddProductsDrawer.tsx
"use client";
import React, { useState, useEffect } from "react";
import { Drawer, Button, Checkbox, Input, Empty, message } from "antd";
import { getInstance } from "@/app/api/instance";
import { albumApi } from "@/app/api/album";
import { useQueryClient } from "@tanstack/react-query";

interface ProductOption {
  id: string;
  title: string;
  thumbnail_url: string;
}

interface AddProductsDrawerProps {
  open: boolean;
  albumId: string;
  albumCategory: string;
  existingProductIds: string[];
  onClose: () => void;
}

export default function AddProductsDrawer({
  open,
  albumId,
  albumCategory,
  existingProductIds,
  onClose,
}: AddProductsDrawerProps) {
  const queryClient = useQueryClient();
  const [products, setProducts] = useState<ProductOption[]>([]);
  const [loading, setLoading] = useState(false);
  const [submitting, setSubmitting] = useState(false);
  const [selected, setSelected] = useState<string[]>([]);
  const [search, setSearch] = useState("");

  useEffect(() => {
    if (!open || !albumCategory) return;
    setLoading(true);
    setSelected([]);
    setSearch("");
    getInstance()
      .get("/products/original/my", { params: { category: albumCategory, size: 100 } })
      .then((res) => {
        if (res?.data?.success) {
          const all: ProductOption[] = res.data.data?.data ?? [];
          // Only show products NOT already in the album
          setProducts(all.filter((p) => !existingProductIds.includes(p.id)));
        }
      })
      .catch(() => {})
      .finally(() => setLoading(false));
  }, [open, albumCategory, existingProductIds]);

  const handleAdd = async () => {
    if (selected.length === 0) {
      message.warning("Vui lòng chọn ít nhất một tác phẩm");
      return;
    }
    setSubmitting(true);
    try {
      await albumApi.addProducts(albumId, selected);
      message.success(`Đã thêm ${selected.length} tác phẩm vào album`);
      queryClient.invalidateQueries({ queryKey: ["album-detail", albumId] });
      onClose();
    } catch (err: any) {
      message.error(err?.response?.data?.message ?? "Thêm tác phẩm thất bại");
    } finally {
      setSubmitting(false);
    }
  };

  const filtered = products.filter((p) =>
    p.title?.toLowerCase().includes(search.toLowerCase())
  );

  const toggleSelect = (id: string) => {
    setSelected((prev) => (prev.includes(id) ? prev.filter((x) => x !== id) : [...prev, id]));
  };

  return (
    <Drawer
      title="Thêm tác phẩm vào album"
      placement="right"
      width={400}
      open={open}
      onClose={onClose}
      footer={
        <div className="flex justify-end gap-2">
          <Button onClick={onClose}>Huỷ</Button>
          <Button
            type="primary"
            loading={submitting}
            disabled={selected.length === 0}
            onClick={handleAdd}
          >
            Thêm {selected.length > 0 ? `${selected.length} tác phẩm` : ""}
          </Button>
        </div>
      }
    >
      <Input
        placeholder="Tìm tác phẩm..."
        value={search}
        onChange={(e) => setSearch(e.target.value)}
        className="mb-4"
      />

      {loading ? (
        <div className="space-y-2">
          {Array.from({ length: 4 }).map((_, i) => (
            <div key={i} className="h-16 bg-gray-100 rounded animate-pulse" />
          ))}
        </div>
      ) : filtered.length === 0 ? (
        <Empty description="Không còn sản phẩm nào phù hợp với danh mục này" />
      ) : (
        <div className="space-y-1">
          {filtered.map((p) => (
            <div
              key={p.id}
              className="flex items-center gap-3 p-2 rounded hover:bg-gray-50 cursor-pointer"
              onClick={() => toggleSelect(p.id)}
            >
              <Checkbox checked={selected.includes(p.id)} />
              {p.thumbnail_url && (
                <img src={p.thumbnail_url} alt={p.title} className="w-10 h-10 object-cover rounded" />
              )}
              <span className="text-sm flex-1 truncate">{p.title}</span>
            </div>
          ))}
        </div>
      )}
    </Drawer>
  );
}
```

- [ ] **Step 2: Create `AlbumDetailPage.tsx`**

```tsx
// common/pages/Album/AlbumDetailPage.tsx
"use client";
import React, { useState } from "react";
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { Button, Dropdown, Tag, Tooltip, message, Spin } from "antd";
import type { MenuProps } from "antd";
import {
  ArrowLeftOutlined,
  EllipsisOutlined,
  LinkOutlined,
  EditOutlined,
  DeleteOutlined,
  EyeOutlined,
  EyeInvisibleOutlined,
  PlusOutlined,
  CloseOutlined,
  LockOutlined,
  GlobalOutlined,
} from "@ant-design/icons";
import { useSelector } from "react-redux";
import { useRouter } from "next/navigation";
import { albumApi } from "@/app/api/album";
import { RootState } from "@/store";
import AddProductsDrawer from "./AddProductsDrawer";
import EditAlbumModal from "../../app/[locale]/(profile)/profile/my-albums/components/EditAlbumModal";

interface AlbumDetailPageProps {
  albumId: string;
}

const CATEGORY_LABELS: Record<string, string> = {
  music: "Âm nhạc", literature: "Văn học", film: "Phim", art: "Mỹ thuật",
  computer: "Phần mềm", photography: "Nhiếp ảnh", stage: "Sân khấu",
  architecture: "Kiến trúc", journalism: "Báo chí", coursebook: "Giáo trình",
};

export default function AlbumDetailPage({ albumId }: AlbumDetailPageProps) {
  const router = useRouter();
  const queryClient = useQueryClient();
  const currentUser = useSelector((state: RootState) => state.user.information);
  const [drawerOpen, setDrawerOpen] = useState(false);
  const [editOpen, setEditOpen] = useState(false);

  const { data: albumData, isLoading, isError } = useQuery({
    queryKey: ["album-detail", albumId],
    queryFn: () => albumApi.getById(albumId).then((r) => r.data.data),
    retry: false,
  });

  const deleteMutation = useMutation({
    mutationFn: () => albumApi.remove(albumId),
    onSuccess: () => {
      message.success("Đã xoá album");
      router.push("/profile/my-albums");
    },
    onError: () => message.error("Xoá album thất bại"),
  });

  const togglePublicMutation = useMutation({
    mutationFn: (isPublic: boolean) => {
      const fd = new FormData();
      fd.append("name", albumData.name);
      fd.append("description", albumData.description ?? "");
      fd.append("is_public", isPublic ? "true" : "false");
      return albumApi.update(albumId, fd);
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["album-detail", albumId] });
      message.success("Đã cập nhật chế độ hiển thị");
    },
  });

  const removeProductMutation = useMutation({
    mutationFn: (productId: string) => albumApi.removeProducts(albumId, [productId]),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["album-detail", albumId] });
    },
    onError: () => message.error("Xoá tác phẩm thất bại"),
  });

  if (isLoading) {
    return (
      <div className="flex items-center justify-center py-20">
        <Spin size="large" />
      </div>
    );
  }

  if (isError || !albumData) {
    return (
      <div className="py-20 text-center">
        <p className="text-lg font-semibold">Album không tồn tại</p>
        <Button className="mt-4" onClick={() => router.back()}>Quay lại</Button>
      </div>
    );
  }

  const isOwner = currentUser?.id === albumData.user_id;

  if (!albumData.is_public && !isOwner) {
    return (
      <div className="py-20 text-center">
        <LockOutlined className="text-5xl text-gray-400 mb-4" />
        <p className="text-lg font-semibold">Album này là riêng tư.</p>
        <Button className="mt-4" onClick={() => router.back()}>Quay lại</Button>
      </div>
    );
  }

  const copyLink = () => {
    if (!albumData.is_public) {
      message.warning("Hãy chuyển album sang công khai để chia sẻ");
      return;
    }
    navigator.clipboard.writeText(window.location.href);
    message.success("Đã sao chép link");
  };

  const ownerActions: MenuProps["items"] = [
    {
      key: "edit",
      icon: <EditOutlined />,
      label: "Sửa thông tin",
      onClick: () => setEditOpen(true),
    },
    {
      key: "toggle-public",
      icon: albumData.is_public ? <EyeInvisibleOutlined /> : <EyeOutlined />,
      label: albumData.is_public ? "Chuyển sang riêng tư" : "Chuyển sang công khai",
      onClick: () => togglePublicMutation.mutate(!albumData.is_public),
    },
    { type: "divider" },
    {
      key: "delete",
      icon: <DeleteOutlined />,
      label: "Xoá album",
      danger: true,
      onClick: () => deleteMutation.mutate(),
    },
  ];

  const existingProductIds: string[] = (albumData.products ?? []).map((p: any) => p.id);

  return (
    <div className="max-w-5xl mx-auto px-4 py-6">
      {/* Top bar */}
      <div className="flex items-center justify-between mb-6">
        <Button icon={<ArrowLeftOutlined />} onClick={() => router.back()} variant="text">
          Quay lại
        </Button>
        {isOwner && (
          <Dropdown menu={{ items: ownerActions }} trigger={["click"]}>
            <Button icon={<EllipsisOutlined />} />
          </Dropdown>
        )}
      </div>

      {/* Album header */}
      <div className="flex gap-6 mb-8 flex-col sm:flex-row">
        <div className="w-full sm:w-48 h-48 flex-shrink-0 rounded-xl overflow-hidden bg-gray-100 dark:bg-gray-700">
          {albumData.cover_url ? (
            <img src={albumData.cover_url} alt={albumData.name} className="w-full h-full object-cover" />
          ) : (
            <div className="w-full h-full flex items-center justify-center text-5xl">🎵</div>
          )}
        </div>
        <div className="flex-1">
          <h1 className="text-2xl font-bold mb-1">{albumData.name}</h1>
          <p className="text-gray-500 text-sm mb-2">
            by <span className="font-medium">{albumData.user_name}</span>
          </p>
          <div className="flex items-center gap-2 mb-3">
            <Tag>{CATEGORY_LABELS[albumData.category] ?? albumData.category}</Tag>
            <Tag>{(albumData.products ?? []).length} tác phẩm</Tag>
            {albumData.is_public ? (
              <Tag icon={<GlobalOutlined />} color="green">Công khai</Tag>
            ) : (
              <Tag icon={<LockOutlined />}>Riêng tư</Tag>
            )}
          </div>
          {albumData.description && (
            <p className="text-gray-600 dark:text-gray-400 text-sm mb-4">{albumData.description}</p>
          )}
          <div className="flex gap-2">
            <Tooltip title={!albumData.is_public ? "Hãy chuyển album sang công khai để chia sẻ" : ""}>
              <Button icon={<LinkOutlined />} onClick={copyLink}>
                Sao chép link
              </Button>
            </Tooltip>
            {isOwner && (
              <Button icon={<EditOutlined />} onClick={() => setEditOpen(true)}>
                Sửa thông tin
              </Button>
            )}
          </div>
        </div>
      </div>

      {/* Products section */}
      <div className="mb-4 flex items-center justify-between">
        <h2 className="text-lg font-semibold">Tác phẩm trong album</h2>
        {isOwner && (
          <Button icon={<PlusOutlined />} onClick={() => setDrawerOpen(true)}>
            Thêm tác phẩm
          </Button>
        )}
      </div>

      {(albumData.products ?? []).length === 0 ? (
        <div className="py-10 text-center text-gray-400">
          <p>Chưa có tác phẩm nào trong album</p>
          {isOwner && (
            <Button className="mt-3" onClick={() => setDrawerOpen(true)} icon={<PlusOutlined />}>
              Thêm tác phẩm
            </Button>
          )}
        </div>
      ) : (
        <div className="grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-4 gap-4">
          {(albumData.products ?? []).map((product: any) => (
            <div key={product.id} className="relative group rounded-lg overflow-hidden border border-gray-200 dark:border-gray-700">
              <div className="aspect-square bg-gray-100 dark:bg-gray-800">
                {product.thumbnail_url ? (
                  <img
                    src={product.thumbnail_url}
                    alt={product.title}
                    className="w-full h-full object-cover"
                  />
                ) : (
                  <div className="w-full h-full flex items-center justify-center text-3xl text-gray-400">🎵</div>
                )}
              </div>
              {isOwner && (
                <button
                  className="absolute top-1 right-1 bg-black/60 text-white rounded-full w-6 h-6 flex items-center justify-center opacity-0 group-hover:opacity-100 transition-opacity text-xs"
                  onClick={() => removeProductMutation.mutate(product.id)}
                >
                  <CloseOutlined />
                </button>
              )}
              <div className="p-2">
                <p className="text-xs font-medium truncate">{product.title}</p>
              </div>
            </div>
          ))}
        </div>
      )}

      {/* Drawers / Modals */}
      {isOwner && (
        <>
          <AddProductsDrawer
            open={drawerOpen}
            albumId={albumId}
            albumCategory={albumData.category}
            existingProductIds={existingProductIds}
            onClose={() => setDrawerOpen(false)}
          />
          <EditAlbumModal
            open={editOpen}
            album={albumData ? { ...albumData, id: albumId, product_count: (albumData.products ?? []).length } : null}
            onClose={() => setEditOpen(false)}
            onUpdated={() => {
              queryClient.invalidateQueries({ queryKey: ["album-detail", albumId] });
              setEditOpen(false);
            }}
          />
        </>
      )}
    </div>
  );
}
```

- [ ] **Step 3: Create `albums/[id]/page.tsx`**

First, ensure the `(album)` route group exists:

```bash
mkdir -p "/Users/macos/workspace-user/marketplace-frontend/app/[locale]/(album)/albums/[id]"
```

Then create the page:

```tsx
// app/[locale]/(album)/albums/[id]/page.tsx
import MainLayout from "@/common/components/Layouts/MainLayout";
import AlbumDetailPage from "@/common/pages/Album/AlbumDetailPage";

interface AlbumPageProps {
  params: { id: string; locale: string };
}

export default function AlbumPage({ params }: AlbumPageProps) {
  return (
    <MainLayout>
      <AlbumDetailPage albumId={params.id} />
    </MainLayout>
  );
}
```

- [ ] **Step 4: Verify TypeScript compiles**

```bash
cd /Users/macos/workspace-user/marketplace-frontend
npx tsc --noEmit 2>&1 | grep -E "(AlbumDetail|AddProducts|albums/\[id\])"
```

Expected: no output (no errors).

- [ ] **Step 5: Commit**

```bash
cd /Users/macos/workspace-user/marketplace-frontend
git add \
  common/pages/Album/AlbumDetailPage.tsx \
  common/pages/Album/AddProductsDrawer.tsx
# Use shell escaping for special chars in git add:
git add "app/[locale]/(album)/albums/[id]/page.tsx"
git commit -m "$(cat <<'EOF'
feat(marketplace-frontend): add Album detail page with owner controls and add-products drawer

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Spec Coverage Check

| Spec Section | Covered By |
|---|---|
| MongoDB schema `albumcol` | Task 1 |
| CRUD query helpers | Task 1 |
| `POST /albums` with multipart cover | Task 2 |
| `GET /albums/my` paginated | Task 3 |
| `GET /albums/:id` with product enrichment + access control | Task 3 |
| `PUT /albums/:id` ownership check + optional cover | Task 4 |
| `DELETE /albums/:id` soft delete | Task 4 |
| `POST /albums/:id/products` ownership validation of each product | Task 5 |
| `DELETE /albums/:id/products` | Task 5 |
| Register router in `ver1.go` | Task 6 |
| `albumApi` frontend module | Task 7 |
| `MY_ALBUMS` in NavigationPath + sidebar entries | Task 8 |
| "Album của tôi" in SettingsLayout menu | Task 8 |
| My Albums list page with grid, pagination, empty state | Task 9 |
| CreateAlbumModal with product picker | Task 9 |
| EditAlbumModal pre-filled | Task 9 |
| Album detail — owner controls (actions dropdown, edit, copy link, add/remove products) | Task 10 |
| Album detail — public read-only view | Task 10 |
| Album detail — 403 page for private + non-owner | Task 10 |
| AddProductsDrawer — filters to category, excludes existing products | Task 10 |
| Empty state "Không còn sản phẩm nào phù hợp" | Task 10 |
| Removed product reappears in drawer (query invalidation) | Task 10 |

All spec requirements covered. No TBDs or placeholders remain.
