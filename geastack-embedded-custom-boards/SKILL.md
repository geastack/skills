---
name: geastack-embedded-custom-boards
description: Compose an app-local GeaStack board from the chip catalog - gea chips list/info/add/remove, .gea/targets board definitions, extends bases, buses, GPIO pins, flash size and partitions, microSD and launcher-button options, headless (no-display) boards, and the generated board.h. Use when bringing up custom or hand-wired hardware including bare devkit modules, choosing a display/touch/power/IMU/audio controller, changing pins on a composed board, or deciding between a composed board and a new built-in target.
---

# GeaStack Custom Board Composition

An application can compose its own board from a supported native base instead of
adding a built-in target to `@geastack/targets`. The definition is a JSON file
in the app; the CLI validates it against `@geastack/chips/catalog.json`,
generates `board.h` and `target.cmake` into the build directory, and compiles
only the selected portable sources and adapter bindings.

**Composed board or new built-in target?** Compose when the hardware is yours,
one-off, or still moving — nothing has to ship in a package, and `gea chips`
edits it. Add a built-in target (`$geastack-embedded-target-bringup`) when the
board needs sources, sdkconfig, or partitions that no composition option covers.

Wanting the board supported for everyone is no longer a reason to write a
target: a composed definition can itself be promoted into `@geastack/targets`,
after which an alias is a target id and a USB serial and no app carries a file
for it. See "Promoting a Composed Board Into the Library" in the composition
reference. Reach for a built-in target only when composition cannot describe the
hardware.

## Setup

The guided path writes both files for you:

```sh
npx gea setup
# -> "Custom board target" -> choose MCU and compatible chips
#    -> interface and pin questions from @geastack/chips/catalog.json
#    -> optional microSD and launcher button
#    -> detect USB connection, review, write .gea/targets/<alias>.json
#                                      and the alias in .gea/boards.json
```

The alias points at the definition, relative to `.gea/boards.json`:

```json
{
  "my-board": {
    "target": "my-board",
    "targetDefinition": "targets/my-board.json",
    "appPlatform": "esp32"
  }
}
```

## Editing Chips

```sh
npx gea chips list                              # catalog, with what this board uses
npx gea chips info co5300                       # interfaces, configuration, adapter support
npx gea chips add co5300 ft3168 --board my-board
npx gea chips add ft3168 --board my-board --replace
npx gea chips remove ft3168 --board my-board
```

`--set <chip-id>.<path>=<value>` supplies configuration values
non-interactively (`--set co5300.pins.cs=12`, `--set co5300.interface=qspi`),
repeatable. A missing value in a non-interactive shell is an error naming the
exact `--set` to pass, and unused `--set` entries are an error rather than a
silent no-op. Adding a chip whose role is already filled needs `--replace`.

These commands edit the target definition only. They never copy native sources
into the application: the target adapter compiles the selected drivers straight
out of the installed `@geastack/chips` package.

Two failures are worth recognizing:

- *"Chip 'X' has no esp32-idf binding"* — the chip is catalogued as portable
  source but has no adapter entry for this MCU. See
  `$geastack-embedded-driver-development`.
- *"Board 'X' is a preset and has no editable target definition"* — the alias
  points at a built-in target, not a composed one.

## Definition Shape

Every chip role is optional, and so are `buses`, `storage` and `controls`: the
definition describes the hardware the board has. A bare module with no panel,
no touch, no PMIC, no IMU and no codec is the short end of the same spectrum —
see "Headless Boards" below.

```json
{
  "id": "manual-amoled",
  "extends": "esp32-s3",
  "adapter": "esp32-idf",
  "mcu": "esp32s3",
  "buses": { "i2c": { "sda": 15, "scl": 14 } },
  "chips": {
    "touch":   { "driver": "ft3168", "interface": "i2c",
                 "pins": { "reset": 9, "interrupt": 38 } },
    "power":   { "driver": "axp2101", "interface": "i2c" },
    "imu":     { "driver": "qmi8658", "interface": "i2c" },
    "audio":   { "driver": "es8311", "interface": "i2s",
                 "pins": { "mclk": 16, "bclk": 41, "ws": 45,
                           "dout": 40, "din": 42, "powerAmplifier": 46 } },
    "display": { "driver": "co5300", "interface": "qspi",
                 "width": 410, "height": 502, "spiHost": "spi2",
                 "pins": { "cs": 12, "pclk": 11, "data0": 4, "data1": 5,
                           "data2": 6, "data3": 7, "reset": 8, "te": 13 } }
  },
  "storage": {
    "microSD": { "interface": "sdmmc-1bit",
                 "pins": { "clk": 2, "cmd": 1, "data0": 3 } }
  },
  "controls": { "launcherButton": { "pin": 0, "activeLevel": 0 } }
}
```

