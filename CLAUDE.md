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

## Build

ZMK source is pinned in `config/west.yml` to a **fork**: `refil/zmk` @ `adv360-z3.5-2`.

- `make` — both halves via Docker/podman. `make left` — left only, faster. Needs Docker Desktop
  running; on Windows run from WSL2. Output lands in `firmware/`.
- **Or just push the branch** and download the `firmware-clique` artifact from Actions. Usually
  faster than getting a local container going, and it compiles both halves.

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
