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
BASE=0, COLEMAK=1, DEVLEFT=2, NPAD=3, SYSTEM=4, NAV=5, WM=6, OS_MAC=7, WM_MAC=8
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

- `WMK(key)` — sends Alt+key on macOS, Super+key on Linux (focus, workspaces)
- `WMSK(key)` — sends Alt+Shift+key on macOS, Super+Shift+key on Linux (move windows)
- `WMCK(key)` — sends Alt+Ctrl+Shift+key on macOS, Super+Ctrl+key on Linux (join/group windows)

Note: Named `WMK`/`WMSK` (not `WM`/`WMS`) to avoid colliding with the `WM` layer define.

### OS Switching

Runtime OS switching via conditional layers (no separate macOS/Linux firmware builds needed):
- `OS_MAC` (layer 7): ghost flag layer (all `&trans`), toggled via `&tog OS_MAC` on SYSTEM layer
- `WM_MAC` (layer 8): macOS WM bindings using `LA()` instead of `LG()`
- Conditional layer: when WM + OS_MAC are both active, WM_MAC auto-activates on top
- Default is Linux (Super+key). Toggle OS_MAC for macOS (Alt+key)

Legacy compile-time switching (`-DDTS_EXTRA_CPPFLAGS=-DOS_MACOS`) is still supported via the `#ifdef` macros for keyboards that haven't migrated (e.g., Glove80).

### Modifier Macros

- `HYPER` = Ctrl+Shift+GUI+Alt
- `MEH` = Ctrl+Shift+Alt
- `SUPER` = Ctrl+GUI+Alt
### Bluetooth

- `bt_0`..`bt_3`: tap-dance (tap=select BLE profile, double-tap=disconnect)
- `bt_select_0`..`bt_select_3`: macros (switch output to BLE + select profile)
- `ctrlaltdel`: Ctrl+Alt+Del macro

### Home Row Mods (all balanced, hold-trigger-on-release)

- `hml`/`hmr`: left/right hand (280ms tapping-term, 150ms prior-idle)
- `hmls`/`hmrs`: left/right shift-specific (200ms tapping-term, 100ms prior-idle — faster for capitals)
- `hmlt`/`hmrt`: left/right thumb variants (280ms)
- `ltl`/`ltr`: left/right layer-tap (`&mo`, `&kp`)
- `ltlt`/`ltrt`: left/right thumb layer-tap variants

### Tap-Dance

- `caps_shift`: tap=RShift, double-tap=Caps Word

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
