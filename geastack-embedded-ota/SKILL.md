---
name: geastack-embedded-ota
description: Deploy GeaStack firmware without a cable - gea ota over WiFi or BLE, OTA slots and staging, partition layouts, erase/boot/reboot maintenance, gea flash slot provisioning, and the ota.ble manifest flag. Use when updating a board over the air, provisioning several app images into OTA slots, debugging an OTA that will not connect, or planning a board's partition table for OTA.
---

# GeaStack Embedded OTA

`gea ota` builds the app and installs it on a board that is already running,
with no cable. Firmware slot operations apply to ESP32 boards. GeaOS targets
with `bootMode: "ram-only"` also support WiFi application replacement, without
firmware slots or reboot flags.

```sh
npx gea ota --board <alias> [--app <id>] [--monitor] [--logs]
            [--transport wifi|ble] [--device <name>] [--host <ip>]
```

## Choosing The Transport

The default is **BLE when the app enables it, WiFi otherwise**:

| transport | reaches the board via | requires |
| --- | --- | --- |
| `wifi` | `POST /ota` on port 8080 at `transports.ota.host` | the app links a network binding, and the alias records a host |
| `ble` | the firmware's BLE OTA service | running firmware built with BLE OTA enabled (usually `gea.ota.ble: true`); a macOS host with `swift` |

BLE transport is selected by `--transport ble`, or defaults from the selected
app’s manifest flag. `GEA_EMBEDDED_BLE_OTA=1` enables BLE OTA in the build but
does not select the upload transport; pass `--transport ble` explicitly.
`--device <name>` names the peripheral. The firmware already running on the
board must expose the chosen OTA service; enabling it only in the new image
cannot make the old image reachable.

The WiFi path has a trap worth internalizing: GeaStack links the network stack
only when the app's code reaches `wifi`, `fetch`, `http`, `websocket`, or `rtc`
(see `$geastack-embedded-build-flags`). An app with no network code produces
firmware that can never be reached over WiFi, even though the alias still holds
the address an earlier app answered on. `gea.ota.ble: true` is the way to keep a
network-free app updatable without a cable — it links the BLE OTA service and
sets the BLE capability whether or not the app itself uses BLE.

## Everyday Loop

```sh
npx gea boards set amoled host 192.168.1.42     # record the address once
npx gea ota --board amoled --app watch
npx gea ota --board amoled --app watch --logs   # then follow the log over WiFi
npx gea ota --board amoled --app watch --monitor
```

`gea boards discover` reports each plugged-in board's current IP, and `--save`
writes it back into the alias.

## Slots

A board with OTA has two app partitions, `ota_0` and `ota_1`, plus `otadata`
recording which one boots. Beyond replacing the running app, the CLI can stage
images into a specific slot over WiFi. Pass `--transport wifi` for slot/erase
operations even when the app defaults to BLE; BLE updates do not implement
explicit slot staging:

```sh
npx gea ota --board amoled --transport wifi --app watch --slot ota_1            # stage only
npx gea ota --board amoled --transport wifi --app watch --slot ota_1 --boot     # stage + mark bootable
npx gea ota --board amoled --transport wifi --app watch --slot ota_1 --reboot   # stage + boot + restart
npx gea ota --board amoled --transport wifi --erase-slot ota_1
```

`--image <file.bin>` uploads a prebuilt application image; `--no-build` reuses
the selected app’s existing build. USB flashing can also stage several
prebuilt images at once:

```sh
npx gea flash --board amoled --slot ota_1 --image ./watch.bin
npx gea flash --board amoled --slot-image ota_0=./launcher.bin --slot-image ota_1=./watch.bin
npx gea flash --board amoled --erase-slot ota_1
npx gea flash --board amoled --restore-boot
```

`--restore-boot` puts `otadata` back to the factory/boot slot after slot
experiments.

Boards whose adapter only supports replacing the application in RAM reject slot
and reboot flags with a message saying so, rather than pretending to stage.

## Partitions

OTA needs the partition table to carry `otadata` and two app slots big enough
for the image. The 32MB AMOLED layout is the reference — two 8MB app slots
plus a SPIFFS `storage` partition:

```
nvs,       data, nvs,     0x9000,    0x6000
otadata,   data, ota,     0xf000,    0x2000
phy_init,  data, phy,     0x11000,   0x1000
ota_0,     app,  ota_0,   0x20000,   0x800000
ota_1,     app,  ota_1,   0x820000,  0x800000
storage,   data, spiffs,  0x1020000, 0xFE0000
```

Two mistakes recur: sizing the slots to fill the flash (which runs past the chip
end) and omitting a writable filesystem partition. `localStorage` uses SPIFFS when
`GEA_STORAGE_SPIFFS` is enabled and mounted, otherwise a small NVS blob. NVS
persists too, but capacity/write failures can leave new values unsaved. See
`$geastack-embedded-target-bringup`.

App tables can override the board: `gea.targets.esp32.partitionsByTarget`
selects by concrete target id, then `partitions` is the fallback. Both accept
an app-relative CSV or partition object. Composed-board partition objects are
next, then built-in tables. Verify the selected table fits that board’s flash;
an OTA application upload does not install a changed partition table. Provision
a changed layout over USB before relying on it for OTA.

## HTTP Surface

The port-8080 server holds `POST /ota`, `POST /ota/erase`, `GET /ota/status`,
`GET /screenshot`, and the display knobs. Port 8081 carries the diagnostics log
stream `gea logs` reads, but only when networking and
`GEA_EMBEDDED_DIAGNOSTICS_ENABLED` are enabled; diagnostics defaults off.
Details in `$geastack-embedded-device-control`.

## Debugging

| symptom | check |
| --- | --- |
| "does not define transports.ota.host" | `gea boards set <alias> host <ip>`, or pass `--host` |
| connection refused over WiFi | the running app links no network stack; use USB or BLE OTA |
| BLE OTA unavailable | `gea.ota.ble` not set, or `swift` missing (`gea doctor` warns) |
| "OTA is only available for ESP32 boards" | the alias resolves to a non-`esp32-idf` adapter |
| image does not boot after staging | the slot was staged but not marked bootable; add `--boot` |
| flash full / image too large | slot size in `partitions.csv` |

Always confirm with a dry run first:

```sh
npx gea ota --board amoled --app watch --dry-run
```
