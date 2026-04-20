# Membership Redesign — 6 User Types + Strike Turn + Pricing Fields Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend the membership system to support 6 distinct user types (free-KYC + 5 paid tiers), add a Strike Turn quota feature, configure membership-tier-specific pricing for publication/scan/strike, add negotiation/exploitation fee percentages, and redesign the PackageForm UI with a duration-type selector.

**Architecture:** All schema changes are additive (new fields with `omitempty`), preserving backward compatibility with existing documents. `DurationType` is repurposed from storage-only to general (membership now uses "monthly"/"yearly"/"custom"). Pricing per membership tier lives ON the membership package itself; free-tier pricing stays on each respective package type (publication, scan_turn). Strike follows the exact same pattern as `legal_advice_turn` / `copyright_scan_count`.

**Tech Stack:** Go 1.24 / Gin / MongoDB (mgm); Next.js 15 / React 19 / TypeScript / Ant Design 5 / TanStack Query; 4 repos: marketplace-backend, marketplace-admin-api, marketplace-admin, marketplace-frontend (frontend changes are minor).

---

## Pre-task Context

### Current User Types (derived from `is_member` + package's `is_company` + `DurationType`)

| User Type | `is_member` | `is_company` (User) | `DurationType` |
|---|---|---|---|
| Free KYC | false | false | — |
| Individual Monthly | true | false | monthly |
| Individual Yearly | true | false | yearly |
| Company Monthly | true | true | monthly |
| Company Yearly | true | true | yearly |
| Company Custom | true | true | custom |

### Quota / Pricing Fields to Add to Package (type=membership)

| Field | Purpose |
|---|---|
| `duration_type` | "monthly"/"yearly"/"custom" — already exists, repurpose for membership |
| `legal_advice_initial_turn` | Turns added per membership purchase (was hardcoded 1) |
| `strike_count` | Initial strike turns granted (-1 not stored here; use `unlimited_strike`) |
| `unlimited_strike` | true = company plan, set user.strike_turn = -1 |
| `publication_base_price` | Base price/work for users on this tier |
| `publication_software_min_price` | Software/DB min price for this tier |
| `publication_software_max_price` | Software/DB max price for this tier |
| `scan_turn_additional_price` | Price per extra scan turn for this tier |
| `strike_turn_additional_price` | Price per extra strike turn for this tier |
| `negotiation_fee_percent` | % commission on negotiation (0–1) |
| `exploitation_fee_percent` | % commission on exploitation delegation (0–1) |

### New User Fields

| Field | Purpose |
|---|---|
| `is_company` | Copied from pkg.IsCompany on membership purchase; cleared on expiry |
| `strike_turn` | Current strike turns remaining; -1 = unlimited (company plans) |

---

## File Map

| File | Action | Repos |
|---|---|---|
| `schema/packagecol/enum.go` | Add `PackageTypeStrikeTurn`, `MembershipDurationType` type + constants | backend + admin-api |
| `schema/packagecol/model.go` | Add 11 new fields | backend + admin-api |
| `schema/usercol/model.go` | Add `IsCompany`, `StrikeTurn` | backend only |
| `schema/purchasecol/enum.go` | Add `TransactionTypeStrikeTurn` | backend only |
| `schema/purchasecol/metadata.go` | Add `StrikeTurnPaymentMetadata` | backend only |
| `business/payment/vnpay/handle_membership.go` | Set `is_company`, `strike_turn`, use `legal_advice_initial_turn` | backend |
| `business/payment/vnpay/handle_strike_turn.go` | **New** — mirror `handle_scan_turn.go` | backend |
| `business/payment/vnpay/vnpay_ipn.go` | Add `case TransactionTypeStrikeTurn` | backend |
| `business/payment/vnpay/vnpay_create_link.go` | Add strike turn case + `doCreatePaymentLinkForStrikeTurn` | backend |
| `services/cronjob/membership_storage_cron.go` | Reset `strike_turn` + `is_company` on membership expiry | backend |
| `business/package/create.go` | Add new fields to request + Package construction | admin-api |
| `business/package/update.go` | Add new fields to update request | admin-api |
| `business/package/list.go` | Add new fields to `PackageResponseData` + mapping | admin-api |
| `business/package/detail.go` | Add new fields to `PackageDetailResponseData` + mapping | admin-api |
| `business/striketurnpackage/router.go` | **New** | admin-api |
| `business/striketurnpackage/create.go` | **New** | admin-api |
| `business/striketurnpackage/update.go` | **New** | admin-api |
| `business/striketurnpackage/list.go` | **New** | admin-api |
| `business/striketurnpackage/delete.go` | **New** | admin-api |
| `business/role/keys.go` | Add strike turn package router keys | admin-api |
| `routers/ver1.go` | Register `strike-turn-packages` route group | admin-api |
| `app/api/package.ts` | Add new fields to all interfaces | admin |
| `components/dashboard/membership-management/PackageForm.tsx` | Add duration_type selector + all new fields | admin |

---

## Task 1: packagecol Schema — Add Enum Constants and New Fields

**Files:**
- Modify: `marketplace-backend/schema/packagecol/enum.go`
- Modify: `marketplace-backend/schema/packagecol/model.go`
- Modify: `marketplace-admin-api/schema/packagecol/enum.go`
- Modify: `marketplace-admin-api/schema/packagecol/model.go`

> Note: Both repos have identical `packagecol` packages. Apply the same changes to each.

- [ ] **Step 1.1: Update marketplace-backend/schema/packagecol/enum.go**

Replace the full file:

```go
package packagecol

// PackageType discriminates package kind in the unified "packages" collection.
type PackageType string

const (
	PackageTypeMembership      PackageType = "membership"
	PackageTypeStorage         PackageType = "storage"
	PackageTypeLegalAdviceTurn PackageType = "legal_advice_turn"
	PackageTypeScanTurn        PackageType = "scan_turn"   // lượt quét vi phạm bản quyền
	PackageTypeStrikeTurn      PackageType = "strike_turn" // lượt yêu cầu đánh gậy (strike request)
	PackageTypePublication     PackageType = "publication"
	PackageTypeEncodingFee     PackageType = "encoding_fee"
	PackageTypeExploitation    PackageType = "exploitation"
	PackageTypeHardCopy        PackageType = "hard_copy"
)

// MembershipDurationType indicates how the subscription period is computed for membership packages.
type MembershipDurationType string

const (
	MembershipDurationMonthly MembershipDurationType = "monthly" // recurring monthly (duration_months field)
	MembershipDurationYearly  MembershipDurationType = "yearly"  // recurring yearly (typically duration_months = 12)
	MembershipDurationCustom  MembershipDurationType = "custom"  // arbitrary period (duration_days + duration_months, enterprise only)
)

func (t PackageType) String() string {
	return string(t)
}

// AllPackageTypes returns every package kind used for partner affiliate rate mapping.
func AllPackageTypes() []PackageType {
	return []PackageType{
		PackageTypeMembership,
		PackageTypeStorage,
		PackageTypeLegalAdviceTurn,
		PackageTypeScanTurn,
		PackageTypeStrikeTurn,
		PackageTypePublication,
		PackageTypeEncodingFee,
		PackageTypeExploitation,
		PackageTypeHardCopy,
	}
}
```

- [ ] **Step 1.2: Update marketplace-backend/schema/packagecol/model.go**

Replace the full file:

