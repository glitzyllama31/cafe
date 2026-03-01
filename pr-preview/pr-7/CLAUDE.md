# CLAUDE.md — Development Guide

## Overview

Cafe Creamery is a browser game split across two files: `index.html` (HTML + JS, ~2560 lines) and `style.css` (~990 lines). There is no build step — open `index.html` directly in a browser to test. Optimized for iPad landscape.

## File Layout

```
index.html
  ├── <head>      Google Fonts links, viewport meta, style.css link
  ├── <svg>       SVG <defs> block (gradients, filters, patterns)
  ├── <canvas>    Particle overlay canvas (#particleCanvas)
  ├── <body>      Top bar, three-column layout, result bar, overlays
  └── <script>    lines ~195-2560  JS (state, SVG rendering, particles,
                  drag-and-drop, audio, game logic, shop)

style.css
  ├── :root       Design tokens (colors, flavour triads)
  ├── Base         Reset, body (dark shop bg), typography
  ├── Top bar      #topBar, .top-controls, #dayLabel, #moneyLabel, timer
  ├── Layout       Three-column (.col-left, .col-center, .col-right)
  ├── Components   Cards, buttons, shelf, drag items, drop zone, dialogue
  ├── Result bar   #resultBar (slim feedback strip below game columns)
  ├── Overlays     Day-end, shop (spring-bounce transitions)
  ├── Animations   scoopDrop, servePulse, dangerPulse, wrongShake, coinBounce
  └── Responsive   @media (max-width: 800px) for non-iPad screens

.github/workflows/pr-preview.yml
  └── PR preview deployment to GitHub Pages via rossjrw/pr-preview-action
```

## Architecture

### HTML Layout
```
+------------------------------------------------------------------+
| #topBar: Day | Timer+Bar | Money | [Start Day] [Shop]            |
+------------------------------------------------------------------+
| #tutorialBanner (contextual guidance text)                        |
+------------------------------------------------------------------+
|  .col-left (220px)  | .col-center (flex:1) | .col-right (240px)  |
|                      |                      |                     |
|  #customerCard       | #builder             | #ingredientShelf    |
|  - #customerName     |  - #builderHint      |  [Containers]       |
|  - .dialogueBox      |  - #orderTabs        |  [Flavours 0/3]     |
|  - #customerVisual   |  - #builderDropZone  |  [Toppings]         |
|  - #customerAvatar   |    - #builderVisual  |  [Clear scoops]     |
|  - #customerProgress |  - #stations (A + B) |                     |
|                      |  - Serve button      |                     |
|  #notepadPanel       |                      |                     |
+------------------------------------------------------------------+
| #resultBar (slim text feedback + #endOfDayPanel)                  |
+------------------------------------------------------------------+
```

Overlays (fixed, z-index 100): `#dayEndOverlay`, `#shopOverlay` — toggled via `.active` class with opacity transition.

### State Variables (top of `<script>`)
- `day`, `money`, `customersServedToday` — core progression
- `currentOrder` / `currentOrders[]` — the customer's order(s) for current round
- `playerOrders[]` — player's built orders (one per ice cream in multi-order)
- `activeOrderIndex` — which ice cream is being built (tab system)
- `playerOrder` — reference to `playerOrders[activeOrderIndex]` (convenience alias)
- `dayTimerSeconds` / `dayTimeLeft` / `dayTimerId` — continuous day timer
- `customerBuildStart` — timestamp for tip calculation and patience indicator
- `phase` — state machine: `"idle"` | `"dialogue"` | `"build"`
- `roundActive` — boolean, true during active build phase
- `hasSecondStation`, `hasNotepad` — shop upgrade flags
- `unlockedFlavours`, `unlockedToppings` — `Set` objects
- `currentCustomerName` — personality label for current customer (affects dialogue)
- `dayEarnings[]` — per-customer earning records for day-end breakdown
- `inTutorial` — true on Day 1 for tutorial messages
- `DragManager` — drag-and-drop state (active, ghost, source, type, value, threshold)
- `particles[]` — active particle instances for canvas animation
- `currentCustomerSeed` — seeded random for deterministic character appearance
- `audioCtx` — Web Audio context (lazy-initialized)

### Constants
- `ALL_FLAVOURS` / `ALL_TOPPINGS` — full arrays of game items
- `FLAVOUR_COLORS` — main/shadow/shine triads per flavour (for SVG rendering)
- `SKIN_TONES`, `HAIR_COLORS`, `SHIRT_COLORS` — character avatar randomization pools

