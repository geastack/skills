# Flag Map

## `gea`

```
gea create <name>                              scaffold a new Gea project
gea setup [--board <alias>] [--target <id>]    guided board setup, or configure a board's build
gea doctor [--json] [--strict]                 check packages and toolchains

gea build  --board <alias> [--app <id>] [--verbose] [--configure-only] [--output file]
gea flash  --board <alias> [--app <id>] [--monitor] [--manual-boot] [--no-reset] [--flash-baud N]
           [--slot ota_N [--image f]] [--slot-image ota_N=f ...] [--erase-slot ota_N] [--restore-boot]
gea run    --board <alias> [--app <id>]        build + flash + monitor
gea ota    --board <alias> [--app <id>] [--monitor] [--logs]
           [--transport wifi|ble] [--device <name>] [--host <ip>]
           [--slot ota_N [--boot] [--reboot]] [--erase-slot ota_N]
gea clean  --board <alias> [--app <id>]

gea logs       --board <alias> [--follow] [--transport auto|usb|wifi]
gea monitor    --board <alias> [--timestamps] [--log-file f]
gea screenshot [file.png] --board <alias> [--transport auto|usb|wifi]
gea devctl <verb> ... --board <alias>

gea apps    list|inspect|pack|index|launcher|icons|icon-sheet|apple-icons
gea boards  list|show|add|set|remove|rename|discover
gea targets list|show <id>
gea chips   list|info|add|remove
gea heap-report [logs...] [--out file] [--map elf.map]
gea inspect [<app-id>] [--json]
gea list apps|targets|boards                   legacy alias
```

Global options: `--project <dir>`, `--boards-config <file>`,
`--global` | `--local` (which file a `gea boards` write lands in), `--dry-run`,
`--json`, `--verbose`.

Board aliases: `~/.geastack/boards.json` (this machine) merged with
`<project>/.gea/boards.json` (project overrides).

## `gea create` / `create-geastack`

- `--starter counter|blank|example`
- `--example <example-id>`
- `--dir <path>`, `--id <app-id>`, `--name <display-name>`
- `--examples-repo <git-url-or-local-path>`, `--examples-ref <git-ref>`
- `--targets web,esp32,rp2350,geaos,macos,ios,android`
- `--core-dependency <specifier>`, `--cli-dependency <specifier>`
- `--install` / `--no-install`
- `--dry-run`, `--yes`, `--force`

## App Manifest Build Inputs

- `gea.defines`: object or `NAME=value` array applied to the whole native build.
- `gea.cssDevicePixelRatio`: positive layout/font ratio for the app.
- `gea.nativeSources`: app-relative or package-resolved native files.
- `gea.compilerPlugins`: app-relative compiler plugins.
- `gea.targets.esp32`: `true` for defaults or an object with `componentDirs`,
  `componentRequires`, `linkOptions`, `ldFragments`, `embedFiles`, `sdkconfig`,
  `prebuild`, `output`, `partitions`, and `partitionsByTarget`.

App sdkconfig overrides matching board defaults and CLI board-policy settings.
App partitions override board partitions, with `partitionsByTarget[targetId]`
taking precedence over the app’s general `partitions`. `--output` is relative
to the working directory and overrides the app-relative manifest `output`.
See `$geastack-embedded-app-builder` for the field shapes.

## Environment

Project and toolchain:

- `GEA_PROJECT_ROOT`, `GEA_PROJECT_BUILD_ROOT`
- `GEA_HOME` (relocates `~/.geastack`)
- `GEA_BOARDS_CONFIG`
- `GEA_EXTRA_APP_DIRS`
- `GEA_VERBOSE`
- `GEA_EXAMPLES_REPO`, `GEA_EXAMPLES_REF`

ESP-IDF:

- `IDF_PATH`, `IDF_PYTHON_ENV_PATH`, `IDF_TOOLS_PATH`
- `GEA_EMBEDDED_IDF_EXPORT`, `ESP_IDF_EXPORT`
- `GEA_ESP_IDF_DIR`, `GEA_ESP_IDF_VERSION`
- `GEA_IDF_BUILD_VARIANT`, `GEA_IDF_GENERATOR`, `GEA_IDF_JOBS`, `GEA_IDF_CCACHE`
- `CMAKE`

Device and serial:

