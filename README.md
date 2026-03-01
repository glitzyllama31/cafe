# Cafe Creamery

A browser-based memory game where customers order ice cream and you rebuild their orders from memory. Features a cafe scene layout with wooden counter, glass display case, drag-and-drop building, hand-crafted SVG character and food assets, particle effects, and illustrated customer characters.

## How to Play

1. **Open `index.html`** in any modern browser — no build step or server needed (optimized for iPad landscape)
2. A customer arrives and describes their order via a speech bubble
3. Start building immediately — the speech bubble auto-dismisses when you add your first ingredient
4. Pick ingredients from the **counter shelf** (containers, sauce bottles) and **glass display tray** (flavour tubs, toppings) — drag or tap to add
5. Click **Serve** to submit. You earn money based on accuracy and speed
6. Tap the **rubbish bin** to clear your current order and start over

## Game Mechanics

### Day System
Each day has a countdown timer (100 seconds on Day 1, scaling down to a minimum of 60 seconds). Serve as many customers as possible before time runs out. The timer shows an SVG face icon (happy/neutral/sad) that changes with remaining time.

### Scoring
- Orders have 3 scored parts: **container** (cone/cup), **scoops** (flavor + count), **toppings**
- Base earnings scale with accuracy (up to $12 per single order)
- Tips (up to $6) are awarded for perfect orders served quickly (within a 15-second window)
- Completely wrong orders result in an $8 refund penalty
- Perfect orders trigger confetti particles; bad orders shake the screen

### Difficulty Scaling
| Day | Min Scoops | Flavors | Min Toppings |
|-----|-----------|---------|-------------|
| 1-2 | 1 | Often same | 0 |
| 3-4 | 1 | Always mixed (2+ scoops) | 0 |
| 5-6 | 2 | Always mixed | 1 |
| 7+  | 3 (if 3+ flavors) | Always mixed | 1 |

### Customer Patience
During the build phase, the illustrated customer character (120px) changes mouth expression — smiling under 8 seconds, neutral at 8-15 seconds, and frowning beyond 15 seconds. There are 4 unique character designs mapped to the 7 personality types.

### Multi-Ice-Cream Orders
Starting at Day 5 (requires Station B upgrade), customers may order 2 or 3 ice creams. A tab system lets you switch between orders during the build phase.

## Shop Upgrades

Between days a parchment-scroll shop overlay lets you spend earnings:

| Item | Cost | Effect |
|------|------|--------|
| Extra Station (B) | $60 | Unlocks multi-ice-cream orders |
| Notepad | $45 | Floating drawing canvas for scribbling notes during orders |
| New Flavors | $30 each | Adds strawberry or mint chip to the order pool |
| New Toppings | $22 each | Adds chocolate flake, caramel drizzle, chocolate sauce, or cookie |

## Layout

```
+--------------------------------------------------------------+
| [emoji %] ═══timer bar═══  Day N    $money   [Shop]         |  wooden plank bar
+--------------------------------------------------------------+
|                                                              |
|  [Built ice cream]   "Speech bubble       [Customer avatar   |  warm cafe wall
|   above counter]      with order text"     80x80, right]     |
|                                                              |
+══════════ wooden counter shelf (containers + sauces) ════════+
| [flavour tubs in glass display case] | [toppings]      [bin] |
+--------------------------------------------------------------+
```

## Tech Stack

- **HTML + JS** in `index.html` (~3000 lines) — all game logic inline
- **CSS** in `style.css` — scene layout with premium wood/glass visuals
- **Google Fonts**: Fredoka (display) + Nunito (body)
- **SVG assets** in `assets/` — hand-crafted ice cream scoops, containers, toppings, character illustrations, UI icons
- **Pointer Events** drag-and-drop system (mouse + touch, tap fallback, bin hit-testing)
- **Canvas** particle system (confetti, scoop bursts, money sparkles) + notepad drawing
- **Web Audio API** for sound effects (pickup, drop, invalid, ding)
- **GitHub Actions** PR preview deployments via GitHub Pages
- No frameworks or build tools — open `index.html` directly

## Project Structure

```
cafe/
  index.html                        # HTML structure + all JS
  style.css                         # All CSS — scene layout + premium styles
  assets/                           # Hand-crafted SVG assets
    containers/                     #   cone.svg, cup.svg
    scoops/                         #   vanilla, chocolate, strawberry, mint-chip
    toppings/                       #   sprinkles, flake, caramel-drizzle, chocolate-sauce, cookie
    characters/                     #   4 illustrated customer designs (student, commuter, kid, teen)
    ui/                             #   coin, face-happy/neutral/sad, speech-bubble, etc.
  .github/workflows/pr-preview.yml  # PR preview deployment to GitHub Pages
  README.md                         # This file
  CLAUDE.md                         # Development guide for AI assistants
```