```go
package packagecol

import (
	"context"
	"time"

	"api/internal/mongodb"
	"api/internal/timer"
)

// Package is the unified package model: membership, storage, legal_advice_turn, scan_turn, strike_turn.
// Stored in MongoDB collection "packages". Use Type to discriminate; only fields for that type are populated.
type Package struct {
	mongodb.DefaultModel `json:",inline" bson:",inline,omitnested"`

	Type PackageType `json:"type" bson:"type"`

	// Common
	Code        string    `json:"code" bson:"code"`
	Name        string    `json:"name" bson:"name"`
	DisplayName string    `json:"display_name,omitempty" bson:"display_name,omitempty"`
	Description string    `json:"description" bson:"description"`
	Price       float64   `json:"price" bson:"price"`
	TaxRate     float64   `json:"tax_rate,omitempty" bson:"tax_rate,omitempty"`
	Currency    string    `json:"currency" bson:"currency"`
	IsActive    bool      `json:"is_active" bson:"is_active"`
	// DurationType: for membership packages = "monthly"|"yearly"|"custom" (see MembershipDurationType).
	// For storage packages = existing values (e.g. "monthly", "yearly").
	DurationType string    `json:"duration_type,omitempty" bson:"duration_type,omitempty"`
	CreatedAt    time.Time `json:"created_at" bson:"created_at"`
	UpdatedAt    time.Time `json:"updated_at" bson:"updated_at"`

	// Membership-only: basic quota
	DurationDays             int      `json:"duration_days,omitempty" bson:"duration_days,omitempty"`
	DurationMonths           int      `json:"duration_months,omitempty" bson:"duration_months,omitempty"`
	Benefits                 []string `json:"benefits,omitempty" bson:"benefits,omitempty"`
	UnlimitedProductCreation bool     `json:"unlimited_product_creation,omitempty" bson:"unlimited_product_creation,omitempty"`
	ProductCreationCount     int      `json:"product_creation_count,omitempty" bson:"product_creation_count,omitempty"`
	ScanViolationCount       int      `json:"scan_violation_count,omitempty" bson:"scan_violation_count,omitempty"`
	StorageCapacity          int      `json:"storage_capacity,omitempty" bson:"storage_capacity,omitempty"`
	IsPopular                bool     `json:"is_popular,omitempty" bson:"is_popular,omitempty"`
	IsCompany                bool     `json:"is_company,omitempty" bson:"is_company,omitempty"`
	IAPAppleCode             string   `json:"iap_apple_code,omitempty" bson:"iap_apple_code,omitempty"`

	// Membership-only: legal advice reward
	// Number of legal advice turns added to user.legal_advice_turn on membership purchase.
	// Defaults to 1 if zero (backward-compatible).
	LegalAdviceInitialTurn int `json:"legal_advice_initial_turn,omitempty" bson:"legal_advice_initial_turn,omitempty"`

	// Membership-only: strike turns (đánh gậy)
	// StrikeCount: initial turns granted on purchase (ignored when UnlimitedStrike is true).
	StrikeCount     int  `json:"strike_count,omitempty" bson:"strike_count,omitempty"`
	// UnlimitedStrike: if true, user.strike_turn is set to -1 (unlimited) — for company plans.
	UnlimitedStrike bool `json:"unlimited_strike,omitempty" bson:"unlimited_strike,omitempty"`

	// Membership-only: per-tier pricing config for other package types.
	// Free-tier prices live on their respective package types (publication price, scan_turn price, etc.).
	PublicationBasePrice        float64 `json:"publication_base_price,omitempty" bson:"publication_base_price,omitempty"`               // giá công bố cơ bản/tác phẩm
	PublicationSoftwareMinPrice float64 `json:"publication_software_min_price,omitempty" bson:"publication_software_min_price,omitempty"` // giá phần mềm tối thiểu
	PublicationSoftwareMaxPrice float64 `json:"publication_software_max_price,omitempty" bson:"publication_software_max_price,omitempty"` // giá phần mềm tối đa
	ScanTurnAdditionalPrice     float64 `json:"scan_turn_additional_price,omitempty" bson:"scan_turn_additional_price,omitempty"`         // giá mua thêm 1 lượt quét
	StrikeTurnAdditionalPrice   float64 `json:"strike_turn_additional_price,omitempty" bson:"strike_turn_additional_price,omitempty"`     // giá mua thêm 1 lượt đánh gậy

	// Membership-only: commission percentages
	NegotiationFeePercent  float64 `json:"negotiation_fee_percent,omitempty" bson:"negotiation_fee_percent,omitempty"`   // % phí đàm phán (0–1)
	ExploitationFeePercent float64 `json:"exploitation_fee_percent,omitempty" bson:"exploitation_fee_percent,omitempty"` // % phí khai thác ủy quyền (0–1)

	// Storage-only
	StorageSizeGB float64 `json:"storage_size_gb,omitempty" bson:"storage_size_gb,omitempty"`

	// Legal advice turn-only / scan turn-only / strike turn-only
	Turns int `json:"turns,omitempty" bson:"turns,omitempty"`
}

// CollectionName returns the MongoDB collection name (unified for all package types).
func (Package) CollectionName() string {
	return "packages"
}

func (p *Package) BeforeCreate(ctx context.Context) error {
	now := timer.Now()
	p.CreatedAt = now
	p.UpdatedAt = now
	return nil
}

func (p *Package) BeforeUpdate(ctx context.Context) error {
	p.UpdatedAt = timer.Now()
	return nil
}

func (p *Package) CalculateTotalPrice() float64 {
	if p.TaxRate == 0 {
		return p.Price
	}
	return p.Price * (1 + p.TaxRate)
}

func (p *Package) CalculateTaxAmount() float64 {
	return p.Price * p.TaxRate
}
```

- [ ] **Step 1.3: Apply identical changes to marketplace-admin-api/schema/packagecol/enum.go and model.go**

The file contents are identical to Steps 1.1 and 1.2 — copy verbatim, only the import path differs if present in model.go. Both repos import `"api/internal/mongodb"` and `"api/internal/timer"`.

- [ ] **Step 1.4: Build both backends**

```bash
cd /Users/macos/workspace-user/marketplace-backend && go build ./...
cd /Users/macos/workspace-user/marketplace-admin-api && go build ./...
```

Expected: no errors.

- [ ] **Step 1.5: Commit marketplace-backend**

```bash
cd /Users/macos/workspace-user/marketplace-backend
git add schema/packagecol/enum.go schema/packagecol/model.go
git commit -m "$(cat <<'EOF'
chore(marketplace-backend): add DurationType constants, StrikeTurn, and pricing fields to Package schema

- Add PackageTypeStrikeTurn enum constant and MembershipDurationType type (monthly/yearly/custom)
- Add StrikeCount, UnlimitedStrike, LegalAdviceInitialTurn to membership fields
- Add per-tier pricing fields: PublicationBasePrice, PublicationSoftwareMinPrice/MaxPrice,
  ScanTurnAdditionalPrice, StrikeTurnAdditionalPrice
- Add commission fields: NegotiationFeePercent, ExploitationFeePercent
- Move DurationType comment to common section (used by both storage and membership)

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

- [ ] **Step 1.6: Commit marketplace-admin-api**

```bash
cd /Users/macos/workspace-user/marketplace-admin-api
git add schema/packagecol/enum.go schema/packagecol/model.go
git commit -m "$(cat <<'EOF'
chore(marketplace-admin-api): add DurationType constants, StrikeTurn, and pricing fields to Package schema

Same schema additions as marketplace-backend: PackageTypeStrikeTurn, MembershipDurationType,
StrikeCount, UnlimitedStrike, LegalAdviceInitialTurn, per-tier pricing fields, commission fields.

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 2: usercol Schema — Add IsCompany and StrikeTurn to User

**Files:**
- Modify: `marketplace-backend/schema/usercol/model.go`

- [ ] **Step 2.1: Add fields to usercol/model.go**

In [marketplace-backend/schema/usercol/model.go](marketplace-backend/schema/usercol/model.go), find the membership block (around line 92):

```go
	// Membership information
	IsMember              bool      `json:"is_member" bson:"is_member"`                             // Có phải là thành viên không
	MembershipPackageID   string    `json:"membership_package_id" bson:"membership_package_id"`     // ID của gói membership đang sử dụng
	MembershipPackageCode string    `json:"membership_package_code" bson:"membership_package_code"` // Code của gói (monthly, yearly)
	MembershipStartDate   time.Time `json:"membership_start_date" bson:"membership_start_date"`     // Ngày bắt đầu membership
	MembershipExpireDate  time.Time `json:"membership_expire_date" bson:"membership_expire_date"`   // Ngày hết hạn membership
```

Replace with:

```go
	// Membership information
	IsMember              bool      `json:"is_member" bson:"is_member"`                             // Có phải là thành viên không
	IsCompany             bool      `json:"is_company" bson:"is_company"`                           // Gói doanh nghiệp (copied from pkg.IsCompany on purchase; cleared on expiry)
	MembershipPackageID   string    `json:"membership_package_id" bson:"membership_package_id"`     // ID của gói membership đang sử dụng
	MembershipPackageCode string    `json:"membership_package_code" bson:"membership_package_code"` // Code của gói (monthly, yearly, custom...)
	MembershipStartDate   time.Time `json:"membership_start_date" bson:"membership_start_date"`     // Ngày bắt đầu membership
	MembershipExpireDate  time.Time `json:"membership_expire_date" bson:"membership_expire_date"`   // Ngày hết hạn membership
```

Also add after `CopyrightScanCount` (end of the struct, around line 128):

```go
	// Copyright scan count - số lượt quét vi phạm bản quyền
	CopyrightScanCount int `json:"copyright_scan_count" bson:"copyright_scan_count"`

	// StrikeTurn - số lượt yêu cầu đánh gậy (strike request) còn lại.
	// -1 = không giới hạn (dành cho gói doanh nghiệp, được set khi mua gói có UnlimitedStrike=true).
	// 0  = hết lượt.
	StrikeTurn int `json:"strike_turn" bson:"strike_turn"`
```

- [ ] **Step 2.2: Build**

```bash
cd /Users/macos/workspace-user/marketplace-backend && go build ./...
```

Expected: no errors.

- [ ] **Step 2.3: Commit**

```bash
cd /Users/macos/workspace-user/marketplace-backend
git add schema/usercol/model.go
git commit -m "$(cat <<'EOF'
chore(marketplace-backend): add is_company and strike_turn to User schema

- is_company: copied from pkg.IsCompany on membership purchase, cleared on expiry
- strike_turn: int with -1=unlimited sentinel for company plans

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: purchasecol Schema — Add Strike Turn Transaction Type and Metadata

**Files:**
- Modify: `marketplace-backend/schema/purchasecol/enum.go`
- Modify: `marketplace-backend/schema/purchasecol/metadata.go`

- [ ] **Step 3.1: Add TransactionTypeStrikeTurn to enum.go**

In [marketplace-backend/schema/purchasecol/enum.go](marketplace-backend/schema/purchasecol/enum.go), find the `TransactionTypeScanTurn` constant and add after it:

```go
	TransactionTypeScanTurn    TransactionType = "scan_turn"    // mua lượt quét vi phạm
	TransactionTypeStrikeTurn  TransactionType = "strike_turn"  // mua lượt đánh gậy (strike request)
```

Also add to `StringToTransactionType` switch:

```go
	case "scan_turn":
		return TransactionTypeScanTurn, nil
	case "strike_turn":
		return TransactionTypeStrikeTurn, nil
```

- [ ] **Step 3.2: Add StrikeTurnPaymentMetadata to metadata.go**

In [marketplace-backend/schema/purchasecol/metadata.go](marketplace-backend/schema/purchasecol/metadata.go), add after `ScanTurnPaymentMetadata`:

```go
// StrikeTurnPaymentMetadata: buying strike (đánh gậy) turn(s) via a package; tops up user.strike_turn.
// If user already has unlimited (-1), the purchased turns are added as a positive count (replacing -1 to allow deduction).
type StrikeTurnPaymentMetadata struct {
	PackageCode string `json:"package_code" bson:"package_code"`
	Turns       int    `json:"turns" bson:"turns"`
}
```

- [ ] **Step 3.3: Build**

```bash
cd /Users/macos/workspace-user/marketplace-backend && go build ./...
```

Expected: no errors.

- [ ] **Step 3.4: Commit**

```bash
cd /Users/macos/workspace-user/marketplace-backend
git add schema/purchasecol/enum.go schema/purchasecol/metadata.go
git commit -m "$(cat <<'EOF'
chore(marketplace-backend): add strike_turn transaction type and StrikeTurnPaymentMetadata

