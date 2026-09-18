# zmk-shared AI Context

This module is consumed by the board repos beside it under `~/code/zmk/`: corne42, glove80 and piantor. Check how they use a behavior before changing it.

`dts/shared_behaviors.dtsi` defines every shared behavior and macro, with its timings. Read values there, not here.

## Shared Behaviors Reference

### Layer Defines

Per-board, at the top of each keyboard's `.keymap`. A layer's index is its
block's position in `keymap`, so one shared list forced every board into the
same order — a seven-layer Glove80 could not have `WM` at index eight, and
adding a layer to one keyboard meant editing all three. The names are identical
everywhere; only the numbering is local.

### Home Row Mod Order (GASC)

Pinky GUI, ring Alt, middle Shift, index Ctrl, mirrored on the right hand. The keycodes are the same on both OSes, and so are the drawer labels.

Keymaps use real keycodes directly (e.g., `&hml LGUI A`), not `MOD_*` defines.

### WM Macros

- `WMK(key)` — sends Alt+key on macOS, Super+key on Linux (focus, workspaces)
- `WMSK(key)` — sends Alt+Shift+key on macOS, Super+Shift+key on Linux (move windows)
- `WMCK(key)` — sends Alt+Ctrl+Shift+key on macOS, Super+Ctrl+key on Linux (join/group windows)

Note: Named `WMK`/`WMSK` (not `WM`/`WMS`) to avoid colliding with the `WM` layer define.

### OS Switching

Runtime OS switching via conditional layers (no separate macOS/Linux firmware builds needed):

- `OS_MAC_LAYER`: ghost flag layer (all `&trans`), toggled via `&tog OS_MAC_LAYER` on SYSTEM layer
- `WM_MAC_LAYER`: macOS WM bindings using `LA()` instead of `LG()`. It must be numbered **above** both `WM` and `OS_MAC_LAYER`, since a conditional layer activates on top
- Conditional layer: when WM + OS_MAC_LAYER are both active, WM_MAC_LAYER auto-activates on top
- Default is Linux (Super+key). Toggle OS_MAC_LAYER for macOS (Alt+key)

The `_LAYER` suffix is not decoration — `WM` is already a layer define, so the macOS layers cannot
take the bare names. Writing `&tog OS_MAC` does not compile.

Legacy compile-time switching (`-DDTS_EXTRA_CPPFLAGS=-DOS_MACOS`) is still supported via the `#ifdef` macros for keyboards that haven't migrated (e.g., Glove80).

### tmux Macros

`tmux_*` macros send the prefix (Ctrl+Space) then one key, for the actions tmux
only exposes through its prefix table. Anything reachable as a plain chord is
bound directly in the TMUX layer instead, because a chord auto-repeats when held
and a macro fires once per press.

### Colemak-DH and WM Layers

Colemak-DH toggles with a combo on the two inner thumbs, active on BASE and COLEMAK. It redefines only the letters and HRM letters; everything else is `&trans` and falls through to BASE.

WM is held on the left outermost thumb (`&mo WM`) and sends through `WMK()`, `WMSK()` and `WMCK()`. It uses QWERTY positions, so it works under either base layout.

## Build Tools

Every board repo is driven by `zmk`, run from inside it. There is no Makefile. Every derived path comes from the single `config/*.keymap`.

`zmk draw` renders and does not parse. `keymap parse` cannot read these keymaps —
the conditional-layers node holds layer defines rather than integers — so each
`<stem>_keymap.yaml` is hand-written and a change to a keymap does not reach the
drawing. `zmk check` is the only thing that catches it: it compares every cell by
class, and checks that each `&mo`, layer-tap and `&magic` is labelled with the
layer it opens and that the layer marks one of the keys reaching it.

## Cross-Repo Workflow

When changing shared behaviors:

1. Edit `dts/shared_behaviors.dtsi`
2. Test in ONE keyboard repo first: `cd ~/code/zmk/corne42 && zmk build`
3. Then rebuild others: `cd ~/code/zmk/glove80 && zmk build`, etc.

Local edits take effect immediately — `zmk` bind-mounts this directory into the Docker container via `ZMK_EXTRA_MODULES`. No push/pull cycle needed locally.

**Push this repo before the board repo.** CI has no bind-mount and resolves the module through each board's `config/west.yml`, which declares it from GitHub at `main`. Pushing the board first makes its run compile against a `main` that lacks the change. This repo has no workflow of its own, so nothing validates it until a board pushes.

## Build Pitfalls

**`ZEPHYR_EXTRA_MODULES` vs `ZMK_EXTRA_MODULES`** (⚠️ CRITICAL): Never pass `-DZEPHYR_EXTRA_MODULES` from the command line — ZMK uses this variable internally to register its own modules (board definitions like `nice_nano`). A CLI `-D` flag overrides `set()` in CMakeLists, clobbering ZMK's module list and causing "Invalid BOARD" / "No board named 'nice_nano' found" errors. Always use `-DZMK_EXTRA_MODULES` instead — ZMK prepends this to its own list.

**CMake cache poisoning**: A bad `-D` flag persists in `CMakeCache.txt` even after fixing the script. After changing any CMake flags, the next build MUST be `zmk build --pristine` to clear the cache. `--pristine` only wipes the build directory — it does NOT re-download the west workspace, so it's fast. `zmk clean` destroys the board's entire west workspace (~5-10 min re-download) and is almost never the right fix for board errors.

## Guardrails

These apply in every board repo. Each board's own CLAUDE.md adds only what differs there.

- **Rebuild firmware after every keymap change** — run `zmk sync` (align + draw + build) before committing, then `zmk check`. Source changes without a build are useless; the UF2 file is what gets flashed. `check` is what catches the drawing falling behind, which `sync` alone does not.
- Changes to `shared_behaviors.dtsi` affect ALL keyboards — test carefully
- Each keyboard defines its own `KEYS_L`, `KEYS_R`, `THUMBS_L`, `THUMBS_R` in its keymap (position numbers differ per keyboard)
- **The gate decides what may be pressed FIRST, not just what resolves a hold.** A key outside `hold-trigger-key-positions` resolves the hold-tap as a *tap*. So a layer with modifiers on one hand and targets on the other cannot be entered by pressing the modifier first — it types the tap. That is what `ltltb`/`ltrtb` exist for.
- **A layer's index comes from the order of its block in `keymap`, not from its define.** Count the blocks in that board's keymap rather than assuming. Append a new layer last or the define points somewhere else, and nothing errors — the layer just does the wrong thing.
- The `hold-trigger-key-positions` in HRM behaviors reference the position macros, which must be defined before `#include "shared_behaviors.dtsi"`
- **On boards with runtime OS switching (corne42, piantor)**: combos must include `OS_MAC_LAYER` in their `layers` property or they won't fire in macOS mode; Shift uses `hmls`/`hmrs` (faster timing) instead of `hml`/`hmr`. Glove80 uses compile-time OS switching and has no `OS_MAC` layer, so neither applies there.
