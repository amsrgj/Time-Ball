# Time Ball

Physics puzzle game: draw a line, drop the ball, race the clock to the goal.

## Project shape

- **Single file**: `index.html` (~1700 lines). Vanilla HTML/CSS/JS, no framework, no build step, no dependencies.
- **Why**: portability. Works as a static page, ships as a PWA, and can be wrapped in a WKWebView for App Store submission without a JS bundler in the loop.
- **Don't add a build pipeline** unless we actually need one. Inline everything in `index.html`.

## Run locally

`.claude/launch.json` defines a `static-html` server that runs `python3 -m http.server 8080 --directory /Users/papos/line-ball-game`. Use `preview_start static-html`, then open http://localhost:8080.

## Architecture

The whole game lives inside one IIFE in `<script>` at the bottom of `index.html`. Major sections, in order:

1. **`LEVELS`** — array of 18 circuit definitions. Each has `spawn(W,H)`, `goal(W,H)`, `walls(W,H)`, `ink` multiplier, and `medals: { platinum, gold, silver, bronze }` in ms. Coordinates are W/H-relative so layouts scale to any screen.
2. **Persistent storage** — keys: `timeball.bestTimes.v3`, `timeball.ghosts.v3`, `timeball.settings.v3`, `timeball.progress.v3`, `timeball.plays.v3`. Bump the version suffix on schema changes.
3. **Screen routing** — state machine: `welcome → menu → {circuits, records, stats, settings, about, game}`. `showScreen(name)` toggles `.active` and triggers per-screen render hooks. Trophy modal hides automatically when leaving `game`.
4. **Menu attract demo** — small canvas at top of `#screen-menu` runs a scripted physics loop (`startMenuDemo`/`stopMenuDemo`). Lower gravity (600 vs 1400) than main game.
5. **Game loop** — `requestAnimationFrame` drives a variable-step outer loop with a fixed-step (1/240 s) inner physics loop. Stable across frame rates.
6. **Physics** — `closestOnSeg` finds the closest point on a segment to the ball; `resolveCollisions` pushes the ball out of penetration and reflects velocity along the normal with restitution + tangential friction. Walls expand to 4 edge segments via the `segments()` generator. Player strokes contribute their own segments. No screen-edge bouncing — the ball can fly off any side, and `checkOffscreen` triggers a reset after a brief "Lost!" toast.
7. **Recording / ghost** — every 30ms while running, `maybeSampleGhost` pushes `[t_ms, x/W, y/H]` into `recording`. On a new best, `recording` is saved as the level's ghost. Replay via `ghostPosAt(t)` with binary-search lerp.
8. **Trophy modal** — Gran Turismo–style win animation (SVG cup, scale-in + Y-axis spin, tier-colored). Triggered from `checkGoal`. Buttons: Share / Repeat / Next / ✕.
9. **Share** — `MediaRecorder` over `canvas.captureStream(30)`. Codec preference: `mp4 h264 → webm vp9 → webm vp8 → webm`. Falls back to text-only Web Share, then clipboard, if no recorder/share API.

## Medals

- **Platinum** = world record. Currently the per-level `platinum` threshold is treated as the world record (no backend). When you wire a real backend, change `hasPlatinum(idx)` and the `medalFor` platinum branch to consult the server. Platinum win shows "Congratulations, you beat the world record!" and a 🏆 in share text.
- **Gold / silver / bronze** are time tiers (≤ threshold = that tier).
- **Π easter egg**: Circuit 6 has bronze=3140ms (3.14s) as a nod to π. Don't change.
- Per-level platinum is **explicit**, not auto-derived. If you tune medals, set all four explicitly.

## State machine cheat sheet

- `won` — goal reached, physics frozen, trophy shown.
- `lost` — ball went off-screen; brief delay then `repeatAttempt()`.
- `paused` — physics frozen but no overlay.
- `tool` — `'draw' | 'erase'`. Erase removes whole strokes you drag over and refunds their ink.
- **Reset** clears strokes; **Repeat** keeps them; **Next** advances the level.

## Conventions

- No emojis in source unless they're part of the UI (medal glyphs, button icons are intentional).
- All persisted positions are W/H-normalized so resize/rotation doesn't break them.
- `let` for mutable state, `const` for handles. No classes.
- CSS uses `--platinum / --gold / --silver / --bronze` variables — extend those rather than hard-coding hex.
- Coordinate space inside the canvas is in CSS px; rendering scales by `DPR` once via `setTransform`.

## Open work

- **Native iOS Option B** — never started. Plan was a SwiftUI WKWebView wrapper around `index.html`, then App Store submission (Apple Developer account, signing, screenshots, privacy policy, review). Wrapper would let us ship this as-is.
- **Real global leaderboards** — would need a small backend (Cloudflare Workers + KV would be enough). Until then, "global" is the developer-set platinum time.
- **Medal time tuning** — current platinum/gold/silver/bronze values are calibrated guesses, not measured percentiles. Adjust based on real play data.
- **Menu demo geometry** — the ball-into-goal trajectory is sensitive to the curve points in `DEMO_LINE_NORM`. Tune if it ever fails to land cleanly.

## Things that would break the game

- Adding a build step (defeats the single-file model).
- Renaming storage keys without bumping the `.vN` suffix (silently wipes records).
- Removing `closestOnSeg` from the IIFE scope — both the game and the menu demo use it.
- Tunneling: max ball velocity is capped at 2400 px/s. Keep `MAX_V * STEP < ball.r` to prevent the ball from teleporting through thin lines.
