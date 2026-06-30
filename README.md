# Lofty Sale Ordering Tool — Frontend Prototype

## Tổng quan

Prototype HTML/CSS/JS (single file `index.html`) cho **Lofty Sale Ordering Tool** — công cụ tạo và quản lý subscription cho sales team của Lofty (nền tảng CRM bất động sản).

Hỗ trợ 2 flow chính:
1. **New Sale** — tạo đơn mới cho khách hàng chưa có subscription
2. **Upsell** — sửa đổi subscription hiện có của khách (đổi tier, thêm seats/MLS, thêm add-ons)

Tool có flow đầy đủ: configure → review → confirm → done, kèm hệ thống discount linh hoạt (Role authority, Fixed/Percentage, per-fee adjust price).

## Tech Stack

- **Single HTML file** (`index.html`, ~4200 lines) — không framework, không build step
- **Font**: Inter (Google Fonts CDN)
- **Style**: CSS inline trong `<style>` tag, sử dụng CSS variables
- **Logic**: Vanilla JS, data-driven rendering, heavy event delegation
- **Color palette**: warm beige bg `#FAF8F5`, ink `#1A1A1A`, orange accent `#C4754B`/`#B35F37` (pen icons), emerald `#047857` (active status), indigo `#3730a3` (badges)

## File structure

```
lofty-sale-tool-project/
├── index.html                          # Main app (single file, ~4200 lines)
├── README.md                           # This file
├── CLAUDE.md                           # Instructions for Claude Code
├── .claude/launch.json                 # Preview server config
└── docs/
    ├── finance-backend-requirements.md  # FBE pain points & requirements report
    ├── apply-discount-update-order-single-instance.md  # Discount logic (business)
    ├── apply-discount-technical-doc.md   # Discount logic (technical, Java)
    ├── live-quote-discount.html          # Earlier prototype for discount UX
    └── Lofty_Order_Addons.xlsx           # Add-on source data
```

## Quick start

```bash
cd lofty-sale-tool-project
python3 -m http.server 8766
# Open http://localhost:8766/index.html
```

---

# Architecture

## Top-level navigation
- **Header stepper**: `1 Configure → 2 Review → 3 Done`
- **Tab toggle** (top of main content): `New Sale | Upsell`
- **Live Quote sidebar** (right column, sticky): updates in real-time

---

## NEW SALE Flow

### Data layer

**`allPlans[]`** — 9 plans:
1. Lofty Agent(New) — $299/mo, 2 seats, AI Bundle Agent $120
2. Lofty Team — $649/mo, 5 seats, AI Bundle Team $260 [MOST CHOSEN]
3. Lofty Broker 15 — $999/mo, 15 seats, AI Copilot $10/seat
4. Lofty Broker 50 — $1299/mo, 50 seats
5. Lofty Enterprise(New) — $2299/mo, 100 seats
6. CRM Only Seats, Blaze Website, Multi-Team (CRM-Only), Multi-Team (IDX)

Each plan: `{id, name, packageFee, includedSeats, seatPrice, freeMls, mlsPrice, addonBundle, addonPrice, defaultSeats, defaultMls, onboardingFee, aiCopilot, noPackageFee, noSeatFee, noMls, hasInstances, hasSeatFee, instancePrice, freeInstances, defaultInstances, maxSeats, included[]}`

**`addons[]`** — **6 add-ons** (simplified from 25):
| Addon | Variants | Charge Type |
|---|---|---|
| Website | Hyperlocal $149 · Team Hyperlocal $70 · Full Website $79 | prorated /mo |
| Smart Homeowner | Package $0.05 · Team Package $0.05 | full amount /mo |
| AI Platform Bundle | Agent $120 · Team $260 · Broker 15 $400 · Broker 50 $1000 · Enterprise $1201 | prorated /mo (simpleAdjustTier) |
| SEO | Premium $250 (one-time) · Recurring $299 /mo | mixed |
| Website Design | Launch $499 · Grow $1299 · Scale $2999 | onetime |
| Blaze DFY | Launch $499 · Grow $1299 · Scale $2999 | onetime |

