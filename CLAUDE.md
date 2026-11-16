# zmk-shared AI Context

This module is consumed by the board repos cloned beside it: `../corne42`, `../glove80` and `../piantor`. Check how they use a behavior before changing it.

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

### WM Macros, OS Switching and tmux Macros

README.md owns these. Its `WMK()`, `WMSK()`, `WMCK()` section gives what each macro sends, the
two ways a board picks the OS, and the constraints runtime switching puts on a keymap. Its
tap-dances and macros section lists the `tmux_*` macros.

The macOS layers are named `OS_MAC_LAYER` and `WM_MAC_LAYER` because `WM` is already a layer
define, so they cannot take the bare names. Writing `&tog OS_MAC` does not compile.

### Colemak-DH and WM Layers

Colemak-DH toggles with a combo on the two inner thumbs, active on BASE and COLEMAK. It redefines only the letters and HRM letters; everything else is `&trans` and falls through to BASE.

WM is held on the left outermost thumb (`&mo WM`) and sends through `WMK()`, `WMSK()` and `WMCK()`. It uses QWERTY positions, so it works under either base layout.

## Build Tools

Every board repo is driven by `zmk`, run from inside it. There is no Makefile. Every derived path comes from the single `config/*.keymap`.

`zmk draw` renders and does not parse. Each `<stem>_keymap.yaml` is hand-written.
`keymap parse` reads these keymaps and exits 0, but it writes raw bindings such as
`&tmux_zoom` where the drawing carries a legend such as `ZOOM`. So a change to a
keymap does not reach the drawing. `zmk check` is the only thing that catches it:
it compares every cell by class, and checks that each `&mo`, layer-tap and
`&magic` is labeled with the layer it opens and that the layer marks one of the
keys reaching it.

## Cross-Repo Workflow

When changing shared behaviors:

1. Edit `dts/shared_behaviors.dtsi`
2. Test in ONE keyboard repo first: `cd ../corne42 && zmk build`
3. Then rebuild others: `cd ../glove80 && zmk build`, etc.

**Push this repo before the board repo.** README.md's section on changing a behavior says why: a
local build mounts this working tree, while CI fetches the module at `main` and this repo's own
workflow compiles nothing.

## Build Pitfalls

**"Invalid BOARD" or "No board named 'nice_nano' found"** means the build was passed
`-DZEPHYR_EXTRA_MODULES`. README.md's Install section says why that drops ZMK's board definitions
and what to pass instead.

**CMake cache poisoning**: A bad `-D` flag persists in `CMakeCache.txt` even after fixing the script. After changing any CMake flags, the next build MUST be `zmk build --pristine` to clear the cache. `--pristine` only wipes the build directory — it does NOT re-download the west workspace, so it's fast. `zmk clean` destroys the board's entire west workspace (~5-10 min re-download) and is almost never the right fix for board errors.

## Guardrails

These apply in every board repo. Each board's own CLAUDE.md adds only what differs there.

- **Rebuild firmware after every keymap change** — run `zmk sync` (align + draw + build) before committing, then `zmk check`. Source changes without a build are useless; the UF2 file is what gets flashed. `check` is what catches the drawing falling behind, which `sync` alone does not.
- Changes to `shared_behaviors.dtsi` affect ALL keyboards — test carefully
- Each keyboard defines its own `KEYS_L`, `KEYS_R`, `THUMBS_L`, `THUMBS_R` in its keymap (position numbers differ per keyboard)
- **The gate decides what may be pressed first, not just what resolves a hold.** The comment above `ltltb` in `dts/shared_behaviors.dtsi` explains it, and why `ltltb`/`ltrtb` exist.
- **A layer's index comes from the order of its block in `keymap`, not from its define.** Count the blocks in that board's keymap rather than assuming. Append a new layer last or the define points somewhere else, and nothing errors — the layer just does the wrong thing.
- The `hold-trigger-key-positions` in HRM behaviors reference the position macros, which must be defined before `#include "shared_behaviors.dtsi"`
- **On boards with runtime OS switching**, every combo lists `OS_MAC_LAYER` in its `layers`. README.md's `WMK()` section gives the reason and the layer-numbering constraint beside it.
- Shift uses `hmls`/`hmrs` (faster timing) instead of `hml`/`hmr` on every board.
