# Adv360-Pro-ZMK — Nick's fork

ZMK firmware for a Kinesis Advantage 360 Pro. Fork of `KinesisCorporation/Adv360-Pro-ZMK`
(upstream default branch is `V3.0`, not `main`).

## Branches

| Branch | Purpose | Rule |
|---|---|---|
| `stock` | Pristine Kinesis, tracks `upstream/V3.0` | **Never commit here.** Only `git merge --ff-only upstream/V3.0`. |
| `V3.0` | Fork default / shared baseline. Bug fixes and docs every variant wants. | Branch everything personal off this. |
| `primary-mode` | **The daily driver.** Symbol layer + Game layer. | Flash from here. |
| `experiment/<name>` | Try-outs | Branch off `V3.0` or `primary-mode`. |

Two properties worth knowing:

- `.github/workflows/build.yml` triggers on `[push, pull_request, workflow_dispatch]` with
  **no branch filter** — every pushed branch produces its own `firmware-clique` artifact.
  That's the cheapest way to compile-check and to A/B two variants.
- `&macro_ver` (Mod layer, **position 50** — the `V` keycap) types
  `<date>-<branch[0:4]>-<commit>-clique`, so the keyboard can report which build is flashed.
  `bin/get_version.sh` **silently drops any character that isn't alphanumeric or `.`**, so keep
  the first four characters of a branch name distinct and alphanumeric.

Branch switching is clean: `firmware/` is gitignored and `config/version.dtsi` is generated
during `make` then `git checkout`-reverted, so neither leaves a stray diff.

## Syncing with upstream

`upstream` is `KinesisCorporation/Adv360-Pro-ZMK`, `origin` is `NickSt/Adv360-Pro-ZMK`. Updates
flow `stock` → `V3.0` → `primary-mode`:

```sh
git fetch upstream
git checkout stock        && git merge --ff-only upstream/V3.0
git checkout V3.0         && git merge stock
git checkout primary-mode && git merge V3.0
```

Nothing is silently overwritten — merges conflict, they don't clobber. But the conflict surface
is lopsided, and worth knowing before starting:

| File | Our divergence | Upstream churn | Expect |
|---|---|---|---|
| `config/adv360.keymap` | 72+/71− on an 89-line file | 4 of last 20 commits | conflicts every sync |
| `config/info.json` | 81+/79− | 2 of last 20 | conflicts every sync |
| `CLAUDE.md`, `.gitattributes` | new files | never existed upstream | never conflict |

Our keymap is effectively a full rewrite of upstream's, so resolve it by **keeping ours** and
hand-porting anything genuinely new (added includes, new behavior definitions, board changes) —
not by accepting theirs. `config/info.json` only feeds the web GUI and never reaches the
firmware, so `--ours` is always safe there.

Prefer putting shared fixes and docs on `V3.0` and merging forward, rather than committing them
to `primary-mode` directly — otherwise a build run from `V3.0` or `stock` re-breaks.

## Editing the keymap

`config/adv360.keymap` is **the only file to edit** for keymap changes.

- **Do not use the Nick Coutsos keymap-editor web GUI.** Its bot rewrites the whole file; commit
  `9b7d63f` was authored by it and silently dropped `#include <dt-bindings/zmk/pointing.h>` while
  `config/macros.dtsi` still referenced `MB1`. It also cannot model `#define`s, custom hold-taps,
  or combos and will drop them.
- **The file is CRLF.** Any script that rewrites it must open with `newline=''` and emit `\r\n`,
  or the whole file shows as changed.
- `config/adv360_left.keymap` / `adv360_right.keymap` are one-line `#include "adv360.keymap"`
  stubs. Never touch them.
- `config/keymap.json` is **dead** — legacy Kinesis-editor format, nothing in the build path reads
  it, stale since `7287c4d`. Leave it alone.
- `config/info.json` is the visual layout for the web GUI only. Not used by the firmware build.
- `config/boards/arm/adv360/macros.dtsi` is a **stale duplicate**. The live one is
  `config/macros.dtsi` (resolved via `-DZMK_CONFIG`).

### Key positions

Bindings are one flat 76-entry array. Row breaks in the file are cosmetic, but the row widths are
fixed: **14, 14, 18, 14, 16**. Canonical numbering is in `assets/key-positions.md`.