Follows the same pattern as scan_turn: TransactionTypeStrikeTurn enum constant +
StrikeTurnPaymentMetadata struct for payment fulfillment.

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: Update handle_membership.go — Set IsCompany, StrikeTurn, LegalAdviceInitialTurn

**Files:**
- Modify: `marketplace-backend/business/payment/vnpay/handle_membership.go`

- [ ] **Step 4.1: Replace handleMembershipPayment body**

Replace the full file [marketplace-backend/business/payment/vnpay/handle_membership.go](marketplace-backend/business/payment/vnpay/handle_membership.go):

```go
package vnpay

import (
	"context"
	"errors"
	"fmt"
	"time"

	"api/internal/plog"
	"api/schema/affiliatecommissioncol"
	"api/schema/packagecol"
	"api/schema/purchasecol"
	"api/schema/usercol"
	"api/services/affiliate"

	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/bson/primitive"
)

// handleMembershipPayment processes membership package payment after VNPay confirmation
func handleMembershipPayment(ctx context.Context, paymentItem *purchasecol.Payment) error {
	logger := plog.NewBizLogger("[business][payment][vnpay][membership]")

	metadata := purchasecol.GetPaymentMetadata[purchasecol.MembershipPaymentMetadata](*paymentItem)
	if metadata.PackageId == "" {
		return errors.New("membership package metadata missing")
	}

	// Try to find package by code first, then fallback to ID for backward compatibility
	pkg, err := packagecol.FindByTypeAndCode(ctx, packagecol.PackageTypeMembership, metadata.PackageId)
	if err != nil {
		pkg, err = packagecol.FindByTypeAndID(ctx, packagecol.PackageTypeMembership, metadata.PackageId)
		if err != nil {
			return fmt.Errorf("membership package not found: %v", err)
		}
	}

	userObjID, err := primitive.ObjectIDFromHex(paymentItem.UserId)
	if err != nil {
		return fmt.Errorf("invalid user ID: %v", err)
	}

	now := time.Now()
	expireDate := now
	if pkg.DurationMonths > 0 || pkg.DurationDays > 0 {
		expireDate = expireDate.AddDate(0, pkg.DurationMonths, pkg.DurationDays)
	}

	// Load current user to compute legal advice turns (additive reward per purchase)
	user, loadErr := usercol.FindWithUserID(ctx, paymentItem.UserId)

	// Legal advice turns: add pkg.LegalAdviceInitialTurn (default 1 for backward compat)
	legalAdviceAdd := pkg.LegalAdviceInitialTurn
	if legalAdviceAdd <= 0 {
		legalAdviceAdd = 1
	}
	currentLegalAdviceTurn := 0
	if loadErr == nil && user.LegalAdviceTurn >= 0 {
		currentLegalAdviceTurn = user.LegalAdviceTurn
	}

	// Strike turns: replace with package's quota; -1 for unlimited (company plans)
	var strikeTurn int
	if pkg.UnlimitedStrike {
		strikeTurn = -1
	} else {
		strikeTurn = pkg.StrikeCount
	}

	updateData := bson.M{
		"is_member":                        true,
		"is_company":                       pkg.IsCompany,
		"membership_package_id":            pkg.GetIDString(),
		"membership_package_code":          pkg.Code,
		"membership_start_date":            now,
		"membership_expire_date":           expireDate,
		"remaining_product_creation_count": -1,
		"copyright_scan_count":             pkg.ScanViolationCount,
		"storage_limit_gb":                 float64(pkg.StorageCapacity),
		"legal_advice_turn":                currentLegalAdviceTurn + legalAdviceAdd,
		"strike_turn":                      strikeTurn,
	}

	_, err = usercol.UpdateByID(ctx, userObjID, updateData)
	if err != nil {
		logger.Err(err).
			Str("user_id", paymentItem.UserId).
			Msg("failed to update membership info")
		return fmt.Errorf("failed to update membership info: %v", err)
	}

	logger.Info().
		Str("user_id", paymentItem.UserId).
		Str("package_code", pkg.Code).
		Str("expires_at", expireDate.Format(time.RFC3339)).
		Bool("is_company", pkg.IsCompany).
		Int("strike_turn", strikeTurn).
		Msg("membership activated successfully")

	actor, actorErr := usercol.FindWithUserID(ctx, paymentItem.UserId)
	if actorErr == nil {
		tryAwardAffiliateCommission(ctx, paymentItem, actor, affiliatecommissioncol.ActionMembership, affiliate.CommissionMeta{
			ProductID:    pkg.GetIDString(),
			ProductTitle: fmt.Sprintf("Gói thành viên (%s)", pkg.Code),
			Notes:        fmt.Sprintf("membership_package_code=%s", pkg.Code),
		})
	}
	return nil
}
```

- [ ] **Step 4.2: Build**

```bash
cd /Users/macos/workspace-user/marketplace-backend && go build ./...
```

Expected: no errors.

- [ ] **Step 4.3: Commit**

```bash
cd /Users/macos/workspace-user/marketplace-backend
git add business/payment/vnpay/handle_membership.go
git commit -m "$(cat <<'EOF'
feat(marketplace-backend): set is_company, strike_turn, and configurable legal_advice_turn on membership purchase

- is_company: copied from pkg.IsCompany
- strike_turn: -1 if pkg.UnlimitedStrike, else pkg.StrikeCount (replaces previous value)
- legal_advice_turn: additive, using pkg.LegalAdviceInitialTurn (defaults to 1 if unconfigured)

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: Add Strike Turn Payment Flow (backend)

**Files:**
- Create: `marketplace-backend/business/payment/vnpay/handle_strike_turn.go`
- Modify: `marketplace-backend/business/payment/vnpay/vnpay_create_link.go`
- Modify: `marketplace-backend/business/payment/vnpay/vnpay_ipn.go`
- Modify: `marketplace-backend/services/cronjob/membership_storage_cron.go`

- [ ] **Step 5.1: Create handle_strike_turn.go**

Create new file [marketplace-backend/business/payment/vnpay/handle_strike_turn.go](marketplace-backend/business/payment/vnpay/handle_strike_turn.go):

```go
package vnpay

import (
	"context"
	"fmt"
	"time"

	"api/internal/plog"
	"api/schema/affiliatecommissioncol"
	"api/schema/purchasecol"
	"api/schema/usercol"
	"api/services/affiliate"

	"go.mongodb.org/mongo-driver/bson"
)

// handleStrikeTurnPayment processes strike (đánh gậy) turn purchase after VNPay confirmation.
// Adds the package's turns to user.strike_turn.
// Special case: if user currently has -1 (unlimited), the purchase adds positive turns
// (replacing -1 so deduction is possible).
func handleStrikeTurnPayment(ctx context.Context, paymentItem *purchasecol.Payment) error {
	logger := plog.NewBizLogger("[business][payment][vnpay][strike_turn]")

	metadata := purchasecol.GetPaymentMetadata[purchasecol.StrikeTurnPaymentMetadata](*paymentItem)
	turns := metadata.Turns
	if turns <= 0 {
		turns = 1
	}

	user, err := usercol.FindWithUserID(ctx, paymentItem.UserId)
	if err != nil {
		return err
	}

	var newCount int
	if user.StrikeTurn < 0 {
		// Was unlimited: purchasing turns gives them the exact purchased amount (no longer unlimited)
		newCount = turns
	} else {
		newCount = user.StrikeTurn + turns
	}

	if err := usercol.UpdateFields(ctx, user.GetIDString(), bson.M{
		"strike_turn": newCount,
		"updated_at":  time.Now(),
	}); err != nil {
		logger.Err(err).Str("user_id", paymentItem.UserId).Msg("failed to update strike_turn")
		return err
	}

	logger.Info().
		Str("user_id", paymentItem.UserId).
		Int("added_turns", turns).
		Int("new_strike_turn", newCount).
		Msg("strike turn payment applied")

	tryAwardAffiliateCommission(ctx, paymentItem, user, affiliatecommissioncol.ActionScanTurn, affiliate.CommissionMeta{
		ProductTitle: "Mua lượt đánh gậy",
		Notes:        fmt.Sprintf("turns=%d", turns),
	})
	return nil
}
```

- [ ] **Step 5.2: Add doCreatePaymentLinkForStrikeTurn to vnpay_create_link.go**

In [marketplace-backend/business/payment/vnpay/vnpay_create_link.go](marketplace-backend/business/payment/vnpay/vnpay_create_link.go), find the `switch req.Type` dispatcher block and add:

```go
	case purchasecol.TransactionTypeScanTurn:
		return doCreatePaymentLinkForScanTurn(ctx, ip, userId, req, st)
	case purchasecol.TransactionTypeStrikeTurn:
		return doCreatePaymentLinkForStrikeTurn(ctx, ip, userId, req, st)
