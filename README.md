# Cafe Creamery

A browser-based memory game where customers order ice cream and you rebuild their orders from memory. Features drag-and-drop building, SVG illustrations, particle effects, and randomized character avatars.

## How to Play

1. **Open `index.html`** in any modern browser — no build step or server needed (optimized for iPad landscape)
2. A customer arrives and describes their ice cream order via dialogue
3. Click **Okay** to dismiss the order — it disappears from view
4. Rebuild the order from memory: **drag** (or tap) ingredients from the right shelf onto the builder zone — container first, then scoops, then toppings
5. Click **Serve Ice Cream** to submit. You earn money based on accuracy and speed

## Game Mechanics

### Day System
Each day has a countdown timer (100 seconds on Day 1, scaling down to a minimum of 60 seconds). Serve as many customers as possible before time runs out. There is no per-customer timer — speed is rewarded through tips instead.

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
During the build phase, the SVG customer avatar changes expression — smiling under 8 seconds, neutral at 8-15 seconds, and frowning beyond 15 seconds.

### Multi-Ice-Cream Orders
Starting at Day 5 (requires Station B upgrade), customers may order 2 or 3 ice creams. A tab system lets you switch between orders during the build phase.

## Shop Upgrades

Between days you can spend earnings on upgrades:

| Item | Cost | Effect |
|------|------|--------|
| Extra Station (B) | $60 | Unlocks multi-ice-cream orders + container hints during build |
| Notepad | $45 | Drawing canvas for scribbling notes during order memorization |
| New Flavors | $30 each | Adds strawberry or mint chip to the order pool |
| New Toppings | $22 each | Adds chocolate flake, caramel drizzle, or chocolate sauce |

## Tech Stack

- **HTML + JS** in `index.html` (~2560 lines) — all game logic inline
- **CSS** in `style.css` (~990 lines) — extracted for maintainability
- **Google Fonts**: Fredoka (display) + Nunito (body)
- **SVG** ice cream illustrations with radial gradients, filters, and waffle patterns
- **Pointer Events** drag-and-drop system (mouse + touch, tap fallback)
- **Canvas** particle system (confetti, scoop bursts, money sparkles) + notepad drawing
- **Web Audio API** for sound effects (pickup, drop, invalid, ding)
- **GitHub Actions** PR preview deployments via GitHub Pages
- No frameworks or build tools — open `index.html` directly

## Project Structure

```
cafe/
  index.html                        # HTML structure + all JS (~2560 lines)
  style.css                         # All CSS (~990 lines)
  .github/workflows/pr-preview.yml  # PR preview deployment to GitHub Pages
  README.md                         # This file
  CLAUDE.md                         # Development guide for AI assistants
```
