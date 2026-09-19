# Target Checklist

## Add Or Change A Built-In Target

1. Add or edit the entry in `targets.json` at the `@geastack/targets` package
   root.
2. Prefer a packaged composed-board `definition` when the catalog covers the
   hardware; otherwise create or update the target folder under `targets/`.
3. Confirm `targetPath` resolves from the package root.
4. Set `appPlatform` to the manifest platform apps enable — `esp32`, `rp2350`,
   or `geaos`.
5. Set `adapter` — `esp32-idf` or `rp2350-pico`.
6. For ESP32, set `idfTarget` and `esptoolChip`.
7. Add `ipcTaskStackSize` when the board needs a non-default IPC stack.
8. Add `composable: true` only if the target is meant to be an `extends` base
   for composed boards.
9. Add or update target-local tests under `targets/<id>/test`; board resolution
   is tested in `cli/test`.

## Board Alias Transports

- USB flashing and monitor: `transports.usbSerial.serial`
- Post-flash restart quirk: `transports.usbSerial.restartAfterFlash` (`"manual"`)
- WiFi OTA, logs, screenshot: `transports.ota.host`
- GeaOS telnet: `transports.telnet.host`, `transports.telnet.port`
- Fastboot: `transports.fastboot.serial`
- MTK: `transports.mtk.workdir`, `bootSlot`, `method`, `monitorGlob`

`gea boards set <alias> <key> <value>` accepts the shorthands `host`/`ip`
(→ `transports.ota.host`), `serial`/`usb` (→ `transports.usbSerial.serial`),
`restart` (→ `transports.usbSerial.restartAfterFlash`), plus `target`,
`adapter`, or any dotted path such as `transports.telnet.port`. An empty value
removes the field.

## ESP32 sdkconfig Defaults

Check:

- `CONFIG_IDF_TARGET`
- flash size
- console transport
- FreeRTOS tick rate
- CPU frequency
- main and IPC task stack sizes
- partition table
- PSRAM settings
- WiFi/LwIP memory budget
- BLE/NimBLE roles when used
- optimization and assertions
- development logging defaults
- heap tracing defaults
- `CONFIG_GEA_EMBEDDED_PRODUCTION_LOCKDOWN`

Note that the WiFi, BLE, and audio subsystems are only linked when the app's
compiler analysis reports a matching binding, so an sdkconfig that enables them
does not by itself put them in the image.

## Partitions

A board with OTA needs `otadata` plus two app slots. The 32MB AMOLED layout is
a useful reference — two 8MB app slots plus a `storage`
SPIFFS partition for persistent app data:

```
nvs,       data, nvs,     0x9000,    0x6000
otadata,   data, ota,     0xf000,    0x2000
phy_init,  data, phy,     0x11000,   0x1000
ota_0,     app,  ota_0,   0x20000,   0x800000
ota_1,     app,  ota_1,   0x820000,  0x800000
storage,   data, spiffs,  0x1020000, 0xFE0000
```

With `GEA_STORAGE_SPIFFS` enabled and `storage` mounted, `localStorage` uses
SPIFFS. Otherwise it uses a persistent but small NVS blob shared with other
settings; capacity/write failures can leave recent changes unsaved.

On composed boards, declare the real flash size and a fitting partition object.
App `gea.targets.esp32.partitionsByTarget[targetId]` overrides general app
`partitions`, which overrides the board table. App tables accept a CSV path or
object; board tables accept objects without `data` payloads. USB flashing
installs a changed layout; ordinary OTA only replaces an application image.

## Hardware Verification

- Build a tiny app first.
- Flash at default baud; retry with `--flash-baud 115200` for connection issues.
- Monitor boot logs (`npx gea monitor --board <alias>`).
- `npx gea devctl i2cscan --board <alias>` to confirm the bus sees every chip.
- Capture a screenshot if display changes are involved.
- Watch `gea.perf` / `perf-lite` lines if frame pacing changes are involved.
- Run a heap report if memory pressure or fragmentation changes are involved.
