# Album Feature Design

**Date:** 2026-04-13  
**Scope:** Full-stack — `marketplace-backend` (Go/Gin/MongoDB) + `marketplace-frontend` (Next.js)  
**Status:** Approved

---

## 1. Overview

Users can create curated albums of their own products, grouped by category. Albums are private by default; the owner can toggle them public to share a URL. The album detail page (`/albums/[id]`) serves both the owner (with edit controls) and the public (read-only) — Spotify-style conditional controls based on auth state.

---

## 2. Backend

### 2.1 MongoDB Schema — `albumcol`

**File:** `marketplace-backend/schema/albumcol/model.go`

```go
type Album struct {
    mongodb.DefaultModel   `json:",inline" bson:",inline,omitnested"`
    CreatedAt   time.Time            `json:"created_at" bson:"created_at"`
    UpdatedAt   time.Time            `json:"updated_at" bson:"updated_at"`

    UserID      primitive.ObjectID   `json:"user_id" bson:"user_id"`
    Name        string               `json:"name" bson:"name"`
    Description string               `json:"description" bson:"description"`
    Category    string               `json:"category" bson:"category"`   // category_code: "music", "literature", etc.
    CoverURL    string               `json:"cover_url" bson:"cover_url"` // S3-hosted image
    IsPublic    bool                 `json:"is_public" bson:"is_public"` // default false
    ProductIDs  []primitive.ObjectID `json:"product_ids" bson:"product_ids"`
    IsDeleted   bool                 `json:"is_deleted" bson:"is_deleted"`
}

func (Album) CollectionName() string { return "album" }
```

**File:** `marketplace-backend/schema/albumcol/query.go`

Standard CRUD helpers:
- `Create(ctx, doc)` — insert
- `FindByID(ctx, id)` — single doc
- `FindByUserID(ctx, userID, page, size)` — paginated list, filter `is_deleted=false`
- `FindPublicByID(ctx, id)` — single doc, require `is_public=true`
- `Update(ctx, doc)` — full update
- `SoftDelete(ctx, id, userID)` — set `is_deleted=true`
- `AddProducts(ctx, id, userID, productIDs[])` — `$addToSet` into `product_ids`
- `RemoveProducts(ctx, id, userID, productIDs[])` — `$pull` from `product_ids`

---

### 2.2 API Endpoints — `business/album/`

**Router:** `marketplace-backend/business/album/router.go`

```
POST   /albums                    AuthMiddleware()   → create
GET    /albums/my                 AuthMiddleware()   → list my albums
GET    /albums/:id                (no auth)          → get detail (owner or public)
PUT    /albums/:id                AuthMiddleware()   → update info
DELETE /albums/:id                AuthMiddleware()   → soft delete
POST   /albums/:id/products       AuthMiddleware()   → add products
DELETE /albums/:id/products       AuthMiddleware()   → remove products
```

**Handler files:**
- `create.go` — validates name + category required; accepts `multipart/form-data` for cover image upload (reuses existing S3 upload pattern); creates album doc; returns created album
- `list_my.go` — paginates by `user_id`; returns array + total
- `get_detail.go` — if `is_public=false` and caller is not owner → 403; enriches `product_ids` with product data via lookup on `original_product` collection; returns album + `products[]`
- `update.go` — verifies ownership; accepts `multipart/form-data` (cover optional); updates mutable fields: name, description, is_public, cover_url
- `delete.go` — verifies ownership; soft delete
- `add_products.go` — verifies ownership; validates each product_id belongs to caller (query `original_product` with `user_id` + `_id IN [...]` filter); if **any** product_id is invalid or not owned → reject entire batch with 400 + list of invalid IDs; on success: `$addToSet` to avoid duplicates
- `remove_products.go` — verifies ownership; `$pull` product_ids from album

**Ownership check:** every mutating handler extracts `user_id` from JWT middleware context, queries album by `_id`, asserts `album.UserID == callerUserID` before proceeding → 403 if mismatch.

**Product enrichment in `get_detail.go`:**
```
1. Fetch album doc
2. If len(product_ids) > 0: query originalproductcol WHERE _id IN product_ids
3. Return album metadata + products array (title, thumbnail_url, category, copyright_status, etc.)
```

**Register in `routers/ver1.go`:**
```go
albumRouter := r.Group("albums")
album.AddRouter(albumRouter)
```

