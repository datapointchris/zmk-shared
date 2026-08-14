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
| --- | --- | --- | --- | --- |
| Corne42 | 42 (3x6+3) | Nice!Nano v2 | upstream `main` | Runtime (conditional layers) |
| Glove80 | 80 | Built-in (glove80_lh/rh) | MoErgo fork `main` | Compile-time (shelved) |
| Piantor Pro BT | 42 (3x6+3) | Custom board defs | upstream `v0.3` | Runtime (conditional layers) |

## Shared Behaviors

All defined in `dts/shared_behaviors.dtsi` and available to every keyboard via `#include "shared_behaviors.dtsi"`.

### Modifier Macros

| Define | Keys | Usage |
| --- | --- | --- |
| `HYPER` | Ctrl+Shift+GUI+Alt | Window manager |
| `MEH` | Ctrl+Shift+Alt | Shortcuts |
| `SUPER` | Ctrl+GUI+Alt | Shortcuts |

### Bluetooth

| Behavior | Description |
| --- | --- |
| `bt_0`..`bt_3` | Tap-dance: tap = select + BLE output, double-tap = disconnect |
| `bt_select_0`..`bt_select_3` | Macros: switch to BLE output + select profile |
| `ctrlaltdel` | Macro: Ctrl+Alt+Del |

### Home Row Mods

All use balanced flavor, hold-trigger-on-release. Standard mods use 280ms tapping-term, 175ms quick-tap, 150ms require-prior-idle. Shift-specific variants use faster timing (200ms/175ms/100ms).

| Behavior | Hand | Timing | Trigger Keys |
| --- | --- | --- | --- |
| `hml` | Left | Standard (280ms) | Right keys + all thumbs |
| `hmr` | Right | Standard (280ms) | Left keys + all thumbs |
| `hmls` | Left (shift) | Fast (200ms) | Right keys + all thumbs |
| `hmrs` | Right (shift) | Fast (200ms) | Left keys + all thumbs |
| `hmlt` | Left thumb | Standard (280ms) | Right keys + right thumbs |
| `hmrt` | Right thumb | Standard (280ms) | Left keys + left thumbs |

### Layer-Tap (same timing as HRM)

| Behavior | Hand | Bindings | Trigger Keys |
| --- | --- | --- | --- |
| `ltl` | Left | `&mo`, `&kp` | Right keys + all thumbs |
| `ltr` | Right | `&mo`, `&kp` | Left keys + all thumbs |
| `ltlt` | Left thumb | `&mo`, `&kp` | Right keys + right thumbs |
| `ltrt` | Right thumb | `&mo`, `&kp` | Left keys + left thumbs |

### Tap-Dance

| Behavior | Tap | Double-tap |
| --- | --- | --- |
| `caps_shift` | Right Shift | Caps Word |

## Layer Architecture

Layers are numbered 0-8. Glove80 may not use all layers.

| # | Layer | Description |
| --- | --- | --- |
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

Every keyboard repo is driven by the same tool, run from inside it:

| Tool | Purpose | Install |
| --- | --- | --- |
| `zmk` | Build, draw, check and flash | Shared tool |
| `keymap-align` | Align keymap .dtsi columns | Shared tool |
| `keymap` (keymap-drawer) | Generate SVG from YAML | `uv tool install keymap-drawer` |

`zmk --help` is the command reference. There is no Makefile — every path a repo
needs derives from its single `config/*.keymap`, which is what the three
constants at the top of each Makefile used to hold.

```sh
zmk check     # what is missing or has drifted
zmk sync      # align + draw + build
zmk flash     # pick halves with fzf, then write each
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
3. Update that keyboard's `<stem>_keymap.yaml` to match — it is hand-written,
   because `keymap parse` cannot read these keymaps
4. Run `zmk sync` to align, redraw and build
5. Run `zmk check` to confirm the drawing did not fall behind the keymap
6. Run `zmk flash` and double-tap reset on each half as it asks

## Files

```text
dts/shared_behaviors.dtsi  - All shared behaviors, macros, and layer defines
zephyr/module.yml          - Registers this repo as a Zephyr module
keymap_drawer.config.yaml  - Shared keymap-drawer styling (dark theme)
```
