---
name: geastack-embedded-canvas
description: Build and tune fast GeaStack canvas and Display.ctx apps, animation loops, image drawing, typed-array rendering, dirty-region behavior, frame rate, flush configuration, rgb/rgb565 colors, text measurement, and immediate-mode graphics. Use when creating or optimizing a canvas app, using the JSX canvas element, drawing with Display.ctx, fixing frame pacing, reducing allocations, tuning flush rows/depth, or porting browser canvas logic to GeaStack embedded hardware.
---

# GeaStack Embedded Canvas

## Choose The Surface

Use `Display.ctx` for full-screen, high-frequency immediate drawing. Use a JSX
`<canvas>` when the canvas is one leaf inside a retained UI and should
participate in layout, clipping, and event routing.

Read `references/canvas-api.md` for supported methods and common loops.

## Fast Full-Screen Pattern

`rgb565` is an ambient global — do not import it.

```ts
import { Display } from '@geastack/core'

Display.setFrameRate(60)
Display.setFlushConfig({ rows: 48, depth: 2 })

const ctx = Display.ctx
const w = Math.floor(window.innerWidth)
const h = Math.floor(window.innerHeight)

const xs = new Uint16Array(32)
const ys = new Uint16Array(32)
const colors: Rgb565[] = []
for (let i = 0; i < xs.length; i++) {
  xs[i] = 8 + Math.floor(Math.random() * Math.max(1, w - 16))
  ys[i] = 8 + Math.floor(Math.random() * Math.max(1, h - 16))
  colors.push(rgb565(255, 64, 32))
}

function frame(): void {
  ctx.beginBatch()
  ctx.clear()
  ctx.fillCirclesRgb565(xs, ys, 8, colors, xs.length)
  ctx.endBatch()
  requestAnimationFrame(frame)
}

requestAnimationFrame(frame)
```

## Performance Rules

- Allocate typed arrays, images, labels, and palettes outside the frame loop.
- Prefer `rgb565()` for stored hot palettes and `rgb()` for occasional numeric
  colors. An `Rgb565` is already in the panel's pixel format, so the numeric
  canvas APIs draw it with zero conversion.
- Use `beginBatch()` / `endBatch()` around a frame's draw calls.
- Avoid per-frame string concatenation, JSON parsing, image decoding, DOM
  queries, and object churn.
- Use `Display.invalidate()` once per frame during full-screen pans or
  animations where everything changes — it skips the dirty-rect diff and the
  previous-frame copy, both of which are waste when everything changed. Stop
  calling it when motion settles so idle gets the cheap diff back.
- Keep native-color circle palettes as `Rgb565[]`; a numeric typed array
  erases the native-color marker and can select a conversion path.
- Batch hundreds of triangles through `fillTrianglesRgb565Sorted` instead of a
  `fillTriangleRgb565` loop. This batch API takes `0xRRGGBBAA` colors (for
  example `rgb()` values in a `Uint32Array`), not native `rgb565()` values.
- Use `Display.setVSync(true)` only when the board wires a TE/VBlank line.
- Use `Display.setTextRasterCache(true)` for large static text (font-size ≳40px)
  overlapping animated content.
- On packed 4-bit grayscale targets (e-paper), `Display.setTextSolidBackdrop(0x000000)`
  pre-blends glyphs against black and stamps them as byte copies. Pass the
  actual solid backdrop as `0xRRGGBB` for other colors.
- Use `loadImage(..., { opaque: true })` for tile/sprite art when alpha edges do
  not need blending; `drawImageCircle` renders an opaque JPEG as a round tile
  without an alpha plane.

## Text Layout

`fillText` anchors the **top-left** at `(x, y)` and `textAlign` is a no-op on
this canvas. To center text:

```ts
const x = cx - ctx.measureText(label) / 2
const y = cy - ctx.measureTextInkCenter(label)
```

`measureTextInkCenter` returns the offset to the vertical center of the ink.
Subtracting half the font size instead sags, because the line box carries
ascender and descender the glyphs do not fill.

## Flush And Frame Tuning

Start with app-level runtime knobs:

```ts
Display.setFrameRate(60)
Display.setFlushConfig({ rows: 48, depth: 2 })
Display.setMemoryConfig({ commandBufferCommands: 4096, backgroundCache: true })
Display.setAA(1)
```

Move to target compile definitions only when the target itself needs a default:

- `GEA_EMBEDDED_DEFAULT_FRAME_INTERVAL_US`
- `GEA_EMBEDDED_DIRTY_REGION_MAX_RECTS`
- `GEA_EMBEDDED_FLUSH_ANCHOR_LEFT_PARTIAL_RECTS`
- `GEA_EMBEDDED_RENDER_PARALLEL_MIN_ROWS`
- `GEA_EMBEDDED_RENDER_PARALLEL_MIN_PIXELS`
- `GEA_EMBEDDED_FRAME_SCHEDULER_USE_ESP_TIMER`
- `GEA_EMBEDDED_FRAME_SCHEDULER_PERF_LITE`
- `GEA_EMBEDDED_FRAME_SCHEDULER_DROP_CATCHUP_FRAMES`

Use `$geastack-embedded-build-flags` before changing these globally.

## Verification

Check both correctness and pacing:

```sh
npx gea run --board <alias> --app <id>         # build, flash, monitor
npx gea logs --board <alias> --follow          # perf lines
npx gea screenshot frame.png --board <alias>
npx gea devctl state --board <alias>
npx gea heap-report run.log --out heap.html --map .gea/build/<target>/app-builds/<app>/gea_embedded.map
```

Look for `gea.perf:`, `perf:`, and `perf-lite:` lines in the log. Generate a
heap report when animation stutters grow over time.