---

### 2.3 Response Shapes

**Album summary (list):**
```json
{
  "id": "...",
  "name": "Mùa đông 2025",
  "description": "...",
  "category": "music",
  "cover_url": "https://...",
  "is_public": false,
  "product_count": 5,
  "created_at": "..."
}
```

**Album detail:**
```json
{
  "id": "...",
  "name": "...",
  "category": "music",
  "cover_url": "...",
  "is_public": true,
  "description": "...",
  "user_id": "...",
  "user_name": "...",
  "user_avatar": "...",
  "products": [
    { "id": "...", "title": "...", "thumbnail_url": "...", "category": {...}, "copyright_status": "..." }
  ],
  "created_at": "..."
}
```

---

## 3. Frontend

### 3.1 Routing

| Route | Description |
|---|---|
| `/profile/my-albums` | My albums list (auth required) |
| `/albums/[id]` | Album detail — owner sees edit controls, others see read-only |

No separate owner/public URL — same page, conditional controls based on `currentUser.id === album.user_id`.

### 3.2 Sidebar Navigation

Added to both `Sidebar.tsx` and `NavMobile.tsx` under **CÁ NHÂN** section:

```
Tác phẩm của tôi       → /management-studio/original
Album của tôi          → /profile/my-albums        ← NEW
Chương trình Affiliate → /affiliate
Thông tin cá nhân      → /profile/settings
```

`data.tsx`: new item `{ key: "my-albums", href: "/profile/my-albums", label: "Album của tôi", iconDefault: "/icons/menu/icon-album.svg" }` (fallback to existing pencil icon if SVG not available).

`NavigationPath.ts`: `MY_ALBUMS: "/profile/my-albums"`

### 3.3 Page: My Albums List (`/profile/my-albums`)

**File:** `app/[locale]/(profile)/profile/my-albums/page.tsx`

**Layout:**
- Wrapped in `RouteProtection` + `MainLayout` + `SettingsLayout`
- Header row: "Album của tôi" title + `[+ Tạo Album]` button (primary, top-right)
- Grid: 3 cols desktop / 2 tablet / 1 mobile, gap-4
- Pagination at bottom

**Album card:**
```
┌──────────────────────────┐
│   cover image (16:9)     │
│   🔒 or 🌐 badge         │  top-left
├──────────────────────────┤
│   Album Name             │
│   Âm nhạc · 5 tác phẩm  │
│   [Sửa] [Xoá]           │
└──────────────────────────┘
```
- Click card body (not edit/delete) → `/albums/[id]`
- `[Xoá]` shows Ant Design Popconfirm before deleting

**Empty state:** illustration + "Bạn chưa có album nào" + "Tạo album đầu tiên" CTA button

### 3.4 Create Album Modal

Triggered by `[+ Tạo Album]` button. Ant Design `Modal` component, scrollable.

