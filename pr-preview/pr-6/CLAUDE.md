# CLAUDE.md — Development Guide

## Overview

Cafe Creamery is a browser game split across two files: `index.html` (HTML + JS, ~2570 lines) and `style.css` (~1020 lines). There is no build step — open `index.html` directly in a browser to test. Optimized for iPad landscape.

## File Layout

```
index.html
  ├── <head>      Google Fonts links, viewport meta, style.css link
  ├── <svg>       SVG <defs> block (gradients, filters, patterns)
  ├── <canvas>    Particle overlay canvas
  ├── <body>      Three-column game layout, overlays
  └── <script>    lines ~195-2570  JS (state, SVG rendering, particles,
                  drag-and-drop, audio, game logic, shop)

style.css
  ├── :root       Design tokens (colors, flavour triads)
  ├── Base         Reset, body (dark shop bg), typography
  ├── Layout       Three-column (.col-left, .col-center, .col-right)
  ├── Components   Cards, buttons, shelf, drag items, drop zone
  ├── Overlays     Day-end, shop (spring-bounce transitions)
  ├── Animations   scoopDrop, servePulse, dangerPulse, wrongShake, coinBounce
  └── Responsive   @media for non-iPad screens
```

## Architecture

### Three-Column Layout
```
+------------------------------------------------------------------+
| Top Bar: Day | Timer | Money                                      |
+------------------------------------------------------------------+
|  CUSTOMER (220px)  |  BUILDER (flex:1)  |  INGREDIENT SHELF (240px)|
|  - Avatar (SVG)    |  - Drop zone       |  [Containers]            |
|  - Dialogue        |  - Ice cream SVG   |  [Flavours 0/3]          |
|  - Notepad         |  - Order tabs      |  [Toppings]              |
|                    |  - Serve button    |  [Clear scoops]          |
+------------------------------------------------------------------+
| Bottom: Results | Controls                                        |
+------------------------------------------------------------------+
```

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
- `DragManager` — drag-and-drop state (active, ghost, source, type, value, threshold)
- `particles[]` — active particle instances for canvas animation
- `currentCustomerSeed` — seeded random for deterministic character appearance

### Order Object Shape
```js
{ container: "cone"|"cup", scoops: ["vanilla", ...], toppings: ["sprinkles", ...] }
```
- `scoops` is an array (order matters for display, multiset comparison for scoring)
- `toppings` is an array in orders, a `Set` in `playerOrder`

### Key Functions

| Function | Purpose |
|----------|---------|
| **Game flow** | |
| `startDay()` | Reset day state, init day timer, clear earnings |
| `startCustomerRound()` | Generate orders, show dialogue, start day timer on first customer |
| `startDayTimer()` | 1-second interval: ticks timer, updates bar, checks patience, ends day at 0 |
| `endDayFromTimer()` | Called when day timer hits 0 — auto-submits if mid-build, ends day |
| `evaluateOrder()` | Scores all ice creams, calculates earnings/tips, triggers particles |
| `scoreOneOrder()` | Scores a single order (container + scoops + toppings = 3 parts) |
| **Builder** | |
| `addIngredient(type, value)` | Unified handler for all ingredient adds (drag or tap) |
| `updateBuilderSteps()` | Sequential gating: containers → flavours → toppings |
| `renderBuilderVisual()` | Creates SVG ice cream in the builder drop zone |
| `renderOrderTabs()` | Draws tab buttons for multi-ice-cream orders |
| `switchToOrder(i)` | Saves current builder state, loads order `i` |
| **SVG rendering** | |
| `createIceCreamSVG(container, scoops, toppings, scale)` | Full ice cream SVG with gradients + toppings |
| `createMiniScoopSVG(flavour)` | 36×28 mini scoop for shelf tiles |
| `createMiniToppingSVG(topping)` | 36×28 mini topping for shelf tiles |
| `createMiniContainerSVG(type)` | 36×32 mini cone/cup for shelf tiles |
| `createCharacterSVG(seed, patience)` | Randomized SVG character bust |
| `appendSprinklesSVG/FlakeSVG/DrizzleSVG()` | Topping overlays on top scoop |
| **Shelf** | |
| `refreshShelfContainers/Flavours/Toppings()` | Rebuild shelf tile grids |
| `refreshSelectionStates()` | Toggle `.selected` on tiles based on playerOrder |
| **Drag-and-drop** | |
| `handlePointerDown/Move/Up()` | Pointer event handlers on `.drag-item` tiles |
| `canDrop(type, value)` | Validates whether a drop is allowed (gating logic) |
| **Particles** | |
| `scoopPlaceEffect(x, y, color)` | 8 circles in flavour color on ingredient add |
| `celebratePerfect(x, y)` | 45 confetti particles on perfect order |
| `sparkleMoney(x, y)` | 10 gold stars on money earn |
| **Audio** | |
| `playSound(type)` | Web Audio: "ding", "pickup", "drop", "invalid" |
| **Shop** | |
| `renderShopItems()` | Populates the shop overlay with purchasable items |

