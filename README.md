# zmk-shared

Shared ZMK module providing common behaviors, macros, and configuration for all keyboard repos.

## Ecosystem Layout

```text
~/code/zmk/
├── shared/     ← you are here (datapointchris/zmk-shared)
├── corne42/    (datapointchris/zmk-config-corne42)
├── glove80/    (datapointchris/zmk-config-glove80)
└── piantor/    (datapointchris/zmk-config-piantor-pro-bt)
```

## Keyboards

| Keyboard | Keys | Board | ZMK Source | Layers |
|---|---|---|---|---|
| Corne42 | 42 (3x6+3) | Nice!Nano v2 | upstream `main` | 5: BASE, DEVLEFT, NPAD, SYSTEM, NAV |
| Glove80 | 80 | Built-in (glove80_lh/rh) | MoErgo fork `main` | 7: all |
| Piantor Pro BT | 42 (3x6+3) | Custom board defs | upstream `v0.3` | 7: all |

Piantor also builds firmware for Corne Choc Pro and Sofle Choc Pro (same keymap, different boards).

## Shared Behaviors

All defined in `dts/shared_behaviors.dtsi` and available to every keyboard via `#include "shared_behaviors.dtsi"`.

### Modifier Macros

| Define | Keys | Usage |
|---|---|---|
| `HYPER` | Ctrl+Shift+GUI+Alt | Window manager |
| `MEH` | Ctrl+Shift+Alt | Shortcuts |
| `SUPER` | Ctrl+GUI+Alt | Shortcuts |
| `AERO` | Ctrl+Shift+Alt+; | AeroSpace WM trigger |

### Bluetooth

| Behavior | Description |
|---|---|
| `bt_0`..`bt_3` | Tap-dance: tap = select + BLE output, double-tap = disconnect |
| `bt_select_0`..`bt_select_3` | Macros: switch to BLE output + select profile |
| `ctrlaltdel` | Macro: Ctrl+Alt+Del |

### Home Row Mods

All use balanced flavor, 280ms tapping-term, 175ms quick-tap, 150ms require-prior-idle, hold-trigger-on-release.

| Behavior | Hand | Bindings | Trigger Keys |
|---|---|---|---|
| `hml` | Left | `&kp`, `&kp` | Right keys + all thumbs |
| `hmr` | Right | `&kp`, `&kp` | Left keys + all thumbs |
| `hmlt` | Left thumb | `&kp`, `&kp` | Right keys + right thumbs |
| `hmrt` | Right thumb | `&kp`, `&kp` | Left keys + left thumbs |

### Layer-Tap (same timing as HRM)

| Behavior | Hand | Bindings | Trigger Keys |
|---|---|---|---|
| `ltl` | Left | `&mo`, `&kp` | Right keys + all thumbs |
| `ltr` | Right | `&mo`, `&kp` | Left keys + all thumbs |
| `ltlt` | Left thumb | `&mo`, `&kp` | Right keys + right thumbs |
| `ltrt` | Right thumb | `&mo`, `&kp` | Left keys + left thumbs |

### Tap-Dance

| Behavior | Tap | Double-tap |
|---|---|---|
| `caps_shift` | Right Shift | Caps Word |
| `caps_aero` | AERO modifier | Caps Word |

## Layer Architecture

Layers are numbered 0-6. Not all keyboards use every layer.

| # | Layer | Description |
|---|---|---|
| 0 | BASE | QWERTY + home row mods + combos for brackets/symbols |
| 1 | DEVLEFT | Programming symbols (left hand active, right hand blank) |
| 2 | DEVRIGHT | Programming symbols (right hand active, left hand nav) |
| 3 | NPAD | Number pad (right) + nav/media (left) |
| 4 | SYSTEM | Bluetooth, bootloader, media, RGB |
| 5 | MOUSE | Mouse movement, scroll, clicks |
| 6 | NAV | Arrow keys + F-keys + sticky modifiers |

## Build System

Each keyboard repo uses the same toolchain via Make:

| Tool | Purpose | Install |
|---|---|---|
| `zmk-build` | Docker-based west build | Shared tool |
| `keymap-align` | Align keymap .dtsi columns | Shared tool |
| `keymap` (keymap-drawer) | Generate SVG from YAML | `pipx install keymap-drawer` |

### Make Targets (same in all keyboard repos)

```sh
make align    # Align keymap formatting
make draw     # Generate SVG visualization from YAML
make build    # Build firmware via zmk-build (Docker)
make sync     # align + draw + build
make clean    # Remove .uf2 files
```

## How It Works

Each keyboard's `config/west.yml` pulls this repo as a Zephyr module:

```yaml
projects:
  - name: zmk-shared
    remote: datapointchris
    revision: main
```

The `zephyr/module.yml` registers the DTS root, making `shared_behaviors.dtsi` available to all keymaps via `#include "shared_behaviors.dtsi"`.

## Keymap Editing Workflow

1. Edit shared behavior in `dts/shared_behaviors.dtsi` (affects ALL keyboards)
2. Or edit a specific keyboard's keymap in its repo
3. Run `make align` to format the keymap
4. Run `make draw` to regenerate the SVG
5. Run `make build` to compile firmware
6. Flash `.uf2` files to keyboard

## Files

```text
dts/shared_behaviors.dtsi  - All shared behaviors, macros, and layer defines
zephyr/module.yml          - Registers this repo as a Zephyr module
keymap_drawer.config.yaml  - Shared keymap-drawer styling (dark theme)
```
