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

- `loadState()` — reads `localStorage['sprite-stamp/v1']` inside `try/catch`, parses inside `try/catch`, validates every field, returns a fully-formed state. Never throws. Bad or absent data returns a blank board.
- `saveState()` — serialises `pixels` to 256 hex chars plus `color`, `erase`, `mirror`; `try/catch` around `setItem` (quota, private mode).
- `measure()` — computes `cell` from the viewport, sets the canvas CSS size to `cell * 16` px, sets the backing size to `cell * 16 * dpr`, resets the transform, updates `layout`.
- `render()` — checker for empty cells, flat fill for painted cells, hairline grid, dashed mirror axis when `state.mirror`.
- `cellFromPointer(ev)` — `getBoundingClientRect()` every time; returns `{x, y}` or `null` when outside or when `rect.width` is 0.
- `write(x, y, value)` — bounds-checked write to `pixels`, plus the mirror twin at `15 - x` when mirror is on. The only function that mutates board content.
- `line(x0, y0, x1, y1)` — Bresenham, calls `write` per cell.
- `beginStroke / moveStroke / endStroke` — pointer capture, previous-cell tracking, undo snapshot on begin, save + status on end.
- `setColor(i) / toggleErase() / toggleMirror() / undo() / clear()` — state transitions, each ending in `render()` and `saveState()`.
- `toPng()` — 256×256 offscreen canvas, 16×16 `fillRect` blocks, empty cells left untouched so they stay transparent, returns a data URI.
- `openExport() / closeExport()` — panel show/hide, focus handling, Escape and backdrop close.
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
- `todo` `toPng()` at 256×256 with transparency
- `todo` export panel, download link, Escape/backdrop close, status line

**Increment 6 — ship**
- `todo` 390×844 pass: reach, tap sizes, rotate, no horizontal scroll
- `todo` hostile-localStorage pass over all eight bad values
- `todo` `screenshot.png` taken from the shipped build at phone width, mirror on
- `todo` README verified as a cold sequence
- `todo` LICENSE, repo description, topics, Pages enabled, demo link loads

## Open threads

- Export size fixed at 256×256 rather than a native 16×16 file. A 16×16 PNG is the purer artifact but shows as a speck in a file browser. Decision stands; the README states the size.
- `download` on a `data:` URI is unreliable on iOS Safari. Mitigation shipped in v0: the panel shows the image so press-and-hold works. If a critic finds the link silently failing, the fallback text is already there.
- Exact palette hexes are provisional. Constraint to hold: the white entry must stay distinguishable from the empty-cell checker, and the selection ring must read on both the black and the white swatch.
- Cell-size floor of 12 px means a 256 px board on very short viewports. Below that the page scrolls. Not yet tested on a 320 px-wide viewport.
