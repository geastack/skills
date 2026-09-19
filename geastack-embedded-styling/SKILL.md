---
name: geastack-embedded-styling
description: Design, implement, and review CSS, inline Style objects, fonts, layout, transforms, overflow, responsive sizing, launcher icons, and visual polish for GeaStack embedded TSX apps. Use when styling GeaStack apps for small screens, fixing layout/rendering issues, adding fonts or image assets, tuning CSS for device pixel ratio, or checking whether a CSS feature is supported by the embedded renderer.
---

# GeaStack Embedded Styling

## Workflow

Inspect the app before styling:

- Read the entrypoint to see which CSS files are imported.
- Read components and stores to understand dynamic classes, inline `style`,
  data attributes, and measured values.
- Read existing examples for the same target class when possible.
- Read `references/style-surface.md` when unsure whether a property is
  supported.

Keep embedded layouts deterministic:

- Use `width: 100vw`, `height: 100vh`, and `overflow: hidden` for full-screen
  app roots.
- Use fixed or bounded dimensions for repeated controls, rows, badges, panels.
- Prefer CSS classes for stable visual styling; use inline `style` for
  state-derived values.
- Keep text sizes realistic for the target display; avoid viewport-scaled fonts.
- Use `@font-face` with local assets already in the app when the target needs
  custom typography.

## Supported Style Habits

The `Style` interface in `@geastack/core` is the authority. It supports:

- Layout: `display` (`block | flex | grid | none`), `flexDirection`, `flexWrap`,
  `justifyContent`, `alignItems`, `alignSelf`, `flex`, `gap`, `width`,
  `height`, `minWidth`, `minHeight`, `maxWidth`, `maxHeight`.
- Spacing: `padding` plus `paddingTop/Right/Bottom/Left`, `margin` plus
  `marginTop/Right/Bottom/Left`.
- Positioning: `position: relative | absolute`, `top`, `left`, `right`,
  `bottom`, `zIndex`.
- Visuals: `backgroundColor`, `color`, `opacity`, `blinkInterval`,
  `borderWidth`, `borderColor`, `borderRadius` plus the four corner radii.
- Text: `fontFamily`, `fontSize`, `textAlign` (`left | center | right`).
- Scrolling/clipping: `overflow`, `overflowX`, `overflowY`.
- Motion and transforms: `transform`, `rotate`, `scale`, `translateX`,
  `translateY`, `transformOrigin`, `filter`.

If a CSS feature is not present in the type declarations or nearby tests, verify
it in the renderer before depending on it.

## Embedded Constraints

Avoid web-only assumptions:

- Do not rely on browser layout APIs unless declared in `@geastack/core`. There
  is no `dom` lib: app tsconfigs use `"types": []` and `"jsx": "preserve"`, and
  every global comes from `@geastack/core`'s ambient declarations.
- Avoid huge shadows, blur-heavy layers, and unnecessary translucent overdraw on
  hardware targets.
- Keep absolute-positioned overlays clipped and inside the viewport.
- For virtual lists, let `<virtual-list>` measure row height from the rendered
  slot when possible.
- For large static text over changing regions, consider
  `Display.setTextRasterCache(true)` in the app. On e-paper/grayscale targets,
  `Display.setTextSolidBackdrop(0x000000)` pre-blends text against black.
  Pass the actual solid backdrop as `0xRRGGBB` for other colors.

## Assets

For fonts and images:

- Keep paths relative to the app stylesheet or component.
- Make image files part of the app directory so CMake asset collection sees
  PNG/JPG/JPEG/GIF changes.
- Use launcher icons through `gea.icons`; do not hide icon changes in CSS.
- If a font or image change does not appear on hardware, rebuild the native
  target — a web build alone will not update the firmware's font tables
  (`GEA_EMBEDDED_HAS_GENERATED_FONTS`).

## Verification

Use a layered check:

```sh
npm run check
npx gea build --board <alias> --dry-run
npx gea run --board <alias> --app <id>
npx gea screenshot shot.png --board <alias>
npx gea devctl node <class> --board <alias>    # what the renderer actually laid out
npx gea devctl hit <x> <y> --board <alias>
```

`gea devctl node` and `hit` are the fastest way to tell a styling bug from a hit-testing
bug. For renderer changes in the core repo, run the focused
native/render pipeline test that matches the changed feature before broad tests.
