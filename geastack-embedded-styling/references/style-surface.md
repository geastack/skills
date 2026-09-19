# Style Surface

## Value Types

`StyleLength` supports numbers and the string forms `px`, `%`, `vw`, `vh`,
`dvw`, `dvh`, plus `calc(...)`, `min(...)`, `max(...)`, and `clamp(...)`.
`StyleBox` is a `StyleLength` or a string (for shorthand boxes such as
`padding: '8px 12px'`).

`class` accepts a string or a class map:
`Record<string, string | number | boolean | null | undefined>`.

`data-*` attributes are typed through `DataAttributes` and accept
`string | number | boolean`.

## Common CSS Shape

```css
html,
body {
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100%;
  overflow: hidden;
}

.app {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}
```

## Font Pattern

```css
@font-face {
  font-family: 'Inter';
  src: url('../assets/fonts/Inter-Regular.ttf');
}
```

The bundled counter starter ships `assets/fonts/Inter-Regular.ttf`; keep font
files inside the app directory so the native build collects them.

## Inline Style Pattern

```tsx
<div
  class="watch-root"
  style={{
    width: '100vw',
    height: '100vh',
    backgroundColor: '#000000',
    overflow: 'hidden',
    fontFamily: 'Inter',
    fontSize: '15px'
  }}
/>
```

## Intrinsic Elements

`body`, `div`, `p`, `h1`-`h6`, `button`, `input`, `textarea`, `img`, `audio`,
`canvas`, `camera`, `virtual-list`, and the inline-level text tags `span`, `a`,
`b`, `i`, `em`, `strong`, `small`, `label`, `code`, `u`, `sub`, `sup`, `mark`.

Inline-level tags are ordinary view nodes the layout flows in a row rather than
a column.

## Device Pixel Ratio

Use logical CSS pixels in app code. For app-wide scaling, set positive
`gea.cssDevicePixelRatio` in `package.json`. The ESP32 CLI passes it to both
`GEA_EMBEDDED_CSS_DEVICE_PIXEL_RATIO` (font generation) and
`GEA_EMBEDDED_CSS_LAYOUT_DEVICE_PIXEL_RATIO` (layout), so glyph atlases and
layout agree. Without it, board font settings and the default layout ratio
remain in effect.

`Display.getDevicePixelRatio()` and `setDevicePixelRatio()` adjust runtime
layout; they do not regenerate font atlases. Rebuild firmware when changing
the manifest ratio or fonts.
