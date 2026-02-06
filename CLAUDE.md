# zmk-shared AI Context

## Directory Layout

All ZMK keyboard repos live under `~/code/zmk/`:

```text
~/code/zmk/
├── shared/     ← this repo
├── corne42/    Corne42 config
├── glove80/    Glove80 config
└── piantor/    Piantor Pro BT config (+ Corne Choc Pro, Sofle Choc Pro)
```

When working on shared behaviors, check sibling repos under `~/code/zmk/` to understand how they're used.

## Key Files

| File | Purpose |
|---|---|
| `dts/shared_behaviors.dtsi` | All shared behaviors, macros, layer defines |
| `zephyr/module.yml` | Zephyr module registration (DTS root) |
| `keymap_drawer.config.yaml` | Shared keymap-drawer styling config |

## Shared Behaviors Reference

### Layer Defines
```text
BASE=0, DEVLEFT=1, DEVRIGHT=2, NPAD=3, SYSTEM=4, MOUSE=5, NAV=6
```

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
- Layer defines are shared, but not all keyboards use all 7 layers (Corne42 skips DEVRIGHT and MOUSE)
- The `hold-trigger-key-positions` in HRM behaviors reference the position macros, which must be defined before `#include "shared_behaviors.dtsi"`
