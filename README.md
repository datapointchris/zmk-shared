# zmk-shared

The ZMK behaviors that several keyboard configs share, packaged as one Zephyr module. A keymap
includes `shared_behaviors.dtsi` and binds home-row mods, layer-taps, Bluetooth tap-dances, tmux
prefix macros and window-manager chords that follow the OS.

The module holds nothing that depends on a board. Key positions, layers, combos and build targets
belong to each keyboard's config repo. The module reads a board's key positions through four macros
the keymap defines before including it. [zmk](https://github.com/datapointchris/zmk) builds, draws,
checks and flashes a config repo, and mounts this module into every local build.

## Install

A config repo reaches the module two ways. A local build mounts a checkout of it, and CI fetches it
through west.

```bash
git clone https://github.com/datapointchris/zmk-shared shared   # beside the config repos, named shared
```

zmk looks for the module at `../shared` from the config repo it runs in. `ZMK_SHARED_DIR` names a
checkout anywhere else. A bare `west build` takes it as `-DZMK_EXTRA_MODULES=<path>`, which ZMK
prepends to its own module list. Passing `-DZEPHYR_EXTRA_MODULES` instead overwrites that list,
which drops ZMK's own board definitions, and the build cannot find its board.

CI resolves the module through each config repo's `config/west.yml`, beside the entry for ZMK:

```yaml
manifest:
  remotes:
    - name: datapointchris
      url-base: https://github.com/datapointchris
  projects:
    - name: zmk-shared
      remote: datapointchris
      revision: main
```

`zephyr/module.yml` declares the checkout a devicetree root. Zephyr puts its `dts/` directory on the
include path, so a keymap includes `shared_behaviors.dtsi` by its bare name.

## Starting from scratch

```bash
# from your package manager: git, docker (runnable without sudo), python3, and mikefarah's yq
curl -fsSL https://raw.githubusercontent.com/datapointchris/zmk/main/install.sh | bash  # zmk, into ~/.local/bin
mkdir -p ~/zmk && cd ~/zmk                                              # one directory for the module and its config repos
git clone https://github.com/datapointchris/zmk-shared shared            # zmk finds it as ../shared
git clone https://github.com/datapointchris/zmk-config-corne42 corne42   # any config repo that includes the module
cd corne42                                                              # zmk acts on the config repo it runs in
zmk build                                                               # compiles the keymap against ../shared; one .uf2 per target
```

`zmk build` refuses when `../shared` holds no `zephyr/module.yml`, and prints the path it looked in.
The first build fetches several gigabytes of ZMK and Zephyr into `~/.cache/zmk/corne42/`. Later
builds are incremental, and each `.uf2` lands beside `build.yaml`.

From here an edit to `dts/shared_behaviors.dtsi` reaches the next `zmk build` with no commit.

## A keymap names its key positions, then includes the module

```c
#define KEYS_L 0 1 2 3 4 5 12 13 14 15 16 17 24 25 26 27 28 29
#define KEYS_R 6 7 8 9 10 11 18 19 20 21 22 23 30 31 32 33 34 35
#define THUMBS_L 36 37 38
#define THUMBS_R 39 40 41

#include <behaviors.dtsi>
#include <dt-bindings/zmk/keys.h>
#include <dt-bindings/zmk/bt.h>
#include <dt-bindings/zmk/outputs.h>
#include "shared_behaviors.dtsi"
```

Every hold-tap in the module names its trigger positions through those four macros, so the keymap
defines them before the include. They are the only thing the module reads from a board. The
positions above are a 42-key Corne's. The headers go first because the module binds ZMK's own
behaviors, such as `&bt`, `&out` and `&caps_word`, and spells keys, Bluetooth commands and outputs
with the names those headers define.

The boards bind home-row mods GASC from the pinky inward: GUI, Alt, Shift, Ctrl, mirrored on the
right. They pass real keycodes, as in `&hml LGUI A`.

The module defines no layer. A layer-tap takes its layer as an argument, as in `&ltlt NPAD BSPC`,
and each keymap numbers its own layers at its top. A layer's number is its block's position in that
keymap, so one shared list would force every board into one order.

## Hold-taps — the next key's position decides hold or tap

| Behavior | What it is | Hold | A hold resolves from |
| --- | --- | --- | --- |
| `hml`, `hmr` | home-row mod, left and right | `&kp` | the other hand's keys, both thumb clusters |
| `hmls`, `hmrs` | the Shift home-row mod, on faster timing | `&kp` | the other hand's keys, both thumb clusters |
| `hmlt`, `hmrt` | thumb mod | `&kp` | the other hand's keys and thumbs |
| `ltl`, `ltr` | layer-tap on a letter key | `&mo` | the other hand's keys, both thumb clusters |
| `ltlt`, `ltrt` | layer-tap on a thumb | `&mo` | the other hand's keys and thumbs |
| `ltltb`, `ltrtb` | thumb layer-tap for a layer on both hands | `&mo` | every key outside its own thumb cluster |

Every tap is `&kp`. Every hold-tap is balanced and triggers on release. The tapping term is 280 ms,
with 175 ms quick-tap and 150 ms of prior idle. The Shift pair runs a 200 ms tapping term and 100 ms
of prior idle. `dts/shared_behaviors.dtsi` is where those values are read and changed.

The position list is the gate. A key pressed outside it resolves the hold-tap as a tap, so a
same-hand roll types two letters rather than a modifier and a letter.

The gate also decides what may be pressed first. Take a layer held on the left thumb with `ltlt`.
A left-hand key on that layer, pressed before anything on the right, resolves the thumb as its tap.
`ltltb` and `ltrtb` exist for a layer like that. They gate on everything but their own thumb
cluster. The cost is that their tap can roll into the layer from either hand, so they belong on
thumbs whose tap is not followed at once by a letter.

## Tap-dances and macros — one key for an action that takes several

| Behavior | Tap | Double-tap |
| --- | --- | --- |
| `bt_0` … `bt_3` | output to Bluetooth, then select that profile | disconnect that profile |
| `caps_shift` | Right Shift | Caps Word |

The Bluetooth tap-dances wait 200 ms for the second tap, and `caps_shift` waits 280 ms. Their tap
halves stand alone as the macros `bt_select_0` … `bt_select_3`. `ctrlaltdel` sends
Ctrl+Alt+Delete.

`HYPER` (Ctrl+Shift+GUI+Alt), `MEH` (Ctrl+Shift+Alt) and `SUPER` (Ctrl+GUI+Alt) are modifier
chords, bound as `&kp HYPER`.

The `tmux_*` macros send tmux's prefix, Ctrl+Space, then one key. They cover the actions tmux
exposes only through its prefix table. An action reachable as a plain chord is bound as that chord
in the keymap instead, because a held chord auto-repeats and a macro fires once per press.

| Macro | Key after the prefix |
| --- | --- |
| `tmux_split_v` | `\|` |
| `tmux_split_h` | `-` |
| `tmux_zoom` | `z` |
| `tmux_newwin` | `c` |
| `tmux_rename` | `,` |
| `tmux_win_left` | `{` |
| `tmux_win_right` | `}` |
| `tmux_finder` | `w` |
| `tmux_ses_left` | `<` |
| `tmux_ses_right` | `>` |
| `tmux_picker` | `s` |
| `tmux_copy` | `[` |

## `WMK()`, `WMSK()`, `WMCK()` — the window-manager chord follows the OS

| Macro | Linux, the default | macOS, built with `OS_MACOS` | For |
| --- | --- | --- | --- |
| `WMK(key)` | Super+key | Alt+key | focus and workspaces |
| `WMSK(key)` | Super+Shift+key | Alt+Shift+key | moving a window |
| `WMCK(key)` | Super+Ctrl+key | Alt+Ctrl+Shift+key | joining and grouping windows |

The split follows the window managers: i3 and sway bind Super, and AeroSpace binds Alt. A keymap
binds the macros as `&kp WMK(H)`. They are not named `WM` and `WMS` because the boards use `WM` as a
layer define.

A board picks the OS in one of two ways:

- **At build time.** A target whose `cmake-args` carry `-DDTS_EXTRA_CPPFLAGS=-DOS_MACOS` compiles
  the macOS chords. Changing OS means flashing the other firmware. The Glove80 config builds each
  half both ways.
- **At runtime.** The firmware is built without the flag, so the macros send Super. The keymap adds
  a flag layer, `OS_MAC_LAYER`, with every key `&trans`, toggled from its SYSTEM layer. It adds
  `WM_MAC_LAYER`, which rebinds the window-manager keys with `LA()`. A conditional layer raises
  `WM_MAC_LAYER` while `WM` and `OS_MAC_LAYER` are both active. The Corne and Piantor configs work
  this way.

Runtime switching puts two constraints on the keymap. `WM_MAC_LAYER` takes a higher number than
`WM`, because ZMK resolves a key from the highest active layer down. Every combo lists
`OS_MAC_LAYER` in its `layers`, because ZMK matches a combo against the highest active layer and
the toggled flag layer sits above `BASE`. A combo without it does not fire in macOS mode.

## Each config repo compiles the module against the ZMK it pins

- [zmk-config-corne42](https://github.com/datapointchris/zmk-config-corne42) builds against
  upstream ZMK at `main`, and switches OS at runtime.
- [zmk-config-piantor-pro-bt](https://github.com/datapointchris/zmk-config-piantor-pro-bt) pins an
  upstream ZMK release rather than `main`, carries its own board definition, and switches OS at
  runtime.
- [zmk-config-glove80](https://github.com/datapointchris/zmk-config-glove80) builds against
  MoErgo's fork of ZMK, and switches OS at build time.

A config repo owns everything positional: its key-position macros, its layers and their order, its
combos, its OS-switching layers, its build targets and its drawing. This repo owns the behaviors
those keymaps bind.

Each build compiles `shared_behaviors.dtsi` against the ZMK its own `config/west.yml` pins. A
property added to a behavior here has to exist in every one of those ZMKs, or Zephyr rejects it as
undeclared in that build. From the directory holding the clones:

```bash
rg -l zmk-shared */config/west.yml       # every config repo that includes the module
rg -A2 'name: zmk$' */config/west.yml    # the ZMK each one pins
rg -lw hmls */config/*.keymap            # the keymaps binding a behavior, before changing it
```

## Changing a behavior — a local build compiles the working tree, CI only what is pushed

```bash
cd ../corne42   # one board first
zmk sync        # align, draw and build, with ../shared mounted as it is on disk
zmk check       # whether the drawing still matches the keymap; exit 1 on any drift
zmk flash       # pick halves with fzf, then write each
```

zmk mounts `../shared` read-only into the build container and passes it to ZMK as
`ZMK_EXTRA_MODULES`. A local build compiles the working tree, so a change reaches it with no commit.
Every other config repo that binds what changed takes the same `zmk sync` and `zmk check`.

CI has no mount. It fetches the module through `config/west.yml` at `main`. Push this repo before a
config repo that depends on the change, or that repo's CI run compiles against a `main` without it.

This repo's own CI runs lint hooks and compiles nothing. A behavior change is first compiled by a
board's build, local or CI.

`zmk check` knows this module's layer-taps by name. It reads `ltl`, `ltr`, `ltlt`, `ltrt`, `ltltb`
and `ltrtb` as keys holding a layer open, and checks each one's drawn hold legend against the layer
it opens. A layer-tap added here under another name escapes that check until zmk learns the name.

`zmk draw` reads `keymap_drawer.config.yaml` from the config repo it runs in. Each config repo
carries its own, so no drawing reads the copy in this repo.
