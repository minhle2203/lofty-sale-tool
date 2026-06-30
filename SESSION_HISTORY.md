# Session History — Lofty Sale Ordering Tool

Toàn bộ lịch sử build từ prototype gốc → state hiện tại. Đọc file này để pick up context khi mở Claude ở tài khoản/session khác.

> **Đọc trước**: `CLAUDE.md` (conventions + state vars + key functions) và `README.md` (architecture + flow chi tiết). File này bổ sung **chronological history** và **rationale các quyết định**.

---

## Project Identity
- **Tên**: Lofty Sale Ordering Tool (prototype)
- **Repo**: https://github.com/minhle2203/lofty-sale-tool
- **Path local**: `/Users/minhle/Documents/Web/lofty-sale-tool-project`
- **File chính**: `index.html` (single-file HTML/CSS/JS, ~4200 lines)
- **Branch**: `main` (chưa có feature branches)
- **Owner**: minhle2203 (GitHub) · marcus.le@lofty.com

---

## Starting Point

User mở chat với 4 file từ Documents/Web/:
- `Claude.md`, `Readme.md`, `index.html`, `lofty-sale-tool-project.tar.gz`

Initial state:
- Step 1: Configure (plan carousel + addons grid với variant content panel cũ)
- Step 2: Review (placeholder)
- Step 3: Done (chưa có)
- Discount: prototype 2-step modal với Role + Fee dropdown + Type + Value + Reason
- 12 plans, 25 add-ons (nhiều add-on TBD/null price)
- Chưa có Upsell flow

---

## Major Milestones (theo thứ tự)

### 1. Discount logic fix (initial bug fix)
Add-on có `price: null` đang tính là $0 trong totals → khi add vào không cập nhật giá.

**Fix**: Đánh dấu `isTBD: true` cho variants không có price → render "TBD" thay vì $0, exclude khỏi totals.

### 2. Step 3 Done page (New Sale)
Build "Waiting for customer payment" page với contract ID, quote ID, payment link, activation status list, Start another quote / Back to dashboard buttons.

### 3. Layout refinements
- `.page-layout` max-width 1280 → 1240px
- `.sidebar` width 340 → 400px
- `.plan-card` width 260 → 232px, padding 20 → 16px
- `.plan-name` font 20 → 18px
- Live Quote add-on separator (1px line + padding)
- Form validation cho Customer + Order Settings ở Step 2 (required fields, email regex)

### 4. Big update — Upsell flow build
Documents cung cấp: `INTEGRATION_PROMPT.md`, `HANDOFF.md`, `live-quote-discount.html`. Build toàn bộ Upsell:

**Step 1 (Configure)**:
- Customer search section với 3 mock customers (Benjamin, Sarah, Marcus)
- Customer card với name + Single Instance badge + status badge (ACTIVE/PENDING SETUP) + Change customer button
- OWNS section + RECENT ACTIVITY collapsible
- **Addons to upsell**: STAGED FOR UPSELL + AVAILABLE ADDONS (19 items, grid 3 cột) + CURRENTLY OWNED
- **Main package**: TIER row + Change tier (expand grid 12 plans) + SEATS stepper + MLS FEEDS stepper
- Mỗi row "changed state" có "was X" strikethrough + ↻ undo + uplift cell

**Live Quote (upsell)**: CURRENT SUBSCRIPTION → MRR UPLIFT box (gradient green) → PENDING CHANGES cards → Starting next bill → Charged today (blue box) → Review order button

**Step 2 (Review)**: REVIEW CHANGES card với ADD/UPD badges + PAYMENT card + NOTE textarea + Back/Apply buttons

**Step 3 (Done)**: Upsell submitted page với green checkmark + CUSTOMER'S NEXT BILL + Added to subscription list

**Confirm modal**: "Apply changes to subscription" với summary table + Applying… delay → Step 3

### 5. Discount system (full)
- State: `packageDiscounts`, `addonDiscounts`, `upsellDiscounts {tier, seat, mls, addons{}, addonOnetime{}}`
- Modal 2-step (Configure → Review) với Role authority validation
- Pen icons (`%` then SVG pen) cạnh mỗi price
- Auto-drop scenarios: tier change confirm, addon SKU change confirm, customer reset, etc.
- Compute: `computeDiscountedFee`, `computeAddonDiscounted`, `computeUpsellFinal`
- Seat fee fixed discount × charged_seats (match backend `getDiscountPrice()`)