**Fields:**
| Field | Type | Required | Notes |
|---|---|---|---|
| Tên album | Text input | ✅ | max 100 chars |
| Thể loại | Select | ✅ | Drives product picker below |
| Mô tả | Textarea | ❌ | max 500 chars |
| Ảnh bìa | Upload (drag+drop) | ✅ | Preview inline, accepts image/* |
| Chọn tác phẩm | Checklist | ❌ | Renders after category selected |

**Product picker behavior:**
- Fetches `getMyOriginalProducts({ category })` when category changes
- Filters out products already in the album (`product_ids`) — for create, this is an empty set so all matching products shown
- Each row: thumbnail (40x40) + title + copyright status badge
- Empty state when no products match: "Bạn chưa có tác phẩm nào trong danh mục này"
- Search input to filter product list client-side

**Submit:** `POST /albums` with `multipart/form-data`. On success: invalidate `my-albums` query, close modal, show success toast, redirect to `/albums/[id]`.

### 3.5 Page: Album Detail (`/albums/[id]`)

**File:** `app/[locale]/(album)/albums/[id]/page.tsx`

**Owner view (currentUser.id === album.user_id):**

```
┌────────────────────────────────────────────────────────┐
│  [← Quay lại]                             [⋯ Actions] │
│                                                        │
│  ┌──────────────┐   Album Name                         │
│  │  cover 1:1   │   by @username                       │
│  │              │   Âm nhạc · 5 tác phẩm               │
│  └──────────────┘   🔒 Riêng tư                        │
│                     Mô tả ngắn...                      │
│                     [🔗 Sao chép link] [✏️ Sửa thông tin]│
│                                                        │
│  ──── Tác phẩm trong album ────  [+ Thêm tác phẩm]   │
│                                                        │
│  [grid of product cards with ✕ remove button overlay] │
└────────────────────────────────────────────────────────┘
```

- `[⋯ Actions]` dropdown: "Sửa thông tin", "Chuyển sang công khai / riêng tư", "Xoá album"
- `[🔗 Sao chép link]`: copies `window.location.href` to clipboard; only enabled when `is_public=true`. If private → shows tooltip "Hãy chuyển album sang công khai để chia sẻ"
- Product cards: on hover, show `✕` remove button (top-right corner of card). Click → calls `DELETE /albums/:id/products`, optimistic UI update
- `[+ Thêm tác phẩm]` opens **Add Products Drawer** (right-side Ant Design Drawer):
  - Lists products in album's category NOT yet in album
  - If all products already added: empty state "Không còn sản phẩm nào phù hợp với danh mục này"
  - When a product is removed from album → it reappears in this drawer (query re-fetches)
  - Checkbox list + "Thêm X tác phẩm" confirm button

**Public view (not owner, album.is_public=true):**
- Same layout without: `[⋯ Actions]`, `[+ Thêm tác phẩm]`, `✕` on cards, `[✏️ Sửa thông tin]`
- `[🔗 Sao chép link]` still visible (shareable)

**Private + not owner:** render a 403 page "Album này là riêng tư."

### 3.6 Edit Album Info Modal

Same form as Create (minus cover upload being optional on edit — show current cover, allow replace). Pre-filled with current values. On submit: `PUT /albums/:id`.

### 3.7 Frontend API — `app/api/album.ts`

```ts
export const albumApi = {
  create(data: FormData)
  getMyAlbums(params: { page?: number; size?: number })
  getById(id: string)
  update(id: string, data: FormData)
  remove(id: string)
  addProducts(id: string, productIds: string[])
  removeProducts(id: string, productIds: string[])
}
```

All calls go through `getInstance()` (existing Axios instance with auth interceptors).

### 3.8 State Management

TanStack Query throughout — consistent with rest of app:
- `["my-albums"]` — list page
- `["album-detail", id]` — detail page
- Mutations invalidate the relevant queries on success

---

## 4. Error Handling

| Scenario | Behavior |
|---|---|
| Album not found | 404 response → frontend shows "Album không tồn tại" |
| Private album, not owner | 403 → "Album này là riêng tư" page |
| Product not owned by user | 400 → toast "Không thể thêm sản phẩm không thuộc về bạn" |
| Cover upload fails | Toast error, form stays open |
| Remove last product | Allowed — album can be empty |

---

## 5. File Structure

### Backend (new files)
```
marketplace-backend/
  business/album/
    router.go
    create.go
    list_my.go
    get_detail.go
    update.go
    delete.go
    add_products.go
    remove_products.go
  schema/albumcol/
    model.go
    query.go
```

### Frontend (new files)
```
marketplace-frontend/
  app/api/album.ts
  app/[locale]/(album)/albums/[id]/page.tsx
  app/[locale]/(profile)/profile/my-albums/page.tsx
  app/[locale]/(profile)/profile/my-albums/components/
    AlbumCard.tsx
    CreateAlbumModal.tsx
    EditAlbumModal.tsx
  common/pages/Album/
    AlbumDetailPage.tsx
    AddProductsDrawer.tsx
```

### Modified files
```
marketplace-backend/routers/ver1.go           ← register album router
marketplace-frontend/config/NavigationPath.ts ← add MY_ALBUMS
marketplace-frontend/common/components/Header/data.tsx     ← add my-albums item
marketplace-frontend/common/components/Layouts/Sidebar.tsx ← add to CÁ NHÂN section
marketplace-frontend/common/components/Header/NavMobile.tsx ← add to personalItems
```

---

## 6. Out of Scope

- Album reordering (drag-and-drop product sort) — can be added later via `position` field
- Multi-category albums — one category per album for now (keeps product picker focused)
- Album likes/comments — future social feature
- Admin moderation of public albums