### Game Flow
```
startDay() → [player clicks "Serve first customer"]
  → startCustomerRound() → [dialogue typewriter plays, character avatar shown]
  → [player clicks "Okay"] → phase="build", customerBuildStart set
  → [player drags/taps ingredients on shelf → addIngredient()]
  → [player clicks "Serve" OR day timer expires]
  → evaluateOrder() → [particles, screen shake, results shown]
  → [player clicks "Next customer" OR day timer expires]
  → ... repeat until dayTimeLeft <= 0 ...
  → endDaySummary() → [day-end overlay with earnings breakdown]
  → [player clicks "Continue to Shop"] → openShop()
  → [player buys upgrades, clicks "Sleep & Next Day"]
  → closeShopAndNextDay() → day++ → startDay()
```

### Drag-and-Drop Flow
```
pointerdown on .drag-item → record start position, capture pointer
  → pointermove: if distance > 8px threshold → create ghost, track position
    → hit-test #builderDropZone with elementFromPoint
    → toggle .drop-active / .drop-invalid on drop zone
  → pointerup:
    → if never exceeded threshold → TAP: addIngredient() directly
    → if dragging + over valid zone → DROP: snap-in animation, addIngredient()
    → if dragging + not over zone → CANCEL: fly-back animation to source tile
```

## CSS Conventions

- Design tokens in `:root` — `--bg-shop`, `--bg-surface`, `--primary-from/to`, `--gold-from/to`, `--good`, `--bad`, `--ink`
- Flavour triads: `--vanilla-main/shadow/shine`, etc.
- `.card` class for cream-white panels with top highlight `::before`
- `.hidden` utility class uses `display: none !important`
- `.drag-item` tiles (72×76px) with 3D box-shadow bottom edge
- `.shelf-section` groups inside `#ingredientShelf` (`.col-right`)
- `#builderDropZone` states: `.has-container`, `.drop-active`, `.drop-invalid`
- `button.primary` has pink gradient + pulse animation when enabled
- Animations: `scoopDrop`, `servePulse`, `dangerPulse`, `wrongShake`, `coinBounce`, `shake`
- `touch-action: manipulation` on `html` to prevent iOS double-tap zoom

## SVG System

Inline `<defs>` block provides shared resources:
- `<radialGradient id="grad-{flavour}">` — uses CSS variable triads
- `<filter id="scoopShadow">` — drop shadow for depth
- `<filter id="shineBlur">` — gaussian blur for specular highlights
- `<pattern id="wafflePattern">` — cross-hatch for cone texture

Character avatars are generated via `createCharacterSVG(seed, patience)` with seeded randomization for: skin tone (5), hair color (5), hair style (5), glasses, hats, shirt color (6). Mouth path changes with patience state.

## Testing

Open `index.html` in a browser (ideally iPad Safari or browser with touch simulation). Key scenarios:

1. Dark shop background visible, cream cards floating above, Fredoka/Nunito fonts loaded
2. Ingredients on right shelf panel, builder in center, customer on left
3. Drag ingredient from shelf → ghost follows pointer → drop on builder → SVG ice cream renders with particle burst
4. Tap (without dragging) also adds ingredients as fallback
5. Sequential gating: flavours disabled until container selected, toppings until scoops added
6. Clear scoops button removes scoops and re-disables toppings
7. Day timer counts down, bar changes color (green → orange → red with pulse)
8. Serving perfect order triggers confetti particles; bad order triggers screen shake
9. Customer avatars are unique SVG characters with expression changes on patience
10. Day-end overlay shows per-customer earnings breakdown
11. Shop purchases persist across days, new flavours/toppings appear on shelf
12. Multi-order tabs work correctly (Day 5+ with Station B)
13. Notepad canvas draws properly and clears per customer
14. Timer expiry mid-build auto-submits; mid-dialogue ends day
15. No double-tap zoom on iPad/iPhone

## Git Workflow

- `main` is the default branch
- Feature work goes on branches like `feature/aaa-visual-overhaul`
- Create PRs with descriptive titles and test plans
- Keep commits focused with clear messages explaining "why"