Each variant: `{name, price, unit, billing, chargeType, feeName, hideMainFee?, extraFees?, extraBadges?}`
- `chargeType`: `'prorated'` | `'full'` | `'onetime'`
- `simpleAdjustTier` (on AI Bundle): skip expand inline panel in Adjust Tier modal

**`selectedAddons[]`** — items in current quote:
```js
{addonId, variantName, variantLabel, price, feeName, qty, isOneTime, isTBD, isDefault}
```
- `isDefault: true` for AI Bundle auto-added on plan select
- `isOneTime: true` for one-time fees (SEO Premium, Website Design, Blaze DFY)

### Step 1: Configure (`#newSaleView`)
- Plan carousel (9 cards, arrows/dots navigation)
- Click a plan → auto-add AI Bundle for that tier, populate Live Quote
- Add-on simple list below: ADDED (N) section + AVAILABLE section. Click toggles add/remove (uses default first-priced variant).
- Each addon in Live Quote = own section with name + variant pill + X remove + fee row + "Adjust tier" button + pen icon

### Step 2: Review (`#step2`)
- PLAN & ADD-ONS table (Initial Payment + Recurring Payment)
- CONTRACT TERMS chips (Duration, Payment Cycle, First Month Bill)
- CUSTOMER form (Email, First/Last name, Phone, Client type, Company)
- ORDER SETTINGS (Upsell Owner, Quote Expires, Note)
- HOW WILL THE CUSTOMER PAY? (4 options: Charge now, Email link [default], Generate link, Send quote [coming soon])
- Validation: required fields + email format
- Place order → confirm modal → success alert → Step 3

### Step 3: Done (`#step3`)
- "Waiting for customer payment" page
- Contract ID, Quote ID, payment link with copy button
- Activation status list (Main subscription, Onboarding Plan, each addon)
- Start another quote / Back to dashboard buttons

---

## UPSELL Flow

### Customer data

**`UPSELL_CUSTOMERS[]`** — 3 mock customers:

| ID | Name | Email | Current Plan | Seats | MLS | Cycle Days |
|---|---|---|---|---|---|---|
| benjamin | Benjamin Carter | benjamin.carter@homethy.com | Lofty Agent(New) | 2 | 5 | 14 |
| sarah | Sarah Nguyen | sarah.n@brokerlux.com | Lofty Team | 5 | 5 | 8 |
| marcus | Marcus Le | minhledesign2203@gmail.com | Lofty Enterprise(New) | 100 | 10 | 5 |

Each customer: `{id, name, email, acctId, location, customerSince, planId, planName, seats, mls, status, instanceType, monthlyTotal, term, cycleEnds, cycleDaysRemaining, cycleLength, nextBill, nextBillShort, renews, payment, ownedAddons[], activity[]}`

### Step 1: Configure (`#upsellView`)

**Customer search section** (`#upsellSearchSection`):
- Title + description
- Search input "Search by email or ID"
- Hint chips (Benjamin, Sarah, Marcus)
- Real-time results card (acctId · location · "Current: PlanName · N seats" in indigo)

**Customer config section** (`#upsellConfigSection`, shown after pick):
- **Customer card** (`#us-customer-card`):
  - Name + "Single Instance" badge (indigo) + status badge (ACTIVE green / PENDING SETUP orange) + Change customer button
  - Email · Location · Customer since DATE (truncated with ellipsis)
  - Current plan row: planName · N seats · N MLS · term
  - $X,XXX /mo next bill (big price)
  - Cycle ends · Next bill · Renews
  - VISA •••• 1111 · exp MM/YY
  - OWNS (N): listed line items
  - RECENT ACTIVITY (N) collapsible — list of ACTIVE/UPGRADED/CANCELED entries

