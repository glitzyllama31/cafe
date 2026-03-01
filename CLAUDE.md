# CLAUDE.md — Development Guide

## Overview

Cafe Creamery is a browser game split across two files: `index.html` (HTML + JS, ~3000 lines) and `style.css` (~700 lines). There is no build step — open `index.html` directly in a browser to test. Optimized for iPad landscape.

## File Layout

```
index.html
  ├── <head>      Google Fonts links, viewport meta, style.css link
  ├── <svg>       SVG <defs> block (gradients, filters, patterns)
  ├── <canvas>    Particle overlay canvas (#particleCanvas)
  ├── <body>      Top bar, scene area, counter shelf, ingredient tray, overlays
  └── <script>    JS (state, SVG rendering, particles,
                  drag-and-drop, audio, game logic, shop)

style.css
  ├── :root       Design tokens (colors, flavour triads, wood, glass, metal)
  ├── Base         Reset, body (warm cafe wall bg), typography
  ├── Top bar      #topBar (wooden plank), emoji timer, day/money labels
  ├── Scene area   #sceneArea (customer, speech bubble, builder)
  ├── Counter      #counterShelf (wood-grain shelf with containers + sauces)
  ├── Tray         #ingredientTray (glass display case with tubs + toppings + bin)
  ├── Overlays     Day-end, shop (parchment scroll with wood rollers)
  ├── Notepad      #notepadPanel (floating overlay)
  ├── Animations   scoopDrop, servePulse, dangerPulse, wrongShake, coinBounce
  └── Responsive   @media (max-width: 800px) for non-iPad screens

.github/workflows/pr-preview.yml
  └── PR preview deployment to GitHub Pages via rossjrw/pr-preview-action
```

## Architecture

### HTML Layout
```
+--------------------------------------------------------------+
| #topBar: [emoji %] ═══timer bar═══  Day N  $money  [Shop]   |  wooden plank
+--------------------------------------------------------------+
| #tutorialBanner (contextual guidance text)                    |
+--------------------------------------------------------------+
|                                                              |
|  #builderArea       #speechBubble          #customerArea     |  #sceneArea
|  [orderTabs]        "Order text..."        [avatar 80x80]   |  (warm cafe wall)
|  [builderVisual]    [customerVisual]       [name]            |
|  [Serve btn]                               [progress dots]   |
|                                                              |
+══════════════ #counterShelf (wooden shelf) ═══════════════════+
| [cone] [cup stack] [caramel bottle] [choc sauce bottle]      |  wood grain
+--------------------------------------------------------------+
| #ingredientTray (glass display case)                          |
| [vanilla tub][choc tub][straw tub][mint tub] | [sprinkles]  |
|              #flavourTubs                    | [flake]  [bin]|
|                                              | #toppingItems |
+--------------------------------------------------------------+
| #resultBar (slim text feedback + #endOfDayPanel)              |
+--------------------------------------------------------------+
```