```
 0 =      1 N1     2 N2    3 N3    4 N4     5 N5   6 tog  ||  7 mo    8 N6   9 N7  10 N8   11 N9  12 N0  13 -
14 TAB   15 Q     16 W    17 E    18 R     19 T   20 ---  || 21 ---  22 Y   23 U  24 I    25 O   26 P   27 \
28 ESC   29 A     30 S    31 D    32 F     33 G   34 ---  || 39 ---  40 H   41 J  42 K    43 L   44 ;   45 '
          L-thumb 35 36                                           R-thumb 37 38
46 LSHFT 47 Z     48 X    49 C    50 V     51 B            || 54 N   55 M   56 ,  57 .    58 /   59 RSHFT
          L-thumb 52                                              R-thumb 53
60 mo    61 GRAVE 62 CAPS 63 LEFT 64 RIGHT                 || 71 UP  72 DOWN 73 [  74 ]   75 mo
          L-thumb 65 66 67                                        R-thumb 68 69 70
```

Rows 4 and 5 are **physically narrower** — no inner column (no 6th/7th slot). That constrains any
layout that shifts the alpha block rightward; overflow has to go to the thumb clusters.

When rewriting layers wholesale, lay tokens on an 18-slot grid:
`row1/2` pos 0-6→slot 0-6 and 7-13→slot 11-17; `row3` 28-34→0-6, 35-38→7-10, 39-45→11-17;
`row4` 46-51→0-5, 52→8, 53→9, 54-59→12-17; `row5` 60-64→0-4, 65-67→6-8, 68-70→9-11, 71-75→13-17.

### Layers

`#define`d at the top of the keymap: `BASE 0`, `KPAD 1`, `SYM 2`, `MOD 3`, `GAME 4`.
Add new layers **at the end** so existing indices keep their meaning.

On `primary-mode`:

- **SYM** is the old Fn layer with its dead `&trans` alpha block filled in. F-keys stay on row 1.
  Brackets on the right hand with each pair adjacent, opener left of closer: home row `( ) { }`,
  row above `[ ] < >`. Left hand carries operators. Row 4 right binds the auto-pair macros that
  already existed in `config/macros.dtsi` (`macro_parens`, `macro_brackets`, `macro_braces`,
  `macro_dquotes`) but were never bound to anything.
  Row 5 is left `&trans` so Backspace/Del/Enter/Space stay live while held.
  Reachable four ways: `&mo SYM` at 60, 75 (outer corners) and 34, 39 (inner keys, opposite-hand).
- **GAME** shifts the alpha block down one row and right one column so `W` sits on position 31
  (the `D` keycap), which frees row 1 for F1–F12 *and* row 2 for the full digit row. `G`/`B`/`V`
  overflow to the small left thumbs (35/36/52); Space/Shift/Ctrl take the big ones (65/66/67).
  `M` and `I` sit in the freed far-left column. Enter with `MOD`+`G` (mod layer pos 33), exit with
  `&to BASE` at pos 6. **No hold-taps, mod-taps or combos** on this layer — keys must fire instantly.

There are no combos anywhere in the repo. If adding any, keep them off the GAME positions
(28–34, 46–51, 60–64 and the left thumb cluster).

`primary-mode` also carries four `status = "reserved"` placeholder layers (`extra1`–`extra4`,
indices 5–8) after GAME. They are the spare slots ZMK Studio / Clique needs to add layers at
runtime — normal builds ignore them entirely, Studio-enabled builds expose them. Upstream shipped
four of these; `9b7d63f` removed them, which silently capped Clique at the layers already defined.

**Reserved layers must stay last**, because layer index is devicetree order. That's why they live
on `primary-mode` rather than `V3.0`: a baseline branch can't hold them, since any downstream
branch adding a real layer would have to insert it *before* the reserved block and would collide
on every merge. A new real layer goes after GAME but **before** `extra1`.

## Changing bindings without reflashing

