# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Single-file browser game: `index.html`. No build step, no dependencies, no package manager. Open the file directly in a browser or serve it with any static server.

To serve locally:
```
python3 -m http.server 7823
```

## Architecture

Everything lives in one HTML file with three sections:

**CSS** — all styles are inline in `<style>`. Key selectors:
- `#game-area` — full-screen click target; the single event handler for all gameplay input lives here
- `#square` — the clickable target; visibility and pointer-events are controlled exclusively via the `.visible` CSS class (never inline `pointer-events`)
- `#flash`, `#miss-label`, `#reaction-label` — feedback overlays, all `pointer-events: none`
- `.screen.active` — toggles overlay screens (start, between-round, game-over)

**Game state machine** — a single `phase` string drives all logic:
```
IDLE → WAITING → SQUARE_APPEARING → SQUARE_VISIBLE → WAITING → ... → GAME_OVER
                                                    ↘ BETWEEN_ROUNDS ↗
```
All input handlers guard on `phase` before acting. `onMiss(type)` accepts `'slow'` (timeout) or `'misclick'` (background click) to show different feedback.

**Input handling** — one `addPointerHandler(gameArea, handler)` listener using `pointerdown` uses `getBoundingClientRect()` coordinate comparison to decide hit vs miss. There is no separate listener on `#square`. This avoids all event propagation / `stopPropagation` reliability issues and works correctly for both mouse and touch.

**Difficulty curve** (`calcDisplayTime(round)`):
- Round 1: 1000ms
- Rounds 2–3: −50ms/round
- Round 4+: −25ms/round
- Floor: 300ms

**Round structure** — 5 successful hits advance the round; misses cost a life but do not count toward round completion (`hitsInRound` vs `attemptsInRound` are tracked separately).

**localStorage keys**: `rtg_best_round`, `rtg_best_reaction`, `rtg_best_avg` — read on game-over, written only when the current session beats the stored value.

**Generation counter** (`gameGen`) — incremented on every `init()` call; delayed callbacks (game-over timer) capture the generation at scheduling time and no-op if it has changed, preventing stale callbacks from affecting a restarted game.
