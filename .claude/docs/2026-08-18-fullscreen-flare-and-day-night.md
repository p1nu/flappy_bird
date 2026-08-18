# Fullscreen canvas, milestone flare, and the day/night cycle

**Date:** 2026-08-18
**Branches:** `pp-001-display` → merged (PR #1) · `pp-002-effect` → merged (PR #2) · current work uncommitted
**File touched throughout:** `index.html` (single-file game)

---

## 1. Fullscreen canvas

**Ask:** make the canvas cover the whole screen.

**First approach (merged as PR #1):** kept the classic 288 × 512 logical frame as a *minimum* and let the world grow in whichever direction the window had room to spare, at a whole-pixel zoom.

Changes: `fit()` rewritten; `W`, `H`, `GROUND_Y`, `BIRD_X` became per-resize values instead of constants; background layers (clouds, skyline, bushes) tiled across the real width and anchored to `GROUND_Y`; game-over panel centred on `W`; screens positioned through a new `uiY()` helper.

Two follow-on fixes that fell out of the wider view:

- **Pipe seeding** — pipes used to spawn just past the right edge. On a wide window that edge is far away, so the run opened with several seconds of empty sky. They now seed from a fixed distance *ahead of the bird*, so the opening beat is identical on every screen.
- **Tap-hint hand** — re-anchored to the bird instead of screen centre, where it had been left stranded.

## 2. Fixing unreachable pipe gaps (the important one)

**Symptom the user hit:** on a 1893 × 980 window, gaps were sometimes impossible to make.

**Root cause:** the world was growing *vertically*, so the pipe band became ~700 px tall while the bird can only climb about **222 px** in the 78 ticks between pipes. The original code only limited downward steps (`MAX_GAP_DROP`), because in a 400 px field an upward jump was never a problem.

**Fix — the current geometry model:** the playable band is **always the classic 400 px**, on every screen. Spare room goes into **zoom** and extra **width**, never a taller playfield.

- `fit()` picks one uniform zoom `SC` from whichever screen dimension runs out of room for 288 × 512 first.
- New `SKY_TOP` marks the ceiling of the playable band; pipe placement, the bird's ceiling clamp, and `uiY()` all hang off it.
- Zoom is deliberately **fractional** with `image-rendering: pixelated` — nearest-neighbour keeps the blocks hard, and snapping to whole numbers would leave a large unfilled margin.
- Side benefit: the canvas draws in logical pixels and the browser scales it, so the HUD, panels, and bird grow with the screen (the user's second ask).

Verified across 10 viewports: the gap band measures **180 px on every one**, inside the 222 px climb budget.

## 3. Milestone flare (merged as PR #2)

**Ask:** confetti or similar at milestones, escalating with the medal.

`medalFor()`'s if-chain became a **`MEDALS` table**, and a tier's *index* in it is the strength of its flare — the reward and the celebration cannot drift apart, and a new medal is a one-line addition.

| Tier | Flare |
| --- | --- |
| Bronze 10, Silver 20 | confetti burst off the bird, banner, rising fanfare, score kicks up a size |
| Gold 30, Platinum 50 | + confetti keeps raining |
| Diamond 100, Emerald 200 | + fireworks pop in sequence |
| Master 500 → Legendary 2000 | + rotating sunburst, colour wash |

Supporting changes:

- The top six medals had been sharing two placeholder colours (diamond was green, emerald blue-grey). Each tier now has its own palette so the ladder reads as a ladder.
- Confetti got a dark outline like the rest of the art — without it the pale silver and platinum palettes were invisible against the sky.
- The banner suppresses itself on game over so it never collides with the "GAME OVER" title.

Everything is cosmetic: nothing touches the bird, the pipes, or the clock.

## 4. Day/night cycle

**Ask:** flip every 50 pipes; a run that ended at night starts the next one at night.

- **Palette crossfade, not a cut.** A `dusk` value eases toward the target over ~1.4 s. Every colour that changes lives in a `DAY`/`NIGHT` table and is mixed once per frame into a live `PAL`; the draw code only reads `PAL`.
- **Phase belongs to the world, not the run.** `night` and `dusk` sit outside `reset()`, and `reset()` snaps `dusk` to the current phase — no sunrise on the title screen.
- **Persisted** in `localStorage` under `flappyNight`, alongside `flappyBest`, via a `setNight()` helper that owns the key.
- Night is not just darker: skyline windows light up amber, 70 stars twinkle and drift, a moon rises.

## 5. Debug harness, extracted

Super mode (invincibility, jump-to-tier, day/night toggle, forced crash, a readout) was built inline, then **lifted out of `index.html` into `debug.js`**.

`index.html` now exposes five inert seams and nothing else:

```js
const DEBUG = { invincible, scoreLocked, key, draw, onReset };
```

- Loaded **only** by `index.html?debug`, so a clean clone never requests it — no 404, no console noise.
- `debug.js` is in `.gitignore` (confirmed with `git check-ignore`) so it cannot be committed by accident.
- Any run the harness has touched is barred from writing the high score; the readout says `NOT SAVING`.

## 6. README

Rewritten as a showcase: centred title, two-up day/night hero images, the medal ladder table, the legendary flare screenshot, sections on the cycle and the fullscreen geometry (including *why* the band is pinned at 400 px), a "How it works" tour, and the debug key table. Screenshots rendered into `docs/`.

Note: markdownlint flags `MD033/no-inline-html` throughout — inherent to the centred layout; GitHub renders it fine.

---

## Verification approach

No test framework in the project, so everything was checked by driving the real game in headless Chrome:

- **Syntax + geometry invariants** — extract the `<script>`, parse it, then simulate `fit()` across 10 viewports asserting full coverage, preserved aspect, and a 180 px gap band.
- **Tier sweep** — fire all nine medals, confirming particle peaks escalate 30 → 271 and drain to zero (no leaks).
- **Autopilot soak** — 20,000 ticks with a steering bot, watching for errors and confirming pipes still kill.
- **Persistence** — two Chrome runs sharing a `--user-data-dir` to test a genuine page reload.
- **Screenshots** at every stage to check the result actually looks right.

Two harness bugs were mistaken for game bugs along the way and worth remembering: an autopilot aiming at the *gap centre* clips the top pipe (its flap arc is 46 px), and a literal newline inside a generated JS string silently kills the whole injected script.

## Files changed

| File | State |
| --- | --- |
| `index.html` | the whole game — geometry, flare, day/night, `DEBUG` seams |
| `debug.js` | new, git-ignored, loaded by `?debug` |
| `.gitignore` | new — covers `debug.js` |
| `README.md` | rewritten |
| `docs/day.png`, `night.png`, `flare.png` | new screenshots |

## Open items

- `docs/` and `.gitignore` are untracked — need `git add` on the next commit.
- Resizing mid-run still moves `BIRD_X` and `GROUND_Y` under a live bird; auto-pausing on resize was offered and not taken up.
- The file is CRLF in the editor; patch tooling normalises to LF and restores CRLF on write so diffs stay small.