```

Then add the new function at the end of the file (after `doCreatePaymentLinkForScanTurn`):

```go
// doCreatePaymentLinkForStrikeTurn creates payment link for buying strike (đánh gậy) turn(s) via an admin-configured package.
func doCreatePaymentLinkForStrikeTurn(ctx context.Context, ip, userId string, req CreatePaymentLinkReq, st partnerdiscount.State) (*CreatePaymentLinkRes, error) {
	transactionId := utils.GenerateCode(12)

	pkg, err := packagecol.FindByTypeAndCode(ctx, packagecol.PackageTypeStrikeTurn, req.Id)
	if err != nil {
		pkg, err = packagecol.FindByTypeAndID(ctx, packagecol.PackageTypeStrikeTurn, req.Id)
		if err != nil {
			return nil, errors.New("strike turn package not found")
		}
	}

	if !pkg.IsActive {
		return nil, errors.New("strike turn package is not active")
	}

	originalPreVAT := pkg.Price
	preTaxAfterCode := originalPreVAT

	var discountAmount float64 = 0
	var discountCode *discountcodecol.DiscountCode
	if req.DiscountCode != nil && *req.DiscountCode != "" {
		discountCode, err = discountcodecol.FindValidByCodeAndScope(ctx, strings.ToUpper(*req.DiscountCode), discountcodecol.ServiceScopeStrikeTurn)
		if err == nil {
			if discountCode.UsagePerUser > 0 {
				usageCount, countErr := purchasecol.CountDiscountUsageByUser(ctx, userId, strings.ToUpper(*req.DiscountCode))
				if countErr == nil && usageCount >= int64(discountCode.UsagePerUser) {
					return nil, errors.New("DISCOUNT_CODE_USAGE_LIMIT_REACHED")
				}
			}
			discountAmount = discountCode.CalculateDiscount(pkg.Price)
			preTaxAfterCode = originalPreVAT - discountAmount
			if preTaxAfterCode < 0 {
				preTaxAfterCode = 0
			}
		}
	}

	preTaxFinal, partnerDisc := partnerdiscount.AdjustPreTaxAfterCode(originalPreVAT, preTaxAfterCode, st)
	totalAmount := preTaxFinal * 1.08

	link, err := vnpay.GeneratePaymentURL(
		transactionId,
		req.RedirectURL,
		int64(totalAmount),
		ip,
	)
	if err != nil {
		return nil, err
	}

	metadata, err := purchasecol.ConvertMetadataToJSON(purchasecol.StrikeTurnPaymentMetadata{
		PackageCode: pkg.Code,
		Turns:       pkg.Turns,
	})
	if err != nil {
		return nil, err
	}

	message := fmt.Sprintf("Thanh toán %s", pkg.Name)
	priceBeforeVAT := preTaxFinal
	vatAmount := totalAmount - priceBeforeVAT

	paymentItem := purchasecol.Payment{
		CreatedAt:            time.Now(),
		UpdatedAt:            time.Now(),
		Method:               purchasecol.PaymentMethodVNPay,
		Status:               purchasecol.PaymentStatusPending,
		TransactionType:      purchasecol.TransactionTypeStrikeTurn,
		TransactionId:        transactionId,
		Metadata:             metadata,
		BankId:               "",
		Link:                 link,
		Message:              message,
		VAT:                  vatAmount,
		TotalAmountBeforeTax: priceBeforeVAT,
		TotalFee:             totalAmount,
		Symbol:               "VND",
		UserId:               userId,
	}
	if discountCode != nil && discountAmount > 0 {
		discountCodeStr := strings.ToUpper(*req.DiscountCode)
		paymentItem.DiscountCode = &discountCodeStr
		paymentItem.DiscountAmount = discountAmount
	}
	if partnerDisc > 0 {
		paymentItem.PartnerReferralDiscountAmount = partnerDisc
	}

	if _, err = purchasecol.Create(ctx, &paymentItem); err != nil {
		return nil, err
	}

	res := &CreatePaymentLinkRes{
		URL:           link,
		TransactionId: transactionId,
		TMNCode:       vnpay.GetClient().TmnCode,
		IsSandbox:     vnpay.GetClient().IsSandbox,
	}
	return res, nil
}
```

> **Note on discountcodecol.ServiceScopeStrikeTurn**: Check [marketplace-backend/schema/discountcodecol/](marketplace-backend/schema/discountcodecol/) for the `ServiceScope` constants. If `ServiceScopeStrikeTurn` does not exist yet, add it following the same pattern as `ServiceScopeScanTurn`. If the discount scope feature is not needed for strike turns initially, pass `discountcodecol.ServiceScopeScanTurn` as a temporary placeholder and add the proper constant in the same commit.

- [ ] **Step 5.3: Add TransactionTypeStrikeTurn case to vnpay_ipn.go**

In [marketplace-backend/business/payment/vnpay/vnpay_ipn.go](marketplace-backend/business/payment/vnpay/vnpay_ipn.go), find the `case purchasecol.TransactionTypeScanTurn:` block (around line 408) and add after it:

```go
	case purchasecol.TransactionTypeStrikeTurn:
		logger.Info().
			Str("payment_id", paymentItem.GetIDString()).
			Str("user_id", paymentItem.UserId).
			Str("response_code", data.VnpResponseCode).
			Msg("processing strike turn payment")

		if data.VnpResponseCode != "00" {
			paymentItem.Status = purchasecol.PaymentStatusFailed
			paymentItem.TransactionIdRef = data.VnpTransactionNo
		} else {
			paymentItem.Status = purchasecol.PaymentStatusPaid
			paymentItem.BankId = data.VnpBankCode
			paymentItem.TransactionIdRef = data.VnpTransactionNo
			paymentItem.PaymentAt = time.Now()

			if err := handleStrikeTurnPayment(ctx, paymentItem); err != nil {
				logger.Err(err).
					Str("payment_id", paymentItem.GetIDString()).
					Str("user_id", paymentItem.UserId).
					Msg("failed to add strike turns")
				return &VNPayIPNRes{
					Message: "System Error: " + err.Error(),
					RspCode: "99",
				}, nil
			}

			go sendPaymentSuccessEmail(ctx, paymentItem, "Lượt đánh gậy", "Thanh toán lượt yêu cầu đánh gậy")
			go sendPaymentTelegramNotification(ctx, paymentItem, "Lượt đánh gậy")
		}
```

- [ ] **Step 5.4: Reset strike_turn and is_company on membership expiry (cronjob)**

In [marketplace-backend/services/cronjob/membership_storage_cron.go](marketplace-backend/services/cronjob/membership_storage_cron.go), find the `set` bson.M inside `sweepLapsedMembership` and add the new fields:

```go
	set := bson.M{
		"is_member":               false,
		"is_company":              false,
		"membership_package_id":   "",
		"membership_package_code": "",
		"membership_start_date":   time.Time{},
		"membership_expire_date":  time.Time{},
		"storage_limit_gb":        document.DefaultStorageLimitGB,
		"strike_turn":             0,
		"updated_at":              timer.Now(),
	}
```

- [ ] **Step 5.5: Build**

```bash
cd /Users/macos/workspace-user/marketplace-backend && go build ./...
```

Expected: no errors. If `discountcodecol.ServiceScopeStrikeTurn` is missing, add the constant to the discountcodecol enum file before building.

- [ ] **Step 5.6: Commit**

```bash
cd /Users/macos/workspace-user/marketplace-backend
git add \
  business/payment/vnpay/handle_strike_turn.go \
  business/payment/vnpay/vnpay_create_link.go \
  business/payment/vnpay/vnpay_ipn.go \
  services/cronjob/membership_storage_cron.go
git commit -m "$(cat <<'EOF'
feat(marketplace-backend): add strike turn payment flow and expiry cleanup

- New handleStrikeTurnPayment: increments user.strike_turn (replaces -1 if was unlimited)
- New doCreatePaymentLinkForStrikeTurn: mirrors scan turn payment link creation
- vnpay_ipn.go: add TransactionTypeStrikeTurn case
- sweepLapsedMembership: reset strike_turn=0 and is_company=false on membership expiry

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 6: Admin API — Update Package CRUD with New Fields

**Files:**
- Modify: `marketplace-admin-api/business/package/create.go`
- Modify: `marketplace-admin-api/business/package/update.go`
- Modify: `marketplace-admin-api/business/package/list.go`
- Modify: `marketplace-admin-api/business/package/detail.go`

- [ ] **Step 6.1: Update create.go — add new fields to request struct and Package construction**

In [marketplace-admin-api/business/package/create.go](marketplace-admin-api/business/package/create.go), replace `CreateRequestData` and the `pkg` construction:

```go
type CreateRequestData struct {
	Code                     string   `json:"code" binding:"required"`
	Name                     string   `json:"name" binding:"required"`
	DisplayName              string   `json:"display_name" binding:"required"`
	Description              string   `json:"description"`
	Price                    float64  `json:"price" binding:"required"`
	TaxRate                  float64  `json:"tax_rate"`
	Currency                 string   `json:"currency"`
	DurationDays             int      `json:"duration_days"`
	DurationMonths           int      `json:"duration_months"`
	DurationType             string   `json:"duration_type"`           // "monthly"|"yearly"|"custom"
	Benefits                 []string `json:"benefits"`
	UnlimitedProductCreation bool     `json:"unlimited_product_creation"`
	ProductCreationCount     int      `json:"product_creation_count"`
	ScanViolationCount       int      `json:"scan_violation_count"`
	StorageCapacity          int      `json:"storage_capacity"`
	IsActive                 bool     `json:"is_active"`
	IsPopular                bool     `json:"is_popular"`
	IsCompany                bool     `json:"is_company"`
	IAPAppleCode             string   `json:"iap_apple_code"`
	// Strike
	StrikeCount     int  `json:"strike_count"`
	UnlimitedStrike bool `json:"unlimited_strike"`
	// Legal advice reward
	LegalAdviceInitialTurn int `json:"legal_advice_initial_turn"`
	// Per-tier pricing
	PublicationBasePrice        float64 `json:"publication_base_price"`
	PublicationSoftwareMinPrice float64 `json:"publication_software_min_price"`
	PublicationSoftwareMaxPrice float64 `json:"publication_software_max_price"`
	ScanTurnAdditionalPrice     float64 `json:"scan_turn_additional_price"`
	StrikeTurnAdditionalPrice   float64 `json:"strike_turn_additional_price"`
	// Commission
	NegotiationFeePercent  float64 `json:"negotiation_fee_percent"`
	ExploitationFeePercent float64 `json:"exploitation_fee_percent"`
}
```

