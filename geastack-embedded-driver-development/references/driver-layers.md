# Driver Layers

## Repository Layout

| layer | location | package |
| --- | --- | --- |
| generic chip logic | `core/packages/chips/<category>/<chip>/` | `@geastack/chips` |
| chip catalog | `core/packages/chips/catalog.json` | `@geastack/chips` |
| app-facing declarations | `core/packages/core/index.d.ts` | `@geastack/core` |
| host facades | `core/packages/host/` | `@geastack/host` |
| ESP-IDF bindings | `targets/targets/esp32/chip_bindings/<category>/` | `@geastack/targets` |
| shared ESP32 sources | `targets/targets/esp32/` | `@geastack/targets` |
| board composition | `targets/targets/<target-id>/` | `@geastack/targets` |

## Current Chips

Catalogued in `@geastack/chips` (an `esp32-idf` adapter means it is selectable
by a composed ESP32-S3 board):

| chip | category | interface | esp32-idf binding |
| --- | --- | --- | --- |
| `co5300` | display | qspi | yes |
| `sh8601` | display | qspi | binding source present, no adapter entry |
| `rm690b0` | display | qspi | no |
| `ft3168` | touch | i2c | yes |
| `ft6336` | touch | i2c | no |
| `axp2101` | power | i2c | yes |
| `eta6098` | power | i2c | no |
| `qmi8658` | imu | i2c | yes |
| `mtk_factory_imu` | imu | host | no |
| `es8311` | audio | i2s | yes |
| `pcf85063` | rtc | i2c | no |

Two gaps are worth knowing. A chip can be catalogued with no `adapters` entry
(`sh8601`, `rm690b0`, `ft6336`, `eta6098`, `pcf85063`, `mtk_factory_imu`): the
portable source ships, but `gea chips add` refuses it on ESP32 because there is
no binding to link. And a binding source can exist with no catalog entry at all
(`touch/cst9217.cpp`, `imu/bmi270.cpp`, `gps/lc76g.cpp`): the target can link it
by hand, but no composed board can select it. Closing either gap — adding the
`adapters.esp32-idf.bindingSources` entry, or adding the catalog entry — is what
makes a chip available to `gea chips add`.

## Shared ESP32 Host Facades

`targets/targets/esp32/host_sources.cmake` lists the platform-universal host
sources: apps, audio, BLE, display, fetch, geolocation, image, media, memory,
input, GPIO/addressable LEDs, IMU, RTC, tile loader, timers, touch, WebSocket,
HTTP, and WiFi. GPIO/LED support is always linked on ESP32; it is not gated by
network, BLE or audio capability analysis.

## Display Driver Pattern

`display.cpp` depends on a small interface — `QspiPanel`
(`chip_bindings/displays/qspi_panel.h`). Each panel binding implements the
interface and exports a factory or driver-name function. Target CMake picks the
driver by linking one binding source file.

This avoids board-specific `#if` chains in shared display code.

## Optional Hardware

Camera is hardware-gated: it is not listed in `host_sources.cmake` for every
ESP32 target. Add optional hardware sources only to targets that have the
physical device and the required sdkconfig/component dependencies.

## Non-ESP Targets

RP2350 targets (`rp2350-waveshare-touch-amoled-2.41`, `rp2350-tufty-2350`) use
the `rp2350-pico` adapter. A chip shared with an ESP32 board should reuse the
generic `@geastack/chips` logic and implement its own target binding rather
than importing ESP-IDF code.