Overlays (fixed, z-index 100): `#dayEndOverlay`, `#shopOverlay` (parchment scroll) — toggled via `.active` class.

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
- `SAUCE_TOPPINGS` — `Set` of toppings rendered as bottles on counter shelf

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
| `startCustomerRound()` | Generate orders, show speech bubble + customer visual, enable builder |
| `startDayTimer()` | 1s interval: tick timer, update emoji display, check patience, end day at 0 |
| `updateTimerDisplay()` | Update emoji face + percentage text based on remaining time |
| `endDayFromTimer()` | Timer hits 0: auto-submit if building, end day if in dialogue/idle |
| `evaluateOrder()` | Score all ice creams, calc earnings/tips, trigger particles/shake |
| `scoreOneOrder()` | Score a single order (container + scoops + toppings = 3 parts) |
| `endDaySummary()` | Show day-end overlay with per-customer earnings breakdown |
| `resetPlayerOrder()` | Clear player's order state for a new customer round |
| **Builder** | |
| `addIngredient(type, value)` | Unified handler; auto-dismisses speech bubble on first add |
| `updateBuilderSteps()` | Sequential gating: containers → flavours → toppings |
| `setBuilderEnabled(enabled)` | Enable/disable counter + tray items and submit button |
| `renderBuilderVisual()` | Create SVG ice cream in #builderVisual |
| `renderCustomerVisual()` | Show mini ice cream previews in speech bubble during dialogue |
| `renderOrderTabs()` | Draw tab buttons for multi-ice-cream orders |
| `switchToOrder(i)` | Save current builder state, load order `i` |
| `maybeEnableServe()` | Check if all orders are ready and enable Serve button |
| **SVG rendering** | |
| `createIceCreamSVG(container, scoops, toppings, scale)` | Full ice cream with gradients + toppings |
| `createSauceBottleSVG(type)` | 56×70 sauce bottle for counter shelf |
| `createStandingConeSVG()` | 56×70 upright cone for counter shelf |
| `createCupStackSVG()` | 56×70 stacked cups for counter shelf |
| `createFlavourTubSVG(flavour)` | 72×56 ice cream tub for ingredient tray |
| `createSprinklesPileSVG()` | 56×56 sprinkles in bowl for tray |
| `createFlakeItemSVG()` | 56×56 flake on wrapper for tray |
| `createRubbishBinSVG(isOpen)` | 52×60 bin with open/closed lid states |
| `createCharacterSVG(seed, patience)` | Randomized SVG character bust (80px) |
| `renderCharacterAvatar(patience)` | Render SVG avatar into `#customerAvatar` |
| `appendSprinklesSVG/FlakeSVG/DrizzleSVG()` | Topping overlays on top scoop |
| **Counter/Tray** | |
| `refreshCounterShelf()` | Render containers + sauce bottles on wooden counter |
| `refreshIngredientTray()` | Render flavour tubs + non-sauce toppings + bin in glass tray |
| `refreshSelectionStates()` | Toggle `.selected` on items based on playerOrder |
| **Drag-and-drop** | |
| `handlePointerDown/Move/Up()` | Pointer event handlers on `.counter-item` / `.tray-item` |
| `canDrop(type, value)` | Validate drop (allows `phase === "dialogue"` for auto-dismiss) |
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
| `openShop()` / `closeShopAndNextDay()` | Open/close parchment shop overlay, advance day |
| `renderShopItems()` | Populate shop with purchasable items |
| **Utility** | |
| `typewriterEffect(el, text, speed, cb)` | Typewriter animation for dialogue |
| `updateCustomerProgress()` | Render served/current progress dots |
| `updateTopBar()` / `setTutorialMessage(msg)` | Update HUD elements |

### Game Flow
```
startDay() → [player clicks "Start Day N" in top bar]
  → startCustomerRound() → [speech bubble + typewriter, builder enabled]
  → [player taps/drags first ingredient → auto-dismiss speech bubble, phase="build"]
  → [player drags/taps ingredients from counter/tray → addIngredient()]
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
pointerdown on .counter-item / .tray-item → record start position, capture pointer
  → pointermove: if distance > 8px threshold → create ghost, track position
    → hit-test #sceneArea / #builderArea with elementFromPoint
    → hit-test #rubbishBin → toggle .drag-hover + open lid SVG
  → pointerup:
    → if never exceeded threshold → TAP: addIngredient() directly
    → if dragging + over scene → DROP: snap-in animation, addIngredient()
    → if dragging + over bin → CANCEL: shrink ghost + "invalid" sound
    → if dragging + not over zone → CANCEL: fly-back animation to source
```

## CSS Conventions

- Design tokens in `:root` — colors, flavour triads, wood/glass/metal/wall variables
- `.hidden` utility class uses `display: none !important`
- `.counter-item` (56×70px) on `#counterShelf` for containers + sauce bottles
- `.tray-item` (72×56px) in `#ingredientTray` for flavour tubs + non-sauce toppings
- `#rubbishBin` (52×60px) with `.drag-hover` state for red glow + open lid
- `#speechBubble` — organic border-radius, lined-paper rules, `.fading` for auto-dismiss
- `#counterShelf` — wood gradient with grain lines, highlight bevel, edge shadow
- `#ingredientTray` — dark interior, metal frame, glass reflection + frost pseudo-elements
- `#topBar` — wooden plank gradient with emoji timer display
- `#shopPanel` — parchment scroll with wooden roller bars (::before/::after)
- `button.primary` has pink gradient + `servePulse` animation when enabled
- Overlays use `.active` class with opacity transition (not `.hidden`)
- Animations: `scoopDrop`, `servePulse`, `dangerPulse`, `wrongShake`, `coinBounce`, `shake`
- `touch-action: manipulation` on `html` + JS `touchend` handler to prevent iOS double-tap zoom
- `@media (max-width: 800px)` responsive layout

