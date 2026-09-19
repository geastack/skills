---
name: geastack-embedded-device-control
description: Drive a running GeaStack board with gea devctl, gea logs, gea monitor, and gea screenshot - read state and memory, inject taps/drags/keys, inspect laid-out nodes and hit targets, move files to and from the device, scan I2C, set the boot app and clock, and change brightness/HBM/VSync over USB GEADEV or WiFi. Use when automating or inspecting a board that is already flashed, scripting a UI walkthrough, capturing screenshots, pushing assets to an SD card, or debugging what the renderer actually laid out.
---

# GeaStack Embedded Device Control

`gea devctl` drives a board that is **already running an app**. It is the
debugging and automation surface, separate from building (`gea build`),
installing (`gea flash`, `gea ota`) and reading output (`gea monitor`,
`gea logs`).

```sh
npx gea devctl <verb> [args] --board <alias> [--transport auto|usb|wifi]
npx gea devctl                                # prints the verb list
```

Use a board alias for persistent identity; `--port <path>` is a temporary USB
override, not a stable board identifier. Register aliases with
`gea setup` or `gea boards add`; see what is plugged in with
`gea boards discover`.

Read `references/devctl-reference.md` for the full verb table and the wire
protocols.

## Transports

| transport | reaches the board via | carries |
| --- | --- | --- |
| `usb` | USB serial, GEADEV line protocol | every verb |
| `wifi` | HTTP on port 8080 at the board's recorded address | the three display knobs only |

Everything except `brightness`, `hbm`, and `vsync` is GEADEV-only and needs the
cable. Asking for another verb over WiFi fails with a message rather than
hanging.

Display knobs default to the cable when the board is attached and fall back to
its recorded address otherwise. That default matters: GeaStack links WiFi only
when the app's code reaches a network binding, so an app that never touches the
network produces a board with no networking at all, while the alias still
records the address some earlier app answered on.

Pass `--transport usb` / `--transport wifi` to override, or `--host <ip>` to aim
at an address without consulting the alias (naming a host implies WiFi).

## Common Loops

Inspect a live board:

```sh
npx gea devctl ping --board amoled          # does it answer at all
npx gea devctl app --board amoled           # which app is running
npx gea devctl summary --board amoled
npx gea devctl mem --board amoled           # heap and PSRAM
npx gea devctl i2cscan --board amoled
```

Drive the UI and check what the renderer did:

```sh
npx gea devctl tap 205 300 --board amoled
npx gea devctl drag 205 400 205 120 12 20 --board amoled
npx gea devctl back --board amoled
npx gea devctl node button --board amoled   # nodes matching a class
npx gea devctl hit 205 300 --board amoled   # what is under a point
npx gea screenshot after-tap.png --board amoled
```

Coordinates are in the panel's own pixels, origin top left.

Move data:

```sh
npx gea devctl storage get theme --board amoled
npx gea devctl storage set theme dark --board amoled
npx gea devctl ls /sdcard --board amoled
npx gea devctl push ./map.bin /sdcard/map.bin --board amoled
npx gea devctl pull /sdcard/log.txt ./log.txt --board amoled
```

Both transfers verify a checksum, so a truncated file is an error rather than a
silently short one. `--base64` selects text-safe upload for `push`.

Board defaults:

```sh
npx gea devctl set-default watch --board amoled   # app launched on boot
npx gea devctl set-time --board amoled            # defaults to host time
```

`Clock.epochMs()` in an app returns a small value until the clock has been set
this way (or by the companion app) — `Date.now()` is monotonic on ESP32, not
wall-clock.

## Reading Output

```sh
npx gea logs --board amoled --follow        # WiFi when the board has an IP, else USB
npx gea monitor --board amoled --timestamps --log-file run.log
```

`gea logs` over WiFi needs both networking and
`GEA_EMBEDDED_DIAGNOSTICS_ENABLED`: the port-8081 server defaults off, and
`GEA_EMBEDDED_DIAGNOSTICS_DISABLED` takes precedence. Use `--transport usb`
when it is unavailable. `gea monitor` is the raw USB serial monitor. The serial
port is opened without touching DTR or RTS,
because a DTR/RTS sequence drops a USB-Serial-JTAG chip (C3, S3, P4) into ROM
download mode where it stops answering.

## Scripting Notes

- Every display knob replies with JSON on stdout, so a script can parse it.
- Passing no value to a knob reports the current setting instead of changing it.
- Pass arguments separately or use a shell array. Default zsh does not split
  `devctl $ARGS` when `ARGS="hbm on"`; Bash word-splitting behaves differently.
  Avoid relying on that shell-specific behavior.
- `push` reports progress on stderr, so keep stdout clean for parsed output.
