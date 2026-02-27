# Cafe Creamery

A browser-based memory game where customers order ice cream and you rebuild their orders from memory.

## How to Play

1. **Open `index.html`** in any modern browser — no build step or server needed
2. A customer arrives and describes their ice cream order via dialogue
3. Click **Okay** to dismiss the order — it disappears from view
4. Rebuild the order from memory: pick the container, scoop flavors, and toppings
5. Click **Serve Ice Cream** to submit. You earn money based on accuracy and speed

## Game Mechanics

### Day System
Each day has a countdown timer (100 seconds on Day 1, scaling down to a minimum of 60 seconds). Serve as many customers as possible before time runs out. There is no per-customer timer — speed is rewarded through tips instead.

### Scoring
- Orders have 3 scored parts: **container** (cone/cup), **scoops** (flavor + count), **toppings**
- Base earnings scale with accuracy (up to $12 per single order)
- Tips (up to $6) are awarded for perfect orders served quickly (within a 15-second window)
- Completely wrong orders result in an $8 refund penalty

### Difficulty Scaling
| Day | Min Scoops | Flavors | Min Toppings |
|-----|-----------|---------|-------------|
| 1-2 | 1 | Often same | 0 |
| 3-4 | 1 | Always mixed (2+ scoops) | 0 |
| 5-6 | 2 | Always mixed | 1 |
| 7+  | 3 (if 3+ flavors) | Always mixed | 1 |

### Customer Patience
During the build phase, the customer avatar visually reacts to how long you take — happy under 8 seconds, neutral at 8-15 seconds, and visibly impatient (with a shake animation) beyond 15 seconds.

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

- Single HTML file (`index.html`) — all CSS, HTML, and JS inline
- No dependencies, frameworks, or build tools
- Pure CSS ice cream illustrations and topping art
- HTML5 Canvas for the notepad feature
- Web Audio API for the order-arrival ding sound

## Project Structure

```
cafe/
  index.html    # The entire game (~2300 lines)
  README.md     # This file
  CLAUDE.md     # Development guide for AI assistants
```
