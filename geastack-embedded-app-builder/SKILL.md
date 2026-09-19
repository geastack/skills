---
name: geastack-embedded-app-builder
description: Build, modify, and review GeaStack embedded apps written in TypeScript or TSX with @geastack/core, package.json gea manifests, gea create, stores, components, host APIs, nativeSources, launcher metadata, and hardware deploy loops. Use when creating a new GEA app, editing an app entrypoint, adding scripts, setting target compatibility, wiring icons or launcher metadata, or making app-level changes for embedded devices.
---

# GeaStack Embedded App Builder

GeaStack is npm-first: an app installs `@geastack/cli`, and the CLI pulls the
core runtime, compiler, chips, host, engine, elements, and target packages from
npm. App builds resolve installed packages; they do not require a GeaStack
source checkout or accept source-root override flags.

## Workflow

Start by identifying the app root and manifest:

- Prefer `npx gea inspect --json` when inside an app.
- Otherwise inspect `package.json` for `gea.id`, `gea.entry`, and `gea.targets`.
- Check `tsconfig.json`, the entrypoint, styles, stores, and component folders
  before editing.
- Read `references/app-anatomy.md` for manifest and command details when
  creating or restructuring an app.
- Read `references/runtime-surface.md` when using host APIs such as `Display`,
  `Memory`, `WiFi`, `Camera`, `BLE`, `localStorage`, fetch, or WebSocket.

For a new app, prefer the CLI scaffold:

```sh
npx gea create my-app
npx gea create my-app --starter blank --targets web,esp32 --yes --no-install
npx gea create my-app --starter example --example analog-clock
```

`gea create` runs a guided flow when given no options; `--starter counter`
copies the bundled component-counter starter, `--starter blank` generates a
minimal entry, and `--starter example` fetches one of the published examples.

If creating files manually, include the same essentials:

- an entry file matching `gea.entry` (the counter starter uses `src/index.tsx`;
  older examples use a top-level `index.tsx`).
- `package.json` with explicit `gea.targets` booleans or configuration objects.
- `tsconfig.json` with `"jsx": "preserve"` and `"types": []`. Do **not** set
  `jsxImportSource`: the JSX namespace is ambient, declared by `@geastack/core`.
- a stylesheet, icons when launcher-visible, and `.gea/boards.json` for
  project-local board aliases.
- `vite.config.ts` only when the app also builds for the web target.

## App Patterns

Use the existing app style:

- Retained UI: import `mount` and a `Component`/`ReactiveComponent` class,
  import CSS, then `mount(App)`.
- Immediate graphics: use `$geastack-embedded-canvas` for `Display.ctx`, frame
  loops, and typed-array drawing.
- Keep app state in focused stores or `ReactiveComponent` when state belongs to
  one component.
- Use `document.querySelector`, `classList`, `setAttribute`, event handlers, and
  typed refs only as declared by `@geastack/core`.
- Prefer typed arrays, records, and explicit narrow types in hot code so the
  compiler can lower to native structures.
- Use `JSON.parse(text) as Type` for structured data; do not hand-roll JSON
  scanning.

## Manifest Rules

Keep the app manifest boring and explicit:

```json
{
  "gea": {
    "id": "watch-face",
    "name": "Watch Face",
    "entry": "src/index.tsx",
    "targets": {
      "web": true,
      "esp32": true,
      "rp2350": false,
      "geaos": true,
      "macos": false,
      "ios": false
    },
    "ota": { "ble": true }
  }
}
```

- `id` is the stable command-line app id.
- `entry` must exist.
- `targets` gates CLI routing. Each entry is `true` for defaults, `false` to
  disable, or a configuration object (enabled unless `enabled: false`). Known
  platforms: `web`, `esp32`, `rp2350`, `geaos`, `macos`, `ios`, `android`,
  `windows`, `xbox`. Enable only platforms the app supports.
- `ota.ble: true` makes the firmware link the BLE OTA service so
  `gea ota --transport ble` can reach the board without a cable.
- `icons` maps sizes such as `32`, `64`, `128`, `256`, `512` to files.
- `launcher` can set `description`, `order`, `accent`, and `hidden`.
- `nativeSources` accepts app-relative or package-resolved `.c`, `.cc`, `.cpp`,
  `.cxx`, `.m`, `.mm`, or `.S` files. An existing app-relative file wins.
- `compilerPlugins` lists app-local compiler plugins for native host bindings.
- `defines` applies macros to the whole native build; `cssDevicePixelRatio`
  keeps app layout and generated font scaling aligned.
- `targets.esp32` can configure components, linker options, embedded files,
  sdkconfig, prebuild commands, output, and partition tables (including
  `partitionsByTarget`). See `references/app-anatomy.md`.
- `runtime` is legacy; most manifests omit it. Apple-native apps may set it.

The firmware's capabilities are otherwise **derived**, not declared: the
compiler analyses the entry and reports which host bindings the program reaches,
and only then is WiFi/LwIP, the BLE host, or the audio pipeline linked. Adding a
`fetch` call changes the firmware's size and boot behavior; see
`$geastack-embedded-build-flags`.

## Verification

Run the smallest useful checks for the change:

```sh
npm run check
npx gea inspect --json
npx gea doctor
npx gea build --board <alias> --dry-run
```

For hardware-facing changes, use `npx gea flash --board <alias> --monitor` (or
`npx gea run --board <alias>`, which is flash plus monitor) only after a dry run
or build succeeds.