The left half is built with ZMK Studio enabled, so most binding changes need no rebuild:
plug the **left** half in over USB, open [Clique](https://clique.kinesis-ergo.com/) or
[zmk.studio](https://zmk.studio), and unlock with **`Mod` + `Esc`** (`&studio_unlock`, mod layer
pos 28). Clique is Kinesis's own UI built on ZMK Studio's protocol — the two are interchangeable
clients, not different mechanisms. The Coutsos keymap-editor is a different thing again: it edits
*repo files*, never the keyboard, and still needs a flash.

Studio can reassign keys on existing layers, rename layers, and enable reserved ones, using any
behavior compiled into the firmware — including every macro in `config/macros.dtsi`, bound or not,
since Studio builds auto-enable `ZMK_BEHAVIORS_KEEP_ALL`. It **cannot** define new behaviors,
combos, or conditional layers, or change Kconfig. Those still need a rebuild and flash.

Two traps:

- **Studio settings override the firmware keymap, and keep overriding it.** Once anything is saved
  in Studio, later `.keymap` changes you flash will not take effect until you use **Restore Stock
  Settings** in the Studio/Clique UI. This interacts badly with per-branch CI builds: flash an
  `experiment/*` branch with Studio settings stored and you'll see the old keymap and think the
  build failed. Restore Stock Settings before evaluating a freshly flashed branch.
- **Studio has no export/import and no profiles** ([zmk-studio#124](https://github.com/zmkfirmware/zmk-studio/issues/124)).
  "Restore Stock Settings" is a wipe, not a restore. Nothing tweaked only in Studio exists anywhere
  but on that keyboard. This repo is the profile system — a branch per variant, built by CI. Treat
  Studio as a scratchpad and port anything worth keeping back into the keymap.

## Build

ZMK source is pinned in `config/west.yml` to a **fork**: `refil/zmk` @ `adv360-z3.5-2`.

- `make` — both halves. `make left` — left only, faster. Output lands in `firmware/`.
- **Or just push the branch** and download the `firmware-clique` artifact from Actions. Needs
  nothing installed locally and compiles both halves.

Local builds work on Windows with **podman**, run from **Git Bash** — no Docker Desktop, and no
WSL2 shell (`podman machine` uses WSL2 underneath, but you never interact with it). One-time
setup is `scoop install make` plus `podman machine start`.

The first `make` builds the container image — pulls `zmkfirmware/zmk-build-arm:stable`, then
`west init/update/zephyr-export` — and takes several minutes. The Zephyr tree is baked into the
image, so later runs are just the compile, about a minute.

Two Windows fixes are already in the repo. Don't undo them:

- `export MSYS_NO_PATHCONV=1` at the top of the `Makefile`. Without it Git Bash rewrites the
  container-side half of `-v host:/app/config` into a Windows path, and podman fails with
  `invalid option type "\Program Files\Git\app\config;ro"`.
- `.gitattributes` pins `*.sh` to LF. Under `core.autocrlf=true` the scripts check out CRLF and
  the container dies immediately with `/usr/bin/env: 'bash\r': No such file or directory`.
  CI never hit this — GitHub's checkout doesn't convert line endings.

The `:z` SELinux mount flags the `Makefile` adds on non-Darwin hosts are harmless under podman
on WSL, and `$(PWD)` as a POSIX path (`/d/source/...`) is accepted as a mount source.

Sizes, as a reference point when adding features: left **32.8%** of 792 KB flash and 29.1% RAM,
right **22.4%** and 12.7%.

Validating a keymap change without a container: every `&kp` keycode must resolve against
`app/include/dt-bindings/zmk/keys.h` in the pinned fork, and every `&macro_*` against
`config/macros.dtsi`. Checking all 76 positions per layer catches structural mistakes. This is a
good pre-flight but is **not** a substitute for a compile.

## Flash

Left half first, then right — order matters. Full sequence in README §"Flashing firmware".

1. Hold `Mod` (pos 7) + **pos 20** → left half mounts as a USB drive → copy `*-left-clique.uf2`.
2. Power off both halves (unplug, switches off), then turn the left one back on.
3. Hold `Mod` + **pos 21** → copy `*-right-clique.uf2`, then power-cycle the right half.

Positions 20 and 21 are the two inner-column keys on the QWERTY row (just right of `T`, just left
of `Y`) — they carry `&bootloader` on the Mod layer. Naming is inconsistent across sources: the
README calls them `macro1`/`macro3`, `info.json` labels them `mod3`/`mod4`. Go by position.

Physical reset buttons are the fallback. `settings-reset.uf2` at the repo root clears settings if a
layer gets stuck. To roll back entirely, build and flash from `stock`.

## Gotchas

- `CONFIG_ZMK_POINTING=y` is set **only** in `adv360_left_defconfig`, not the right. Anything using
  `&mkp` needs care on the right half.
- ZMK properties are hyphenated. `quick_tap_ms` with underscores parses fine and is **silently
  ignored** — was a real bug here.
- The left half is the central/BLE side and is the one built with ZMK Studio
  (`-S studio-rpc-usb-uart -DCONFIG_ZMK_STUDIO=y`).
