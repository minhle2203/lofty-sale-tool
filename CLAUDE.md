# CLAUDE.md — Instructions for Claude Code

## Project Overview
**Lofty Sale Ordering Tool** — single-file HTML prototype (`index.html`) for sales reps to create quotes for Lofty (real estate CRM platform). Two flows: **New Sale** (brand new subscription) and **Upsell** (modify existing customer subscription).

No build step, no framework. Pure vanilla HTML + CSS + JS. Open `index.html` directly in browser.

## How to work
- **Edit `index.html` directly.** It's the entire app (~4200 lines).
- **Preview**: `python3 -m http.server 8766` from project root → `http://localhost:8766/index.html`
- All CSS in `<style>`, all JS in `<script>` — same file.
- After every edit, sync to `~/index.html` if using Claude Preview tool:
  `cp /Users/minhle/Documents/Web/lofty-sale-tool-project/index.html /Users/minhle/index.html`

## Key conventions
- **CSS**: use CSS variables from `:root` (`--bg`, `--white`, `--text-primary`, `--text-secondary`, `--text-muted`, `--border`, `--border-light`, `--orange`, `--accent-strong` #B35F37, `--accent-soft`, etc.)
- **JS**: vanilla, no framework. Heavy use of event delegation (preferred over per-element listeners since HTML is re-rendered often).
- **Font**: Inter (Google Fonts CDN)
- **Color palette**: warm beige bg #FAF8F5, ink #1A1A1A, orange accent #C4754B (used as `--accent-strong` #B35F37 for pen icons), emerald #047857 (status active), indigo #3730a3 (badges)
- **Guard NaN**: always use `(value||0)` in price calculations to prevent NaN propagation
- **Capture phase**: stepper buttons (`+/-`) on plan cards use capture-phase listeners (`addEventListener(..., true)`) to avoid bubbling to plan card click
- **Backward compat first**: when refactoring, keep existing IDs/state keys to avoid breaking other parts. Add new fields, don't remove old ones unless certain.

## State management (global vars)

### New Sale flow
- `selectedPlan` — string, plan id (`'agent'`, `'team'`, `'broker15'`, ...)
- `currentSeats`, `currentMls`, `currentInstances` — numbers
- `selectedAddons[]` — array of `{addonId, variantName, variantLabel, price, feeName, qty, isOneTime, isTBD, isDefault}`
- `packageDiscounts` — `{package, seat, mls, activation, aibundle: {type:'fixed'|'percent', value, reason}}`
- `addonDiscounts` — `{addonId: {type, value, reason}}`
- `currentTier` — `'rep'|'senior'|'manager'|'admin'` (simulated user role for discount auth)

### Upsell flow
- `currentMode` — `'newsale'|'upsell'`
- `upsellCustomer` — picked customer object (null if none)
- `upsellNewSeats`, `upsellNewMls` — proposed new values
- `upsellNewAddons[]` — staged addons (each: `{id, name, variant, price, qty, tbd, extraFees?}`)
- `upsellExpandedAddons` — Set of addon ids with variant picker open in Available list
- `upsellStagedSkuOpen` — Set of staged item ids with SKU picker inline-expanded
- `upsellTierChangeMode` — bool, true when tier picker grid shown
- `upsellSelectedTier` — new tier id (null if unchanged)
- `upsellDiscounts` — `{tier, seat, mls, addons{}, addonOnetime{}}`

### Modal state
- `discModalCtx` — `{kind, feeKey, addonId, isEdit}` — current discount modal context
- `adjTierCtx` — `{addonId, flow, currentVariant, selected, selectedQty, recommended}` — current adjust tier modal
- `adjPriceCtx` — `{kind, feeKey, addonId, original}` — current adjust price modal (legacy/simple)

## Core data arrays
- `allPlans[]` — 12 plans with `id, name, packageFee, seatPrice, mlsPrice, freeMls, includedSeats, addonBundle, addonPrice, hasInstances, hasSeatFee, noPackageFee, noMls, noSeatFee, maxSeats, aiCopilot, onboardingFee, defaultSeats, defaultMls, defaultInstances`
- `addons[]` — **6 add-ons** (New Sale): Website, Smart Homeowner, AI Platform Bundle, SEO, Website Design, Blaze DFY. Each has `{id, name, count, variants[], simpleAdjustTier?, subLabel?}`. Each variant: `{name, price, unit, billing, chargeType, feeName, hideMainFee?, extraFees?, extraBadges?}`
- `UPSELL_ADDONS_CATALOG[]` — 19 add-ons (Upsell flow), separate from `addons[]`
- `UPSELL_CUSTOMERS[]` — 3 mock customers (Benjamin, Sarah, Marcus) with `{id, name, email, acctId, location, planId, planName, seats, mls, status, customerSince, monthlyTotal, cycleEnds, cycleDaysRemaining, cycleLength, nextBill, nextBillShort, payment, ownedAddons[], activity[]}`
- `AI_BUNDLE_VARIANT_FOR_PLAN` — map `{agent: 'AI Platform Bundle — Agent', team: '...', ...}`
- `DISC_TIERS` — `{rep, senior, manager, admin}` discount authority limits

## Modal inventory
1. **`#discModalOverlay`** — Apply Discount modal (simplified single-step). Title = fee/addon name. Fields: Role + Type + Value + Reason. Buttons: Apply + Cancel (vertical).
2. **`#adjPriceOverlay`** — Adjust Price modal (legacy, mostly inactive — call `openAdjustPriceSimple()` directly to use)
3. **`#adjTierOverlay`** — Adjust Tier modal (variant picker, supports both `simpleAdjustTier` mode for AI Bundle and expand mode for others)
4. **`#modalOverlay`** — Send payment link modal (New Sale)
5. **`#upsellConfirmOverlay`** — Apply changes to subscription confirm modal (Upsell)
6. **Step 2 + Step 3 views** — full-page views (not modals): `#step2`, `#step3`, `#upsellReviewView`, `#upsellDoneView`

## Common tasks
- **Add new plan**: add object to `allPlans[]` array. Cards auto-render via `renderPlanCards()`.
- **Add new addon (New Sale)**: add to `addons[]`. List auto-renders via `refreshAddonsList()`. Each variant needs `chargeType` ('prorated' | 'full' | 'onetime') for proper badge.
- **Add new variant to addon**: append to `addon.variants[]`. Adjust tier modal auto-picks up.
- **Add `simpleAdjustTier`**: set `addon.simpleAdjustTier:true` to skip the expand inline panel (just shows variant list with badges + price). Add `addon.subLabel:'1 account'` to override "1 unit" sub.
- **Change discount logic**: see `computeDiscountedFee(base, disc, feeKey, plan)` and `computeAddonDiscounted(base, disc)`. Seat discount fixed type multiplies × charged seats.
- **Add new upsell customer**: append to `UPSELL_CUSTOMERS[]` with required fields.
- **Change UI in Live Quote**: edit `updateAll()` (New Sale) or `renderUpsellSidebar()` (Upsell). Step 2 mirrors Step 1 via `renderReview()` (New Sale) or via separate `renderUpsellReview()` (Upsell).
- **Add pen icon next to a price**: use the SVG path inside `<button class="fee-pen" onclick="openDiscModal(...)">` (color #B35F37, 16x16).
- **Sync Step 2 sidebar with Step 1**: `updateAll()` auto-calls `renderReview()` if Step 2 visible. `renderUpsellSidebar()` for upsell.

## Key functions (where to look)
| Function | Purpose |
|---|---|
| `renderPlanCards()` | Render 12 plan cards in carousel |
| `selectPackage()` (inside plan card click) | Plan selection — auto-adds AI Bundle, resets state |
| `updateAll()` | Main render for New Sale Live Quote — fees + addons + totals |
| `addOrUpdateAddon(addonId, variant)` | Add/replace addon in `selectedAddons[]` |
| `refreshAddonsList(query)` | Re-render addon list (ADDED + AVAILABLE sections) |
| `openDiscModal(kind, feeKey, isEdit, addonId)` | Open Apply Discount modal (single-step) |
| `discCommit()` | Save discount to state + refresh views |
| `openAdjustTier(addonId, opts)` | Open Adjust tier variant picker |
| `commitAdjustTier()` | Apply variant change + qty |
| `goToStep(n)` | Switch New Sale steps (1→2→3) |
| `renderReview()` | Render Step 2 review (New Sale) |
| `setMode(mode)` | Toggle 'newsale'/'upsell' |
| `pickUpsellCustomer(c)` | Init upsell flow with customer |
| `renderUpsellConfig(c)` | Build upsell Step 1 HTML (customer card + addons + main package) |
| `wireUpsellInteractions(c)` | Wire all upsell event handlers (delegation) |
| `renderUpsellSidebar(c)` | Render Live Quote for upsell (CURRENT SUBSCRIPTION + MRR UPLIFT + PENDING CHANGES + Charged today + Review order button) |
| `refreshUpsellAddonUI()` | Re-render addon grid + staged section |
| `refreshUpsellMainBody(c)` | Re-render TIER/SEATS/MLS FEEDS rows in Main package section |
| `goToUpsellStep(n)` | Upsell step nav (2=Review, 3=Done) |
| `computeUpsellSummary(c)` | Compute upsell totals (uplift, next bill, changes list) |
| `showUpsellConfirm(c)` | Open Apply changes confirm modal |
| `renderUpsellDone(c)` | Render Step 3 Done page |
| `resetUpsell()` | Clear all upsell state, restore default sidebar |

## Event handling patterns
- **Stepper buttons** (`+/-` on plan cards, `.qs-btn` in quote sidebar): document-level capture-phase delegation
- **Plan card click**: delegation on `#plansCarousel`, ignores button clicks
- **Addon list click**: delegation on `#addonsListCard` — toggles add/remove with default variant
- **Modal overlay click**: each modal has its own click handler that closes when target = overlay itself
- **Upsell main body (tier/seats/mls)**: delegation on `#upsellMainBody` — handles all stepper, undo, discount, tier-change interactions
- **Upsell addon section**: delegation on `#upsellAddonGrid` and `#upsellStagedContainer`

## Pen icon SVG (color #B35F37, 16x16)
```html
<button class="fee-pen" onclick="openDiscModal(...)" title="Adjust price">
  <svg viewBox="0 0 16 16">
    <path d="M11.3 2.7a1.4 1.4 0 0 1 2 0l0 0a1.4 1.4 0 0 1 0 2L5 13l-3 1 1-3 8.3-8.3z"/>
  </svg>
</button>
```

## Discount kinds (for `openDiscModal`)
- `'main'` + feeKey ('package'|'seat'|'mls'|'activation'|'aibundle') — package fees
- `'addon'` + addonId — New Sale addon
- `'upsell-tier'` — Upsell tier upgrade
- `'upsell-seat'` — Upsell added seats
- `'upsell-mls'` — Upsell added MLS feeds
- `'upsell-addon'` + addonId — Upsell staged addon
- `'upsell-addon-onetime'` + addonId — Upsell addon one-time activation fee

## Pitfalls to avoid
1. **Don't reference removed DOM IDs**: `#discFee` (Fee dropdown) and `#discStep2` were removed when modal was simplified. Use `discModalCtx.feeKey` and skip review step.
2. **Don't remove `id="discBtnApply"`**: `refreshDiscAlerts()` uses it to enable/disable the Apply button.
3. **Step 2 sidebar (`#quoteFilled2`) must mirror Step 1 structure**: if you add a new section to Step 1 sidebar, mirror it in Step 2 HTML or copy via `renderReview()`.
4. **`renderUpsellSidebar` replaces `quoteFilled.innerHTML` and sets `style.display='block'`**: must clear both when resetting (see `resetUpsell()`).
5. **Modal innerHTML mustn't destroy `id="..."` on inner elements**: subsequent calls to `getElementById(...)` will fail. Use template strings that preserve IDs.
6. **AI Bundle is a default addon** with `isDefault:true`, auto-added on plan select. Removable like any other addon. Discount stored in `addonDiscounts['ai-bundle']` (not `packageDiscounts['aibundle']` anymore).
7. **`AI_BUNDLE_VARIANT_FOR_PLAN`**: maps plan.id → Title-Case variant name for matching with `addons[ai-bundle].variants`. Plan `addonBundle` is uppercase for display only.
8. **Restore `quoteFilled.innerHTML` from snapshot**: `ORIGINAL_QUOTE_FILLED_HTML` captured at init. Used by `resetUpsell()` to restore New Sale structure after upsell completion.

## File structure
- `index.html` — the entire app
- `README.md` — architecture documentation
- `CLAUDE.md` — this file
- `docs/` — reference business docs (finance requirements, discount technical doc, addon Excel source, etc.)
- `.claude/launch.json` — preview server config

## Reference docs (in /docs)
- `finance-backend-requirements.md` — FBE pain points, requirements from Sales/CS/RevOps
- `apply-discount-update-order-single-instance.md` — business-side discount logic
- `apply-discount-technical-doc.md` — backend technical doc (Vietnamese, Java)
- `live-quote-discount.html` — previous prototype reference for discount UX
- `Lofty_Order_Addons.xlsx` — source data for add-ons

## Git
- Repo: https://github.com/minhle2203/lofty-sale-tool (public)
- Branch: `main`
- Commit + push as you go: `git add -A && git commit -m "..." && git push`
