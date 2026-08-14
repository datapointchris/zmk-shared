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
| --- | --- |
| `dts/shared_behaviors.dtsi` | All shared behaviors, macros, layer defines, WM macros |
| `zephyr/module.yml` | Zephyr module registration (DTS root) |
| `keymap_drawer.config.yaml` | Shared keymap-drawer styling config |

## Shared Behaviors Reference

### Layer Defines

```text
BASE=0, COLEMAK=1, DEVLEFT=2, NPAD=3, SYSTEM=4, NAV=5, WM=6, OS_MAC_LAYER=7, WM_MAC_LAYER=8, TMUX=9
```

### Home Row Mod Order (GASC)

All keyboards use a unified GASC order — the same keycodes on both OSes:

| Finger | Modifier (Left) | Modifier (Right) |
| -------- | ----------------- | ------------------- |
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

- `OS_MAC_LAYER` (layer 7): ghost flag layer (all `&trans`), toggled via `&tog OS_MAC_LAYER` on SYSTEM layer
- `WM_MAC_LAYER` (layer 8): macOS WM bindings using `LA()` instead of `LG()`
- Conditional layer: when WM + OS_MAC_LAYER are both active, WM_MAC_LAYER auto-activates on top
- Default is Linux (Super+key). Toggle OS_MAC_LAYER for macOS (Alt+key)

The `_LAYER` suffix is not decoration — `WM` is already a layer define, so the macOS layers cannot
take the bare names. Writing `&tog OS_MAC` does not compile.

Legacy compile-time switching (`-DDTS_EXTRA_CPPFLAGS=-DOS_MACOS`) is still supported via the `#ifdef` macros for keyboards that haven't migrated (e.g., Glove80).

### Modifier Macros

- `HYPER` = Ctrl+Shift+GUI+Alt
- `MEH` = Ctrl+Shift+Alt
- `SUPER` = Ctrl+GUI+Alt

### Bluetooth

- `bt_0`..`bt_3`: tap-dance (tap=select BLE profile, double-tap=disconnect)
- `bt_select_0`..`bt_select_3`: macros (switch output to BLE + select profile)
- `ctrlaltdel`: Ctrl+Alt+Del macro

### tmux Macros

`tmux_*` macros send the prefix (Ctrl+Space) then one key, for the actions tmux
only exposes through its prefix table. Anything reachable as a plain chord is
bound directly in the TMUX layer instead, because a chord auto-repeats when held
and a macro fires once per press.

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

Activated by holding left outermost thumb (`&mo WM`). Sends OS-appropriate WM keycodes using the `WMK()`, `WMSK()` and `WMCK()` macros. Uses QWERTY key positions so it works regardless of base layout.

## Build Tools

All keyboard repos are driven by `zmk`, run from inside them. There is no
Makefile — every derived path comes from the single `config/*.keymap`, which is
what each Makefile's three constants used to hold.

- `zmk`: `build`, `flash`, `align`, `draw`, `sync`, `check`, `clean`
- `keymap-align`: column alignment for .keymap files, called by `zmk align`
- `keymap` (keymap-drawer): YAML → SVG, called by `zmk draw`

`zmk draw` renders and does not parse. `keymap parse` cannot read these keymaps —
the conditional-layers node holds layer defines rather than integers — so each
`<stem>_keymap.yaml` is hand-written and a layer added to a keymap does not reach
the drawing. `zmk check` compares the counts, and it is the only thing that does.

## Cross-Repo Workflow

When changing shared behaviors:

1. Edit `dts/shared_behaviors.dtsi`
2. Test in ONE keyboard repo first: `cd ~/code/zmk/corne42 && zmk build`
3. Then rebuild others: `cd ~/code/zmk/glove80 && zmk build`, etc.

Local edits take effect immediately — `zmk` bind-mounts this directory into the Docker container via `ZMK_EXTRA_MODULES`. No push/pull cycle needed locally.

**Push this repo before the board repo.** CI has no bind-mount and resolves the module through each board's `config/west.yml`, which declares it from GitHub at `main`. Pushing the board first makes its run compile against a `main` that lacks the change. This repo has no workflow of its own, so nothing validates it until a board pushes.

## Build Pitfalls

**`ZEPHYR_EXTRA_MODULES` vs `ZMK_EXTRA_MODULES`** (⚠️ CRITICAL): Never pass `-DZEPHYR_EXTRA_MODULES` from the command line — ZMK uses this variable internally to register its own modules (board definitions like `nice_nano`). A CLI `-D` flag overrides `set()` in CMakeLists, clobbering ZMK's module list and causing "Invalid BOARD" / "No board named 'nice_nano' found" errors. Always use `-DZMK_EXTRA_MODULES` instead — ZMK prepends this to its own list.

**CMake cache poisoning**: A bad `-D` flag persists in `CMakeCache.txt` even after fixing the script. After changing any CMake flags, the next build MUST use `--pristine` to clear the cache. `--pristine` only wipes the build directory — it does NOT re-download the west workspace, so it's fast. `--clean` destroys the entire west workspace (~5-10 min re-download) and is almost never the right fix for board errors.

**The local checkout wins over the west manifest**: every board's `west.yml` declares `zmk-shared` from the `datapointchris` remote, but `zmk` bind-mounts `~/code/zmk/shared/` into the container via `-DZMK_EXTRA_MODULES`, which takes precedence. Local edits take effect immediately — no push/pull cycle for shared behavior changes. The manifest entry is what a build without the bind-mount would fall back to.

## Guardrails

- **Rebuild firmware after every keymap change** — run `zmk sync` (align + draw + build) before committing, then `zmk check`. Source changes without a build are useless; the UF2 file is what gets flashed. `check` is what catches the drawing falling behind, which `sync` alone does not. Applies to every board.
- Changes to `shared_behaviors.dtsi` affect ALL keyboards — test carefully
- Each keyboard defines its own `KEYS_L`, `KEYS_R`, `THUMBS_L`, `THUMBS_R` in its keymap (position numbers differ per keyboard)
- Layer defines are shared, but a board uses only what its keymap declares — corne42 carries all ten, piantor the first nine, glove80 the first seven. Count the layer blocks in a board's keymap rather than assuming.
- **A layer's index comes from the order of its block in `keymap`, not from its define.** Append a new layer last or the define points somewhere else, and nothing errors — the layer just does the wrong thing.
- The `hold-trigger-key-positions` in HRM behaviors reference the position macros, which must be defined before `#include "shared_behaviors.dtsi"`
- Keymap YAML files show GASC modifier labels: GUI, Alt, Shift, Ctrl (same on both OSes)
- **On boards with runtime OS switching (corne42, piantor)**: combos must include `OS_MAC_LAYER` in their `layers` property or they won't fire in macOS mode; Shift uses `hmls`/`hmrs` (faster timing) instead of `hml`/`hmr`. Glove80 uses compile-time OS switching and has no `OS_MAC` layer, so neither applies there.
