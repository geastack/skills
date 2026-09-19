---
name: geastack-embedded-debugging
description: Debug GeaStack embedded apps, boards, flashing, monitoring, serial ports, OTA, ESP-IDF setup, board resolution failures, perf logs, heap pressure, screenshots, memory reports, production lockdown, and hardware-specific boot/reset issues. Use when diagnosing build failures, deploy failures, runtime crashes, blank screens, slow frames, WiFi/BLE/media issues, or target hardware behavior.
---

# GeaStack Embedded Debugging

Everything is a `gea` command. There are no board shell scripts and no
`esp32-*.py` helpers any more; device access lives in `@geastack/cli` under
`src/device` and `src/esp32`.

## Triage Order

Start with non-destructive checks:

```sh
npx gea doctor
npx gea inspect <app-id> --json
npx gea targets list --json
npx gea boards list --json
npx gea boards discover
npx gea build --board <alias> --app <app-id> --dry-run
```

`gea boards discover` is usually the fastest first move: it says which
registered board is on which USB port, what app it is running, and its IP.

Then build and deploy with logs visible:

```sh
npx gea build --board <alias> --app <app-id> --verbose
npx gea run --board <alias> --app <app-id>
npx gea logs --board <alias> --follow
```

Read `references/debug-playbook.md` for common failure classes.

## Toolchain Checks

`gea doctor` checks the project root, Node >= 20.19, the installed GeaStack
packages (`@geastack/targets`, `@geastack/core`, `@geastack/compiler`,
`@geastack/geatsc-plugin-gea` required; `@geastack/chips`, `@geastack/engine`,
`@geastack/host`, `@geastack/elements`, `@geastack/geaos` optional), the
`serialport` npm package, ESP-IDF and its python env, `cmake`, `ninja`,
`ccache`, `arm-none-eabi-gcc` and `picotool` (RP2350), `swift` (BLE OTA,
macOS), the board config files, and the app catalog and manifest.

A `[fail]` on a required check exits non-zero; `--strict` also fails on
warnings. The CLI’s offline ESP-IDF fallback is **v6.0.2**; setup can resolve a
newer stable release or use `--idf-version` / `GEA_ESP_IDF_VERSION`. Check the
resolved version and board requirements. If `idf.py` is missing, set
`IDF_PATH`, or point `GEA_EMBEDDED_IDF_EXPORT` / `ESP_IDF_EXPORT` at the export
script.

## Board Resolution

Aliases are merged from two files, project over home:

- `~/.geastack/boards.json` — every board on this machine (`GEA_HOME` relocates
  the directory)
- `<project>/.gea/boards.json` — project overrides

```json
{
  "amoled": {
    "target": "esp32-s3-touch-amoled-2.06",
    "adapter": "esp32-idf",
    "transports": {
      "usbSerial": { "serial": "USB_SERIAL_NUMBER" },
      "ota": { "host": "192.168.1.42" }
    }
  }
}
```

Use USB serial numbers, not `/dev` paths — the numeric suffix changes on every
re-plug. The resolver rejects `transports.usbSerial.path`.

```sh
npx gea boards list          # what resolves, and from which file
npx gea boards show <alias>
npx gea boards discover      # which registered board is plugged in where
npx gea boards set <alias> host 192.168.1.42
```

The resolver itself is `cli/src/boards/resolve.mjs`; built-in target metadata is
`targets.json` at the `@geastack/targets` package root.

## Flashing

Common recovery flags:

- `--flash-baud 115200` for flaky cables or boards.
- `--manual-boot` when holding BOOT/IO0 manually.
- `--no-reset` to leave a device in bootloader after USB flashing.
- `--port <path>` when auto resolution is ambiguous.

Some native USB Serial/JTAG boards need a manual reset or power cycle after
flash; respect `transports.usbSerial.restartAfterFlash = "manual"`.

Never let another tool assert DTR/RTS on a USB-Serial-JTAG board (C3, S3, P4):
that drops the chip into ROM download mode, where it stops answering and looks
hung. Power-cycle to recover.

## Runtime Diagnostics

Look for:

- `gea.perf:` app frame and phase timing.
- `perf:` or `perf-lite:` frame scheduler output.
- heap diagnostics and free block information.
- assertions and panic backtraces.
- app frame task stack high-water marks via `Memory`.

```sh
npx gea logs --board <alias> --follow            # WiFi if the board has an IP, else USB
npx gea monitor --board <alias> --timestamps --log-file run.log
npx gea screenshot shot.png --board <alias>
npx gea devctl summary --board <alias>
npx gea devctl mem --board <alias>
npx gea heap-report run.log --out heap-report.html --map .gea/build/<target>/app-builds/<app>/gea_embedded.map
```

`gea devctl` carries the rest of the live-board surface (state, i2cscan, input
injection, node/hit inspection, file transfer). See
`$geastack-embedded-device-control`.

## Verification

When fixing a bug, preserve the failing evidence:

- exact command
- board alias and target id
- app id
- log excerpt
- screenshot when visual
- heap report when memory-related

Then run the smallest focused test or command that proves the fix.
