# Composition Reference

## Catalog Entries

From `@geastack/chips/catalog.json` (schema version 1). "esp32-idf" means the
chip has an adapter binding and can be selected by a composed ESP32-S3 board.

| chip | category | interface | esp32-idf | configuration |
| --- | --- | --- | --- | --- |
| `co5300` | display | `qspi` | yes | `width`, `height`, `spiHost` (`spi2`\|`spi3`, default `spi2`), `pins.cs`, `pins.pclk`, `pins.data0`-`data3`, `pins.reset`, `pins.te` (optional) |
| `ft3168` | touch | `i2c` | yes | `pins.reset`, `pins.interrupt` |
| `axp2101` | power | `i2c` | yes | none |
| `qmi8658` | imu | `i2c` | yes | none |
| `es8311` | audio | `i2s` | yes | `pins.mclk`, `pins.bclk`, `pins.ws`, `pins.dout`, `pins.din`, `pins.powerAmplifier` (optional) |
| `sh8601` | display | `qspi` | no | none yet |
| `rm690b0` | display | `qspi` | no | none yet |
| `ft6336` | touch | `i2c` | no | none yet |
| `eta6098` | power | `i2c` | no | none yet |
| `pcf85063` | rtc | `i2c` | no | none yet |
| `mtk_factory_imu` | imu | `host` | no | none yet |

Run `npx gea chips info <id>` for the live entry — the catalog is the authority,
this table is a snapshot.

## Definition Fields

Only `id`, `extends` and `mcu` are required. Every chip role, the I2C bus,
storage and controls are optional: a definition describes the hardware the board
has, so a bare module declares almost nothing.

| field | meaning |
| --- | --- |
| `id` | the target id; matches the `target` in the board alias |
| `extends` | the composable base target. `esp32-s3` today |
| `adapter` | `esp32-idf` |
| `mcu` | `esp32s3` |
| `flashSize` | this module's flash, e.g. `"16MB"`. Overrides the base's |
| `partitions` | Named partition object (no CSV path or `data` payloads). App `partitionsByTarget[targetId]` or `partitions` overrides it |
| `console.transport` | where the log comes out: `uart`, `usb-serial-jtag` or `none`. Follows the module's wiring, not the base's |
| `console.uart` / `.baud` | UART number (default 0) and baud (default 115200) when `transport` is `uart` |
| `psram.mode` / `.speed` | `octal`, `quad` or `none`, and 40/80/120 MHz. Declare the actual wiring; omission inherits base settings, which can corrupt a differently wired module |
| `canvas.width` / `.height` | explicit offscreen surface for a board with no panel; omit `canvas` for no display, with no default surface |
| `buses.i2c.sda` / `.scl` | the shared I2C bus every `i2c` chip hangs off. Required only when an `i2c` chip is selected |
| `chips.<category>.driver` | catalog chip id (`controller` is accepted too) |
| `chips.<category>.interface` | one of the chip's catalog interfaces |
| `chips.<category>.pins.*` | per-chip pins from its `configuration` list |
| `chips.display.width` / `.height` / `.spiHost` | display geometry and SPI host |
| `storage.microSD.interface` | e.g. `sdmmc-1bit` |
| `storage.microSD.pins.clk` / `.cmd` / `.data0` | SD pins |
| `controls.launcherButton.pin` / `.activeLevel` | the physical button that reaches `Input.consumeBackButton()` |

Pins are integers 0 through 48. A pin may be `null` or `"none"` only where the
catalog marks it optional.

## Headless Boards

Omit `chips.display` to omit the panel. Declare `canvas: { width, height }`
for an offscreen surface backed by `targets/esp32/display_headless.cpp`: it
runs the UI and frame loop and can be captured by `gea screenshot`.

Omit both `chips.display` and `canvas` for a board with no display at all.
The generated `GEA_EMBEDDED_NO_DISPLAY=1` suppresses framebuffer allocation,
app-tree initialization and the frame loop. Native boot hooks and their tasks
remain available; TS/TSX UI initialization and rAF are not a service loop for
this mode. No default 240x240 or 410x502 surface is allocated.