## SVG System

Inline `<defs>` block provides shared resources:
- `<radialGradient id="grad-{flavour}">` — uses CSS variable triads
- `<filter id="scoopShadow">` — drop shadow for depth
- `<filter id="shineBlur">` — gaussian blur for specular highlights
- `<pattern id="wafflePattern">` — cross-hatch for cone texture
- `<filter id="woodGrain">` — fractalNoise + soft-light blend for wood
- `<filter id="paperTexture">` — fractalNoise + multiply for aged paper
- `<filter id="subtleBevel">` — specular lighting for 3D sheen
- `<filter id="frostFilter">` — fractalNoise + screen for frost effect

Character avatars are generated via `createCharacterSVG(seed, patience)` with seeded randomization for: skin tone (5), hair color (5), hair style (5), glasses, hats, shirt color (6). Mouth path changes with patience state (smile → flat → frown). Rendered at 80×80px in the scene.

## Customer Personality System

`randomCustomerName()` returns one of 7 types: Sunny Tourist, Local Kid, Sleepy Student, Curious Critic, Hurry-Up Commuter, Ice Cream Expert, Birthday Guest. Each personality has unique dialogue patterns in `generateDialogue()`. The `#tutorialBanner` provides contextual guidance that changes throughout gameplay.

## Testing

Open `index.html` in a browser (ideally iPad Safari or browser with touch simulation). Key scenarios:

1. Warm cafe wall background, wooden counter shelf, glass display tray visible
2. Top bar shows wooden plank with emoji timer, day label, gold money
3. Counter shelf shows sauce bottles + cup stack + cone with wood grain texture
4. Tray shows flavour tubs with per-flavour textures, sprinkles in bowl, flake on wrapper
5. Drag item from counter/tray → ghost follows → drop on scene → ice cream renders
6. Speech bubble auto-fades on first ingredient add (no "Okay" button)
7. Tap (without dragging) also adds ingredients as fallback
8. Sequential gating: flavours disabled until container selected, toppings until scoops added
9. Tap rubbish bin → clears entire order with "invalid" sound
10. Drag item over bin mid-drag → bin glows red + lid opens → release cancels add
11. Day timer counts down with emoji faces, bar changes color (green → orange → red)
12. Serving perfect order triggers confetti particles; bad order triggers screen shake
13. Customer avatar is 80px in scene, changes expression with patience
14. Customer progress dots below avatar (served = green, current = gold)
15. Result text appears in slim result bar below tray
16. Day-end overlay shows per-customer earnings breakdown
17. Shop overlay is a parchment scroll with wooden rollers, "Sleep & Next Day" button
18. Shop purchases persist across days, new items appear on counter/tray
19. Multi-order tabs work correctly (Day 5+ with Station B)
20. Notepad floats as overlay (bottom-left), draws properly and clears per customer
21. Timer expiry mid-build auto-submits; mid-dialogue ends day
22. No double-tap zoom on iPad/iPhone
23. Tutorial banner shows contextual hints on Day 1

## CI / Deployment

- `.github/workflows/pr-preview.yml` deploys PR previews to GitHub Pages
- Requires GitHub Pages enabled with source: `gh-pages` branch
- Uses `rossjrw/pr-preview-action` with concurrency control per PR

## Git Workflow

- `main` is the default branch
- Feature work goes on branches like `feature/scene-counter-layout`
- Create PRs with descriptive titles and test plans
- Keep commits focused with clear messages explaining "why"
- **Always update README.md and CLAUDE.md when changing features, layout, or structure**
