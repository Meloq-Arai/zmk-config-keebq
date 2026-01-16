# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Repository purpose

This repository is a ZMK (Zephyr Mechanical Keyboard) user configuration and custom shield definition for a KeebQ-style keyboard. It is built against the upstream `zmkfirmware/zmk` project and uses a local west workspace under `.zmk` pinned to ZMK revision `v0.3` (see `config/west.yml` and `.zmk/config/west.yml`).

## High-level architecture

### West workspace and upstream sources (`.zmk/` and `zephyr/`)

- `.zmk/` is the Zephyr **west workspace root** for this configuration.
  - `.zmk/.west/config` points west at the manifest in `.zmk/config/west.yml` and filters out some heavy projects.
  - `.zmk/zmk/` is a full checkout of the upstream ZMK firmware; treat this as vendored upstream code.
- `.zmk/config/west.yml` (and the copy at `config/west.yml`) defines the manifest:
  - Uses the `zmkfirmware` remote on GitHub.
  - Imports `zmk/app/west.yml` and sets `self.path: config`, meaning the `config/` directory in this repo is the user-config west project.
- `zephyr/` contains the Zephyr RTOS repository and related modules managed by west. You should not normally edit anything under `zephyr/` directly for this project; changes here are upstream Zephyr changes, not user-config changes.

### User configuration module (`config/`)

`config/` is the main user-config project that west sees as `self`:

- `config/west.yml` mirrors `.zmk/config/west.yml` and pins the ZMK revision (`v0.3`). If you upgrade ZMK, this is where the revision should be changed.
- `config/keebq.keymap` is a keymap file for the KeebQ configuration. It may be left empty or used to define shared layers; board/shield-specific keymaps can instead live under `boards/shields/...`.
- `config/boards/shields/keebq/KeebQ.overlay` defines a shield-level device tree overlay for the `KeebQ` shield name. It wires up:
  - The `chosen` nodes for `zmk,kscan` and `zmk,matrix_transform`.
  - A `kscan` node compatible with `zmk,kscan-gpio-matrix` and associated GPIO/diode direction properties.

This overlay is the configuration used when building with `-DSHIELD=KeebQ`.

### Custom shield definition (`boards/shields/keebq1`)

The `boards/shields/keebq1/` directory defines a more verbose custom shield named `keebq1`, following ZMK's "new shield" templates:

- `keebq1.zmk.yml` – hardware metadata file for the shield:
  - `id: keebq1`, `name: keebq`, `type: shield`.
  - `features:` includes `keys` and `studio`, indicating intended support for ZMK Studio.
  - `requires: [seeed_xiao]`, so this shield targets a Seeed XIAO-based board.
- `keebq1-layouts.dtsi` – defines a `default_layout` node compatible with `zmk,physical-layout` and a `keys` array mapping to `default_transform` and `kscan`.
- `keebq1.overlay` – shield overlay that:
  - Includes `keebq1-layouts.dtsi`.
  - Defines a `kscan` node using `zmk,kscan-gpio-matrix` with `col-gpios` and `row-gpios` using `&xiao_d` pins.
  - Defines `default_transform` (`zmk,matrix-transform`) with `rows`, `columns`, and `map` using `RC(r, c)`.
  - Sets `chosen { zmk,physical-layout = &default_layout; }`.
- `keebq1.keymap` – the default keymap for the `keebq1` shield:
  - Includes `<behaviors.dtsi>` and `<dt-bindings/zmk/keys.h>`.
  - Declares a `keymap` node with a `default_layer` and a small 2x2 example layout (`&kp A`, `&kp B`, `&kp C`, `&kp D`).
- `keebq1.conf` – a Kconfig fragment template for user-tunable options (currently only comments).

When working on the `keebq1` shield you will typically touch **all four** of these files together: overlay, layouts, keymap, and metadata.

### CI and build configuration

- `.github/workflows/build.yml` delegates builds to the reusable workflow `zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3`.
- `build.yaml` at the repo root defines the build matrix consumed by that reusable workflow. Currently it includes a single entry:
  - `board: rp2040`
  - `shield: KeebQ`

The CI build therefore validates the `rp2040` + `KeebQ` combination defined via `build.yaml` and the `config/boards/shields/keebq/KeebQ.overlay` overlay.

## Common commands

All commands below are intended to be run from the **repository root** (`zmk-config-keebq`). They assume the local west workspace lives under `.zmk/` as committed in this repo.

### Sync/initialize the west workspace

Use this when first checking out the repo or when updating to a new ZMK revision:

- Ensure `.zmk` exists and is initialized according to `.zmk/config/west.yml`:
  - `west -d .zmk update`

This tells west to use `.zmk` as the workspace directory (`-d .zmk`), read the manifest from `.zmk/config/west.yml`, and fetch/update all projects including `zmk` and `zephyr`.

### Build firmware

The main build target is the `rp2040` board with the `KeebQ` shield, matching the CI matrix.

- **Build the default firmware (rp2040 + KeebQ):**
  - `west -d .zmk build -p -b rp2040 zmk/app -- -DSHIELD=KeebQ`

Notes:
- `-d .zmk` runs west inside the committed workspace.
- `-p` triggers a **pristine** build (clean build directory) which is useful when switching boards/shields or after large changes.
- `-b rp2040` selects the board; `-DSHIELD=KeebQ` selects the shield and picks up `config/boards/shields/keebq/KeebQ.overlay`.
- The build directory defaults to `.zmk/build/`; you can add `-d .zmk/build/rp2040-keebq` if you want a named directory.

- **Example: build using the `keebq1` shield template** (for development and experimentation):
  - Choose an appropriate XIAO-based board (for example `seeed_xiao_rp2040` if that matches your hardware) and run:
  - `west -d .zmk build -p -b seeed_xiao_rp2040 zmk/app -- -DSHIELD=keebq1`

This uses the overlay/layout/keymap/metadata under `boards/shields/keebq1/` and the metadata in `keebq1.zmk.yml`.

### Linting and tests

This repository does **not** define its own linting or test targets:

- There are no local unit tests or test runners under this repo; validation is primarily via successful `west build` for the configured board/shield combinations and testing on hardware.
- Linting/formatting is inherited from the upstream ZMK project (`.zmk/zmk`) and Zephyr. If you need to run or extend those checks, do so from the ZMK repo in `.zmk/zmk` following the official ZMK documentation.

## Notes for future Warp agents

- Treat `.zmk/` and `zephyr/` as **vendored upstream dependencies**. Avoid editing files there unless you are intentionally modifying ZMK or Zephyr itself; user configuration should live under `config/` and `boards/shields/`.
- When adjusting the default build matrix used by CI, edit `build.yaml` rather than `.github/workflows/build.yml`. Add or modify `board`/`shield` combinations there and keep them in sync with the overlays/keymaps you create.
- For new shield variants, follow the structure used in `boards/shields/keebq1/`: create a directory with matching `.zmk.yml` metadata, `*.overlay`, `*-layouts.dtsi`, `*.keymap`, and optional `*.conf`, and then reference that shield name with `-DSHIELD=<your_shield>` in `west build` and in `build.yaml`.