### 6. UX polishing series
- Icon sizes normalized (28px buttons, 16px pen icon)
- `.qhr.strong` divider trên DUE TODAY (border-top 1.5px solid ink)
- Pen icon align right của DUE TODAY label
- `.us-charged-today` styled như info box (blue bg + border)
- Step 2 sidebar đồng nhất với Step 1 (qhr divider, pen icon)
- `.quote-line` padding 6 → 4px
- `.quote-addon-wrap` padding 8 → 4px
- `.quote-addon-remove` 32 → 20px
- Badge colors: `.adjtier-badge.current` → emerald (#2E7D32 + rgba(46,125,50,.08)), `.us-cust-badge.single/.upgraded` → indigo (#e0e7ff + #3730a3)

### 7. Add-on catalog simplification (HUGE)
User chỉ giữ lại **6 add-ons** cho New Sale (thay vì 25):
- Website (3 variants: Hyperlocal, Team Hyperlocal, Full Website)
- Smart Homeowner (2)
- AI Platform Bundle (5)
- SEO (Premium one-time, Recurring)
- Website Design (3 one-time variants)
- Blaze DFY (3 one-time variants)

Mỗi variant có `chargeType: 'prorated' | 'full' | 'onetime'`.

Auto-add AI Bundle khi pick plan → matching variant via `AI_BUNDLE_VARIANT_FOR_PLAN`.

### 8. Adjust Tier modal redesign
- Standard mode: list variants với expand inline panel (fee row + stepper + total)
- Simple mode (`simpleAdjustTier: true` — AI Bundle): chỉ list variants, không expand
- Badges: CURRENT (emerald) + RECOMMENDED (orange) — RECOMMENDED tự match plan tier
- Total ở footer ($X /mo hoặc $X one-time)

### 9. Adjust Price modal (simple price input)
Tạo modal đơn giản với 1 input ($ prefix) + Apply/Cancel. Tính discount internally = `original - newPrice`, lưu thành `{type:'fixed', value: discountAmount}`. Hiện tại function `openAdjustPriceSimple()` (kept inactive — fallback).

### 10. Discount modal SIMPLIFICATION (cuối cùng)
User yêu cầu đơn giản hóa modal:
- **Bỏ stepper Configure/Review** (header)
- **Bỏ X close button**
- **Title = addon/fee name** (vd "Package Fee", "Website — Hyperlocal Website") thay vì "Apply Discount"
- **Bỏ Step 2 (Review)**: Apply → commit thẳng
- **Bỏ Fee dropdown**: fee key derive từ pen icon click context
- **Vertical layout**: all fields stacked + buttons stacked (Apply trên, Cancel dưới)
- **Width 400px**

Modal cuối: Role → Type → Value → Reason → Apply / Cancel (vertical).

### 11. Documentation + Git
- Init git repo, commit initial
- Push lên `https://github.com/minhle2203/lofty-sale-tool` (public)
- Update CLAUDE.md và README.md với full architecture + state vars + flow chi tiết + pitfalls
- User sau đó edit thủ công docs (plan count 12 → 9, danh sách plans rút gọn)

---

## Current State (snapshot)

### Codebase
- `index.html` — 4214 lines, production-ready prototype
- 2 flows hoàn chỉnh (New Sale + Upsell) với full 3-step navigation
- 6 modals: discount, adjPrice, adjTier, send-payment-link, upsellConfirm, + 2 full-page step views
- 9 plans + 6 New Sale addons + 19 Upsell addons + 3 mock customers
- Discount system: tier authority, fixed/percent, per-fee adjust, auto-drop scenarios

### Docs
- `CLAUDE.md` — instructions for Claude Code (state vars, key functions, conventions, pitfalls)
- `README.md` — architecture + flow chi tiết + sidebar layout diagram
- `SESSION_HISTORY.md` — this file
- `docs/` — business reference docs (finance requirements, discount logic, addon Excel)

### Git
- Branch `main`, 2+ commits
- Pending: docs có thể có thay đổi chưa commit (user edit thủ công)

---

## Key Design Decisions (Rationale)

### Why single-file HTML?
Prototype-first approach. No framework = no build step = open file directly. Easy to share via tar.gz hoặc paste.

### Why split `addons[]` (New Sale) vs `UPSELL_ADDONS_CATALOG[]`?
- New Sale: catalog đơn giản, 6 items với chargeType
- Upsell: catalog phong phú hơn (19 items) cho khả năng upsell rộng
- Tách array → không conflict, mỗi flow tự sở hữu

### Why `isDefault: true` cho AI Bundle?
Plan tự động bundle 1 variant AI Bundle khi user pick plan. Stored trong selectedAddons như addon thường nhưng `isDefault:true` để mark. Vẫn removable.

### Why discount modal simplified (no review step)?
- User feedback: review step thừa cho discount đơn giản (% hoặc fixed)
- Vertical layout dễ scan trên modal nhỏ
- Title = context (fee/addon name) → user biết đang discount cái gì, không cần dropdown nữa

### Why pen icon thay vì % icon?
- Pen icon (edit) intuitive hơn về việc "adjust price"
- Color `#B35F37` (orange muted) blend với accent palette
- 16x16 size nhỏ gọn nằm sau price không gây chú ý quá

### Why event delegation everywhere?
HTML re-rendered thường xuyên (updateAll, refreshAddonsList, refreshUpsellMainBody). Direct listeners sẽ bị mất sau re-render. Delegation trên parent persistent.

### Why `ORIGINAL_QUOTE_FILLED_HTML` snapshot?
Khi reset Upsell, cần restore sidebar về New Sale-compatible structure. Snapshot taken at init.

---

## Lessons Learned / Pitfalls Encountered

1. **innerHTML replacement destroys element IDs** — phải đảm bảo template strings bảo toàn IDs cho subsequent `getElementById()` calls (vd `upsellConfirmApplyAmt` span).

2. **Variant name matching uppercase vs Title Case** — `plan.addonBundle` là uppercase ("AI PLATFORM BUNDLE — AGENT") nhưng `addons.variants[].name` là Title Case ("AI Platform Bundle — Agent"). Fix bằng `AI_BUNDLE_VARIANT_FOR_PLAN` mapping.

3. **Inline `style.display` overrides CSS class** — `resetUpsell` cần clear cả `style.display = ''` và remove `.visible` class.

4. **Step 2 sidebar inconsistency** — `renderReview()` copy innerHTML từ Step 1 sidebar nhưng wrapper structure khác. Phải mirror manually từng phần (qhr divider, pen icon).

5. **AI Bundle special case removed** — initially `packageDiscounts['aibundle']` nhưng phức tạp. Now all addon discounts go through `addonDiscounts[id]` for uniformity.

6. **Capture phase listeners** cho stepper buttons (`+/-` trên plan cards) — prevent bubbling to plan card click handler.

7. **`(value||0)` everywhere** — JS multiply by undefined → NaN propagates. Guard mọi price multiplication.

---

## How to Continue (New Session / New Account)

### Setup
```bash
git clone https://github.com/minhle2203/lofty-sale-tool.git
cd lofty-sale-tool
python3 -m http.server 8766
# Open http://localhost:8766/index.html
```

### Suggested first prompt for new Claude session
```
Đọc CLAUDE.md, README.md, và SESSION_HISTORY.md trước khi sửa code.

Project: Lofty Sale Ordering Tool — single-file HTML prototype.
Hai flow: New Sale (12 plans, 6 addons) + Upsell (3 customers, 19 addons).
Discount system với tier authority. 5 modals + 2 step views.

Toàn bộ logic trong /Users/minhle/Documents/Web/lofty-sale-tool-project/index.html (~4200 lines).
```

### Common workflows
- **Sửa CSS**: tìm class trong `<style>` block (đầu file). Sync sang `~/index.html` nếu dùng preview tool.
- **Sửa render**: tìm `updateAll()` (New Sale Live Quote), `renderUpsellSidebar()` (Upsell), `refreshUpsellMainBody()`, `refreshUpsellAddonUI()`.
- **Sửa modal**: HTML modal nằm cuối file trước `</body>`. JS handlers gần dưới đó.
- **Add new addon**: append vào `addons[]` array (New Sale) hoặc `UPSELL_ADDONS_CATALOG[]` (Upsell). Auto-renders.
- **Add new plan**: append vào `allPlans[]`. Carousel auto-renders. Add entry vào `AI_BUNDLE_VARIANT_FOR_PLAN` nếu plan có AI bundle.

### Test flows quickly (paste vào console)
```js
// Test New Sale → Step 2 → Step 3
document.querySelector('.plan-card[data-plan="team"]').click();
goToStep(2);

// Test Upsell
setMode('upsell');
pickUpsellCustomer(UPSELL_CUSTOMERS.find(c=>c.id==='marcus'));
```

---

## Known Not-Done

- Backend integration (FE-only prototype)
- Data persistence (refresh = lose state)
- Real customer search (3 mock customers only)
- Real auth (tier role is simulated via dropdown in discount modal)
- Mobile responsive (not optimized)
- Audit log (discount reason captured but not transmitted)

---

# Session 2 (June 2026) — Tier pricing, AI Assistant rename, Dashboard

## Summary
Refactor pricing model (CSV-driven), thêm No-AI seat pricing toggle, build Dashboard view với CSV/XLSX export, simulate real-time updates.

## Thay đổi theo thứ tự

### 1. Pen icon Monthly/One-time + bỏ pen DUE TODAY
- Index.html: thêm pen icon bên phải Monthly total (→ open discount modal `feeKey='package'`) và One-time total (`feeKey='activation'`)
- Bỏ pen icon ở DUE TODAY row (cả Step 1 và Step 2 sidebar)

### 2. Xóa 3 Bloom plans
- Bỏ `bloom-elite`, `bloom-starter`, `bloom-growth` khỏi `allPlans[]`
- Còn 9 plans (sau đó còn 6 — xem #11)

### 3. Thêm 3 add-ons mới
- **Dialer Text** (2 variants): Growth 5000 texts/mo $49 · Elite 10000 texts/mo $89 — `subLabel:'seat'`
- **Dialer Call** (4 variants): Growth $15 · Elite $79 · Three Line $119 · Basic $0 — `subLabel:'seat'`
- **Office Add-On** (1 variant): $299/mo
- `renderAdjTierList()` line ~3819: stepper info `× $X/unit` → `× $X/${subLabel}` để dynamic

### 4. Discount Modal — refactor field
- **Bỏ "Simulate User Role"** field (`#discRole`)
- **Thêm Duration field** với 3 options: `Permanent` / `Months` / `1 Month`
- **Thêm Months count** select (2-12), visible khi chọn "Months"
- Discount shape mới: `{type, value, reason, duration:'permanent'|'months'|'1month', months:2-12|null}`

### 5. Discount description row (italic màu cam dưới fee)
- Helper `discDescRow(disc, feeName)`: render dòng "A discounted price of X% was applied to [feeName] for N months"
- Permanent → "permanently"; 1month → "for 1 month"
- Inserted at 5 spots New Sale + Step 2 review (`feeAmtHtml`) + Upsell sidebar (tier/seat/mls/staged addon/pending changes)
- Bỏ pill badge bên cạnh giá

### 6. Hover-only "Adjust Price" button (thay pen icon)
- Pen icon → hover-only black pill button text "Adjust Price" (hoặc "Edit Price" khi đã có discount)
- CSS: `.adjust-price-btn{position:absolute;right:8px;display:none}`, on `.quote-line:hover` → show button + `.q-value{visibility:hidden}`
- **Quan trọng**: button phải là **sibling của `.q-value`**, KHÔNG nested bên trong — vì visibility:hidden inherits xuống children
- Áp dụng cho `penFor()` helper, addon pen, Monthly/One-time totals (Step 1 + Step 2)

### 7. Bỏ hover background
- Bỏ `.quote-line:hover{background}` và `.quote-total-row:hover{background}` (per user request)
- Chỉ hide price + show button khi hover, không đổi background

### 8. AI Copilot → AI Assistant + bỏ surcharge
- **Rename**: `'AI Copilot'` → `'AI Assistant'` trong `allPlans[].included[]` cho agent/team/broker15/broker50/enterprise
- **Bỏ logic +$10/seat surcharge** cho broker15 và broker50: `aiCopilot: {perSeat:10}` → `aiCopilot: null`
- Update render text ở plan card, Live Quote, Step 2 review, breakdown
- `getIncludedAddonIds()`: match cả 'ai assistant' và 'ai copilot' (backward compat)

### 9. Seat prices theo CSV chart (With AI flat rates)
| Plan | Old | New |
|---|---|---|
| Agent | $99 | **$139** |
| Team | $99 | **$139** |
| Broker 15 | $60 | **$98** |
| Broker 50 | $50 | **$106** |
| Enterprise | $19 | **$32** (sau đó → tiered) |

### 10. Bỏ 3 plans nữa
- Xóa `crmonly` (CRM Only Seats), `blaze` (Blaze Website), `multicrm` (Multi-Team CRM-Only)
- Còn **6 main plans**: agent, team, broker15, broker50, enterprise, multiidx

### 11. CSV products analysis (no code change)
- Phân tích `Lofty SKU Catalog(Lofty SKU Catalog)-2.csv` (70 rows)
- Phân loại: Main plan (excluded) vs Add-Ons (29 products)
- Coverage status: 11 đúng, 4 sai/khác, 14 thiếu

### 12. Input editable cho stepper (bug fix)
- **Trước**: không type được số seats/MLS, chỉ click +/−
- **Fix**: `<div class="qs-val">` → `<input class="qs-val" type="number">`
- Document-level change handler: update `currentSeats`/`currentMls`/`currentInstances`/addon qty
- Enter to commit (blur), min=1, clamp với `plan.maxSeats`
- CSS: bỏ spinner default, focus → bg #FBF6EE

### 13. Enterprise tiered seat pricing (CSV bands)
- Helper `tieredSeatsFee(plan, totalSeats)`: marginal/progressive cumulative
- **With AI** (`seatPriceTiers`): 100-250:$32 · 251-500:$30 · 501-1500:$25 · 1501-3000:$22 · 3001-5000:$21 · 5001-10000:$20 · 10001+:$17
- **No AI** (`noAiSeatPriceTiers`): 25/23/19/17/16/15/13
- Áp dụng vào `updateAll()`, `renderReview()`, `showModal()`, `getFeeBase()`, `getOriginalPriceForKind()`

### 14. AI Bundle strikethrough + re-add button
- Click X trên AI Bundle (live quote) → AI Platform Bundle row trong card INCLUDED bị strikethrough
- Hiện **"+" button 16×16** (orange viền tròn) cạnh strikethrough → click re-add AI Bundle
- Pricing tự switch: With AI → No AI (xài `noAiSeatPrice` / `noAiSeatPriceTiers`)
- Data fields thêm: `noAiSeatPrice` cho 5 plans, `noAiSeatPriceTiers` cho enterprise
- Helpers mới: `hasAiBundleSelected()`, `effectiveSeatPrice(plan)`, `refreshCardAiBundleState()`

### 15. Bug fix: stepper rate update theo band
- **Trước**: stepper info luôn `plan.seatPrice` flat ($32) — không update khi cross band
- **Sau**: `effectiveSeatPrice()` lookup tier theo `currentSeats` → seat 502 hiện $25 (band 501-1500)
- Verified 14 boundary cases × 2 modes (With/No AI) match CSV

### 16. Verified totals
- 18 test cases (9 seat values × 2 modes): Monthly + Due Today + Seat Fee đều đúng
- Monthly = packageFee + tieredSeatsFee + addonPrice (nếu có AI)
- Due Today = onboarding fee fixed (không đổi theo seats/AI)

### 17. Dashboard view (new feature)
- **Top-nav buttons**: "New Order" / "Dashboard" toggle (body class `mode-dashboard`)
- **Stats bar**: Total / Valid / Created / Today's MRR Added
- **Filters**: search + Status dropdown + Agent dropdown
- **Table 22 columns**: Team Agent Name (hyperlink), Order ID, Status, Create Time, Start Time, Next Recurring Time, End Time, Upsell Owner, Related Order, Monthly Payment, Site Service, CRM Subscription, Seat Num, Buyer Package, Seller Package, Zipcode Dominator, Sphere Loudspeaker, Other Monthly Charge, Plan Discount, Management Fee, Contract Duration, Operate By
- **Mock data**: 15 orders với đủ field
- **Live update sim**: setInterval 5s flip random Created → Valid + flash row vàng
- Hook `addOrderToDashboard(planName, monthly, seats, operatorEmail)` — chưa wire vào `handlePlaceOrder()`

### 18. Status codes theo backend spec
- 8 codes: 0 Undefine · 1 Created · 2 Valid · 3 Upgraded · 4 Invalid · 5 Paid_End · 6 Price_Change · 7 Pause
- Title Case display: "PAID_END" → "Paid End"
- Badge classes + colors: gray/yellow/green/purple/red/gray/orange/yellow-dark

### 19. Team Agent Name hyperlink
- Visual link: `color:#2563EB; text-decoration:underline`
- `onclick="event.preventDefault()"` — chưa navigate
- Tooltip: "View [Agent Name] detail (coming soon)"

### 20. Top-nav layout
- **Outer nav** full-width white bg + bottom border
- **Inner wrapper** `.top-nav-inner` max-width 1240px (match `.page-layout`)
- Right side: New Order / Dashboard buttons + **v0.0.1 monospace pill** badge

### 21. CSV/XLSX download dropdown
- Replace "Download CSV" button → dropdown "↓ Download ▾" với 2 options:
  - Download CSV (.csv · 2 KB) — text/csv blob
  - Download Excel (.xlsx · native) — via SheetJS CDN `xlsx@0.18.5`
- Export theo filtered rows (search + status + agent)
- Xlsx có column widths + freeze header row

### 22. Operate By → email format
- Mock data đổi từ tên ("Sarah Chen") sang email (`sarah.chen@lofty.com`)
- Pattern match auth thật (Google SSO/Okta sẽ lấy email từ session)

### 23. Order ID 6-digit format
- `ORD-2026-1014` → 6-digit number (433705, 432730, ...)
- Mock formula: `433705 - i * 137 - (i*i*23 % 200)`
- New orders: `434000 + random(9000)`

---

## Hiện trạng Data (session 2 end)

### `allPlans[]` (6 plans)
```js
agent     packageFee:299  includedSeats:2   seatPrice:139 noAiSeatPrice:99  addonPrice:120
team      packageFee:649  includedSeats:5   seatPrice:139 noAiSeatPrice:99  addonPrice:260  mostChosen:true
broker15  packageFee:999  includedSeats:15  seatPrice:98  noAiSeatPrice:70  addonPrice:400
broker50  packageFee:1299 includedSeats:50  seatPrice:106 noAiSeatPrice:60  addonPrice:1000
enterprise packageFee:2299 includedSeats:100 seatPrice:32 noAiSeatPrice:25 addonPrice:1201
  seatPriceTiers:    [100-250:32, 251-500:30, 501-1500:25, 1501-3000:22, 3001-5000:21, 5001-10000:20, 10001+:17]
  noAiSeatPriceTiers:[100-250:25, 251-500:23, 501-1500:19, 1501-3000:17, 3001-5000:16, 5001-10000:15, 10001+:13]
multiidx  packageFee:0    seatPrice:15  hasSeatFee:true  hasInstances:true
```

### `addons[]` (9 add-ons)
1. Website (3) · 2. Smart Homeowner (2) · 3. AI Platform Bundle (5, `simpleAdjustTier`) · 4. SEO (2) · 5. Website Design (3) · 6. Blaze DFY (3) · 7. Dialer Text (2, `subLabel:'seat'`) · 8. Dialer Call (4, `subLabel:'seat'`) · 9. Office Add-On (1)

### `dashOrders[]` (15 mock entries)
- Status distribution: Valid×6, Created×2, Upgraded×2, các status khác ×1
- Operate By: emails `<firstname>.<lastname>@lofty.com`
- Order IDs: 6-digit (431679 → 433705)

---

## Key Helpers (session 2)

| Function | Mục đích |
|---|---|
| `tieredSeatsFee(plan, totalSeats)` | Marginal/progressive seat fee, auto-switch With/No AI |
| `effectiveSeatPrice(plan)` | Stepper "× $X" rate theo band hiện tại |
| `hasAiBundleSelected()` | True nếu 'ai-bundle' trong selectedAddons |
| `discDescRow(disc, feeName)` | Render dòng mô tả discount italic màu cam |
| `refreshCardAiBundleState()` | Toggle strikethrough + "+" button trên card |
| `addOrderToDashboard(plan, mo, seats, email)` | Push order mới vào dashOrders (chưa wire) |
| `renderDashboard()` | Render filtered rows + update stats |
| `downloadDashboardCsv()` / `downloadDashboardXlsx()` | Export filtered rows |
| `setMode('order'|'dashboard')` | Toggle view (đè lên `setMode` cũ — fix conflict cần check) |
| `getFilteredDashRows()` | Filter logic shared cho render + export |

---

## Status update mechanism (per backend codebase research)

User đã research codebase backend và xác nhận:
- **Event-driven (chính)**: billing webhook (ACH result), Kafka consumers (delete_team, delete_member, seat.type, user.display.change), payment webhooks (NpWebhookVoidExecutor), direct service calls (`activateNewContractOrder`, `updatePriceChange`)
- **Cron jobs (giới hạn)**: FreeTrialMonitorSchedule mỗi 5min (chỉ free trial expiration), CancelTeamOrMemberEmptyOrderSchedule
- **Không có daily cron** cho status update — đều near real-time

→ Dashboard FE chỉ cần WebSocket/SSE subscribe → BE đã ready

---

## Pitfalls cần nhớ (session 2)

1. **Adjust Price button** phải là sibling của `.q-value` (không nested) — visibility:hidden inherits
2. **Discount shape backward compat**: discount cũ không có `duration` → default 'permanent' render
3. **Tiered pricing marginal/progressive** (cumulative), NOT flat tier lookup
4. **Re-add AI Bundle button**: dùng `data-readd-ai="1"`, click → push với variant từ `AI_BUNDLE_VARIANT_FOR_PLAN[plan.id]`
5. **Dashboard input handler**: re-query input mỗi lần vì updateAll() re-render DOM → ref cũ detached
6. **Status badge class**: dùng `s.toLowerCase()`, must match CSS (undefine/created/valid/upgraded/invalid/paid_end/price_change/pause)
7. **Top-nav inner wrapper**: full-width outer (cho sticky+border), inner max-width 1240px (cho align với page-layout)
8. **`renderPlanCards()` runs once at init** — AI Bundle state change → dùng `refreshCardAiBundleState()` thay re-render
9. **`setMode()` conflict**: session 1 đã có `setMode('newsale'|'upsell')` cho New Sale/Upsell tabs; session 2 thêm overload cho `'order'|'dashboard'`. CẨN THẬN khi grep — có thể bị conflict. Check lại nếu function bị override.

---

## Phần CHƯA wire (TODO cho session sau)

1. **`handlePlaceOrder()` → `addOrderToDashboard(...)`**: hiện click Place Order chỉ alert + Step 3, chưa push vào dashboard
2. **User authentication**: chưa có. Production cần Google SSO / Okta để lấy email người login → fill Operate By
3. **Backend integration**: persist orders (DB), API endpoint, WebSocket cho real-time
4. **HubSpot sync**: webhook khi submit → HubSpot Custom Object (user defer)
5. **Detail page Team Agent Name**: hyperlink visual, chưa navigate

---

## External CDN added (session 2)
- `https://cdn.jsdelivr.net/npm/xlsx@0.18.5/dist/xlsx.full.min.js` — for Excel export

---

## Suggested first prompt for session 3
```
Đọc CLAUDE.md, README.md, và SESSION_HISTORY.md trước khi sửa code.

Project: Lofty Sale Ordering Tool — single-file HTML prototype.
- 6 main plans (Agent/Team/Broker15/Broker50/Enterprise/Multi-Team IDX)
- 9 New Sale add-ons (Website, Smart Homeowner, AI Platform Bundle, SEO, Website Design, Blaze DFY, Dialer Text, Dialer Call, Office Add-On)
- 3 Upsell mock customers
- Enterprise có tiered seat pricing (With AI + No AI bands)
- Dashboard view với 22 cột, status real-time simulation, CSV/XLSX export
- Discount system: tier authority, fixed/percent + duration (permanent/N months)
- Hover "Adjust Price" button thay pen icon

File chính: /Users/minhle/Documents/Web/lofty-sale-tool-project/index.html (~4700 lines)
Sync sau edit: cp index.html ~/index.html
Preview: python3 -m http.server 8766 từ project root
```

---

## Contact

Original developer's account: marcus.le@lofty.com (GitHub: minhle2203)