- `GEA_ESP32_FLASH_BAUD`, `GEA_ESP32_FLASH_RETRY_SECONDS`
- `GEA_ESP32_MANUAL_BOOT_GRACE_SECONDS`, `GEA_ESP32_MONITOR_WAIT_SECONDS`
- `GEA_SERIAL_BAUD`, `GEA_SERIAL_DEVICES`
- `GEA_EMBEDDED_BLE_OTA`, `GEA_BLE_OTA_DEVICE`

Other: `PICO_TOOLCHAIN_PATH` (RP2350), `GEA_SERIALIZE_HEAVY_BUILDS` /
`GEA_HEAVY_BUILD_LOCK_PATH`, `GEA_XBOX_SELF_SIGNED`.

Removed: `GEA_COLLECTION_ROOT`, `GEA_APPS_ROOT`, `GEA_EXAMPLES_ROOT`,
`GEA_CORE_DIR`/`GEA_CORE_ROOT`, `GEA_COMPILER_DIR`/`GEA_COMPILER_ROOT`,
`GEA_TARGETS_ROOT`, `GEA_SIMULATOR_ROOT`, `GEA_USE_YYJSON`.

## CMake Cache

- `GEA_EMBEDDED_ROOT`
- `GEA_EMBEDDED_APP`, `GEA_EMBEDDED_APP_META`, `GEA_EMBEDDED_APP_DIR`,
  `GEA_EMBEDDED_APP_ROOT`, `GEA_EMBEDDED_APP_ENTRY`
- `GEA_EMBEDDED_APP_NATIVE_SOURCES`, `GEA_EMBEDDED_APP_NATIVE_SOURCE_RELS`,
  `GEA_EMBEDDED_APP_NATIVE_INCLUDE_DIRS`
- `GEA_EMBEDDED_APP_GEATSC_PLUGINS`, `GEA_EMBEDDED_APP_GEATSC_PLUGIN_ARGS`
- `GEA_EMBEDDED_APP_DEFINES`, `GEA_EMBEDDED_APP_LDFRAGMENTS`
- `GEA_EMBEDDED_APP_COMPONENT_DIRS`, `GEA_EMBEDDED_APP_COMPONENT_REQUIRES`
- `GEA_EMBEDDED_APP_LINK_OPTIONS`, `GEA_EMBEDDED_APP_EMBED_FILES`
- `GEA_BOARD_DEFINITION`, `GEA_CUSTOM_TARGET_DIR`
- `GEA_EMBEDDED_CPP_BOARD`, `GEA_EMBEDDED_BOARD_SOURCES`
- `GEA_EMBEDDED_CSS_DEVICE_PIXEL_RATIO` (font generation); the generated
  `GEA_EMBEDDED_CSS_LAYOUT_DEVICE_PIXEL_RATIO` define controls layout
- `GEA_EMBEDDED_HTTPS_SUPPORT`, `GEA_EMBEDDED_ENABLE_HTTPS`,
  `GEA_EMBEDDED_HTTPS_REQUIRES`
- `GEA_EMBEDDED_TARGET_REQUIRES`, `GEA_EMBEDDED_TARGET_EXTRA_REQUIRES`,
  `GEA_EMBEDDED_TARGET_MAIN_DIR`

Removed: `GEA_EMBEDDED_RESIDENT_APPS`.

## Capability Definitions

Set from the compiler's `analyze` pass, not by hand:

- `GEA_EMBEDDED_CAPABILITY_NETWORK`
- `GEA_EMBEDDED_CAPABILITY_BLE`
- `GEA_EMBEDDED_CAPABILITY_AUDIO`
- `GEA_EMBEDDED_CAPABILITY_COMPILE_DEFINITIONS`
- `GEA_EMBEDDED_APP_USES_BLE`, `GEA_EMBEDDED_BLE_OTA`,
  `GEA_EMBEDDED_BLE_OTA_ENABLED`

Target CMake may store analysis results in internal `_GEA_EMBEDDED_HOST_BINDINGS`
and `_GEA_EMBEDDED_HOST_FEATURES` variables. `_GEA_EMBEDDED_HTTPS_SUPPORT_MODE`
is likewise derived internally; these are not user-facing cache controls.

## Common Compile Definitions

Display and rendering:

