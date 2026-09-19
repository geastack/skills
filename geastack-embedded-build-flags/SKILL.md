---
name: geastack-embedded-build-flags
description: Choose, inspect, and update GeaStack CLI flags, gea create options, environment variables, CMake cache variables, target compile definitions, sdkconfig defaults, capability analysis, perf/logging flags, flash options, and compiler knobs. Use when a task mentions flags, build configuration, target tuning, manifests, dry runs, production lockdown, flash baud, or memory/performance compile switches.
---

# GeaStack Embedded Build Flags

## Rule Of Thumb

Find the layer that owns the behavior before changing a flag:

1. App manifest: target eligibility, BLE OTA, native sources/plugins, defines,
   CSS pixel ratio, and per-platform build configuration.
2. **Capability analysis**: what the app's code reaches decides what is linked.
3. `gea` CLI: command routing, board selection, flash/OTA options.
4. Target CMake: cache variables and compile definitions.
5. `sdkconfig.defaults`: ESP-IDF features, memory, logging, toolchain config.
6. Runtime app code: `Display`, `Memory`, and host API knobs.

Read `references/flag-map.md` for the current flag surface.

## Capabilities Are Derived, Not Declared

Before an ESP32 build, the CLI runs the compiler's `analyze` pass over the app
entry with the gea plugin and reads back the host bindings the program reaches.
That result sets the capability defines:

| capability | triggered by a reached binding of |
| --- | --- |
| `GEA_EMBEDDED_CAPABILITY_NETWORK` | `wifi`, `fetch`, `http`, `websocket`, `rtc`, or the `https` feature |
| `GEA_EMBEDDED_CAPABILITY_BLE` | `ble`, `gea.ota.ble: true`, or a build requesting BLE OTA through `--transport ble` / `GEA_EMBEDDED_BLE_OTA=1` |
| `GEA_EMBEDDED_CAPABILITY_AUDIO` | `audio` |

So an app that never calls the network produces firmware with no WiFi stack —
smaller, faster to boot, and unreachable by `gea logs --transport wifi`. Adding
one `fetch` call changes all of that. This is the first thing to check when
firmware size, boot time, or OTA reachability changes unexpectedly.

The older `GEA_EMBEDDED_WIFI_DISABLED` / `GEA_EMBEDDED_BLE_DISABLED` /
`GEA_EMBEDDED_AUDIO_DISABLED` defines still exist as board-level overrides for
hardware that simply lacks the peripheral. Do not use them to trim an app.

## CLI Flags

Use `--dry-run` before changing hardware state.

```sh
npx gea build --board amoled --app watch --dry-run
npx gea flash --board amoled --app watch --monitor --manual-boot
npx gea flash --board amoled --flash-baud 115200
npx gea doctor --strict --json
npx gea build --board amoled --verbose        # or GEA_VERBOSE=1
```

The parser supports `--flag value`, `--flag=value`, repeated flags with
last-value lookup, comma lists, and `--no-name` for boolean false.

Board and app selection:

- `--board <alias>` — required for device commands unless exactly one board is
  registered.
- `--app <id>` — or run inside the app folder, or pass the id positionally.
- `--target <target-id>` — a concrete target from `gea targets list`. Passing a
  *platform* name (`esp32`, `rp2350`, `geaos`) is rejected with a message
  telling you to pass a board alias or a target id instead.
- `--project <dir>`, `--boards-config <file>`.

Flash and slot options: `--monitor`, `--manual-boot`, `--no-reset`,
`--flash-baud <rate>`, `--port <path>`, `--slot ota_N`, `--image <bin>`,
`--slot-image ota_N=<bin>`, `--erase-slot ota_N`, `--restore-boot`, `--boot`,
`--reboot`. See `$geastack-embedded-ota`.

ESP32 builds also accept `--configure-only` and `--output <file>`. App-owned
build changes belong in `gea.defines`, `gea.cssDevicePixelRatio`, or
`gea.targets.esp32` (components, sdkconfig, partitions, prebuild and output),
not in edits to installed target files. See `references/flag-map.md`.

There are no `--collection-root`, `--core-root`, `--compiler-root`,
`--examples-root`, `--simulator-root`, or `--targets-root` options, and no
`--resident-apps`: the CLI resolves packages from `node_modules`, and resident
apps were removed.

## Target Defines

Target compile definitions are appropriate for board-level hardware facts and
defaults, such as:

- display width, height, native width, native height
- pixel format and panel endian
- CSS device pixel ratio
- default frame interval
- flush/dirty-region behavior
- frame scheduler policy
- app frame task core and priority
- WiFi, BLE, audio availability on this hardware
- perf/logging defaults

Do not put app-specific tuning in a target define unless every app on that
target should inherit it.

## WiFi Diagnostics

A network capability does not automatically enable `gea logs` over WiFi.
The port-8081 diagnostics server defaults off. Define
`GEA_EMBEDDED_DIAGNOSTICS_ENABLED` (for example in `gea.defines`) to opt in;
`GEA_EMBEDDED_DIAGNOSTICS_DISABLED` takes precedence. USB monitoring remains
available without the TCP diagnostics task. OTA uses a separate HTTP service.

## Production Versus Development

Development builds intentionally keep logs, assertions, heap tracing, and perf
visibility available. Production lockdown is gated by
`CONFIG_GEA_EMBEDDED_PRODUCTION_LOCKDOWN` in sdkconfig. The current CLI build
policy clears that key and forces INFO logging, so setting it in board defaults
alone does not produce a locked-down CLI build. Inspect the generated sdkconfig
and the active build policy before reporting production lockdown as enabled.

`GEA_EMBEDDED_PERF=0` is the master compile-time switch that forces the app perf
log, scheduler perf log, perf-lite, UI refresh perf, canvas detail perf, rAF
perf, and heap diagnostics off.

## Verification

- Run dry-run commands to confirm routing.
- Run `npx gea doctor` after toolchain or package changes.
- Run focused tests in `cli/test` when parser, resolver, or metadata changes.
- Build one representative app after CMake or sdkconfig changes.
- Read logs with `npx gea logs --board <alias> --follow` after
  logging/perf/memory changes.
