---
name: geastack-embedded-driver-development
description: Add or modify GeaStack embedded chip drivers, ESP32 chip bindings, the chip catalog, board composition, host facades, app-facing TypeScript declarations, nativeSources, and hardware-gated CMake sources. Use when adding a display, touch controller, IMU, PMU, audio codec, GPS, camera, storage, WiFi/BLE/media/input binding, or any new native API exposed to GeaStack apps.
---

# GeaStack Embedded Driver Development

## Decide The Layer

Use the thinnest correct layer:

- Generic chip logic belongs in `@geastack/chips` —
  `core/packages/chips/<category>/<chip>/` in the core repo.
- The chip's catalog entry belongs in `core/packages/chips/catalog.json`.
- ESP-IDF adapters belong in
  `targets/targets/esp32/chip_bindings/<category>/<chip>.cpp`.
- Board composition belongs in the concrete target folder, or in a composed
  target definition (`$geastack-embedded-custom-boards`).
- Platform-universal host facades belong in
  `targets/targets/esp32/host_sources.cmake`.
- Hardware-gated facades stay opt-in in the target's `main/CMakeLists.txt`.
- App-facing TypeScript declarations belong in
  `core/packages/core/index.d.ts` (`@geastack/core`).

Read `references/driver-layers.md` before adding files.

## Generic Chip Rules

Keep generic chip code target-independent:

- Do not include ESP-IDF, FreeRTOS, Pico SDK, or board headers.
- Put register maps, init sequences, state machines, sample conversion, and
  packet formats here.
- Define small bus/runtime interfaces for I2C, SPI, GPIO, sleep/delay,
  interrupts, or DMA as needed.
- Add focused unit tests when behavior can be tested without hardware.

## The Chip Catalog

A chip is only selectable by a composed board once it has a `catalog.json`
entry. The shape is:

```json
{
  "co5300": {
    "label": "CO5300 AMOLED display controller",
    "category": "display",
    "interfaces": ["qspi"],
    "sources": ["displays/co5300/co5300.cpp"],
    "headers": ["displays/co5300/co5300.h"],
    "adapters": {
      "esp32-idf": {
        "mcus": ["esp32s3"],
        "bindingSources": ["chip_bindings/displays/co5300.cpp"]
      }
    },
    "configuration": [
      { "path": "width", "label": "Display width", "type": "integer", "min": 1, "max": 4096 },
      { "path": "spiHost", "label": "SPI host", "type": "choice", "values": ["spi2", "spi3"], "default": "spi2" },
      { "path": "pins.cs", "label": "Chip-select pin", "type": "pin" }
    ]
  }
}
```

Categories in use: `display`, `touch`, `power`, `imu`, `audio`, `rtc`. A chip
with no `adapters` entry is portable source only — `gea chips add` refuses it
with "has no `<adapter>` binding" until a binding exists. The `configuration`
list is what the setup wizard and `gea chips add --set` prompt for, and what
CMake validates (including GPIO collision checks) before generating `board.h`.

## Target Binding Rules

Keep bindings thin:

- Include platform headers only in the binding layer.
- Adapt board buses, pins, interrupts, tasks, and DMA to the generic chip
  interface.
- Read board composition from the active target, not hardcoded globals.
- Link one concrete binding source per target instead of adding `#if` chains to
  shared display/input code.
- Fail clearly when hardware is optional or absent.

## Host Facades

When an app should call a new API:

1. Add or extend the native C++ host facade.
2. Add TypeScript declarations in `@geastack/core` (`index.d.ts`).
3. Ensure the compiler's host-binding analysis sees the binding — the analysis
   result is what makes the CLI link the network, BLE, or audio subsystem
   (`$geastack-embedded-build-flags`). A binding the analysis misses produces a
   link error at best and silently missing hardware at worst.
4. Add the facade source to `targets/targets/esp32/host_sources.cmake` only if
   every ESP32 board can support it.
5. Otherwise append it in the target's `main/CMakeLists.txt` for boards with
   that hardware.
6. Add tests for linker coverage and API shape.

For app-owned native helpers, use `gea.nativeSources` (app-relative or
package-resolved paths) and `gea.compilerPlugins`. ESP-IDF components belong
in `gea.targets.esp32.componentDirs`; add `componentRequires` when app sources
need their public headers/defines. Use `gea.defines` for macros that must agree
across app and framework, and app sdkconfig/linker fragments for app-specific
build requirements. See `$geastack-embedded-app-builder`.

## Verification

Use increasingly real checks:

- Compile generic chip tests or small native tests.
- Run the target metadata tests if CMake sources or manifests change.
- Run `npx gea chips info <id>` to confirm the catalog entry resolves.
- Build a representative app that uses the binding.
- Flash and monitor when the change touches buses, interrupts, display,
  storage, audio, camera, or power.
- Capture a screenshot or `gea devctl i2cscan` for bus/display changes.
- Check heap and stack if adding buffers, tasks, DMA, networking, or media.
