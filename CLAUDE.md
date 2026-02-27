# CLAUDE.md — Development Guide

## Overview

Cafe Creamery is a single-file browser game (`index.html`, ~2300 lines). All CSS, HTML, and JS live in one file. There is no build step — open `index.html` directly in a browser to test.

## File Layout

```
index.html
  ├── <style>     lines ~6-810     CSS (globals, components, animations)
  ├── <body>      lines ~812-920   HTML (game layout, overlays, canvas)
  └── <script>    lines ~922-2285  JS  (state, UI, game logic, shop)
```

## Architecture

### State Variables (top of `<script>`)
- `day`, `money`, `customersServedToday` — core progression
- `currentOrders[]` / `playerOrders[]` — array of order objects (supports multi-ice-cream)
- `activeOrderIndex` — which ice cream is being built (tab system)
- `playerOrder` — reference to `playerOrders[activeOrderIndex]` (convenience alias)
- `dayTimerSeconds` / `dayTimeLeft` / `dayTimerId` — continuous day timer
- `customerBuildStart` — timestamp for tip calculation and patience indicator
- `phase` — state machine: `"idle"` | `"dialogue"` | `"build"`
- `hasSecondStation`, `hasNotepad` — shop upgrade flags
- `unlockedFlavours`, `unlockedToppings` — `Set` objects

### Order Object Shape
```js
{ container: "cone"|"cup", scoops: ["vanilla", ...], toppings: ["sprinkles", ...] }
```
- `scoops` is an array (order matters for display, multiset comparison for scoring)
- `toppings` is an array in orders, a `Set` in `playerOrder`

### Key Functions

| Function | Purpose |
|----------|---------|
| `startDay()` | Reset day state, init day timer, clear earnings |
| `startCustomerRound()` | Generate orders, show dialogue, start day timer on first customer |
| `startDayTimer()` | 1-second interval: ticks timer, updates bar, checks patience, ends day at 0 |
| `endDayFromTimer()` | Called when day timer hits 0 — auto-submits if mid-build, ends day |
| `evaluateOrder()` | Scores all ice creams, calculates earnings/tips, tracks breakdown |
| `scoreOneOrder()` | Scores a single order (container + scoops + toppings = 3 parts) |
| `renderOrderTabs()` | Draws tab buttons for multi-ice-cream orders |
| `switchToOrder(i)` | Saves current builder state, loads order `i` into the builder |
| `renderBuilderVisual()` | Draws the CSS ice cream in the builder panel |
| `renderShopItems()` | Populates the shop overlay with purchasable items |
| `randomOrder()` | Generates an order with difficulty scaling based on `day` |
| `generateDialogue()` | Creates customer speech from order(s) + customer personality |
| `updateStation2Hint()` | Shows container silhouette hint when Station B is owned |

### Game Flow
```
startDay() → [player clicks "Serve first customer"]
  → startCustomerRound() → [dialogue typewriter plays]
  → [player clicks "Okay"] → phase="build", customerBuildStart set
  → [player builds order, switches tabs if multi]
  → [player clicks "Serve" OR day timer expires]
  → evaluateOrder() → [show results]
  → [player clicks "Next customer" OR day timer expires]
  → ... repeat until dayTimeLeft <= 0 ...
  → endDaySummary() → [day-end overlay with earnings breakdown]
  → [player clicks "Continue to Shop"] → openShop()
  → [player buys upgrades, clicks "Sleep & Next Day"]
  → closeShopAndNextDay() → day++ → startDay()
```

## CSS Conventions

- CSS variables defined in `:root` — `--bg`, `--accent`, `--ink`, `--good`, `--bad`
- `.card` class for panels (rounded, backdrop-blur, soft shadow)
- `.hidden` utility class uses `display: none !important`
- Topping tiles use `.topping-tile` with CSS-art children (`.topping-illust-*`)
- Customer patience: `.avatar-neutral`, `.avatar-impatient` (with `@keyframes shake`)
- Order tabs: `.order-tab` / `.order-tab.active`

## Testing

Open `index.html` in a browser. There is no test suite — all testing is manual. Key scenarios to verify:

1. Day timer counts down, bar changes color (green → orange → red)
2. Serving customers earns money, tips for speed on perfect orders
3. Day-end overlay shows per-customer earnings breakdown
4. Shop purchases persist across days
5. Difficulty scales: Day 5+ orders have 2+ scoops and toppings
6. Station B enables container hints and multi-ice-cream orders (Day 5+)
7. Notepad canvas draws properly and clears per customer
8. Timer expiry mid-build auto-submits; mid-dialogue ends day

## Git Workflow

- `main` is the default branch
- Feature work goes on branches like `feature/phase2-game-improvements`
- Create PRs with descriptive titles and test plans
- Keep commits focused with clear messages explaining "why"
