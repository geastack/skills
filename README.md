# GeaStack Skills

Skills for building GeaStack embedded applications and hardware targets.

GeaStack is npm-first: an application installs `@geastack/cli`, and the `gea`
command pulls the core runtime, compiler, chips, host, engine, elements, and
target packages from npm. App workflows use installed packages; driver and
target development uses the corresponding source repositories. The CLI has no
source-root override flags or board shell-script workflow.

Reviewed against the local CLI, core, targets, and compiler sources on
2026-09-19. For an installed app, check its package versions and local CLI help
before using a recently added option. The source authorities are CLI
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

Install one skill by suffixing the skill folder name:

```sh
npx skills add geastack/skills@geastack-embedded-canvas -g -y
npx skills add geastack/skills@geastack-embedded-target-bringup -g -y
```

Check and update installed skills:

```sh
npx skills check
npx skills update
```


Manual fallback:

```sh
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
for skill in geastack-embedded-*; do
  ln -sfn "$PWD/$skill" "${CODEX_HOME:-$HOME/.codex}/skills/$skill"
done
```

Skills installed before the rename carry the old `gea-embedded-*` names; remove
those so the two sets do not both trigger.

Each skill folder is self-contained and includes only the instructions and
references the agent needs.

## License

MIT (see `LICENSE`). Use it, change it, ship closed-source products on it, no
strings attached. The only GeaStack code under a different license is the
embedded board support (`targets` and `@geastack/chips`, GPL-3.0-only):
shipping closed-source firmware through those needs a commercial license.
Contact [contact@geastack.com](mailto:contact@geastack.com) for commercial terms, support and hosted builds.
