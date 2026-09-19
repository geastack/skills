# Runtime Surface

Import from `@geastack/core` unless the API is listed as global. Globals are
declared ambiently by `@geastack/core`'s `declare global` block and must NOT be
imported.

## UI

- `mount(App)`, `Component`, `ReactiveComponent`, `Store`
- JSX tags: `body`, `div`, `span`, `p`, `h1`-`h6`, `a`, `b`, `i`, `em`,
  `strong`, `small`, `label`, `code`, `u`, `sub`, `sup`, `mark`, `button`,
  `input`, `textarea`, `canvas`, `camera`, `img`, `audio`, `virtual-list`
- Events: `onClick`, `onPress`, `onTouchStart`, `onTouchMove`, `onTouchEnd`,
  `onRotary`, `onKeyDown`
- DOM-ish APIs: `document.querySelector`, `querySelectorAll`, `getElementById`,
  `createElement`, `classList`, `scrollIntoView`

## Display And Input

- `Display.ctx`
- `Display.width`, `height`, `nativeWidth`, `nativeHeight`
- `Display.setFrameRate(fps)`, `setFrameIntervalMs(ms)`,
  `setFlushConfig({ rows, depth })`, `setMemoryConfig({ commandBufferCommands, backgroundCache })`
- `Display.setVSync(on)`, `setTextRasterCache(on)`, `setTextSolidBackdrop(rrggbb)`,
  `invalidate()`, `setAA(samples)`
- `Display.getBrightness()`, `setBrightness(0-100)`
- `Display.orientation`, `setOrientation()`, `setSupportedOrientations()`,
  `setAutoRotate()`
- `Display.pixelFormat`, `panelPixelFormat`, `supportedPixelFormats`,
  `setPixelFormat()`, `getDevicePixelRatio()`, `setDevicePixelRatio()`
- `Display.setEpaperRefreshConfig(config)`, `epaperFullRefresh()` (e-paper only)
- `Input.consumeBackButton()`
- `Gpio.configureInput(pin, pullUp?)`, `configureOutput(pin)`, `read(pin)`,
  `write(pin, high)`; `Led.attach(pin, count)`, `setPixel(index, r, g, b)`,
  `show()` for addressable LEDs
- `touch.read()` for a raw `{ touching, x, y }` sample
- global `requestAnimationFrame(callback)`

## Device

- `Memory.stats()`, `Memory.config()`, and individual memory methods
- `Battery.level()`
- `Clock.epochMs()` for wall-clock time after sync (`Date.now()` is monotonic on
  ESP32, so it is NOT wall-clock)
- `Profiler.nowUs()`, `Profiler.nowCycles()` for temporary instrumentation
- `Notify.text()`, `Notify.seq()`
- `Apps.launch(appId)`
- `DeviceControl.exec(command)` (macOS companion only)
- global `localStorage` (persistent on ESP32 through the `storage` SPIFFS
  partition when `GEA_STORAGE_SPIFFS` is enabled and mounted; otherwise NVS)
- globals `setTimeout`, `clearTimeout`, `setInterval`, `clearInterval` —
  native timers, numeric handles, callback takes no arguments
- global `console.log` / `console.error` only

## Connectivity And Media

- `WiFi` / `wifi`: enable, configure, scan, connection status, IP, RSSI
- `BLE` / `bluetooth`: HID keyboard/mouse/MIDI, HID host, advertising,
  connection state; `initBleServers()`, `registerBleServer()`, `BLEServer`
- `fetch`, `fetchAsync`, `fetchReady`, `fetchResult`, `fetchRelease`
- `fetchUploadFileAsync`, `fetchDownloadFileAsync`, upload progress helpers
- `WebSocket`
- `http.createServer()`
- `Audio`, `audioContext`
- `MediaDevices`, `MediaRecorder`, `MediaStream`
- `RTCPeerConnection`
- `Camera` and `<camera>`
- `Geolocation` / `geolocation`
- `Accelerometer`

Reaching any of `wifi`, `fetch`, `http`, `websocket`, `rtc`, `ble`, or `audio`
is what makes the CLI link that subsystem into the firmware. An app that never
touches the network produces a board with no networking at all.

## Images And Files

- `loadImage`, `loadImageWithOpaque`, `loadImageFile`, `loadAssetImage`,
  `imageFromId`
- `writeCacheFile`, `readCacheFile`, `listCacheFiles`, `readFileRange`,
  `readMapArchive`
- `fetchBytes`, `fetchText`

## Colors

- `rgb(r, g, b)` / `rgba(r, g, b, a)` — exported from `@geastack/core`; pack a
  canvas color as `0xRRGGBBAA`.
- `rgb565(r, g, b)` — an ambient **global**, not an import. Returns an `Rgb565`
  already in the board's native pixel format; a literal call folds to a
  constant at build time. Prefer it for hot palettes drawn every frame.
