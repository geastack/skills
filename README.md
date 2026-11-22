# GeaStack Skills

Skills for building GeaStack embedded applications and hardware targets.

GeaStack is npm-first: an application installs `@geastack/cli`, and the `gea`
command pulls the core runtime, compiler, chips, host, engine, elements, and
target packages from npm. App workflows use installed packages; driver and
target development uses the corresponding source repositories.

For an installed app, check its package versions and local CLI help
(`npx gea --help`) before using a recently added option. The skills are based on CLI
`src/manifest.mjs`, `src/boards/custom-target.mjs`, `src/esp32/build.mjs`, and
`src/commands/board.mjs`; core `packages/core/index.d.ts` and
`packages/chips/catalog.json`; and the targets package’s `targets.json`.

## Skills

Building apps:

- `geastack-embedded-app-builder`: create and maintain TS/TSX GeaStack apps.
- `geastack-embedded-styling`: style embedded layouts, fonts, assets, and
  renderer-supported CSS.
- `geastack-embedded-canvas`: build fast `Display.ctx` and `<canvas>` apps.

Hardware:

- `geastack-embedded-target-bringup`: add built-in targets, board aliases,
  `targets.json` metadata, transports, CMake, and sdkconfig.
- `geastack-embedded-custom-boards`: compose an app-local board from the chip
  catalog with `gea chips` and `.gea/targets/<alias>.json`.
- `geastack-embedded-driver-development`: add chip drivers, catalog entries,
  platform bindings, and app-facing host APIs.

Build, deploy, debug:

- `geastack-embedded-build-flags`: choose and reason about CLI, CMake,
  sdkconfig, capability, perf, and compiler flags.
- `geastack-embedded-ota`: update boards over WiFi or BLE, and manage OTA slots
  and partitions.
- `geastack-embedded-device-control`: drive a running board with `gea devctl` —
  state, input injection, UI inspection, file transfer, display knobs.
- `geastack-embedded-debugging`: debug build, flash, monitor, serial, heap, FPS,
  and hardware issues.

## Install

Install the whole repo with the Skills CLI:

```sh
npx skills add geastack/skills -g -y
```

Install one skill by naming its folder with `--skill`:

```sh
npx skills add geastack/skills --skill geastack-embedded-canvas -g -y
npx skills add geastack/skills --skill geastack-embedded-target-bringup -g -y
```

Add `-a claude-code` or `-a codex` to install for one agent only. The Skills CLI
needs Node.js 22.20 or later.

Check and update installed skills:

```sh
npx skills check
npx skills update
```

Manual fallback, run from a clone of this repo. For Codex:

```sh
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
for skill in geastack-embedded-*; do
  ln -sfn "$PWD/$skill" "${CODEX_HOME:-$HOME/.codex}/skills/$skill"
done
```

For Claude Code:

```sh
mkdir -p "$HOME/.claude/skills"
for skill in geastack-embedded-*; do
  ln -sfn "$PWD/$skill" "$HOME/.claude/skills/$skill"
done
```

Skills installed before the rename carry the old `gea-embedded-*` names; remove
those so the two sets do not both trigger.

Each skill folder is self-contained and includes only the instructions and
references the agent needs.

## License

MIT (see `LICENSE`).
