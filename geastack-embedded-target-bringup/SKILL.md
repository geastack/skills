---
name: geastack-embedded-target-bringup
description: Add, configure, and bring up GeaStack embedded boards and targets across the targets package, board aliases, targets.json metadata, ESP32 IDF folders, RP2350 Pico SDK folders, sdkconfig.defaults, CMake, partitions, transports, resolver behavior, and hardware verification. Use when adding a new board, adding a new target id, changing board metadata, wiring flash/monitor/OTA transports, tuning display geometry, or debugging target bring-up.
---

# GeaStack Embedded Target Bringup

A **target** is a built-in board project shipped by `@geastack/targets`. A
**board alias** is machine-local configuration naming one physical unit. For a
board described by the chip catalog, use `$geastack-embedded-custom-boards`.
Such definitions can be app-local or shipped in the target registry through
its `definition` field. Create a native target folder only when composition
cannot describe the required hardware or build behavior.

## Workflow

Start from the closest working target:

- `esp32-s3-devkit-n16r8` for a composed bare S3 module (no panel or canvas).
- `esp32-s3-touch-amoled-{1.75,1.8,2.06,2.41}` for ESP32-S3 AMOLED panels.
- `esp32-s3-lilygo-t-display-s3-long`, `esp32-s3-m5stack-sticks3`,
  `esp32-s3-elecrow-rotary-2.1` for other ESP32-S3 panels.
- `esp32-s3-epaper-1.54`, `esp32-s3-lilygo-t5-epaper-4.7`,
  `esp32-s3-m5stack-papers3`, `esp32-m5stack-m5paper` for e-paper.
- `esp32-p4-waveshare-touch-lcd-{3.5,7}`, `esp32-p4-m5stack-tab5` for ESP32-P4
  display/camera boards.
- `rp2350-waveshare-touch-amoled-2.41`, `rp2350-tufty-2350` for Pico SDK/RP2350.

Read `references/target-checklist.md` before editing target metadata.

## Metadata

Built-in targets live in `targets.json` at the `@geastack/targets` package root
(in the monorepo: `targets/targets.json`). Each entry:

```json
{
  "esp32-s3-example-board": {
    "adapter": "esp32-idf",
    "targetPath": "targets/esp32-s3-example-board",
    "flashSize": "16MB",
    "appPlatform": "esp32",
    "idfTarget": "esp32s3",
    "esptoolChip": "esp32s3",
    "ipcTaskStackSize": "4096"
  }
}
```

Adapters in use: `esp32-idf` and `rp2350-pico`. `composable: true` marks a
target usable as the `extends` base of a composed board — today only
`esp32-s3`. `definition` points to a packaged composed-board JSON file; an
alias’s `targetDefinition` overrides it. `compatibleAppPlatforms` allows
additional manifest platforms (for example, Tufty accepts `esp32` apps too).

## Board Aliases

Aliases are machine-local and never ship in a package. The CLI merges
`~/.geastack/boards.json` (every board on this machine; `GEA_HOME` relocates the
directory) with `<project>/.gea/boards.json` (project overrides).

Register them with the CLI rather than editing JSON by hand:

```sh
npx gea setup                       # guided: pick board, alias, serial, OTA host
npx gea boards add
npx gea boards set <alias> host 192.168.1.42
npx gea boards list                 # what resolves, and from which file
npx gea boards discover             # which registered board is plugged in where
```

```json
{
  "my-board": {
    "target": "esp32-s3-example-board",
    "adapter": "esp32-idf",
    "transports": {
      "usbSerial": { "serial": "USB_SERIAL_NUMBER" },
      "ota": { "host": "192.168.1.42" }
    }
  }
}
```

Prefer USB serial numbers over path globs. The resolver rejects
`transports.usbSerial.path` — the numeric suffix on a serial device changes on
every re-plug and never identifies a particular board.

## Target Folder

ESP32 target folders commonly include:

- `CMakeLists.txt`
- `sdkconfig.defaults`
- `partitions.csv`
- `dependencies.lock`
- `main/CMakeLists.txt`
- `main/idf_component.yml`
- `main/Kconfig.projbuild`
- board-specific `main/*.cpp` (`app_main.cpp`, `touch.cpp`, `i2c.cpp`,
  `launcher_button.cpp`, `sdcard_mount.cpp`)
- focused tests under `test/`

Shared ESP32 sources live under `targets/targets/esp32`; board-specific folders
should contain only composition, config, and hardware glue. The CLI selects
the active app and passes `GEA_EMBEDDED_APP` / `GEA_EMBEDDED_APP_META`
(root;entry;runtime;native sources) to CMake. Reuse the shared source manifest
and generators; CMake may invoke Node helpers for these, so do not duplicate
app discovery or source-list generation.

Build output goes to the *application's*
`.gea/build/<target>/app-builds/<app>/`, not into the package.

## CMake And Defines

Follow the established pattern:

- Set `GEA_EMBEDDED_ROOT` from `CMAKE_CURRENT_LIST_DIR`.
- Include `targets/esp32/gea_framework.cmake`.
- Require `GEA_EMBEDDED_APP` in real configure passes.
- Set `GEA_EMBEDDED_CPP_BOARD` and `GEA_EMBEDDED_CSS_DEVICE_PIXEL_RATIO`.
- Include `targets/esp32/host_sources.cmake` for platform-universal host
  facades.
- Add hardware-gated sources, such as camera, per target.
- Put display geometry, pixel format, perf, frame scheduler, and
  target-specific workarounds in `target_compile_definitions`.

Do not hand-set the capability defines
(`GEA_EMBEDDED_CAPABILITY_NETWORK`/`_BLE`/`_AUDIO`): the CLI derives them from
the app's compiler analysis. Use `$geastack-embedded-build-flags` before
changing compile definitions that affect performance, logging, memory, or
display behavior.

## Verification

Start with metadata and resolution:

```sh
npx gea targets list --json
npx gea targets show <target-id>
npx gea boards list --json
npx gea doctor
```

Then build and deploy:

```sh
npx gea setup --target <target-id>
npx gea build --target <target-id> --app <app-id> --dry-run
npx gea build --target <target-id> --app <app-id> --verbose
npx gea run --board <alias> --app <app-id>
```

Run the target metadata tests after a metadata or CMake-source change; board
resolution, sdkconfig policy, flashing, OTA, and device transports are tested in
`@geastack/cli` (`cli/test`).
