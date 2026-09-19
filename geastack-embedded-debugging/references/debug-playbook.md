# Debug Playbook

## App Not Found Or Target Disabled

- Confirm `package.json` has `gea.id`.
- Run `npx gea inspect <app-id> --json`.
- Confirm `gea.entry` exists.
- Confirm `gea.targets.<platform>` is `true` or a configuration object without
  `enabled: false`.
- Run from inside the app folder, pass `--app <id>`, or point `--project <dir>`
  at the right project. `GEA_EXTRA_APP_DIRS` adds app directories to the scan.

## `--target esp32` Rejected

`esp32`, `rp2350` and `geaos` are *platform* names, not targets. Pass
`--board <alias>` (`gea boards list`) or `--target <target-id>`
(`gea targets list`).

## Board Not Found

- Aliases come from `~/.geastack/boards.json` and `<project>/.gea/boards.json`,
  or only from `--boards-config` when given.
- `npx gea boards list` shows what resolves and the file each alias came from.
- `npx gea boards discover` shows what is actually plugged in.
- Ensure the alias has a `target`, and that the target id exists in the
  `@geastack/targets` `targets.json` (or that `targetDefinition` points at an
  app-local composed target).

## Serial Port Problems

- Prefer `transports.usbSerial.serial`; path-based configs are rejected.
- Use `npx gea boards discover` to map serial numbers to today's ports.
- Pass an explicit `--port` only for temporary debugging.
- Use `--flash-baud 115200` after sync/connect failures.
- A board that stops answering and logs "waiting for download" had DTR/RTS
  asserted by some other tool. Power-cycle it.

## Blank Screen After Flash

- Check whether the board requires a manual restart after flash
  (`transports.usbSerial.restartAfterFlash`).
- Monitor boot logs: `npx gea monitor --board <alias>`.
- Confirm the app image matches the selected app id and target.
- Use `--manual-boot` only when entering the bootloader manually.
- `npx gea screenshot` distinguishes a display failure from an app/render
  failure — a correct screenshot with a dark panel is a backlight/brightness
  problem, so try `npx gea devctl brightness 100 --board <alias>`.

## Board Unreachable Over WiFi

GeaStack links WiFi only when the app's code reaches a network binding. An app
that never calls `fetch`/`WiFi`/`http`/`WebSocket` produces firmware with no
networking, while the alias still records the address an earlier app answered
on. Use the cable: `--transport usb`.

## No UI On A Composed Board

No `chips.display` and no `canvas` means `GEA_EMBEDDED_NO_DISPLAY=1`: no
framebuffer, app-tree initialization or frame loop. Native boot-hook tasks
continue, but a TS/TSX UI or screenshot needs an explicit `canvas` or panel.
Also verify `console` transport and PSRAM wiring in the board definition.

## WiFi OTA Works But Logs Do Not

TCP diagnostics on port 8081 defaults off, separately from HTTP OTA. Use USB
monitoring, or opt in with `gea.defines.GEA_EMBEDDED_DIAGNOSTICS_ENABLED: true`
and rebuild. `GEA_EMBEDDED_DIAGNOSTICS_DISABLED` overrides the opt-in.

## Slow Frames

- Check `gea.perf`, `perf`, and `perf-lite` lines in `npx gea logs --follow`.
- Use `Display.setFrameRate`, `Display.setFlushConfig`, and
  `Display.invalidate` appropriately.
- Avoid per-frame allocations and string work.
- Use `rgb565` (an ambient global) for hot palettes.
- Generate a heap report if stutter grows over time.

## Production Lockdown Surprise

If logs and diagnostics disappear, inspect
`CONFIG_GEA_EMBEDDED_PRODUCTION_LOCKDOWN` in sdkconfig and compile definitions
such as `GEA_EMBEDDED_PERF`, `GEA_FRAME_PERF_LOG`,
`GEA_EMBEDDED_FRAME_SCHEDULER_PERF_LOG`, and `GEA_EMBEDDED_DIAGNOSTICS_DISABLED`.

## Missing Package

`gea doctor` reporting `@geastack/compiler` or `@geastack/geatsc-plugin-gea` as
not installed means the capability analysis cannot run and no ESP32 build will
start. Reinstall project dependencies rather than passing paths — there are no
root-override options.
