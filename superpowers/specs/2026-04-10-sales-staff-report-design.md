# Spec: Báo cáo doanh thu Sub + Mua lẻ (Sales-Staff Report Page)

**Date**: 2026-04-10  
**Scope**: `marketplace-admin` (frontend) + `marketplace-admin-api` (backend) only  
**Route**: `/dashboard/reports/sales-staff` (currently a placeholder)

---

## Context

The business needs a report page that tracks subscription (membership) vs retail purchase revenue, segmented by attribution source — affiliate partner, individual sale staff, or unattributed "unknown" customers. A commission breakdown for affiliate partners is also included. This is the first of 4 new report pages to be built on top of the existing 4 report pages (which remain unchanged).

---

## Key Data Model Facts

### Attribution Segments

| Segment | Detection Logic | Source Field |
|---|---|---|
| **Affiliate** | `user.referred_by` is set AND referrer has `is_partner: true` | `user` collection |
| **Sale staff** | `user.assigned_sales_admin_id` is set | `user` collection |
| **Unknown** | Neither of the above | `user` collection |

### Sub vs Mua lẻ

| Category | Transaction Types |
|---|---|
| **Sub** | `membership` only (`TransactionTypeMembership`) |
| **Mua lẻ** | `storage`, `encoding`, `scan_similarity`, `protection`, `copyright`, `legal_advice`, `scan_turn`, `purchase` |

### Sub Breakdown Logic

| Metric | Logic |
|---|---|
| DK Sub mới | Membership purchases in period where user has NO prior membership purchase before period start |
| Sub cũ (renewal) | Membership purchases in period where user DID have a prior membership purchase before period start |
| Hủy | Users whose `membership_expire_date` falls within period AND made no membership purchase in that period |
| Còn lại sub active | DK Sub mới + Sub cũ (i.e. total successful membership purchases in period) |

### Mua lẻ Breakdown (mapped display labels)

| Transaction Type | Display Label |
|---|---|
| `storage` | Data |
| `copyright`, `publication` | Đăng ký công bố |
| `legal_advice` | Tư vấn |
| `scan_similarity`, `scan_turn` | Quét bản quyền |
| `encoding` | Mã hoá |
| `protection` | Bảo hộ |
| `purchase` | Nhượng quyền |

### Commission

- Source collection: `affiliate_commission` (`partner_id`, `base_amount`, `commission_amount`, `rate`)
- Tax (Thuế): 10% TNCN applied on `commission_amount`
- Net commission: `commission_amount × (1 - 0.10)`

---

## Page Layout (Option A — Single Scrollable Page)

```
┌────────────────────────────────────────────────────┐
│ FilterBar (enhanced)                               │
│  Segment: [All] [Affiliate ▼] [Sale staff ▼] [Unknown]  │
│  Time:    [Từ ngày → Đến ngày] [Tuần trước] [Tháng trước] │
└────────────────────────────────────────────────────┘

┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ Tổng DT  │ │  DT Sub  │ │ DT Mua lẻ│ │Sub active│
│ KpiCard  │ │ KpiCard  │ │ KpiCard  │ │ KpiCard  │
└──────────┘ └──────────┘ └──────────┘ └──────────┘

┌────────────── Báo cáo có các trường ─────────────┐
│ [Sl Sub*] [Sl Mua lẻ] [DT Sub] [DT Mua lẻ] [Tổng DT] │  ← Layer 2 KpiCards, clickable
│                                                  │
│ ▼ (if Sl Sub active):                            │
│   ReportKpiTable: DK Sub mới / Sub cũ / Hủy / Còn lại  │
└────────────────────────────────────────────────────┘

┌────────────── Báo cáo hoa hồng ──────────────────┐
│ (visible only when segment = all or affiliate)   │
│ StandardTable: Partner | DT | % HH | Thuế | Thực nhận │
│ Summary row: Tổng                                │
└────────────────────────────────────────────────────┘
```

*Active card highlighted, breakdown table slides in below card row*

---

## Backend: `marketplace-admin-api`

### New directory
`business/reports/salesstaff/`

### New files

| File | Handler | Route |
|---|---|---|
| `router.go` | — | Registers sub-routes under `/reports/sales-staff` |
| `overview.go` | `Overview()` | `GET /reports/sales-staff/overview` |
| `details.go` | `Details()` | `GET /reports/sales-staff/details` |
| `commission.go` | `Commission()` | `GET /reports/sales-staff/commission` |

### Modified files

| File | Change |
|---|---|
| `business/reports/router.go` | Add `salesstaff.RegisterRoutes(rg)` |
| `business/role/keys.go` (or equivalent) | Add `KeyReportsSalesStaff = "reports_sales_staff_view"` |

### Shared Query Parameters (all 3 endpoints)

| Param | Values | Notes |
|---|---|---|
| `segment` | `all` \| `affiliate` \| `sale` \| `unknown` | Default: `all` |
| `affiliate_id` | string | When `segment=affiliate`; narrows to one partner |
| `sale_id` | string | When `segment=sale`; narrows to one staff member |
| `date_from` | `2006-01-02` | Period start |
| `date_to` | `2006-01-02` | Period end |
| `time_range` | `last_week` \| `last_month` | Preset (mutually exclusive with date_from/to) |

### Response Schemas

**`GET /reports/sales-staff/overview`**
```json
{
  "total_revenue": 1000000,
  "sub_revenue": 600000,
  "retail_revenue": 400000,
  "active_subs_count": 150
}
```