And the `pkg` construction inside `Create()`:

```go
		pkg := &packagecol.Package{
			Type:                     packagecol.PackageTypeMembership,
			Code:                     req.Code,
			Name:                     req.Name,
			DisplayName:              req.DisplayName,
			Description:              req.Description,
			Price:                    req.Price,
			TaxRate:                  req.TaxRate,
			Currency:                 req.Currency,
			DurationDays:             req.DurationDays,
			DurationMonths:           req.DurationMonths,
			DurationType:             req.DurationType,
			Benefits:                 req.Benefits,
			UnlimitedProductCreation: req.UnlimitedProductCreation,
			ProductCreationCount:     req.ProductCreationCount,
			ScanViolationCount:       req.ScanViolationCount,
			StorageCapacity:          req.StorageCapacity,
			IsActive:                 req.IsActive,
			IsPopular:                req.IsPopular,
			IsCompany:                req.IsCompany,
			IAPAppleCode:             req.IAPAppleCode,
			StrikeCount:              req.StrikeCount,
			UnlimitedStrike:          req.UnlimitedStrike,
			LegalAdviceInitialTurn:   req.LegalAdviceInitialTurn,
			PublicationBasePrice:        req.PublicationBasePrice,
			PublicationSoftwareMinPrice: req.PublicationSoftwareMinPrice,
			PublicationSoftwareMaxPrice: req.PublicationSoftwareMaxPrice,
			ScanTurnAdditionalPrice:     req.ScanTurnAdditionalPrice,
			StrikeTurnAdditionalPrice:   req.StrikeTurnAdditionalPrice,
			NegotiationFeePercent:       req.NegotiationFeePercent,
			ExploitationFeePercent:      req.ExploitationFeePercent,
		}
```

Also remove the `binding:"required"` from `DurationDays` and `DurationMonths` (custom packages may only have one set).

- [ ] **Step 6.2: Update update.go — add new fields**

In [marketplace-admin-api/business/package/update.go](marketplace-admin-api/business/package/update.go), add all new fields to the update request struct and the update body. Follow the same pointer-optional pattern already used for existing fields.

Fields to add to `UpdateRequestData`:

```go
	DurationType             *string  `json:"duration_type"`
	StrikeCount              *int     `json:"strike_count"`
	UnlimitedStrike          *bool    `json:"unlimited_strike"`
	LegalAdviceInitialTurn   *int     `json:"legal_advice_initial_turn"`
	PublicationBasePrice        *float64 `json:"publication_base_price"`
	PublicationSoftwareMinPrice *float64 `json:"publication_software_min_price"`
	PublicationSoftwareMaxPrice *float64 `json:"publication_software_max_price"`
	ScanTurnAdditionalPrice     *float64 `json:"scan_turn_additional_price"`
	StrikeTurnAdditionalPrice   *float64 `json:"strike_turn_additional_price"`
	NegotiationFeePercent       *float64 `json:"negotiation_fee_percent"`
	ExploitationFeePercent      *float64 `json:"exploitation_fee_percent"`
```

And corresponding `if req.X != nil { pkg.X = *req.X }` blocks in the handler.

- [ ] **Step 6.3: Update list.go and detail.go — add new fields to response structs**

In [marketplace-admin-api/business/package/list.go](marketplace-admin-api/business/package/list.go), add to `PackageResponseData`:

```go
	DurationType                string  `json:"duration_type"`
	StrikeCount                 int     `json:"strike_count"`
	UnlimitedStrike             bool    `json:"unlimited_strike"`
	LegalAdviceInitialTurn      int     `json:"legal_advice_initial_turn"`
	PublicationBasePrice        float64 `json:"publication_base_price"`
	PublicationSoftwareMinPrice float64 `json:"publication_software_min_price"`
	PublicationSoftwareMaxPrice float64 `json:"publication_software_max_price"`
	ScanTurnAdditionalPrice     float64 `json:"scan_turn_additional_price"`
	StrikeTurnAdditionalPrice   float64 `json:"strike_turn_additional_price"`
	NegotiationFeePercent       float64 `json:"negotiation_fee_percent"`
	ExploitationFeePercent      float64 `json:"exploitation_fee_percent"`
```

And map each field in the response mapping loop:

```go
	DurationType:                pkg.DurationType,
	StrikeCount:                 pkg.StrikeCount,
	UnlimitedStrike:             pkg.UnlimitedStrike,
	LegalAdviceInitialTurn:      pkg.LegalAdviceInitialTurn,
	PublicationBasePrice:        pkg.PublicationBasePrice,
	PublicationSoftwareMinPrice: pkg.PublicationSoftwareMinPrice,
	PublicationSoftwareMaxPrice: pkg.PublicationSoftwareMaxPrice,
	ScanTurnAdditionalPrice:     pkg.ScanTurnAdditionalPrice,
	StrikeTurnAdditionalPrice:   pkg.StrikeTurnAdditionalPrice,
	NegotiationFeePercent:       pkg.NegotiationFeePercent,
	ExploitationFeePercent:      pkg.ExploitationFeePercent,
```

Apply the same additions to `detail.go`'s response struct and mapping (detail uses a separate `PackageDetailResponseData` struct — check the file and mirror the changes).

- [ ] **Step 6.4: Build**

```bash
cd /Users/macos/workspace-user/marketplace-admin-api && go build ./...
```

Expected: no errors.

- [ ] **Step 6.5: Commit**

```bash
cd /Users/macos/workspace-user/marketplace-admin-api
git add \
  business/package/create.go \
  business/package/update.go \
  business/package/list.go \
  business/package/detail.go
git commit -m "$(cat <<'EOF'
feat(marketplace-admin-api): add duration_type, strike, pricing, and commission fields to membership package CRUD

- create: add DurationType, StrikeCount, UnlimitedStrike, LegalAdviceInitialTurn, PublicationBasePrice,
  PublicationSoftwareMinPrice/MaxPrice, ScanTurnAdditionalPrice, StrikeTurnAdditionalPrice,
  NegotiationFeePercent, ExploitationFeePercent
- update: same fields as optional pointers
- list/detail: expose all new fields in response

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 7: Admin API — New Strike Turn Package Domain

**Files:**
- Create: `marketplace-admin-api/business/striketurnpackage/router.go`
- Create: `marketplace-admin-api/business/striketurnpackage/create.go`
- Create: `marketplace-admin-api/business/striketurnpackage/update.go`
- Create: `marketplace-admin-api/business/striketurnpackage/list.go`
- Create: `marketplace-admin-api/business/striketurnpackage/delete.go`
- Modify: `marketplace-admin-api/business/role/keys.go`
- Modify: `marketplace-admin-api/routers/ver1.go`

- [ ] **Step 7.1: Create router.go**

```go
package striketurnpackage

import "github.com/gin-gonic/gin"

func Router(r *gin.RouterGroup) {
	r.GET("/list", List())
	r.POST("/create", Create())
	r.POST("/update/:id", Update())
	r.DELETE("/:id", Delete())
}
```

- [ ] **Step 7.2: Create create.go**

```go
package striketurnpackage

import (
	packagepkg "api/business/package"
	"api/internal/plog"
	"api/internal/rabbitmq"
	"api/internal/rabbitmq/events"
	"api/internal/response"
	"api/schema/packagecol"
	"net/http"
	"strings"

	"github.com/gin-gonic/gin"
)

type CreateReq struct {
	Code        string  `json:"code" binding:"required"`
	Name        string  `json:"name" binding:"required"`
	Description string  `json:"description"`
	Turns       int     `json:"turns" binding:"required,min=1"`
	Price       float64 `json:"price" binding:"required,min=0"`
	Currency    string  `json:"currency"`
	IsActive    bool    `json:"is_active"`
}

func Create() gin.HandlerFunc {
	return func(c *gin.Context) {
		var req CreateReq
		if err := c.ShouldBindJSON(&req); err != nil {
			code := response.ErrorResponse(err.Error())
			c.JSON(code.Code, code)
			return
		}
		codeStr := packagecol.NormalizeCode(req.Code)
		if codeStr == "" {
			code := response.ErrorResponse("code is required")
			c.JSON(code.Code, code)
			return
		}
		if req.Currency == "" {
			req.Currency = "VND"
		}

		ctx := c.Request.Context()
		existing, _ := packagecol.FindByTypeAndCode(ctx, packagecol.PackageTypeStrikeTurn, codeStr)
		if existing != nil {
			code := response.ErrorResponse("package with this code already exists")
			c.JSON(code.Code, code)
			return
		}

		pkg := &packagecol.Package{
			Type:        packagecol.PackageTypeStrikeTurn,
			Code:        codeStr,
			Name:        strings.TrimSpace(req.Name),
			Description: strings.TrimSpace(req.Description),
			Turns:       req.Turns,
			Price:       req.Price,
			Currency:    req.Currency,
			IsActive:    req.IsActive,
		}
		created, err := packagecol.Create(ctx, pkg)
		if err != nil {
			code := response.ErrorResponse(err.Error())
			c.JSON(code.Code, code)
			return
		}
		packagepkg.SavePackageHistory(ctx, c, created, "create", nil)
		if err := rabbitmq.PublishEvent(c.Request.Context(), rabbitmq.EventPackageCreated, events.PackageEventPayload(created)); err != nil {
			plog.NewBizLogger("striketurnpackage.create").Err(err).Msg("rabbitmq publish package.created failed")
		}
		c.JSON(http.StatusOK, response.SuccessResponse(gin.H{"id": created.GetIDString()}))
	}
}
```

- [ ] **Step 7.3: Create update.go**

Mirror `marketplace-admin-api/business/scanturnpackage/update.go` exactly, replacing every `PackageTypeScanTurn` with `PackageTypeStrikeTurn` and every `"scanturnpackage"` logger string with `"striketurnpackage"`.

Read [marketplace-admin-api/business/scanturnpackage/update.go](marketplace-admin-api/business/scanturnpackage/update.go) and copy it verbatim with those substitutions.

- [ ] **Step 7.4: Create list.go**

Mirror `marketplace-admin-api/business/scanturnpackage/list.go`, replacing `PackageTypeScanTurn` → `PackageTypeStrikeTurn`.

- [ ] **Step 7.5: Create delete.go**

Mirror `marketplace-admin-api/business/scanturnpackage/delete.go`, replacing `PackageTypeScanTurn` → `PackageTypeStrikeTurn` and logger strings accordingly.

- [ ] **Step 7.6: Add router keys to role/keys.go**

In [marketplace-admin-api/business/role/keys.go](marketplace-admin-api/business/role/keys.go), find the scan turn package keys (or package keys section) and add:

```go
// Strike Turn Package
const (
	KeyStrikeTurnPackageList   = "strike-turn-package.list"
	KeyStrikeTurnPackageCreate = "strike-turn-package.create"
	KeyStrikeTurnPackageUpdate = "strike-turn-package.update"
	KeyStrikeTurnPackageDelete = "strike-turn-package.delete"
)
```

- [ ] **Step 7.7: Register route in routers/ver1.go**

In [marketplace-admin-api/routers/ver1.go](marketplace-admin-api/routers/ver1.go), add the import and route group after the `scanTurnPkgRouter`:

Add to imports:
```go
	"api/business/striketurnpackage"
