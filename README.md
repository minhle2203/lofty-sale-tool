# Lofty Sale Ordering Tool — Frontend Prototype

## Tổng quan

Đây là prototype HTML/CSS/JS (single file `index.html`) cho **Lofty Sale Ordering Tool** — công cụ tạo đơn hàng (New Sale / Upsell) cho sales team của Lofty (nền tảng CRM bất động sản). Tool này thay thế quy trình quote-to-close thủ công, cho phép sales tạo quote, chọn package + add-ons, review order, và gửi payment link cho khách hàng.

## Tech Stack

- **Single HTML file** (`index.html`) — không framework, không build step
- **Font**: Inter (Google Fonts CDN)
- **Style**: CSS inline trong `<style>` tag, sử dụng CSS variables
- **Logic**: Vanilla JS trong `<script>` tag, data-driven rendering
- **Color palette**: Warm beige (#FAF8F5 bg), #1A1A1A text, #C4754B orange accent, #3D7A3D green

## Cấu trúc file

```
lofty-sale-tool-project/
├── index.html                          # Main app (single file)
├── README.md                           # This file
├── CLAUDE.md                           # Instructions for Claude Code
└── docs/
    ├── finance-backend-requirements.md # FBE pain points & requirements report
    ├── apply-discount-update-order-single-instance.md  # Discount logic (business)
    ├── apply-discount-technical-doc.md  # Discount logic (technical, Java)
    └── Lofty_Order_Addons.xlsx          # Add-on data source
```

## Architecture — index.html

### Data Layer (JS objects)

**`allPlans[]`** — 12 main plan packages:
1. Lofty Agent(New) — $299/mo, 2 seats, AI Bundle Agent $120
2. Lofty Team — $649/mo, 5 seats, AI Bundle Team $260 [MOST CHOSEN]
3. Lofty Broker 15 — $999/mo, 15 seats, AI Copilot surcharge $10/seat
4. Lofty Broker 50 — $1299/mo, 50 seats, AI Copilot surcharge $10/seat
5. Lofty Enterprise(New) — $2299/mo, 100 seats
6. CRM Only Seats — no package fee, seats only
7. Blaze Website — $499 onboarding, seats only, max 1
8. Multi-Team (CRM-Only) — seats + instances fee
9. Multi-Team (IDX) — seats + instances + MLS
10. Bloom Agent Elite — $999/mo package, no seat fee
11. Bloom Agent Starter — $699/mo, no MLS
12. Bloom Agent Growth — $799/mo

Mỗi plan có properties: `id, name, onboardingFee, onboardingPkg, packageFee, includedSeats, seatPrice, defaultSeats, freeMls, mlsPrice, aiCopilot, addonBundle, addonPrice, included[], noPackageFee, noSeatFee, noMls, hasInstances, instancePrice, freeInstances, maxSeats`

**`addons[]`** — 25 add-on categories (từ file Excel `Lofty_Order_Addons.xlsx`):
- Mỗi addon: `{id, name, count, variants[]}`
- Mỗi variant: `{name, price, unit, billing, feeName, detail, note}`
- Categories: Team Add-On, Office Add-On, Branding Bundle, Social Media, Agent Website/Vanity (13 variants), Seat Types, AI Bundle Package (5 tiers), AI Copilot, LeadPondMonthFee (7 variants), Coaching, CMA, Fixed CMA, Dialer (3), Multilingual, Lofty Bloom, Lofty Pay, Sales Assistant, Property Management, Smart Homeowner, Lofty Blast (4 variants), Listing Promotion, Sphere Ads, WordPress IDX Plugin, Enterprise Listing, Zip Code Blast

**`selectedAddons[]`** — Tracking add-ons đã chọn:
- Mỗi entry: `{addonId, variantName, variantLabel, price, feeName, qty, isDefault, isOneTime}`
- `isDefault: true` = included trong package (không có nút X để remove)
- `isOneTime: true` = phí 1 lần, cộng vào DUE TODAY thay vì Monthly

### State Variables

```js
selectedPlan    // string: plan ID đang chọn
currentSeats    // number: số seats hiện tại
currentMls      // number: số MLS boards hiện tại
currentInstances // number: số instances hiện tại
selectedAddons  // array: add-ons đã chọn
```

### UI Flow — 2 Steps

**Step 1: Configure** (`page-layout` đầu tiên)
- Tab: New Sale / Upsell
- Main plan carousel (12 cards, arrows left/right, dots)
- Khi chọn plan: card highlight, stepper +/- cho seats/MLS hiện ra, Live Quote sidebar populate
- Add-ons section: sidebar trái (search + nav list), content phải (variant cards)
- Click add-on → auto-add first variant → hiện trong Live Quote
- Click variant khác → swap trong selectedAddons
- Hover addon trong Live Quote → hiện nút X (nếu không phải default) → click X remove

**Step 2: Review** (`#step2`, hidden by default)
- PLAN & ADD-ONS: Initial Payment table + Recurring Payment table
- CONTRACT TERMS: chip groups (Contract Duration, Payment Cycle, First Month Bill)
- CUSTOMER: form (Email, First name, Last name, Phone, Client type, Company)
- ORDER SETTINGS: Upsell Owner, Quote Expires, Note (textarea 0/1024)
- HOW WILL THE CUSTOMER PAY?: 4 options (Charge now, Email payment link [default], Generate link only, Send quote [coming soon])
- Back button → Step 1
- "Create order & send email" → Modal popup

**Modal: Send payment link**
- Hiện: Customer name, Email, Monthly going forward, Customer will pay
- Cancel / Create order & send email

### Key Functions

| Function | Mô tả |
|----------|--------|
| `renderPlanCards()` | Render 12 plan cards từ allPlans data |
| `renderAddonNav()` | Render addon sidebar navigation |
| `updateAll()` | Recalculate tất cả fees, update card steppers + Live Quote + sidebar |
| `updateArrows()` | Show/hide carousel arrows dựa trên scroll position |
| `addOrUpdateAddon(addonId, variant)` | Add/replace addon trong selectedAddons, gọi updateAll |
| `updateAddonSidebarState()` | Sync sidebar "added" markers với selectedAddons |
| `updateAddonSidebarForPlan(plan)` | Auto-mark included addons khi chọn plan |
| `getIncludedAddonIds(plan)` | Map plan included items → addon IDs |
| `goToStep(n)` | Switch giữa Step 1 và Step 2 |
| `renderReview()` | Populate Step 2 review tables từ current state |
| `showModal()` | Hiện confirmation modal với tóm tắt order |

### Event Handling

- **Stepper buttons** (card + quote sidebar): Capture phase delegation trên `document`
- **Plan card click**: Delegation trên `#plansCarousel`, ignore stepper button clicks
- **Addon sidebar click**: Delegation trên `#addonsSidebar`
- **Remove addon**: Delegation trên `document` cho `.quote-addon-remove`
- **Chip groups**: Direct listeners per chip
- **Quote sidebar steppers** (`qs-btn`): Capture phase delegation

### Fee Calculation Logic

```
Monthly = Package Fee
        + Seats Fee (extraSeats × seatPrice)
        + MLS Fee (extraMls × mlsPrice)
        + Instances Fee (extraInst × instancePrice)
        + AI Copilot (seats × perSeat, nếu có)
        + Σ Monthly Addon Prices (non-oneTime)

One-time = Onboarding Fee + Σ One-time Addon Prices

DUE TODAY = One-time total
Next month = Monthly total
```

Guards: `(value||0)` cho tất cả price multiplications để tránh NaN khi property undefined.

### CSS Architecture

- CSS Variables trong `:root` cho colors, spacing, fonts
- Badge positioning: MOST CHOSEN, ONE-TIME, MONTHLY dùng `position: absolute` trên wrapper `position: relative`, `top: -10px, left: 10px/20px`, `background: var(--card-bg)` để cắt ngang border
- Plan cards: `min-width: 240px`, border highlight khi selected
- Stepper controls: `.fee-detail-static` (hidden khi selected) vs `.fee-detail-stepper` (shown khi selected)
- Live Quote sidebar: `position: sticky; top: 80px`
- Addon remove button: `opacity: 0` → `opacity: 1` on parent hover

## Bugs đã biết / Cần cải thiện

### Bugs
1. **Volume steppers trên plan cards** — đã fix nhiều lần nhưng cần verify lại, capture phase event delegation
2. **AI Copilot addon** — variants đều có price: null, khi add thủ công hiện $0.00 — cần xác nhận business logic

### Features chưa làm
1. **Upsell tab** — chưa có logic, chỉ có UI tab
2. **Discount/coupon** — chưa implement (xem docs/apply-discount-*.md cho logic backend)
3. **Step 3: Done** — chưa có UI confirmation sau khi place order
4. **Responsive/mobile** — chưa optimize cho mobile
5. **Data persistence** — không lưu state, refresh mất hết
6. **Customer search** — "Back to search" button chưa có logic search existing customers
7. **Upsell Owner search** — chỉ là text input, chưa có autocomplete
8. **Validation** — chưa validate required fields trước khi place order
9. **One-time addon fees trong Review table** — cần hiện riêng trong Initial Payment section
10. **Addon quantity stepper** — logic có nhưng chỉ hiện khi qty > 1

## Tài liệu tham khảo (trong /docs)

- **finance-backend-requirements.md** — Báo cáo đầy đủ về Finance Backend pain points, requirements từ Sales/CS/RevOps, mapping với Subscription Object plan và Sales Ordering Tool
- **apply-discount-update-order-single-instance.md** — Business logic cho discount: fixed amount vs percentage, duration, fee types
- **apply-discount-technical-doc.md** — Technical doc chi tiết: DB schema, Java classes, function flow, discount calculation code
- **Lofty_Order_Addons.xlsx** — Source data cho add-ons (2 sheets: All Add-ons, Add-on Tiers)