**`GET /reports/sales-staff/details`**
```json
{
  "sub": {
    "new": 20,
    "renewal": 45,
    "cancelled": 5,
    "active": 65
  },
  "retail": {
    "storage": 10,
    "publication": 8,
    "legal_advice": 3,
    "scan": 12,
    "encoding": 4,
    "protection": 2,
    "purchase": 1
  },
  "sub_revenue": 600000,
  "retail_revenue": 400000,
  "total_revenue": 1000000
}
```

**`GET /reports/sales-staff/commission`**
```json
{
  "data": [
    {
      "partner_id": "...",
      "partner_name": "Nguyen Van A",
      "partner_email": "a@example.com",
      "revenue": 500000,
      "commission_rate": 0.05,
      "commission_amount": 25000,
      "tax": 2500,
      "net_commission": 22500
    }
  ],
  "total": {
    "revenue": 1000000,
    "commission_amount": 50000,
    "tax": 5000,
    "net_commission": 45000
  },
  "pagination": { "page": 1, "limit": 20, "total": 5 }
}
```

### MongoDB Aggregation Strategy

**Step 1 — Resolve user_id list from segment filter** (shared helper, called by all 3 handlers):
- `affiliate`: query `user` where `referred_by != ""` → lookup referrer → filter `is_partner = true` → collect `user_id`s. If `affiliate_id` provided, narrow to users where `referred_by = affiliate_id`.
- `sale`: query `user` where `assigned_sales_admin_id != ""`. If `sale_id` provided, narrow to `assigned_sales_admin_id = sale_id`.
- `unknown`: query `user` where `referred_by = ""` AND `assigned_sales_admin_id = ""`.
- `all`: no user_id filter.

**Step 2 — Overview aggregation** (on `purchase` collection):
- `$match`: `user_id in [resolved list]`, `status = "success"`, `created_at` in date range
- `$group`: sum `total_fee` overall, sum `total_fee` where `type = membership` (sub_revenue), sum rest (retail_revenue)
- Separate query for `active_subs_count`: count `user` where `membership_expire_date > now` AND user_id in resolved list

**Step 3 — Details aggregation** (on `purchase` collection):
- Sub new: membership purchases in period where user has no prior membership purchase (`created_at < date_from`)
- Sub renewal: membership purchases in period where user has a prior membership purchase
- Hủy: users in resolved list where `membership_expire_date` in period AND no membership purchase in period
- Retail: `$group` by `type`, count per type

**Step 4 — Commission aggregation** (on `affiliate_commission` collection):
- `$match`: `created_at` in date range. If `affiliate_id` specified, `$match partner_id = affiliate_id`.
- `$group` by `partner_id`: sum `base_amount`, sum `commission_amount`
- `$addFields`: `tax = commission_amount * 0.10`, `net_commission = commission_amount * 0.90`
- `$lookup` partner name/email from `user` collection
- `$facet` for pagination + total summary row

---

## Frontend: `marketplace-admin`

### Modified files

| File | Change |
|---|---|
| `app/dashboard/(dashboard)/reports/sales-staff/page.tsx` | Replace placeholder with full page implementation |

### New files

| File | Purpose |
|---|---|
| `app/api/reports-sales-staff.ts` | 3 API functions: `overview()`, `details()`, `commission()` |
| `components/dashboard/reports/SalesStaffOverviewCards.tsx` | 4x `KpiMetricCard` summary grid |
| `components/dashboard/reports/SalesStaffDetailsSection.tsx` | Layer 2 clickable cards + Layer 3 expandable `ReportKpiTable` |
| `components/dashboard/reports/SalesStaffCommissionTable.tsx` | `StandardTable` for commission + totals row |

### Reused Components (no modification)

| Component | Usage |
|---|---|
| `KpiMetricCard` | Layer 2 cards + overview cards |
| `ReportKpiTable` | Layer 3 breakdown (slides in on card click) |
| `StandardTable` | Commission table |
| `FilterBar` + `DatePickerInput` | Enhanced with Segment selector — extend, don't fork |

### State

```ts
// Filter state
const [segment, setSegment] = useState<'all' | 'affiliate' | 'sale' | 'unknown'>('all')
const [affiliateId, setAffiliateId] = useState<string | null>(null)
const [saleId, setSaleId] = useState<string | null>(null)
const [dateFrom, setDateFrom] = useState<string | null>(null)
const [dateTo, setDateTo] = useState<string | null>(null)
const [timeRange, setTimeRange] = useState<'last_week' | 'last_month' | null>('last_month')

// Active detail card
const [activeCard, setActiveCard] = useState<'sub' | 'retail' | 'sub_revenue' | 'retail_revenue' | 'total' | null>(null)
```

Data fetching via React Query with filter params as query key (follows existing pattern).

---

## Verification

1. `make dev` in `marketplace-admin-api`
2. `GET /reports/sales-staff/overview?segment=all&time_range=last_month` → 200 with 4 fields
3. `GET /reports/sales-staff/details?segment=affiliate` → sub/retail breakdown
4. `GET /reports/sales-staff/commission?segment=all` → paginated partner rows
5. `npm run dev` in `marketplace-admin`
6. `/dashboard/reports/sales-staff` renders (not placeholder)
7. Filter bar: segment switcher + time range work
8. 4 overview KPI cards load
9. Click "Số lượng sub" card → breakdown table slides in with 4 rows
10. Switch to `Unknown` segment → commission table hides
11. Switch to `Affiliate` → commission table shows, affiliate dropdown appears in filter bar
12. `npm run lint` → 0 errors
13. `make lint` in admin-api → 0 errors