Declare `console` too. A devkit reached through a USB-UART bridge and one with
its native USB-Serial/JTAG port plugged in are different boards, and a headless
board's log is the only thing you have to look at. Inheriting the base's
transport produces firmware that runs perfectly while logging into a port
nobody is connected to — indistinguishable from a board that never booted.

```json
{
  "id": "bare-s3",
  "extends": "esp32-s3",
  "mcu": "esp32s3",
  "flashSize": "32MB",
  "psram": { "mode": "octal", "speed": 80 },
  "canvas": { "width": 240, "height": 240 },
  "console": { "transport": "uart", "uart": 0, "baud": 115200 },
  "controls": { "launcherButton": { "pin": 0, "activeLevel": 0 } }
}
```

This example matches the base’s 32MB flash layout. The shipped
`esp32-s3-devkit-n16r8` definition demonstrates 16MB flash, octal PSRAM, UART
console, a fitting partition table, and no canvas. Use its table when composing
that module; changing `flashSize` alone does not shrink the base’s table.

Omitting `chips.touch` likewise selects `targets/esp32/touch_absent.cpp`, which
reports "nothing is touching" forever rather than making every caller special-
case a board with no panel. `i2c.cpp`, `launcher_button.cpp` and
`sdcard_mount.cpp` are already `GPIO_NUM_NC`-tolerant, so a board with none of
those peripherals compiles and boots without further work.

## Validation Order

1. `extends` resolves to a `composable: true` target.
2. Each `chips.<role>` entry that is *present* names a catalog chip of that
   category. An omitted role means the board has no such chip.
3. The chip's `interface` is one it declares.
4. The chip has an adapter binding for the MCU.
5. Every required `configuration` path is present and in range.
6. No GPIO is claimed twice across the chips, buses, storage and controls the
   board actually declares. An absent role owns no GPIO and cannot collide.

Only then does the CLI generate `board.h` and `target.cmake` into
`.gea/build/<target>/app-builds/<app>/gea-custom-target/`, before CMake
compiles the selected sources. App `gea.defines` takes precedence over the
generated board macros, including canvas dimensions.

## Where Things Live

| file | role |
| --- | --- |
| `.gea/boards.json` | alias -> `target`, `targetDefinition`, `appPlatform` |
| `.gea/targets/<alias>.json` | the composed definition, app-local |
| `.gea/build/<target>/app-builds/<app>/` | sdkconfig and firmware; generated `board.h` and `target.cmake` under `gea-custom-target/` |
| `@geastack/chips/catalog.json` | what can be selected at all |
| `targets/targets/<id>.json` | the composed definition, once promoted |
| `targets/targets.json` | the registry entry that names it |

## Promoting a Composed Board Into the Library

A definition under `.gea/targets/` describes a module for one project. The same
JSON shipped in `@geastack/targets` describes it for every project, and the
alias then shrinks to a target id and a USB serial -- no per-app files at all.

Add the JSON to `targets/targets/<id>.json` and register it in
`targets/targets.json` with a `definition` field beside the usual metadata:

```json
"esp32-s3-devkit-n16r8": {
  "adapter": "esp32-idf",
  "targetPath": "targets/esp32-s3-touch-amoled-2.06",
  "definition": "targets/esp32-s3-devkit-n16r8.json",
  "flashSize": "16MB",
  "appPlatform": "esp32",
  "idfTarget": "esp32s3",
  "esptoolChip": "esp32s3"
}
```

`targetPath` is the project the board borrows -- the same one its `extends` base
points at, since a composed board contributes a definition rather than a source
tree. `definition` is relative to the targets package root.

An alias may still carry its own `targetDefinition`, and it wins: the registry
describes the module in general, while a project-local definition describes the
one on somebody's desk. A registry entry naming a definition that was never
shipped fails closed rather than silently falling back to the base.