- **Addons to upsell** (`#upsellStagedContainer` + `#upsellAddonGrid`):
  - STAGED FOR UPSELL section (top): each staged item shows:
    - Header: name + NEWLY ADDED badge + Change SKU button (inline grid) + X remove
    - Fee rows with stepper + pen icon
    - Modifier row: qty stepper
  - AVAILABLE ADDONS section: 19-item grid (3 cols), each tile has + Add button
    - Click + Add: single-variant → auto-stage; multi-variant → expand inline picker
  - CURRENTLY OWNED section: customer's existing addons (LEGACY badge, not editable)

- **Main package** (`#us-main-card`):
  - Collapsible summary: "Currently {planName} · N seats · N MLS"
  - TIER row: planName + Change tier button. Click → grid of 9 plans (Current badge, dashed blue border on current, solid blue on selected).
  - SEATS row: stepper + $X/seat/mo
  - MLS FEEDS row: stepper + $X/feed/mo
  - When SEATS/MLS/Tier changed: row expands to 3-col with `was X` strikethrough + ↻ undo button + uplift cell (+$X/mo + $X today)

### Live Quote sidebar (upsell mode)

`renderUpsellSidebar(c)` builds:
- CURRENT SUBSCRIPTION: planName · seats · owned addons (each line) → Currently $X /mo
- `<div class="qhr strong">` divider
- MRR UPLIFT box (gradient green if positive): `+$X /mo` huge + ↑X.X%
- `<div class="qhr strong">` divider
- PENDING CHANGES list: each change = card (green-left-border)
  - Name + discount badge (if discounted)
  - +$X.XX/mo (italic bold) + strikethrough if discounted
  - +$X.XX next bill
  - +$X.XX today (blue, prorated)
  - ✎ Edit / ✕ Remove inline (if discount)
- `<div class="qhr strong">` divider
- Starting next bill ({date}) — $X.XX + tax
- "+$X added to next bill (pre-tax)" sub
- Charged today: $X.XX + tax (blue box w/ blue border)
- Review order → button (greyed if uplift=0)

### Step 2: Review (`#upsellReviewView`)

- REVIEW CHANGES card:
  - "What's changing" + "Estimated — final amount calculated by finance at submit"
  - Each change = green-bordered card: ADD/UPD badge + name + +$X/mo + next bill amount
  - Totals: Added to next bill (pre-tax) + Next bill · {date} ($X + tax)
  - Note: "No charge today — difference rolls into the next bill"
- PAYMENT card: VISA ••XXXX · default payment method · Auto-charged at submit
- NOTE (OPTIONAL) textarea (max 1024 chars)
- Back / Apply changes · +$X next bill buttons
- Sidebar mirrors Step 1 sidebar (with ✎ pen icon for DUE TODAY, ✂ qhr divider)

### Confirm modal (`#upsellConfirmOverlay`)
- Title: "Apply changes to subscription"
- Description: "No charge today — difference ($X pre-tax) added to next bill on {date}. Cannot be undone."
- Summary table: CUSTOMER, CARD, CHANGES (N items), NEXT BILL, ADDED TO NEXT BILL (red)
- Cancel + Apply changes (red button) → "Applying..." → Step 3

### Step 3: Done (`#upsellDoneView`)
- Green checkmark icon
- "Upsell submitted." + "N addons added · +$X /mo MRR. No charge today."
- CUSTOMER'S NEXT BILL box (gradient green): $X + tax · on {date} · +$X from this upsell
- "Added to subscription" list with green check + Active badge per item
- Quote # / Contract # (was #...)
- Start another upsell / Back to dashboard buttons

---

## DISCOUNT System

