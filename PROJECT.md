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

- `loadState()` — reads `localStorage['sprite-stamp/v1']` inside `try/catch`, parses inside `try/catch`, requires `v === 1`, validates every field. Returns nothing: it mutates the module-level `state` in place, and simply leaves it at its blank defaults when the record is absent, oversized, unparseable or of another version. Never throws.
- `saveState()` — serialises `pixels` to 256 hex chars plus `color`, `erase`, `mirror`; `try/catch` around `setItem` (quota, private mode).
- `measure()` — computes `cell` from the viewport, sets the canvas CSS size to `cell * 16` px, sets the backing size to `cell * 16 * dpr`, resets the transform, updates `layout`, and publishes the CSS size as the `--board` custom property so every row is exactly as wide as the board. Height decides the cell only while it can clear the 12 px floor; below that the width decides and the page scrolls.
- `render()` — checker for empty cells, flat fill for painted cells, hairline grid, dashed mirror axis when `state.mirror`.
- `cellFromPointer(ev)` — `getBoundingClientRect()` every time; returns `{x, y}` or `null` when outside or when `rect.width` is 0.
- `poke(x, y, value)` — bounds-checked single-cell write to `pixels`, returns whether anything changed. The only function that mutates board content.
- `write(x, y, value)` — one `poke` for the cell, plus a second for the mirror twin at `15 - x` when mirror is on. Every write to the board goes through here.
- `line(x0, y0, x1, y1, value)` — Bresenham, calls `write` per cell.
- `beginStroke / moveStroke / endStroke` — pointer capture, previous-cell tracking, undo snapshot on begin, save + status on end. Only a primary button starts a stroke, and `contextmenu` is suppressed on the canvas.
- `restingText()` — the status line's resting text, derived from `state.erase` rather than being a constant, so the page's only instruction is true in both modes.
- `setColor(i) / toggleErase() / toggleMirror() / undo() / clearBoard()` — state transitions, each ending in `render()` and `saveState()`. `clearBoard()` does nothing at all — no record, no announcement — on an already-empty board.
- `onActivate(el, fn)` — activates a control from `click`, and additionally from `pointerup` for a non-primary touch pointer, because Chromium synthesises no click for a second finger while a first one rests on the board. The following click is swallowed once so nothing fires twice.
- `toPng()` — 256×256 offscreen canvas, 16×16 `fillRect` blocks, empty cells left untouched so they stay transparent, returns a data URI.
- `openExport() / closeExport() / trapTab(ev)` — panel show/hide, Escape and backdrop close, a wrap-around focus trap so the page behind the `aria-modal` panel is unreachable, and focus restored to `#export` on close.
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
- `todo` `screenshot.png` taken from the shipped build at phone width, mirror on
- `todo` README verified as a cold sequence from a fresh clone
- `todo` repo description, topics, Pages enabled, demo link loads

**Increment 7 — cycle-2 polish** (defect list `defects-cycle1.md`, D1–D23)
- `done` contrast: `--line` and `--muted` retoned, toggles and the primary export action given their own weight
- `done` layout: board derived from the width once height runs short, every row aligned to `--board`, column centred on both axes
- `done` export panel: reachable at landscape phone heights, square preview, focus trapped, download confirmed inside the panel
- `done` pointer: second-finger taps activate controls, right and middle clicks no longer paint
- `done` render and state: one checker square per cell, state-derived status line, `v === 1` required, silent Clear on an empty board, swatches named by colour
- `done` `origin` remote re-supplied

## Open threads

- Export size fixed at 256×256 rather than a native 16×16 file. A 16×16 PNG is the purer artifact but shows as a speck in a file browser. Decision stands; the README states the size.
- `download` on a `data:` URI is unreliable on iOS Safari. Mitigation shipped in v0: the panel shows the image so press-and-hold works. If a critic finds the link silently failing, the fallback text is already there.
- Exact palette hexes are provisional. Constraint to hold: the white entry must stay distinguishable from the empty-cell checker, and the selection ring must read on both the black and the white swatch.
- The cell used to come from `min(width, height)`, which on a short viewport bought a 192 px board *and* a scrollbar. It now falls back to the width alone once the height-derived cell would drop under the 12 px floor, and the page scrolls — swept across widths 280–1440 × heights 390–900, square and a whole multiple of 16 at every combination.
- The export panel is taller than a landscape phone viewport: at 390 px of height the whole preview and the whole Close button cannot both be on screen at once, though every part of the panel is reachable by scrolling. Shrinking the preview on short viewports would cost its alignment with the buttons. Left as is.
