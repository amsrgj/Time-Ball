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

1. **Virtual coordinate system** — gameplay runs in a fixed `VW × VH` world (currently 1000 × 1600 portrait). All state — walls, spawn, goal, ball radius, gravity, friction, ink budget, sim time — is in virtual units, independent of screen size. The renderer maps virtual → screen via a single fit-and-letterbox transform (`viewScale`, `viewOffX`, `viewOffY`) recomputed on resize. Pointer events convert screen px → virtual on the way in.
2. **`LEVELS`** — array of 18 circuit definitions. Each has a stable `id` slug, plain-object `spawn:{x,y}`, `goal:{x,y,w,h}`, `walls:[{x,y,w,h},...]` in virtual units, an `ink` multiplier, and `medals: { platinum, gold, silver, bronze }` in ms. Storage maps (`bestTimes`, `ghosts`, `plays`) and `progress.currentLevel` all reference the `id`, never the array position — reorder LEVELS freely without invalidating saved records. `LEVEL_INDEX_BY_ID` gives the reverse lookup. Access via helpers `bestTimeOf(idx)`, `ghostOf(idx)`, `playsOf(idx)`, `medalOf(idx)`, `isCompleted(idx)` rather than indexing the maps directly.
3. **Persistent storage** — keys: `timeball.bestTimes.v4`, `timeball.ghosts.v5`, `timeball.settings.v3`, `timeball.progress.v4`, `timeball.plays.v4`. All per-circuit maps are id-keyed. `loadKeyedMap`/`loadProgress` one-shot migrate from the prior index-keyed schema on first load. Bump the version suffix on any schema change.
4. **Screen routing** — state machine: `welcome → menu → {circuits, records, stats, settings, about, game}`. `showScreen(name)` toggles `.active` and triggers per-screen render hooks. Trophy modal hides automatically when leaving `game`.
5. **Menu attract demo** — small canvas at top of `#screen-menu` runs a scripted physics loop (`startMenuDemo`/`stopMenuDemo`) in its own screen-px coord space (not virtual). Lower gravity (600) than main game.
6. **Game loop** — `requestAnimationFrame` drives an outer loop that accumulates real `dt` into `physicsAcc`; the inner loop runs as many fixed `STEP = 1/240 s` physics ticks as fit, carrying the leftover to the next frame. `simTimeMs` only advances inside those ticks, so the on-screen timer and physics are deterministic and identical across FPS.
7. **Physics** — `closestOnSeg` finds the closest point on a segment to the ball; `resolveCollisions` pushes the ball out of penetration and reflects velocity along the normal with restitution + tangential friction. Walls expand to 4 edge segments via the `segments()` generator. Player strokes contribute their own segments. No edge bouncing — the ball can fly off any side of the virtual world, and `checkOffscreen` (with `OFFSCREEN_MARGIN` vu of slack past `VW`/`VH`) triggers a reset after a brief "Lost!" toast.
8. **Recording / ghost** — every 30ms of `simTimeMs`, `maybeSampleGhost` pushes `[simTimeMs, virtX, virtY]` into `recording`. On a new best, `recording` is saved as the level's ghost. Replay via `ghostPosAt(t)` with binary-search lerp; coords are virtual so they replay identically on any screen.
9. **Trophy modal** — Gran Turismo–style win animation (SVG cup, scale-in + Y-axis spin, tier-colored). Triggered from `checkGoal`. Buttons: Share / Repeat / Next / ✕.
10. **Share** — `MediaRecorder` over `canvas.captureStream(30)`. Codec preference: `mp4 h264 → webm vp9 → webm vp8 → webm`. Falls back to text-only Web Share, then clipboard, if no recorder/share API.

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
- All gameplay state (and persisted positions) is in virtual units, not screen px — resize/rotation just recomputes the viewport transform.
- `let` for mutable state, `const` for handles. No classes.
- CSS uses `--platinum / --gold / --silver / --bronze` variables — extend those rather than hard-coding hex.
- Coordinate space for game content is virtual units; `draw()` applies a single `setTransform(DPR * viewScale, …)` and clips drawing to `[0,VW]×[0,VH]`. Letterbox bands are filled with the outer bg.

## Open work

- **Native iOS Option B** — never started. Plan was a SwiftUI WKWebView wrapper around `index.html`, then App Store submission (Apple Developer account, signing, screenshots, privacy policy, review). Wrapper would let us ship this as-is.
- **Real global leaderboards** — would need a small backend (Cloudflare Workers + KV would be enough). Until then, "global" is the developer-set platinum time.
- **Medal time tuning** — current platinum/gold/silver/bronze values are calibrated guesses, not measured percentiles. Adjust based on real play data.
- **Menu demo geometry** — the ball-into-goal trajectory is sensitive to the curve points in `DEMO_LINE_NORM`. Tune if it ever fails to land cleanly.

## Things that would break the game

- Adding a build step (defeats the single-file model).
- Renaming storage keys without bumping the `.vN` suffix (silently wipes records).
- Removing `closestOnSeg` from the IIFE scope — both the game and the menu demo use it.
- Tunneling: the velocity cap is `MAX_V` vu/s (currently 3500) and the ball radius is `BALL_R` (currently 18). Keep `MAX_V * STEP < BALL_R` to prevent the ball from teleporting through thin lines (3500 * 1/240 ≈ 14.6 < 18 ✓). `clampV()` runs at the start of every step and again after collision resolution so the cap holds regardless of contact density.
- Drawing without applying the viewport transform — anything rendered with raw `(0,0) → (screenW, screenH)` will sit outside the virtual world and won't line up with physics.