```

Add after `scanturnpackage.Router(scanTurnPkgRouter)`:
```go
	strikeTurnPkgRouter := authGroup.Group("strike-turn-packages")
	striketurnpackage.Router(strikeTurnPkgRouter)
```

- [ ] **Step 7.8: Build**

```bash
cd /Users/macos/workspace-user/marketplace-admin-api && go build ./...
```

Expected: no errors.

- [ ] **Step 7.9: Commit**

```bash
cd /Users/macos/workspace-user/marketplace-admin-api
git add \
  business/striketurnpackage/router.go \
  business/striketurnpackage/create.go \
  business/striketurnpackage/update.go \
  business/striketurnpackage/list.go \
  business/striketurnpackage/delete.go \
  business/role/keys.go \
  routers/ver1.go
git commit -m "$(cat <<'EOF'
feat(marketplace-admin-api): add strike turn package management domain

New business/striketurnpackage domain (CRUD) following scanturnpackage pattern.
Routes: GET /strike-turn-packages/list, POST /create, POST /update/:id, DELETE /:id.
Router keys: strike-turn-package.list/create/update/delete added to role/keys.go.

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 8: Admin UI — Update API Types and PackageForm

**Files:**
- Modify: `marketplace-admin/app/api/package.ts`
- Modify: `marketplace-admin/components/dashboard/membership-management/PackageForm.tsx`

- [ ] **Step 8.1: Update app/api/package.ts — add new fields to all interfaces**

In [marketplace-admin/app/api/package.ts](marketplace-admin/app/api/package.ts), add to `PackageData`:

```typescript
  duration_type?: string;           // "monthly" | "yearly" | "custom"
  strike_count?: number;
  unlimited_strike?: boolean;
  legal_advice_initial_turn?: number;
  publication_base_price?: number;
  publication_software_min_price?: number;
  publication_software_max_price?: number;
  scan_turn_additional_price?: number;
  strike_turn_additional_price?: number;
  negotiation_fee_percent?: number;
  exploitation_fee_percent?: number;
```

Add the same fields to `CreatePackageData` (non-optional where sensible, with `?` for optional pricing):

```typescript
  duration_type?: string;
  strike_count?: number;
  unlimited_strike?: boolean;
  legal_advice_initial_turn?: number;
  publication_base_price?: number;
  publication_software_min_price?: number;
  publication_software_max_price?: number;
  scan_turn_additional_price?: number;
  strike_turn_additional_price?: number;
  negotiation_fee_percent?: number;
  exploitation_fee_percent?: number;
```

Add same optional fields to `UpdatePackageData`.

- [ ] **Step 8.2: Redesign PackageForm.tsx**

Replace the full file [marketplace-admin/components/dashboard/membership-management/PackageForm.tsx](marketplace-admin/components/dashboard/membership-management/PackageForm.tsx) with the redesigned version below. The form is organized into 4 collapsible/sectioned areas: **Thông tin cơ bản**, **Quota & Giới hạn**, **Cấu hình giá dịch vụ**, **Phí hoa hồng**.

Key behavioral changes:
1. Add `duration_type` Select with options: "monthly" (Gói tháng) / "yearly" (Gói năm) / "custom" (Tuỳ chọn - doanh nghiệp).
2. When `duration_type = "custom"`, show both `duration_months` + `duration_days` inputs (free form).
3. When `duration_type = "monthly"`, show only `duration_months` (lock to 1 by default).
4. When `duration_type = "yearly"`, show only `duration_months` (lock to 12 by default).
5. Fix duration validation: if `duration_type = "custom"`, require at least one of `duration_days > 0` OR `duration_months > 0`. Others require `duration_months > 0`.
6. When `unlimited_strike = true`, hide `strike_count` input.
7. When `is_company = false`, grey out / disable `unlimited_strike`.

