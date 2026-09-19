# devctl Reference

## Display Knobs

The only verbs that answer on both transports. Passing no value reports instead
of setting. Every one replies with JSON on stdout.

```sh
npx gea devctl brightness --board amoled       # {"brightness":100}
npx gea devctl brightness 40 --board amoled    # {"brightness":40}
npx gea devctl hbm on --board amoled           # {"hbm":true,"supported":true}
npx gea devctl hbm off --board amoled
npx gea devctl vsync on --board amoled         # {"vsync":true}
```

The CLI rejects brightness outside 0-100; the raw HTTP endpoint clamps it.
Unsupported `hbm` reports `supported: false` over USB or HTTP 501 over WiFi.

## Verbs (USB only)

### State and identity

| verb | what it does |
| --- | --- |
| `ping` | confirm the board answers the protocol at all |
| `app` | the id of the app currently running |
| `state` | the app's reported state |
| `mem` | heap and PSRAM figures |
| `summary` | a combined status report |
| `i2cscan` | addresses responding on the I2C bus |
| `reboot` | restart the board |

### Input injection

| verb | arguments |
| --- | --- |
| `tap` | `<x> <y> [holdMs]`, default hold 80 ms |
| `drag` | `<x1> <y1> <x2> <y2> [steps] [delayMs]`, defaults 6 steps and 24 ms |
| `swipe` | `<x> <y1> <y2>` |
| `key` | `<code>` |
| `back` | the platform back action |
| `notify` | `<text>`, the rest of the line is the message |

Coordinates are in the panel's own pixels, origin top left. `notify` shows up in
the app through `Notify.text()` / `Notify.seq()`; `back` is what
`Input.consumeBackButton()` reads.

### UI inspection

| verb | arguments |
| --- | --- |
| `node` | `<class>`, describe nodes matching a class |
| `hit` | `<x> <y>`, report what is under a point |

### Storage and files

| verb | arguments |
| --- | --- |
| `storage get` | `<key>` |
| `storage set` | `<key> <value>` |
| `ls` | `[path]`, defaults to `/sdcard` |
| `rm` | `<path>` |
| `push` | `<local> <remote>`, `--base64` for a text-safe transfer |
| `pull` | `<remote> <local>` |
| `playfile` | `<path>` |

`storage` is the same key/value store the app reaches through the global
`localStorage`. Local paths resolve against the working directory.

### Board defaults

| verb | arguments |
| --- | --- |
| `set-default` | `<app-id>`, the app the board launches on boot |
| `set-time` | `[epochSeconds]`, defaults to the host's current time |

## GEADEV Over USB Serial

A line protocol. The host writes `GEADEV <VERB> [args]`; the board answers a
line beginning `GEADEV:`, with `GEADEV:OK <VERB> ...` on success and
`GEADEV:ERR <VERB> ...` on failure. Replies carry `key=value` pairs.

```text
GEADEV PING          -> GEADEV:PONG app=tilt-breakout ip=192.168.1.100 mac=...
GEADEV BRIGHTNESS    -> GEADEV:OK BRIGHTNESS value=100
GEADEV BRIGHTNESS 40 -> GEADEV:OK BRIGHTNESS value=40 readback=40
GEADEV HBM on        -> GEADEV:OK HBM value=1 supported=1
GEADEV VSYNC         -> GEADEV:OK VSYNC value=1
```

The port is opened without touching DTR or RTS. This is required, not
incidental: on USB-Serial-JTAG boards (ESP32-C3, S3, P4) a DTR/RTS sequence
drops the chip into ROM download mode, where it waits for a firmware image and
appears hung.

Firmware older than the alias-discovery work answers `PING` without `ip` and
`mac`; the address simply reads as unknown.

## HTTP On Port 8080

The same server that serves OTA carries the display knobs, because it is the
only HTTP surface a cable-free board has.

| route | query | reply |
| --- | --- | --- |
| `POST /display/brightness` | `value=0-100` | `{"ok":true,"brightness":40}` |
| `POST /display/hbm` | `on=1\|0` | `{"ok":true,"hbm":true}` |
| `POST /display/vsync` | `on=1\|0` | `{"ok":true,"vsync":true}` |

Omit the query string and the route reports rather than sets.

```sh
curl -X POST "http://192.168.1.100:8080/display/hbm?on=1"
```

The same server also holds `POST /ota`, `POST /ota/erase`, `GET /ota/status`,
`GET /screenshot`, and a flash benchmark. Port **8081** carries the diagnostics
log stream that `gea logs` reads when `GEA_EMBEDDED_DIAGNOSTICS_ENABLED` is
defined. It defaults off independently of OTA;
`GEA_EMBEDDED_DIAGNOSTICS_DISABLED` overrides the opt-in.

One firmware detail before adding a route: ESP-IDF's default server config caps
registered URI handlers at 8, and registration past that limit fails silently,
leaving a route that 404s for no visible reason. The board raises the cap
explicitly.

## Troubleshooting

| symptom | cause |
| --- | --- |
| `No such board` | alias not in `~/.geastack/boards.json` or `<project>/.gea/boards.json`; run `gea boards list` |
| OTA works but WiFi logs fail | TCP diagnostics is disabled; opt in with `GEA_EMBEDDED_DIAGNOSTICS_ENABLED` or use USB |
| a display knob fails with a network error | the running app has no WiFi linked; add `--transport usb` |
| `firmware has no /display/<knob> endpoint` | firmware predates the routes; reflash, or use the cable |
| "verb needs the USB transport" | only the three display knobs answer over WiFi |
| board stops answering, log mentions waiting for download | another tool asserted DTR/RTS; power-cycle |
| a quoted argument arrives as one token | pass the arguments separately, not as one shell variable |
