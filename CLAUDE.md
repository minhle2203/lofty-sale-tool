# CLAUDE.md — Instructions for Claude Code

## Project Overview
Lofty Sale Ordering Tool — single-file HTML prototype (`index.html`). No build step, no framework. Open directly in browser.

## How to work
- Edit `index.html` directly
- Preview: `open index.html` hoặc dùng live server
- All CSS trong `<style>`, all JS trong `<script>` — cùng 1 file
- Data arrays `allPlans[]` và `addons[]` ở đầu `<script>` block

## Key conventions
- CSS: use CSS variables from `:root` (--bg, --white, --text-primary, --text-secondary, --text-muted, --border, --border-light, --orange, --green-badge, etc.)
- JS: vanilla, no framework. Event delegation với capture phase cho steppers. Data-driven rendering.
- Font: Inter (Google Fonts)
- Color: #C4754B for orange accents (badges, one-time, most chosen), #3D7A3D for green
- Guard all price calculations with `(value||0)` to prevent NaN
- Badges (ONE-TIME, MONTHLY, MOST CHOSEN): position absolute on wrapper, top: -10px, background: card-bg to cut through border

## Common tasks
- **Add new plan**: Add object to `allPlans[]` array, cards auto-render
- **Add new addon**: Add object to `addons[]` array with variants, nav + content auto-render
- **Fix pricing**: Check `updateAll()`, `renderReview()`, `showModal()` — same calc logic in 3 places
- **Fix event issues**: Stepper buttons use capture phase (`addEventListener(..., true)`), plan card click ignores `.step-btn` clicks

## File structure
- `index.html` — the app
- `docs/` — reference docs (finance requirements, discount logic, addon data)
- `README.md` — full architecture documentation

## Reference docs
- `docs/finance-backend-requirements.md` — Business context & requirements
- `docs/apply-discount-*.md` — Discount calculation logic (for future implementation)
- `docs/Lofty_Order_Addons.xlsx` — Add-on source data