### Discount kinds (passed to `openDiscModal(kind, feeKey, isEdit, addonId)`)
| Kind | Stored in |
|---|---|
| `'main'` + feeKey ('package', 'seat', 'mls', 'activation', 'aibundle') | `packageDiscounts[feeKey]` |
| `'addon'` + addonId | `addonDiscounts[addonId]` |
| `'upsell-tier'` | `upsellDiscounts.tier` |
| `'upsell-seat'` | `upsellDiscounts.seat` |
| `'upsell-mls'` | `upsellDiscounts.mls` |
| `'upsell-addon'` + addonId | `upsellDiscounts.addons[addonId]` |
| `'upsell-addon-onetime'` + addonId | `upsellDiscounts.addonOnetime[addonId]` |

### Discount shape
```js
{type: 'fixed'|'percent', value: number, reason: string}
```

### Modal (`#discModalOverlay`) — simplified single-step vertical layout
- Title: addon/fee name (e.g. "Package Fee", "Website — Hyperlocal Website")
- No close X, no stepper, no Step 2 Review
- Fields (vertical stack, 400px wide):
  - Simulate User Role * (dropdown: Sales Rep / Senior Rep / Manager / Admin)
  - Discount Type * (dropdown: Fixed Amount / Percentage)
  - Discount Value * ($ or % prefix/suffix swap)
  - Approval Reason (textarea, optional)
- Validation alerts banner (warn if over tier limit, danger if no auth)
- Buttons (vertical): Apply (black, top) + Cancel (outlined, bottom)
- Apply directly commits (no review step)

### Tier authority (`DISC_TIERS`)
- Sales Rep: 0% / $0 (no auth)
- Senior Rep: ≤10% / $100
- Manager: ≤25% / $500
- Admin: unlimited

### Discount calculation
```js
computeDiscountedFee(base, disc, feeKey, plan)  // main package fees
computeAddonDiscounted(base, disc)              // addons (no special logic)
computeUpsellFinal(base, disc, scope, c)        // upsell deltas (seat fixed × added)
```

Seat fee fixed discount = `discount × charged_seats` (matches backend `getDiscountPrice()`).

### Auto-drop scenarios
- **New Sale**: Change plan → all `packageDiscounts` cleared. Remove addon → its `addonDiscounts[id]` cleared.
- **Upsell tier change/undo**: confirm dialog → `upsellDiscounts.tier` cleared
- **Upsell seat/MLS undo**: drop discount silently
- **Staged addon Cancel/Change SKU**: confirm dialog → addon discount + onetime discount cleared
- **Change customer** (Upsell reset): all upsell discounts cleared

### Pen icon (`.fee-pen`)
SVG 16×16, stroke #B35F37, hover #FAF0E8 bg. Filled background when discount applied (`.has-disc`).

---

## ADJUST TIER Modal (`#adjTierOverlay`)

Used for changing addon variant. Two modes:

### Standard mode (default for most addons)
- List of variants with current/recommended badges + price + sub label
- Selected variant expands inline:
  - Fee name + [ONE-TIME pill (orange)] + CHARGE ON ACTIVATION badge (blue: prorated/full)
  - Qty stepper (− N +) × $price/unit
  - Total updates with qty
- Total at footer ($X /mo or $X one-time)
- Cancel + Apply

### Simple mode (`simpleAdjustTier: true`, e.g. AI Bundle)
- No expand inline panel — just variant list
- Each option: name + badges + price + sub label
- Selected has solid border
- Total at footer

### Badges
- `CURRENT` — green emerald, current variant
- `RECOMMENDED` — orange, variant matching selected plan tier (for AI Bundle)

---

## ADJUST PRICE Modal (`#adjPriceOverlay`)

Legacy simple price input modal. Use `openAdjustPriceSimple(kind, feeKey, isEdit, addonId)` directly.

Title: "Adjust {Fee Name}". One input ($ prefix). Apply + Cancel stacked.

Computes discount internally: `discountAmount = original - newPrice`, stores as `{type:'fixed', value: discountAmount}`.

---

## KEY FUNCTIONS REFERENCE

