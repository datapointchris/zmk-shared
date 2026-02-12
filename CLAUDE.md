# zmk-shared AI Context

## Directory Layout

All ZMK keyboard repos live under `~/code/zmk/`:

```text
~/code/zmk/
├── shared/     ← this repo
├── corne42/    Corne42 config
├── glove80/    Glove80 config
└── piantor/    Piantor Pro BT config
```

When working on shared behaviors, check sibling repos under `~/code/zmk/` to understand how they're used.

## Key Files

| File | Purpose |
|---|---|
| `dts/shared_behaviors.dtsi` | All shared behaviors, macros, layer defines, WM macros |
| `zephyr/module.yml` | Zephyr module registration (DTS root) |
| `keymap_drawer.config.yaml` | Shared keymap-drawer styling config |

## Shared Behaviors Reference

### Layer Defines
```text
BASE=0, COLEMAK=1, DEVLEFT=2, NPAD=3, SYSTEM=4, NAV=5, WM=6
```

### Home Row Mod Order (GASC)

All keyboards use a unified GASC order — the same keycodes on both OSes:

| Finger | Modifier (Left) | Modifier (Right) |
|--------|-----------------|-------------------|
| Pinky (A/;) | LGUI | RGUI |
| Ring (S/L) | LALT | RALT |
| Middle (D/K) | LSHIFT | RSHIFT |
| Index (F/J) | LCTRL | RCTRL |

Keymaps use actual keycodes directly (e.g., `&hml LGUI A`), not MOD_* defines.

### WM Macros

- `WMK(key)` — sends Alt+key on macOS, Super+key on Linux
- `WMSK(key)` — sends Alt+Shift+key on macOS, Super+Shift+key on Linux

Note: Named `WMK`/`WMSK` (not `WM`/`WMS`) to avoid colliding with the `WM` layer define.

### OS Build Configuration

The `OS_MACOS` define only affects `WMK()`/`WMSK()` macros (AeroSpace vs i3/sway shortcuts). Home row mods are identical on both OSes. To build for macOS, add `cmake-args: -DDTS_EXTRA_CPPFLAGS=-DOS_MACOS` to `build.yaml`. Both variants can coexist in the same `build.yaml` with different `artifact-name` values.

### Modifier Macros

- `HYPER` = Ctrl+Shift+GUI+Alt
- `MEH` = Ctrl+Shift+Alt
- `SUPER` = Ctrl+GUI+Alt
- `AERO` = Ctrl+Shift+Alt+; (AeroSpace window manager)

### Bluetooth

- `bt_0`..`bt_3`: tap-dance (tap=select BLE profile, double-tap=disconnect)
- `bt_select_0`..`bt_select_3`: macros (switch output to BLE + select profile)
- `ctrlaltdel`: Ctrl+Alt+Del macro

### Home Row Mods (all balanced, 280ms tapping-term, 175ms quick-tap, 150ms require-prior-idle)

- `hml`/`hmr`: left/right hand hold-tap (`&kp`, `&kp`)
- `hmlt`/`hmrt`: left/right thumb variants
- `ltl`/`ltr`: left/right layer-tap (`&mo`, `&kp`)
- `ltlt`/`ltrt`: left/right thumb layer-tap variants

### Tap-Dance

- `caps_shift`: tap=RShift, double-tap=Caps Word
- `caps_aero`: tap=AERO, double-tap=Caps Word

### Colemak-DH Layer

Toggle via combo: press both innermost thumbs (pos 38+39) simultaneously. Only redefines letter keys and HRM letters; everything else is `&trans` (falls through to BASE). Active on both BASE and COLEMAK layers.

### WM Layer

Activated by holding left outermost thumb (`&mo WM`). Sends OS-appropriate WM keycodes using `WM()` and `WMS()` macros. Uses QWERTY key positions so it works regardless of base layout.

## Build Tools

All keyboard repos use the same tools and Make targets:
- `zmk-build`: Docker-based west build
- `keymap-align`: column alignment for .keymap files
- `keymap` (keymap-drawer): YAML → SVG keymap visualization
- Make targets: `align`, `draw`, `build`, `sync` (all three), `clean`

## Cross-Repo Workflow

When changing shared behaviors:
1. Edit `dts/shared_behaviors.dtsi`
2. Test in ONE keyboard repo first: `cd ~/code/zmk/corne42 && make build`
3. Then rebuild others: `cd ~/code/zmk/glove80 && make build`, etc.
4. Each keyboard's west.yml pulls `zmk-shared@main`, so changes are picked up on next `make build`

## Guardrails

- Changes to `shared_behaviors.dtsi` affect ALL keyboards — test carefully
- Each keyboard defines its own `KEYS_L`, `KEYS_R`, `THUMBS_L`, `THUMBS_R` in its keymap (position numbers differ per keyboard)
- Layer defines are shared — all keyboards use all 7 layers
- The `hold-trigger-key-positions` in HRM behaviors reference the position macros, which must be defined before `#include "shared_behaviors.dtsi"`
- Keymap YAML files show GASC modifier labels: GUI, Alt, Shift, Ctrl (same on both OSes)
