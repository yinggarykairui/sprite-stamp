# PROJECT.md — sprite-stamp

Day 020. Issue: [factory-hub#12](https://github.com/yinggarykairui/factory-hub/issues/12).

## Spec being converged on

A 16×16 pixel-art editor in one dependency-free `index.html` at the repo root, served from GitHub Pages and equally openable from `file://`. A fixed 12-colour palette, tap-or-drag painting with gap-free line interpolation, an Erase toggle, a paint-time left–right Mirror toggle with a visible dashed axis, one-step Undo, an undoable Clear, autosave to `localStorage` after every stroke, and an Export panel that produces a 256×256 PNG with transparent empty cells plus a download link. Phone-first at 390×844 with no pinch-zoom required. No layers, no frames, no zoom, no custom palettes, no import, no URL sharing, no cloud — the exclusion list in the issue spec is binding.

## Architecture sketch

One file. One `<canvas>`. One array of truth.

### State

```js
const PALETTE = Object.freeze([
  null,      // 0 = empty (transparent)
  '#17181a', '#4a4f57', '#8c929b', '#ffffff',
  '#b23a34', '#e0714a', '#f0c04c', '#7fae4a',
  '#2f7d5c', '#3f9aa8', '#3f63b5', '#7a54a3'
]);            // indices 1..12 are paint

// Parallel to PALETTE: what a swatch announces to a screen reader.
const COLOUR_NAMES = Object.freeze([
  null,
  'Black', 'Dark grey', 'Grey', 'White',
  'Red', 'Orange', 'Yellow', 'Green',
  'Dark green', 'Teal', 'Blue', 'Purple'
]);

const state = {
  pixels: new Uint8Array(256),  // row-major, pixels[y*16 + x] = palette index
  color:  1,                    // selected palette index, 1..12
  erase:  false,
  mirror: false,
  undo:   null                  // Uint8Array copy taken before the last stroke, or null
};
```

`state.pixels` is the single source of truth. Nothing else stores board content — no per-cell DOM, no offscreen bitmap, no dirty-rect list. `render()` reads `state` and repaints the whole canvas; 256 fills is nothing, and skipping the cache sidesteps the stale-backing-store class of bug entirely.

Layout truth lives in one small record: `layout = { cell, cssPx, dpr }`. The canvas backing store is rebuilt only when `cssPx` or `dpr` changes, and both are part of the key.

### Functions

- `loadState()` — reads `localStorage['sprite-stamp/v1']` inside `try/catch`, parses inside `try/catch`, requires `v === 1`, validates every field. `pixels` is scanned whole before anything is written: one character outside hex 0–12 rejects the whole string and the board stays blank, rather than that cell being zeroed. Returns nothing: it mutates the module-level `state` in place, and simply leaves it at its blank defaults when the record is absent, oversized, unparseable or of another version. Never throws.
- `saveState()` — serialises `pixels` to 256 hex chars plus `color`, `erase`, `mirror`; `try/catch` around `setItem` (quota, private mode).
- `measure()` — computes `cell` from the viewport, sets the canvas CSS size to `cell * 16` px, sets the backing size to `cell * 16 * dpr`, resets the transform, updates `layout`, and publishes the CSS size as the `--board` custom property so every row is exactly as wide as the board. Height decides the cell only while it can clear the 16 px floor (a 256 px board); below that the cell sits at the floor, capped by the width, and the page scrolls. There is no upper clamp: `.wrap`'s 420 px max-width already caps the cell at 26.
- `render()` — checker for empty cells, flat fill for painted cells, a hairline grid stroked twice in alternating light and dark dashes so it reads over any fill, dashed mirror axis when `state.mirror`.
- `cellFromPointer(ev)` — `getBoundingClientRect()` every time; returns `{x, y}` in board coordinates, bounded to 64 cells outside the board so a drag that leaves the board can still be walked as a line, or `null` only when `rect.width` is 0.
- `poke(x, y, value)` — bounds-checked single-cell write to `pixels`, returns whether anything changed. The only function that mutates board content.
- `write(x, y, value)` — one `poke` for the cell, plus a second for the mirror twin at `15 - x` when mirror is on. Every write to the board goes through here.
- `line(x0, y0, x1, y1, value)` — Bresenham, calls `write` per cell.
- `beginStroke / moveStroke / endStroke` — pointer capture, previous-cell tracking, undo snapshot on begin, save + status on end. A sample outside the board is kept rather than dropped, so the line to it still paints the cells it crosses on the way out. Only a primary button starts a stroke, and `contextmenu` is suppressed on the canvas.
- `restingText()` — the status line's resting text, derived from `state.erase` and `state.mirror` rather than being a constant, so the page's only instruction is true in every mode and the one sentence that explains Mirror rests there instead of expiring. Erase leads when both are on.
- `setColor(i) / toggleErase() / toggleMirror() / undo() / clearBoard()` — state transitions, each ending in `saveState()`. The three that change board content or the axis — `toggleMirror()`, `undo()`, `clearBoard()` — also call `render()`; `setColor()` and `toggleErase()` change no pixel, so they only resync the controls and the status line. `clearBoard()` does nothing at all — no record, no announcement — on an already-empty board.
- `onActivate(el, fn)` — activates a control from `click`, and additionally from `pointerup` for a non-primary touch pointer, because Chromium synthesises no click for a second finger while a first one rests on the board. The following click is swallowed once so nothing fires twice.
- `toPng()` — 256×256 offscreen canvas, 16×16 `fillRect` blocks, empty cells left untouched so they stay transparent, returns a data URI.
- `openExport() / closeExport() / trapTab(ev)` — panel show/hide, Escape and backdrop close, a wrap-around focus trap so the page behind the `aria-modal` panel is unreachable, and focus restored to `#export` on close. `openExport()` ends any live stroke first, so a second finger cannot open a picture of a board that is still changing, and it opens the scrim at its own top with `focus({preventScroll: true})`.
- `watchDpr() / onDprChange()` — a `matchMedia('(resolution: Ndppx)')` subscription that re-lays-out when the device pixel ratio changes, re-armed at the new ratio each time.
- `buildUI()` — creates the palette buttons and wires every listener. Called once.

### Files

- `index.html` — everything.
- `README.md`, `LICENSE` (MIT), `screenshot.png`, `PROJECT.md`.

## Done-map

**Increment 1 — skeleton and render** (v0 foundation)
- `done` `index.html` shell, inline CSS, page structure, canvas element
- `done` `PALETTE`, `state`, `measure()`, `render()` drawing a hardcoded test sprite
- `done` resize / orientationchange handling keyed on `{cssPx, dpr}`

**Increment 2 — painting** (v0 is real once this lands)
- `done` `cellFromPointer` with rect-based hit test
- `done` `beginStroke` / `moveStroke` / `endStroke` with pointer capture
- `done` Bresenham interpolation between moves
- `done` `touch-action: none` on the canvas only

**Increment 3 — tools**
- `done` palette buttons, selection ring, `setColor`
- `done` Erase toggle
- `done` Mirror toggle, paint-time twin write, dashed axis in `render()`

**Increment 4 — memory**
- `done` `saveState()` with `try/catch`
- `done` `loadState()` total parse and validation
- `done` one-step Undo, undoable Clear, disabled states

**Increment 5 — export**
- `done` `toPng()` at 256×256 with transparency
- `done` export panel, download link, Escape/backdrop close, status line

**Increment 6 — ship**
- `done` 390×844 pass: reach, tap sizes, rotate, no horizontal scroll
- `done` hostile-localStorage pass over all eight bad values
- `done` LICENSE (MIT), present since `682346b`
- `done` `screenshot.png` taken from the shipped build at phone width, mirror on
- `todo` README verified as a cold sequence from a fresh clone
- `todo` repo description, topics, Pages enabled, demo link loads

**Increment 7 — cycle-2 polish** (the cycle-1 defect list, D1–D23)
- `done` contrast: `--line` and `--muted` retoned, toggles and the primary export action given their own weight
- `done` layout: board derived from the width once height runs short, every row aligned to `--board`, column centred on both axes
- `done` export panel: reachable at landscape phone heights, square preview, focus trapped, download confirmed inside the panel
- `done` pointer: second-finger taps activate controls, right and middle clicks no longer paint
- `done` render and state: one checker square per cell, state-derived status line, `v === 1` required, silent Clear on an empty board, swatches named by colour
- `done` `origin` remote re-supplied

**Increment 8 — cycle-3 defects** (the cycle-2 defect list, L1–L17)
- `done` layout: `MIN_CELL` raised to 16 so the board never drops under 256, and the short-viewport fallback now takes the height budget as a cap, so the board can no longer grow as the viewport shrinks (it used to jump 192 → 352 between h=536 and h=520). Body side padding trimmed to 12 px so 256 still clears a 280 px viewport without a horizontal scrollbar
- `done` landscape: a 844×390 phone now opens on a 256 px board with the palette starting at y=340, inside the fold, instead of a 416 px headless grid
- `done` export panel: opens at its own top (`scrollTop = 0`, `focus({preventScroll: true})`) and the preview is capped at `42vh` of width, so at 844×390 the title, whole preview, download link and Close are all on screen without scrolling
- `done` export panel: a live stroke is ended before the PNG is built, so a second finger cannot open a stale picture of a board that keeps painting
- `done` pointer: a drag that leaves the board is clipped, not dropped — raw cell coordinates are tracked (bounded to 64 cells outside the board) and `poke()` skips whatever falls outside 0–15
- `reverted` render: the review asked for the mirror axis's two-tone dash trick on the hairline grid, to lift its contrast over the twelve palette colours and both checker tones from 1.00–1.28:1 to 2.17–4.49:1. It was built, measured, looked at, and reverted. The spec says the grid "is allowed to disappear over dark pixels, and that is correct — adjacent dark pixels should merge"; the two-tone version chops the sprite into 256 labelled boxes and makes the board read as graph paper. The axis must read over any colour because it is an explanation. The grid must not, because it is scaffolding. A measured finding that contradicts the spec is a finding about the spec
- `done` controls: hover no longer darkens a disabled Undo; the selection ring stays on the armed colour while Erase is on; Undo hands focus to Clear before disabling itself
- `done` status: the Mirror sentence is now the resting text while Mirror is on, so it no longer expires after 2.4 s
- `done` state: `loadState()` rejects a `pixels` string with any character outside hex 0–12 instead of zeroing that cell
- `done` `MAX_CELL` deleted as dead code — `.wrap`'s 420 px max-width caps the cell at 26, so a 28 clamp could never fire
- `done` docs: README "What it does" back inside the 2–5 sentence cap with the screenshot description moved to alt text, PROJECT.md realigned with the code
- `done` recapture `screenshot.png`: the 390×844 layout is unchanged (352 px board, 6×2 palette) but the resting status line now reads the Mirror sentence, so the image was re-rendered against the shipped build

## Open threads

- Export size fixed at 256×256 rather than a native 16×16 file. A 16×16 PNG is the purer artifact but shows as a speck in a file browser. Decision stands; the README states the size.
- `download` on a `data:` URI is unreliable on iOS Safari. Mitigation shipped in v0: the panel shows the image so press-and-hold works. If a critic finds the link silently failing, the fallback text is already there.
- Exact palette hexes are provisional. Constraint to hold: the white entry must stay distinguishable from the empty-cell checker, and the selection ring must read on both the black and the white swatch.
- The board is `max(16, min(byWidth, max(16, byHeight)))` cells: never under the 256 px floor, never wider than the column, and never larger than it was at a taller viewport. Below the floor the page scrolls. Swept across widths 280–1440 × heights 360–900 in 8 px steps, 10001 combinations: square, a whole multiple of 16, ≥ 256, no horizontal scroll, monotonically non-increasing as the height shrinks.
- The export preview is capped at `42vh` of width so a landscape phone can see the whole panel at once. That cost the preview its flush alignment with the buttons under it on short viewports, and it is deliberate: the panel's job is showing the sprite before you save it. At 740×360 — shorter still — the Close button is 14 px below the fold and needs a small scroll; the title, preview and download link are all in view.
- The hairline grid measures 1.00:1 over black and 1.15–1.28:1 over the other eleven colours — under the 3:1 an ordinary UI boundary wants, and deliberately so. The board is a picture, not a table; a stranger reads the sprite, not the cells. If a future cycle wants the cells countable, the answer is a toggle, not a louder default, and a toggle is out of v0's scope.
