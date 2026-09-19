# Canvas API

## Display

- `Display.ctx`: full-screen `CanvasRenderingContext2D`
- `Display.width`, `height` (logical, after orientation), `nativeWidth`,
  `nativeHeight` (physical panel)
- `Display.setFrameRate(fps)`, `setFrameIntervalMs(ms)`, `getFrameRate()`,
  `getFrameIntervalMs()`
- `Display.setFlushConfig({ rows, depth })`
- `Display.setMemoryConfig({ commandBufferCommands?, backgroundCache? })`
- `Display.invalidate()`
- `Display.setVSync(on)`
- `Display.setTextRasterCache(on)`
- `Display.setTextSolidBackdrop(rrggbb)` (negative value disables)
- `Display.setAA(samples)`
- `Display.getBrightness()`, `setBrightness(0-100)`
- `Display.orientation` / `setOrientation()`, `supportedOrientations` /
  `setSupportedOrientations()`, `autoRotate` / `setAutoRotate()`
- `Display.pixelFormat` / `setPixelFormat()`, `panelPixelFormat`,
  `supportedPixelFormats` — `'rgb565' | 'rgb888' | 'rgb8888' | 'argb8888'`
- `Display.getDevicePixelRatio()`, `setDevicePixelRatio()`
- `Display.setEpaperRefreshConfig(config)`, `Display.epaperFullRefresh()`

`DisplayEpaperRefreshConfig`: `fullRefreshEveryPartials`,
`fullRefreshHardCapPartials`, `fastStreakWindowMs`, `partialLut`, `fastLut`,
`grayscale`, `fullOnCover`. Omitted fields keep their current value; no-op on
boards without an e-paper panel.

## Context

- State: `fillStyle`, `strokeStyle`, `globalAlpha`, `lineWidth`, `font`,
  `textBaseline`, `textAlign`
- Frame: `clear()`, `flush()`, `beginBatch()`, `endBatch()`
- Rects: `clearRect`, `fillRect`, `strokeRect`
- Circles: `fillCircle`, `strokeCircle`, `fillCircleRgb565`,
  `fillCirclesRgb565`, `fillCirclesRgb565Uniform`
- Triangles: `fillTriangleRgb565`, `fillTrianglesRgb565Sorted(x0s, y0s, x1s,
  y1s, x2s, y2s, colors, order, count?)` — one recorded command for a whole
  batch, `order` indexes the arrays in paint (depth) order. Colors are
  `0xRRGGBBAA` (`rgb`/`rgba`), not native `rgb565` values
- Paths: `beginPath`, `arc`, `moveTo`, `lineTo`, `closePath`, `fill`, `stroke`
- Text: `fillText`, `measureText`, `measureTextInkCenter`
- Images: `drawImage`, `drawImageCircle`, `drawImageRotated90CW`,
  `drawImageTiledX`

`fillText` anchors the TOP-LEFT at `(x, y)`; `textAlign` is a no-op. Center
horizontally with `measureText` and vertically with `measureTextInkCenter`.

## Image Helpers

- `loadImage(src, options?)`, `loadImageWithOpaque(bytes, opaque)`
- `loadImageFile(path, options?)`, `loadAssetImage(path)`
- `imageFromId(id)`
- `writeCacheFile`, `readCacheFile`, `listCacheFiles`, `readFileRange`

## Colors

- `rgb(r, g, b)` / `rgba(r, g, b, a)`: imported from `@geastack/core`, pack
  `0xRRGGBBAA`.
- `rgb565(r, g, b)`: ambient **global**, no import. Returns `Rgb565`, the
  board's native pixel value; a literal call folds at build time.

## Common Loop Notes

- Use `window.innerWidth` and `window.innerHeight` for logical viewport size.
- Keep the `requestAnimationFrame` callback as a named function or stable
  closure.
- Recompute labels only when their displayed value changes.
- Prefer typed arrays for particle, sprite, and chart coordinates. Preserve
  native circle palette types with `Rgb565[]`; numeric typed arrays lose the
  native-color marker. Sorted triangle batches instead use RGBA numeric arrays.
- On a composed board without a panel, declare an explicit `canvas` to use
  these APIs. Omitting both panel and canvas disables app initialization and
  the frame loop.