```tsx
'use client';

import { useState, useEffect } from 'react';
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { packageApi, PackageData, CreatePackageData, UpdatePackageData } from '@/app/api/package';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Switch } from '@/components/ui/switch';
import { Textarea } from '@/components/ui/textarea';
import { Badge } from '@/components/ui/badge';
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select';
import { toast } from 'sonner';
import { X, Plus } from 'lucide-react';

interface PackageFormProps {
  initialData?: PackageData;
  onSuccess: () => void;
  onCancel: () => void;
}

const DURATION_TYPE_OPTIONS = [
  { value: 'monthly', label: 'Gói tháng (monthly)' },
  { value: 'yearly', label: 'Gói năm (yearly)' },
  { value: 'custom', label: 'Tuỳ chọn — doanh nghiệp (custom)' },
];

const defaultFormData: CreatePackageData = {
  code: '',
  name: '',
  display_name: '',
  description: '',
  price: 0,
  tax_rate: 0.1,
  currency: 'VND',
  duration_type: 'monthly',
  duration_days: 0,
  duration_months: 1,
  benefits: [],
  unlimited_product_creation: false,
  product_creation_count: 0,
  scan_violation_count: 0,
  storage_capacity: 0,
  is_active: true,
  is_popular: false,
  is_company: false,
  iap_apple_code: '',
  strike_count: 0,
  unlimited_strike: false,
  legal_advice_initial_turn: 1,
  publication_base_price: 0,
  publication_software_min_price: 0,
  publication_software_max_price: 0,
  scan_turn_additional_price: 0,
  strike_turn_additional_price: 0,
  negotiation_fee_percent: 0,
  exploitation_fee_percent: 0,
};

export default function PackageForm({ initialData, onSuccess, onCancel }: PackageFormProps) {
  const isEdit = !!initialData;
  const queryClient = useQueryClient();

  const [formData, setFormData] = useState<CreatePackageData>(defaultFormData);
  const [newBenefit, setNewBenefit] = useState('');

  useEffect(() => {
    if (initialData) {
      setFormData({
        ...defaultFormData,
        code: initialData.code || '',
        name: initialData.name || '',
        display_name: initialData.display_name || '',
        description: initialData.description || '',
        price: initialData.price || 0,
        tax_rate: initialData.tax_rate ?? 0.1,
        currency: initialData.currency || 'VND',
        duration_type: initialData.duration_type || 'monthly',
        duration_days: initialData.duration_days || 0,
        duration_months: initialData.duration_months || 1,
        benefits: initialData.benefits || [],
        unlimited_product_creation: initialData.unlimited_product_creation ?? false,
        product_creation_count: initialData.product_creation_count || 0,
        scan_violation_count: initialData.scan_violation_count || 0,
        storage_capacity: initialData.storage_capacity || 0,
        is_active: initialData.is_active ?? true,
        is_popular: initialData.is_popular ?? false,
        is_company: initialData.is_company ?? false,
        iap_apple_code: initialData.iap_apple_code || '',
        strike_count: initialData.strike_count || 0,
        unlimited_strike: initialData.unlimited_strike ?? false,
        legal_advice_initial_turn: initialData.legal_advice_initial_turn ?? 1,
        publication_base_price: initialData.publication_base_price || 0,
        publication_software_min_price: initialData.publication_software_min_price || 0,
        publication_software_max_price: initialData.publication_software_max_price || 0,
        scan_turn_additional_price: initialData.scan_turn_additional_price || 0,
        strike_turn_additional_price: initialData.strike_turn_additional_price || 0,
        negotiation_fee_percent: initialData.negotiation_fee_percent || 0,
        exploitation_fee_percent: initialData.exploitation_fee_percent || 0,
      });
    }
  }, [initialData]);

  const createMutation = useMutation({
    mutationFn: (data: CreatePackageData) => packageApi.create(data),
    onSuccess: () => {
      toast.success('Gói thành viên đã được tạo thành công');
      queryClient.invalidateQueries({ queryKey: ['packages'] });
      onSuccess();
    },
    onError: (error: any) => {
      toast.error('Lỗi khi tạo gói: ' + (error.response?.data?.message || error.message));
    },
  });

  const updateMutation = useMutation({
    mutationFn: ({ id, data }: { id: string; data: UpdatePackageData }) =>
      packageApi.update(id, data),
    onSuccess: () => {
      toast.success('Gói thành viên đã được cập nhật thành công');
      queryClient.invalidateQueries({ queryKey: ['packages'] });
      queryClient.invalidateQueries({ queryKey: ['package', initialData?.id] });
      onSuccess();
    },
    onError: (error: any) => {
      toast.error('Lỗi khi cập nhật gói: ' + (error.response?.data?.message || error.message));
    },
  });

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();

    if (!formData.code || !formData.name || !formData.display_name) {
      toast.error('Vui lòng điền đầy đủ thông tin bắt buộc');
      return;
    }
    if (formData.price <= 0) {
      toast.error('Giá phải lớn hơn 0');
      return;
    }
    if (formData.duration_type === 'custom') {
      if (!formData.duration_months && !formData.duration_days) {
        toast.error('Gói tuỳ chọn cần nhập ít nhất số tháng hoặc số ngày');
        return;
      }
    } else {
      if (!formData.duration_months || formData.duration_months <= 0) {
        toast.error('Thời hạn (tháng) phải lớn hơn 0');
        return;
      }
    }
    if (!formData.unlimited_product_creation && !formData.product_creation_count) {
      toast.error('Số lượng sản phẩm phải lớn hơn 0 nếu không phải không giới hạn');
      return;
    }

    if (isEdit && initialData) {
      updateMutation.mutate({ id: initialData.id, data: formData });
    } else {
      createMutation.mutate(formData);
    }
  };

  const handleInputChange = (field: keyof CreatePackageData, value: any) => {
    setFormData((prev) => ({ ...prev, [field]: value }));
  };

  const handleAddBenefit = () => {
    if (newBenefit.trim()) {
      setFormData((prev) => ({ ...prev, benefits: [...(prev.benefits || []), newBenefit.trim()] }));
      setNewBenefit('');
    }
  };

  const handleRemoveBenefit = (index: number) => {
    setFormData((prev) => ({ ...prev, benefits: prev.benefits?.filter((_, i) => i !== index) || [] }));
  };

  const isLoading = createMutation.isPending || updateMutation.isPending;
  const isCustom = formData.duration_type === 'custom';

  return (
    <form onSubmit={handleSubmit} className="space-y-8">
      {/* ── SECTION 1: Thông tin cơ bản ── */}
      <div className="space-y-4">
        <h3 className="text-sm font-semibold text-muted-foreground uppercase tracking-wide border-b pb-1">
          Thông tin cơ bản
        </h3>
        <div className="grid grid-cols-2 gap-4">
          <div className="space-y-2">
            <Label htmlFor="code">Mã gói *</Label>
            <Input
              id="code"
              value={formData.code}
              onChange={(e) => handleInputChange('code', e.target.value)}
              placeholder="VD: individual_monthly"
              required
              disabled={isEdit}
            />
          </div>
          <div className="space-y-2">
            <Label htmlFor="name">Tên gói *</Label>
            <Input
              id="name"
              value={formData.name}
              onChange={(e) => handleInputChange('name', e.target.value)}
              placeholder="VD: Cá nhân - Tháng"
              required
            />
          </div>
          <div className="space-y-2">
            <Label htmlFor="display_name">Tên hiển thị *</Label>
            <Input
              id="display_name"
              value={formData.display_name}
              onChange={(e) => handleInputChange('display_name', e.target.value)}
              placeholder="VD: Gói thành viên cá nhân tháng"
              required
            />
          </div>
          <div className="space-y-2">
            <Label htmlFor="iap_apple_code">IAP Apple Code</Label>
            <Input
              id="iap_apple_code"
              value={formData.iap_apple_code || ''}
              onChange={(e) => handleInputChange('iap_apple_code', e.target.value)}
              placeholder="VD: com.dipnet.individual.monthly"
            />
          </div>
          <div className="space-y-2">
            <Label htmlFor="price">Giá gốc (trước thuế) *</Label>
            <Input
              id="price"
              type="number"
              step="1000"
              min="0"
              value={formData.price}
              onChange={(e) => handleInputChange('price', parseFloat(e.target.value) || 0)}
              required
            />
          </div>
          <div className="space-y-2">
            <Label htmlFor="tax_rate">Thuế suất (0–1, VD: 0.08)</Label>
            <Input
              id="tax_rate"
              type="number"
              step="0.01"
              min="0"
              max="1"
              value={formData.tax_rate}
              onChange={(e) => handleInputChange('tax_rate', parseFloat(e.target.value) || 0)}
            />
          </div>
        </div>

        {/* Duration type + duration inputs */}
        <div className="grid grid-cols-3 gap-4">
          <div className="space-y-2">
            <Label>Loại thời hạn *</Label>
            <Select
              value={formData.duration_type || 'monthly'}
              onValueChange={(val) => {
                handleInputChange('duration_type', val);
                if (val === 'monthly') handleInputChange('duration_months', 1);
                if (val === 'yearly') handleInputChange('duration_months', 12);
                if (val === 'custom') handleInputChange('is_company', true);
              }}
            >
              <SelectTrigger>
                <SelectValue placeholder="Chọn loại thời hạn" />
              </SelectTrigger>
              <SelectContent>
                {DURATION_TYPE_OPTIONS.map((o) => (
                  <SelectItem key={o.value} value={o.value}>{o.label}</SelectItem>
                ))}
              </SelectContent>
            </Select>
          </div>
          <div className="space-y-2">
            <Label htmlFor="duration_months">
              Số tháng {isCustom ? '(tuỳ chọn nếu đã nhập ngày)' : '*'}
            </Label>
            <Input
              id="duration_months"
              type="number"
              min="0"
              value={formData.duration_months}
              onChange={(e) => handleInputChange('duration_months', parseInt(e.target.value) || 0)}
              disabled={!isCustom && formData.duration_type === 'yearly'}
            />
          </div>
          {isCustom && (
            <div className="space-y-2">
              <Label htmlFor="duration_days">Số ngày thêm (tuỳ chọn)</Label>
              <Input
                id="duration_days"
                type="number"
                min="0"
                value={formData.duration_days}
                onChange={(e) => handleInputChange('duration_days', parseInt(e.target.value) || 0)}
              />
            </div>
          )}
        </div>

        <div className="space-y-2">
          <Label htmlFor="description">Mô tả</Label>
          <Textarea
            id="description"
            value={formData.description}
            onChange={(e) => handleInputChange('description', e.target.value)}
            rows={3}
            placeholder="Mô tả ngắn về gói thành viên..."
          />
        </div>

        {/* Toggles */}
        <div className="grid grid-cols-3 gap-4">
          <div className="flex items-center justify-between rounded border px-3 py-2">
            <Label>Kích hoạt</Label>
            <Switch
              checked={formData.is_active}
              onCheckedChange={(v) => handleInputChange('is_active', v)}
            />
          </div>
          <div className="flex items-center justify-between rounded border px-3 py-2">
            <Label>Phổ biến</Label>
            <Switch
              checked={formData.is_popular}
              onCheckedChange={(v) => handleInputChange('is_popular', v)}
            />
          </div>
          <div className="flex items-center justify-between rounded border px-3 py-2">
            <Label>Gói doanh nghiệp</Label>
            <Switch
              checked={formData.is_company}
              onCheckedChange={(v) => handleInputChange('is_company', v)}
            />
          </div>
        </div>

        {/* Benefits */}
        <div className="space-y-2">
          <Label>Lợi ích</Label>
          <div className="flex gap-2">
            <Input
              value={newBenefit}
              onChange={(e) => setNewBenefit(e.target.value)}
              placeholder="Nhập lợi ích và nhấn +"
              onKeyDown={(e) => { if (e.key === 'Enter') { e.preventDefault(); handleAddBenefit(); } }}
            />
            <Button type="button" onClick={handleAddBenefit} variant="outline">
              <Plus className="h-4 w-4" />
            </Button>
          </div>
          <div className="flex flex-wrap gap-2 mt-2">
            {formData.benefits?.map((b, i) => (
              <Badge key={i} variant="secondary" className="flex items-center gap-1">
                {b}
                <button type="button" onClick={() => handleRemoveBenefit(i)} className="ml-1 hover:text-red-600">
                  <X className="h-3 w-3" />
                </button>
              </Badge>
            ))}
          </div>
        </div>
      </div>

      {/* ── SECTION 2: Quota & Giới hạn ── */}
      <div className="space-y-4">
        <h3 className="text-sm font-semibold text-muted-foreground uppercase tracking-wide border-b pb-1">
          Quota &amp; Giới hạn
        </h3>
        <div className="grid grid-cols-2 gap-4">
          <div className="space-y-2">
            <Label htmlFor="scan_violation_count">Lượt quét vi phạm (bao gồm trong gói)</Label>
            <Input
              id="scan_violation_count"
              type="number"
              min="0"
              value={formData.scan_violation_count}
              onChange={(e) => handleInputChange('scan_violation_count', parseInt(e.target.value) || 0)}
              placeholder="VD: 5 (tháng), 60 (năm), 15 (công ty)"
            />
          </div>
          <div className="space-y-2">
            <Label htmlFor="storage_capacity">Dung lượng lưu trữ (GB)</Label>
            <Input
              id="storage_capacity"
              type="number"
              min="0"
              value={formData.storage_capacity}
              onChange={(e) => handleInputChange('storage_capacity', parseInt(e.target.value) || 0)}
            />
          </div>
          <div className="space-y-2">
            <Label htmlFor="legal_advice_initial_turn">Lượt tư vấn pháp lý tặng khi mua (mặc định 1)</Label>
            <Input
              id="legal_advice_initial_turn"
              type="number"
              min="0"
              value={formData.legal_advice_initial_turn ?? 1}
              onChange={(e) => handleInputChange('legal_advice_initial_turn', parseInt(e.target.value) || 0)}
            />
          </div>
        </div>

        {/* Product creation quota */}
        <div className="space-y-3">
          <div className="flex items-center justify-between rounded border px-3 py-2">
            <Label>Đăng ký công bố không giới hạn</Label>
            <Switch
              checked={formData.unlimited_product_creation}
              onCheckedChange={(v) => handleInputChange('unlimited_product_creation', v)}
            />
          </div>
          {!formData.unlimited_product_creation && (
            <div className="space-y-2">
              <Label htmlFor="product_creation_count">Số lượng sản phẩm được tạo</Label>
              <Input
                id="product_creation_count"
                type="number"
                min="1"
                value={formData.product_creation_count}
                onChange={(e) => handleInputChange('product_creation_count', parseInt(e.target.value) || 0)}
              />
            </div>
          )}
        </div>

        {/* Strike turns */}
        <div className="space-y-3">
          <div className="flex items-center justify-between rounded border px-3 py-2">
            <div>
              <Label>Không giới hạn lượt đánh gậy</Label>
              <p className="text-xs text-muted-foreground">Chỉ áp dụng cho gói doanh nghiệp</p>
            </div>
            <Switch
              checked={formData.unlimited_strike ?? false}
              disabled={!formData.is_company}
              onCheckedChange={(v) => handleInputChange('unlimited_strike', v)}
            />
          </div>
          {!formData.unlimited_strike && (
            <div className="space-y-2">
              <Label htmlFor="strike_count">
                Số lượt đánh gậy tặng khi mua gói
              </Label>
              <Input
                id="strike_count"
                type="number"
                min="0"
                value={formData.strike_count ?? 0}
                onChange={(e) => handleInputChange('strike_count', parseInt(e.target.value) || 0)}
                placeholder="VD: 10 (cá nhân), 0 (miễn phí)"
              />
            </div>
          )}
        </div>
      </div>

      {/* ── SECTION 3: Cấu hình giá dịch vụ ── */}
      <div className="space-y-4">
        <h3 className="text-sm font-semibold text-muted-foreground uppercase tracking-wide border-b pb-1">
          Cấu hình giá dịch vụ cho loại gói này
        </h3>
        <p className="text-xs text-muted-foreground">
          Cấu hình giá áp dụng cho user đang dùng gói này. Giá user miễn phí (chưa có gói) được lấy từ cấu hình riêng của từng gói dịch vụ.
        </p>

        {/* Publication pricing */}
        <div className="space-y-2">
          <Label className="text-xs font-medium text-muted-foreground">Đăng ký công bố tác phẩm</Label>
          <div className="grid grid-cols-3 gap-4">
            <div className="space-y-1">
              <Label htmlFor="publication_base_price" className="text-xs">Giá cơ bản / tác phẩm (VND)</Label>
              <Input
                id="publication_base_price"
                type="number"
                step="1000"
                min="0"
                value={formData.publication_base_price ?? 0}
                onChange={(e) => handleInputChange('publication_base_price', parseFloat(e.target.value) || 0)}
                placeholder="500000"
              />
            </div>
            <div className="space-y-1">
              <Label htmlFor="publication_software_min_price" className="text-xs">Phần mềm / CSDL — giá tối thiểu (VND)</Label>
              <Input
                id="publication_software_min_price"
                type="number"
                step="1000"
                min="0"
                value={formData.publication_software_min_price ?? 0}
                onChange={(e) => handleInputChange('publication_software_min_price', parseFloat(e.target.value) || 0)}
                placeholder="500000"
              />
            </div>
            <div className="space-y-1">
              <Label htmlFor="publication_software_max_price" className="text-xs">Phần mềm / CSDL — giá tối đa (VND)</Label>
              <Input
                id="publication_software_max_price"
                type="number"
                step="1000"
                min="0"
                value={formData.publication_software_max_price ?? 0}
                onChange={(e) => handleInputChange('publication_software_max_price', parseFloat(e.target.value) || 0)}
                placeholder="3000000"
              />
            </div>
          </div>
        </div>

        {/* Add-on prices */}
        <div className="grid grid-cols-2 gap-4">
          <div className="space-y-1">
            <Label htmlFor="scan_turn_additional_price" className="text-xs">Giá mua thêm 1 lượt quét vi phạm (VND)</Label>
            <Input
              id="scan_turn_additional_price"
              type="number"
              step="1000"
              min="0"
              value={formData.scan_turn_additional_price ?? 0}
              onChange={(e) => handleInputChange('scan_turn_additional_price', parseFloat(e.target.value) || 0)}
              placeholder="VD: 200000 (cá nhân), 100000 (DN tháng/năm), 60000 (DN tuỳ chọn)"
            />
          </div>
          <div className="space-y-1">
            <Label htmlFor="strike_turn_additional_price" className="text-xs">Giá mua thêm 1 lượt đánh gậy (VND)</Label>
            <Input
              id="strike_turn_additional_price"
              type="number"
              step="1000"
              min="0"
              value={formData.strike_turn_additional_price ?? 0}
              onChange={(e) => handleInputChange('strike_turn_additional_price', parseFloat(e.target.value) || 0)}
              placeholder="VD: 200000 (cá nhân), 60000 (DN tuỳ chọn)"
            />
          </div>
        </div>
      </div>

      {/* ── SECTION 4: Phí hoa hồng ── */}
      <div className="space-y-4">
        <h3 className="text-sm font-semibold text-muted-foreground uppercase tracking-wide border-b pb-1">
          Phí hoa hồng &amp; dịch vụ
        </h3>
        <div className="grid grid-cols-2 gap-4">
          <div className="space-y-2">
            <Label htmlFor="negotiation_fee_percent">% Phí đàm phán (0–1, VD: 0.15 = 15%)</Label>
            <Input
              id="negotiation_fee_percent"
              type="number"
              step="0.01"
              min="0"
              max="1"
              value={formData.negotiation_fee_percent ?? 0}
              onChange={(e) => handleInputChange('negotiation_fee_percent', parseFloat(e.target.value) || 0)}
              placeholder="0.15"
            />
          </div>
          <div className="space-y-2">
            <Label htmlFor="exploitation_fee_percent">% Phí khai thác ủy quyền (0–1, VD: 0.1 = 10%)</Label>
            <Input
              id="exploitation_fee_percent"
              type="number"
              step="0.01"
              min="0"
              max="1"
              value={formData.exploitation_fee_percent ?? 0}
              onChange={(e) => handleInputChange('exploitation_fee_percent', parseFloat(e.target.value) || 0)}
              placeholder="0.10"
            />
          </div>
        </div>
      </div>

      {/* Submit */}
      <div className="flex justify-end gap-2 pt-2 border-t">
        <Button type="button" variant="outline" onClick={onCancel} disabled={isLoading}>
          Hủy
        </Button>
        <Button type="submit" disabled={isLoading}>
          {isLoading ? 'Đang xử lý...' : isEdit ? 'Cập nhật gói' : 'Tạo gói mới'}
        </Button>
      </div>
    </form>
  );
}
```