### Order Object Shape
```js
{ container: "cone"|"cup", scoops: ["vanilla", ...], toppings: ["sprinkles", ...] }
```
- `scoops` is an array (order matters for display, multiset comparison for scoring)
- `toppings` is an array in generated orders, a `Set` in `playerOrder`

### Key Functions

| Function | Purpose |
|----------|---------|
| **Game flow** | |
| `startDay()` | Reset day state, init timer, set tutorial messages |
| `startCustomerRound()` | Generate orders, show dialogue + customer visual, start timer |
| `startDayTimer()` | 1s interval: tick timer, update bar, check patience, end day at 0 |
| `endDayFromTimer()` | Timer hits 0: auto-submit if building, end day if in dialogue/idle |
| `evaluateOrder()` | Score all ice creams, calc earnings/tips, trigger particles/shake |
| `scoreOneOrder()` | Score a single order (container + scoops + toppings = 3 parts) |
| `endDaySummary()` | Show day-end overlay with per-customer earnings breakdown |
| `resetPlayerOrder()` | Clear player's order state for a new customer round |
| **Builder** | |
| `addIngredient(type, value)` | Unified handler for all ingredient adds (drag or tap) |
| `updateBuilderSteps()` | Sequential gating: containers → flavours → toppings |
| `setBuilderEnabled(enabled)` | Enable/disable shelf tiles and submit button |
| `renderBuilderVisual()` | Create SVG ice cream in the builder drop zone |
| `renderCustomerVisual()` | Show mini ice cream previews in customer card during dialogue |
| `renderOrderTabs()` | Draw tab buttons for multi-ice-cream orders |
| `switchToOrder(i)` | Save current builder state, load order `i` |
| `maybeEnableServe()` | Check if all orders are ready and enable Serve button |
| **SVG rendering** | |
| `createIceCreamSVG(container, scoops, toppings, scale)` | Full ice cream with gradients + toppings |
| `createMiniScoopSVG(flavour)` | 36×28 mini scoop for shelf tiles |
| `createMiniToppingSVG(topping)` | 36×28 mini topping for shelf tiles |
| `createMiniContainerSVG(type)` | 36×32 mini cone/cup for shelf tiles |
| `createCharacterSVG(seed, patience)` | Randomized SVG character bust |
| `renderCharacterAvatar(patience)` | Render SVG avatar into `#customerAvatar` |
| `appendSprinklesSVG/FlakeSVG/DrizzleSVG()` | Topping overlays on top scoop |
| **Shelf** | |
| `refreshShelfContainers/Flavours/Toppings()` | Rebuild shelf tile grids with SVG illustrations |
| `refreshSelectionStates()` | Toggle `.selected` on tiles based on playerOrder |
| **Drag-and-drop** | |
| `handlePointerDown/Move/Up()` | Pointer event handlers on `.drag-item` tiles |
| `canDrop(type, value)` | Validate whether a drop is allowed (gating logic) |
| **Particles** | |
| `scoopPlaceEffect(x, y, color)` | 8 circles in flavour color on ingredient add |
| `celebratePerfect(x, y)` | 45 confetti particles on perfect order |
| `sparkleMoney(x, y)` | 10 gold stars on money earn |
| **Audio** | |
| `playSound(type)` | Web Audio: "ding", "pickup", "drop", "invalid" |
| **Dialogue** | |
| `generateDialogue(orders, customerLabel)` | Build full dialogue from orders + personality |
| `generateSingleOrderDialogue(order)` | Describe one order within a multi-order |
| `randomCustomerName()` | Pick random personality (Sunny Tourist, Local Kid, etc.) |
| **Shop** | |
| `openShop()` / `closeShopAndNextDay()` | Open/close shop overlay, advance day |
| `renderShopItems()` | Populate shop with purchasable items |
| **Utility** | |
| `typewriterEffect(el, text, speed, cb)` | Typewriter animation for dialogue |
| `updateCustomerProgress()` | Render served/current progress dots |
| `updateTopBar()` / `setTutorialMessage(msg)` | Update HUD elements |