- `GEA_EMBEDDED_DISPLAY_WIDTH`, `GEA_EMBEDDED_DISPLAY_HEIGHT`
- `GEA_EMBEDDED_DISPLAY_NATIVE_WIDTH`, `GEA_EMBEDDED_DISPLAY_NATIVE_HEIGHT`
- `GEA_EMBEDDED_TARGET_DISPLAY_WIDTH` / `_HEIGHT` / `_NATIVE_WIDTH` / `_NATIVE_HEIGHT`
- `GEA_EMBEDDED_PIXEL_FORMAT`, `GEA_EMBEDDED_PIXEL_PANEL_ENDIAN`
- `GEA_EMBEDDED_DEFAULT_FRAME_INTERVAL_US`
- `GEA_EMBEDDED_DIRTY_REGION_MAX_RECTS`
- `GEA_EMBEDDED_FLUSH_ANCHOR_LEFT_PARTIAL_RECTS`
- `GEA_EMBEDDED_RENDER_PARALLEL_MIN_ROWS`, `GEA_EMBEDDED_RENDER_PARALLEL_MIN_PIXELS`
- `GEA_EMBEDDED_FRAME_SCHEDULER_USE_ESP_TIMER`
- `GEA_EMBEDDED_FRAME_SCHEDULER_PERF_LITE`
- `GEA_EMBEDDED_FRAME_SCHEDULER_DROP_CATCHUP_FRAMES`
- `GEA_EMBEDDED_FRAME_SCHEDULER_MAX_CATCHUP_FRAMES_BEFORE_YIELD`
- `GEA_EMBEDDED_APP_FRAME_TASK_CORE`, `GEA_EMBEDDED_APP_FRAME_TASK_PRIORITY`
- `GEA_EMBEDDED_SINGLE_THREAD_IMAGE_STORE`
- `GEA_EMBEDDED_NUMBER_F32`
- `GEA_EMBEDDED_FONT_COUNT`, `GEA_EMBEDDED_FONT_FAMILY_COUNT`,
  `GEA_EMBEDDED_HAS_GENERATED_FONTS`

Hardware availability (board-level, not app trimming):

- `GEA_EMBEDDED_WIFI_DISABLED`, `GEA_EMBEDDED_BLE_DISABLED`,
  `GEA_EMBEDDED_AUDIO_DISABLED`
- `GEA_EMBEDDED_DISPLAY_HEADLESS` (explicit offscreen canvas, no panel)
- `GEA_EMBEDDED_NO_DISPLAY` (neither panel nor canvas; skips app tree/frame loop)
- `GEA_BOARD_HAS_POWER` (generated from composed-board hardware)
- `GEA_EMBEDDED_DIAGNOSTICS_ENABLED` (opt in to TCP logs on port 8081)
- `GEA_EMBEDDED_DIAGNOSTICS_DISABLED` (overrides that opt-in)
Production lockdown uses the sdkconfig key
`CONFIG_GEA_EMBEDDED_PRODUCTION_LOCKDOWN`, not a same-named C++ define. The
current CLI build policy clears this key and selects INFO logging; changing a
board default alone does not enable lockdown through that path.

App-level display config generated into `gea_embedded_app_config.h`:

- `GEA_EMBEDDED_DISPLAY_FLUSH_CHUNK_MAX`
- `GEA_EMBEDDED_DISPLAY_FLUSH_QUEUE_DEPTH`

## Perf Flags

- `GEA_EMBEDDED_PERF` (master switch; `0` forces every perf path off)
- `GEA_FRAME_PERF_LOG`
- `GEA_EMBEDDED_FRAME_SCHEDULER_PERF_LOG`
- `GEA_EMBEDDED_FRAME_SCHEDULER_PERF_LITE`
- `GEA_EMBEDDED_UI_REFRESH_PERF`
- `GEA_EMBEDDED_CANVAS_PERF_DETAIL`
- `GEA_EMBEDDED_RAF_PERF`
- `GEA_EMBEDDED_HEAP_DIAGNOSTICS_LOG`, `GEA_EMBEDDED_HEAP_DIAGNOSTICS_LOG_VALUE`

## Compiler Runtime Flags

- `GEA_CPP_USE_TO_CHARS_DOUBLE` selects floating-point `to_chars` support.

Check the installed compiler runtime before adding macros. The current
`compiler/src/targets/cpp/runtime/gea_runtime.h` does not expose the legacy
`GEA_CPP_ENABLE_REGEX`, `GEA_RC_POOL_CACHE_BLOCKS`, or `GEA_CPP_DEBUG_TIMING`
controls; setting them is not a supported tuning mechanism.