| Function | Purpose |
|---|---|
| `renderPlanCards()` | Render 9 plan cards |
| `updateAll()` | Main render for New Sale Live Quote |
| `addOrUpdateAddon(addonId, variant)` | Add/replace addon in selectedAddons |
| `refreshAddonsList(query)` | Re-render addon ADDED+AVAILABLE list |
| `openDiscModal(kind, feeKey, isEdit, addonId)` | Open Apply Discount modal |
| `discCommit()` | Save discount + refresh views |
| `openAdjustTier(addonId, opts)` | Open Adjust Tier variant picker |
| `commitAdjustTier()` | Apply variant change + qty |
| `goToStep(n)` | New Sale step nav |
| `renderReview()` | Render Step 2 review (New Sale) |
| `setMode('newsale'|'upsell')` | Tab toggle |
| `pickUpsellCustomer(c)` | Init upsell flow with customer |
| `renderUpsellConfig(c)` | Build upsell Step 1 HTML |
| `wireUpsellInteractions(c)` | Wire upsell event handlers |
| `renderUpsellSidebar(c)` | Render upsell Live Quote |
| `refreshUpsellAddonUI()` | Re-render addon grid + staged |
| `refreshUpsellMainBody(c)` | Re-render TIER/SEATS/MLS rows |
| `goToUpsellStep(n)` | Upsell step nav (2/3) |
| `computeUpsellSummary(c)` | Compute uplift, next bill, changes list |
| `showUpsellConfirm(c)` | Open confirm modal |
| `renderUpsellDone(c)` | Render Step 3 Done |
| `resetUpsell()` | Clear upsell state + restore sidebar |

---

## Event handling patterns

- **Stepper buttons** on plan cards + Live Quote `.qs-btn`: document-level capture-phase delegation
- **Plan card click**: delegation on `#plansCarousel`
- **Addon list (New Sale)**: delegation on `#addonsListCard` — click toggles add/remove
- **Upsell main body** (TIER, SEATS, MLS, undo): delegation on `#upsellMainBody`
- **Upsell addon grid** (+Add, variant pick, cancel expand): delegation on `#upsellAddonGrid`
- **Staged addon** (Change SKU, Cancel, qty stepper, extra fee stepper): delegation on `#upsellStagedContainer`
- **Adjust Tier**: delegation on `#adjTierList` (variant pick + qty stepper)
- **Modal overlay clicks**: close when target = overlay itself

---

## Live Quote sidebar layout (New Sale, Step 1)

```
● LIVE QUOTE
WHAT YOU'RE BUYING
─────────────────────
{Plan Name} [LOFTY TEAM badge]

Seats Fee                  $X /mo  ✎
[− 2 +]  seats × $99 · 2 free

Package Fee               $X /mo  ✎

MLS Fee                    $X /mo  ✎
[− 5 +]  boards × $5 · 5 free

Onboarding Fee [One-time]  $X     ✎

{Each addon}
  Addon Name [VARIANT PILL]   ×
  Fee Name                $X /mo  ✎
  Adjust tier

Monthly                $X /mo  ✎
One-time                   $X   ✎
═════════════════════ (qhr strong)
DUE TODAY
$X.XX
First bill: 1st of next month

Next month                 $X /mo

[Review order →]
```

---

## Known limitations / not done

- **Backend integration**: prototype is FE-only. No actual API calls.
- **Data persistence**: refresh loses state.
- **Customer search (real)**: mock data only, real search not implemented.
- **Authentication**: tier authority is simulated (dropdown in discount modal).
- **Mobile responsive**: not optimized.
- **Audit log**: discount reason captured in state but not transmitted.

## Reference docs (in /docs)

- **finance-backend-requirements.md** — FBE pain points, requirements
- **apply-discount-update-order-single-instance.md** — discount business logic
- **apply-discount-technical-doc.md** — backend Java implementation
- **live-quote-discount.html** — earlier discount prototype
- **Lofty_Order_Addons.xlsx** — addon source data

## Repository

https://github.com/minhle2203/lofty-sale-tool (public, branch `main`)
