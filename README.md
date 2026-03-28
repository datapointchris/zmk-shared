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

| Keyboard | Keys | Board | ZMK Source | OS Switching |
|---|---|---|---|---|
| Corne42 | 42 (3x6+3) | Nice!Nano v2 | upstream `main` | Runtime (conditional layers) |
| Glove80 | 80 | Built-in (glove80_lh/rh) | MoErgo fork `main` | Compile-time (shelved) |
| Piantor Pro BT | 42 (3x6+3) | Custom board defs | upstream `v0.3` | Runtime (conditional layers) |

## Shared Behaviors

All defined in `dts/shared_behaviors.dtsi` and available to every keyboard via `#include "shared_behaviors.dtsi"`.

### Modifier Macros

| Define | Keys | Usage |
|---|---|---|
| `HYPER` | Ctrl+Shift+GUI+Alt | Window manager |
| `MEH` | Ctrl+Shift+Alt | Shortcuts |
| `SUPER` | Ctrl+GUI+Alt | Shortcuts |

### Bluetooth

| Behavior | Description |
|---|---|
| `bt_0`..`bt_3` | Tap-dance: tap = select + BLE output, double-tap = disconnect |
| `bt_select_0`..`bt_select_3` | Macros: switch to BLE output + select profile |
| `ctrlaltdel` | Macro: Ctrl+Alt+Del |

### Home Row Mods

All use balanced flavor, hold-trigger-on-release. Standard mods use 280ms tapping-term, 175ms quick-tap, 150ms require-prior-idle. Shift-specific variants use faster timing (200ms/175ms/100ms).

| Behavior | Hand | Timing | Trigger Keys |
|---|---|---|---|
| `hml` | Left | Standard (280ms) | Right keys + all thumbs |
| `hmr` | Right | Standard (280ms) | Left keys + all thumbs |
| `hmls` | Left (shift) | Fast (200ms) | Right keys + all thumbs |
| `hmrs` | Right (shift) | Fast (200ms) | Left keys + all thumbs |
| `hmlt` | Left thumb | Standard (280ms) | Right keys + right thumbs |
| `hmrt` | Right thumb | Standard (280ms) | Left keys + left thumbs |

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

## Layer Architecture

Layers are numbered 0-8. Glove80 may not use all layers.

| # | Layer | Description |
|---|---|---|
| 0 | BASE | QWERTY + home row mods (GASC) + combos for brackets/symbols |
| 1 | COLEMAK | Colemak-DH, toggled via inner thumb combo |
| 2 | DEVLEFT | Programming symbols (left hand) |
| 3 | NPAD | Number pad (right) + navigation (left) |
| 4 | SYSTEM | Bluetooth, media, bootloader, RGB, OS toggle |
| 5 | NAV | Arrow keys + F1-F12 + sticky modifiers |
| 6 | WM | Window manager (Linux: Super+key, macOS: see WM_MAC) |
| 7 | OS_MAC | Ghost flag layer (all &trans) — toggles macOS mode |
| 8 | WM_MAC | macOS WM override (Alt+key) — conditional on WM + OS_MAC |

WM layer right side: top row = move, home row = focus, bottom row = join/group. LSHIFT on bottom row for move-to-workspace. Ungroup/flatten on right outer pinky.

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
