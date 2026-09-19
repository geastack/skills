# App Anatomy

## Files

- `package.json`: owns npm scripts, dependencies, and the `gea` manifest.
- the entry file from `gea.entry` (`src/index.tsx` in the bundled starter, a
  top-level `index.tsx` in most published examples).
- `styles.css`: imported by the entrypoint or top-level component.
- `tsconfig.json`: strict TypeScript, `"jsx": "preserve"`, `"types": []`, no
  `dom` lib and no `jsxImportSource` — `@geastack/core` declares the JSX
  namespace and the host globals ambiently.
- `vite.config.ts`: only for apps that also build for the web target.
- `.gea/boards.json`: project-local board aliases (overrides
  `~/.geastack/boards.json`).
- `.gea/targets/<alias>.json`: an app-local composed board definition, when the
  project uses one. See `$geastack-embedded-custom-boards`.
- `.gea/build/<target>/`: generated C++, CMake state, sdkconfig, and firmware.
  Build output, never edited by hand.
- `icons/`: launcher icons, commonly 32, 64, 128, 256, and 512 px PNGs.

## CLI Commands

```sh
npx gea doctor [--json] [--strict]
npx gea setup                                  # guided board setup
npx gea apps list [--json]
npx gea inspect [<app-id>] [--json]
npx gea build --board <alias> [--app <id>]
npx gea flash --board <alias> [--app <id>] [--monitor]
npx gea run --board <alias> [--app <id>]       # build + flash + monitor
npx gea ota --board <alias> [--app <id>]
npx gea monitor --board <alias>
npx gea logs --board <alias> [--follow]
npx gea screenshot shot.png --board <alias>
```

`gea list apps|targets|boards` still works and forwards to
`gea apps|targets|boards list`.

Global options: `--project <dir>`, `--boards-config <file>`, `--global`/`--local`
(where a `gea boards` write lands), `--dry-run`, `--json`, `--verbose`
(or `GEA_VERBOSE=1`).

There are no `--core-root`, `--compiler-root`, `--examples-root`,
`--simulator-root`, `--targets-root`, or `--collection-root` options: the CLI
resolves every package from the project's `node_modules`.

## Manifest Fields

- `gea.id`: stable app id used by build and flash commands.
- `gea.name`: display name.
- `gea.description`: one-line summary.
- `gea.entry`: source entry file.
- `gea.targets`: booleans or configuration objects for `web`, `esp32`,
  `rp2350`, `geaos`, `macos`, `ios`, `android`, `windows`, and `xbox`. Objects
  enable the platform unless `enabled: false`.
- `gea.ota.ble`: link the BLE OTA service into the firmware.
- `gea.icons`: map from numeric icon size to image path.
- `gea.launcher`: launcher metadata: `description`, `order`, `accent`, `hidden`.
- `gea.nativeSources`: app-relative or package-resolved native source files.
- `gea.compilerPlugins`: app-relative compiler plugin paths.
- `gea.defines`: macros as an object or an array of `NAME=value` strings;
  `true` emits a bare define, while false/null object values are omitted.
  These apply to the whole native build so app and framework type layouts agree.
- `gea.cssDevicePixelRatio`: positive app CSS pixel ratio; on ESP32 it sets
  both the layout ratio and the font-generation ratio.
- `gea.runtime`: legacy; usually omitted.

## ESP32 Native Build Configuration

Use a `gea.targets.esp32` object for app-owned build inputs:

| field | purpose |
| --- | --- |
| `componentDirs` | App-relative ESP-IDF component directories |
| `componentRequires` | Component names whose headers/public definitions the app needs; adding a directory alone does not expose them |
| `linkOptions` | Linker flags |
| `ldFragments` | App-relative `.lf` linker fragments |
| `embedFiles` | `{ symbol: appRelativeFile }` bytes embedded in the application image |
| `sdkconfig` | App defaults layered over board defaults, also overriding matching CLI board-policy keys |
| `prebuild` | Shell command run in the app directory before configure and partition processing |
| `output` | App-relative destination for the completed application binary |
| `partitions` | App-relative CSV path or a named partition object |
| `partitionsByTarget` | Same tables keyed by concrete target id, overriding `partitions` for that target |

Partition objects use `{ name: { type, subtype, size, offset?, flags?, data? } }`.
Sizes accept byte counts, hex, or `K`/`M` suffixes. `data` is an app-relative
payload flashed into that partition; unlike `embedFiles`, it is outside the
application image. Selection order is `partitionsByTarget[targetId]`, app
`partitions`, composed-board `partitions`, then the built-in table. Board
aliases are not keys in `partitionsByTarget`.

`--output <file>` overrides the manifest destination and resolves from the
working directory. `--configure-only` on `gea build` stops after configure.
The ordinary ESP32 build directory is
`.gea/build/<target>/app-builds/<app>/`; `GEA_IDF_BUILD_VARIANT` adds a variant
suffix to the app directory. Neither output option relocates this build tree.

## Creation Flags

`gea create` (also published as `create-geastack`) supports:

- `--starter counter|blank|example`
- `--example <example-id>`
- `--dir <path>`
- `--id <app-id>`
- `--name <display-name>`
- `--examples-repo <git-url-or-local-path>`
- `--examples-ref <git-ref>`
- `--targets web,esp32,rp2350,geaos,macos,ios,android`
- `--core-dependency <specifier>`
- `--cli-dependency <specifier>`
- `--install` / `--no-install`
- `--dry-run`
- `--yes`
- `--force`

Run with no options for the guided flow; it derives the source layout, targets,
and runtime configuration from the answers.