### Game Flow
```
startDay() → [player clicks "Start Day N" in top bar]
  → startCustomerRound() → [dialogue typewriter plays, customer visual shown]
  → [player clicks "Okay"] → phase="build", visual hides, customerBuildStart set
  → [player drags/taps ingredients on shelf → addIngredient()]
  → [player clicks "Serve" OR day timer expires]
  → evaluateOrder() → [particles/shake, result text in #resultBar]
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

- Design tokens in `:root` — `--bg-shop`, `--bg-surface`, `--primary-from/to`, `--gold-from/to`, `--good`, `--bad`, `--ink`, `--ink-soft`, `--text-on-dark`
- Flavour triads: `--vanilla-main/shadow/shine`, `--chocolate-*`, `--strawberry-*`, `--mint-*`
- `.card` class for cream-white panels with top highlight `::before`
- `.hidden` utility class uses `display: none !important`
- `.drag-item` tiles (72×76px) with 3D box-shadow bottom edge, `touch-action: none`
- `.shelf-section` groups inside `#ingredientShelf` (`.col-right`)
- `#builderDropZone` states: `.has-container`, `.drop-active`, `.drop-invalid`
- `.top-controls` wraps Start Day / Shop buttons in `#topBar`
- `#resultBar` — slim feedback strip below game columns
- `button.primary` has pink gradient + `servePulse` animation when enabled
- Overlays use `.active` class with opacity transition (not `.hidden`)
- Animations: `scoopDrop`, `servePulse`, `dangerPulse`, `wrongShake`, `coinBounce`, `shake`
- `touch-action: manipulation` on `html` + JS `touchend` handler to prevent iOS double-tap zoom
- `@media (max-width: 800px)` collapses to single-column layout

## SVG System

Inline `<defs>` block provides shared resources:
- `<radialGradient id="grad-{flavour}">` — uses CSS variable triads
- `<filter id="scoopShadow">` — drop shadow for depth
- `<filter id="shineBlur">` — gaussian blur for specular highlights
- `<pattern id="wafflePattern">` — cross-hatch for cone texture

Character avatars are generated via `createCharacterSVG(seed, patience)` with seeded randomization for: skin tone (5), hair color (5), hair style (5), glasses, hats, shirt color (6). Mouth path changes with patience state (smile → flat → frown).

## Customer Personality System

`randomCustomerName()` returns one of 7 types: Sunny Tourist, Local Kid, Sleepy Student, Curious Critic, Hurry-Up Commuter, Ice Cream Expert, Birthday Guest. Each personality has unique dialogue patterns in `generateDialogue()`. The `#tutorialBanner` provides contextual guidance that changes throughout gameplay.

## Testing

Open `index.html` in a browser (ideally iPad Safari or browser with touch simulation). Key scenarios:

1. Dark shop background visible, cream cards floating above, Fredoka/Nunito fonts loaded
2. Start Day / Shop buttons in top bar, timer bar fills top bar center
3. Ingredients on right shelf panel, builder in center, customer on left
4. Drag ingredient from shelf → ghost follows pointer → drop on builder → SVG ice cream renders with particle burst
5. Tap (without dragging) also adds ingredients as fallback
6. Sequential gating: flavours disabled until container selected, toppings until scoops added
7. Clear scoops button removes scoops and re-disables toppings
8. Day timer counts down, bar changes color (green → orange → red with pulse)
9. Serving perfect order triggers confetti particles; bad order triggers screen shake
10. Customer avatars are unique SVG characters with expression changes on patience
11. Customer progress dots appear inside customer card (served = green, current = gold)
12. Result text appears in slim result bar below game columns
13. Day-end overlay shows per-customer earnings breakdown
14. Shop purchases persist across days, new flavours/toppings appear on shelf
15. Multi-order tabs work correctly (Day 5+ with Station B)
16. Notepad canvas draws properly and clears per customer
17. Timer expiry mid-build auto-submits; mid-dialogue ends day
18. No double-tap zoom on iPad/iPhone
19. Tutorial banner shows contextual hints on Day 1

## CI / Deployment

- `.github/workflows/pr-preview.yml` deploys PR previews to GitHub Pages
- Requires GitHub Pages enabled with source: `gh-pages` branch
- Uses `rossjrw/pr-preview-action` with concurrency control per PR

## Git Workflow

- `main` is the default branch
- Feature work goes on branches like `feature/aaa-visual-overhaul`
- Create PRs with descriptive titles and test plans
- Keep commits focused with clear messages explaining "why"
- **Always update README.md and CLAUDE.md when changing features, layout, or structure**