Rules the CLI enforces before CMake configuration:

- `extends` must be a `composable` base. Today that is `esp32-s3` only.
- Only `id`, `extends` and `mcu` are required. An omitted `chips.<role>`,
  `buses.i2c`, `storage.microSD` or `controls.launcherButton` means the board
  does not have it — `null` and `"none"` spell the same thing.
- Each `chips.<role>` key that is present is a catalog **category** — `display`,
  `touch`, `power`, `imu`, `audio`, `rtc`. A chip cannot fill a role of another
  category.
- `interface` must be one the catalog lists for that chip.
- `buses.i2c` is required only when an `i2c` chip is selected.
- Pins are integers 0-48; a pin may be `null` or `"none"` only where the chip
  marks it optional.
- GPIO collisions are rejected before compilation, across the chips, buses,
  storage and controls the board actually declares.
- A chip with no adapter binding for the MCU fails before compilation.

The complete stack the `esp32-s3` base supports today is CO5300 (display),
FT3168 (touch), AXP2101 (power), QMI8658 (IMU), and ES8311 (audio).

## Headless Boards

A board with no `chips.display` has two modes:

- With `canvas: { width, height }`, it has an offscreen surface, a reactive UI
  and a frame loop. Drawing works and screenshots can read the surface; no
  pixels are transmitted to a panel.
- With no `canvas`, it has no display at all (`GEA_EMBEDDED_NO_DISPLAY=1`).
  The runtime creates no framebuffer, app tree or frame loop, and skips
  `AppRunner::initApplication`. Native boot hooks and their own tasks can still
  run. Do not expect TS/TSX UI initialization, rAF, or screenshots in this mode.

This example explicitly opts into the offscreen mode:

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

`canvas` dimensions are explicit; there is no default offscreen surface.
The example uses the base’s 32MB flash layout. For a 16MB N16R8 module, use the
shipped `esp32-s3-devkit-n16r8` definition, which supplies a matching partition
table and no display; add an app-local `canvas` only if rendering is wanted.

`flashSize`, `partitions` and `console` belong to the board because they are its geometry and
wiring: a 16MB module cannot borrow the 32MB layout of the base whose driver
stack it extends, and a module reached over a USB-UART bridge cannot use the
base's native USB-Serial/JTAG console — that combination runs correctly while
logging into a port nobody is connected to, which looks exactly like a board
that never booted. App `gea.targets.esp32.partitionsByTarget[targetId]` wins
over its `partitions`, which wins over the board’s table. Declare PSRAM mode and speed
from the module’s wiring as well; omission leaves the base settings in place.

### Reaching the outside world without a screen

A headless board still has pins, and `Gpio` and `Led` are the platform surface
for them — the only output such a board has, which is why they are linked into
every ESP32 image rather than gated behind a capability flag:

```ts
import { Gpio, Input, Led } from '@geastack/core'

Led.attach(48, 1)                 // a WS2812 strand on one pin
Led.setPixel(0, 0, 40, 0)
Led.show()                        // returns once the strand has latched it

Gpio.configureOutput(2)           // a plain diode, relay or line
Gpio.write(2, true)

Gpio.configureInput(9, true)      // a button of your own, pulled up
const pressed = !Gpio.read(9)
```

Check boolean operation results for unsupported pins or unavailable hardware.
`Gpio.read()` reports the input level. This TypeScript example needs the normal
app initialization path, so retain an explicit `canvas` on a panel-free board.

The BOOT button needs none of this — `controls.launcherButton` already reaches
an app as `Input.consumeBackButton()`, which reads and clears a one-shot flag.
`examples/apps/headless-led` is the whole pattern in one file.

Read `references/composition-reference.md` for per-chip configuration fields.

## Verification

```sh
npx gea chips list --board my-board
npx gea build --board my-board --dry-run
npx gea build --board my-board --verbose        # CMake validates the definition
npx gea flash --board my-board --monitor
npx gea devctl i2cscan --board my-board         # every I2C chip answering?
npx gea screenshot shot.png --board my-board
```

A board that builds but shows nothing is usually a display pin or `spiHost`
mistake; a board whose touch never fires is usually the `interrupt` pin or a
missing `reset`. `i2cscan` separates "wired wrong" from "driver wrong".