- [ ] **Step 8.3: Verify TypeScript types**

```bash
cd /Users/macos/workspace-user/marketplace-admin && npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 8.4: Commit**

```bash
cd /Users/macos/workspace-user/marketplace-admin
git add \
  app/api/package.ts \
  components/dashboard/membership-management/PackageForm.tsx
git commit -m "$(cat <<'EOF'
feat(marketplace-admin): redesign PackageForm with duration type selector and new pricing/quota fields

- Add duration_type Select (monthly/yearly/custom); custom shows free-form day+month inputs
- Add strike quota section: unlimited_strike toggle (company only) + strike_count input
- Add legal_advice_initial_turn input
- Add per-tier pricing section: publication base/software prices, scan and strike add-on prices
- Add commission section: negotiation_fee_percent, exploitation_fee_percent
- Fix duration validation: custom only requires at least one of days/months > 0
- Update CreatePackageData/UpdatePackageData/PackageData types in package.ts

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Self-Review Against Spec

### Spec Coverage

| Requirement | Task |
|---|---|
| 5 gói + 1 user miễn phí | Task 1 (`DurationType`), Task 2 (`IsCompany` on User), Task 6 (admin CRUD) |
| Admin chọn gói tháng/năm/tuỳ chọn | Task 1 (`MembershipDurationType` constants), Task 8 (Select in form) |
| is_company ở User | Task 2 |
| Strike turn system | Tasks 1, 2, 3, 4, 5, 7 |
| Mua thêm lượt đánh gậy | Task 5 (`handle_strike_turn.go`, payment link) |
| Giá đăng ký công bố theo loại member | Tasks 1, 6, 8 (PublicationBasePrice, SoftwareMin/Max) |
| Giá mua thêm quét theo loại member | Tasks 1, 6, 8 (ScanTurnAdditionalPrice) |
| Giá mua thêm gậy theo loại member | Tasks 1, 6, 8 (StrikeTurnAdditionalPrice) |
| % phí đàm phán / khai thác | Tasks 1, 6, 8 |
| Cronjob reset khi hết hạn | Task 5 (cronjob update) |
| Admin API domain mới cho strike turn | Task 7 |
| Build passes before commit | Each task has build step |

### Known Constraints / Notes

1. **`ServiceScopeStrikeTurn` in discountcodecol** — check `marketplace-backend/schema/discountcodecol/enum.go`; if `ServiceScopeScanTurn` exists, add `ServiceScopeStrikeTurn` following the same pattern. Add it in the same commit as Task 5 (Step 5.5).

2. **Free-tier pricing** — lives on the `publication`, `scan_turn` packages' own `price` field. The membership package pricing fields (`publication_base_price` etc.) only apply to users who have an active membership. This is the intended approach per spec discussion.

3. **Monthly quota reset** — not implemented. Admin should configure `scan_violation_count` and `strike_count` as total-for-period (e.g., 60 for a yearly-individual with 5/month), consistent with current system behaviour.

4. **`DurationType` backward compat for storage** — storage packages already use `DurationType`; the new constants are membership-specific and don't conflict with any existing storage values.

5. **Frontend (marketplace-frontend)** — `MembershipPageContent.tsx` reads `is_company` from packages (unchanged) and will display custom-duration packages correctly as long as `display_name` is set properly. No code changes required for basic display.
