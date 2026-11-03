# configs inventory

Complete inventory of `xfetch-cli/configs` at HEAD
`31ffba72ad3c5ce3f1c4380d04fd319bcf325f63` (2026-10-19).

Last verified: 2026-11-03 against `configs@31ffba72ad3c5ce3f1c4380d04fd319bcf325f63` (`main`).

## Tree and counts

```text
configs/
|-- xfetch/
|   |-- defaults/
|   |   `-- config.jsonc                         1 file
|   `-- presets/
|       |-- layouts/                             6 files
|       |-- showcase/                            23 files
|       |-- cyberpunk/                           10 configs + config.jsonc
|       |   |-- banners/                         10 .txt banners
|       |   |-- configs/                         10 .jsonc
|       |   |-- config.jsonc
|       |   |-- README.md
|       |   `-- routes.json
|       `-- massive-roulette/                    149 configs + config.jsonc
|           |-- configs/                         149 .jsonc
|           |-- logos/                           21 .txt logos
|           |-- config.jsonc
|           |-- README.md
|           `-- routes.json
|-- plugins/
|   |-- animate-logo/                            1 preset
|   |-- full-stack/                              1 preset
|   |-- new-modules/                             1 preset
|   |-- roulette-animation-example/              1 config tree + 6 figures + 12 logos
|   `-- wasm-guests/                             2 presets
|-- scripts/                                     ci.sh, ci.ps1, validate.py
|-- CHANGELOG.md
|-- CONTRIBUTING.md
|-- README.md
|-- SUPPORT.md
|-- CODE_OF_CONDUCT.md
`-- LICENSE (MIT)
```

JSON/JSONC totals:

| Area | Files |
| --- | --- |
| `xfetch/defaults/` | 1 |
| `xfetch/presets/layouts/` | 6 |
| `xfetch/presets/showcase/` | 23 |
| `xfetch/presets/cyberpunk/` (configs + config.jsonc + routes.json) | 12 |
| `xfetch/presets/massive-roulette/` (configs + config.jsonc + routes.json) | 151 |
| `plugins/` (presets + configs + routes) | 13 |
| `scripts/` | 0 (Python only) |
| Total validated | 206 |

Hex/other files: 10 cyberpunk banners `.txt`, 21 massive-roulette logos `.txt`,
and the roulette-animation example logo/static files.

## xfetch/defaults/

Single file: `xfetch/defaults/config.jsonc`.

| Property | Value |
| --- | --- |
| Layout | `section` |
| Module groups | `Hardware` (hostname, cpu, gpu, memory, swap, disk, battery), `Software` (os, kernel, packages, shell, wm, terminal, local_ip), `Session` (user, uptime, datetime), plus the ungrouped `palette` item |
| `show_colors` | `true` |
| `palette_style` | `circles` |
| `icons` | Nerd Font glyphs per module; group titles also have icons (`hardware`, `software`, `session`) |
| `colors` | Green for hardware, Yellow for software, Magenta for session, White for palette |
| Plugins | None |

This is the file to copy for a user who wants a hand-written starting config,
but the supported path since v0.6.0 is `xfetch --gen-config`, which fetches the
distro logo from `xfetch-cli/logos` and produces an equivalent config.

## xfetch/presets/layouts/ (6 files)

Focused layout presets. Naming convention: `layout_<name>.jsonc` or
`layout_<name>_full.jsonc`.

| File | Layout | Character |
| --- | --- | --- |
| `layout_box_full.jsonc` | `box` | All modules inside a rounded box with separators. |
| `layout_dots_full.jsonc` | `dots` | Classic layout with `...` separators every 3 items. |
| `layout_pacman_full.jsonc` | `pacman` | Pac-Man header icons (`ᗧ`, `●` x4) and `GAME OVER` footer, full module set. |
| `layout_section.jsonc` | `section` | Hardware / Software / Uptime-Age-DT groups plus palette; text labels as icons. Note: no `_full` suffix. |
| `layout_side_block.jsonc` | `side-block` | Two-column key/value boxes. |
| `layout_tree.jsonc` | `tree` | Hierarchical nested groups. |

## xfetch/presets/showcase/ (23 files)

Single-purpose showcase configs. Names describe intent; most are self-contained
and use core modules only. Files:

| File | Demonstrates |
| --- | --- |
| `arch_compact_cyan.jsonc` | Compact Arch setup, all-cyan palette. |
| `arch_full_blue.jsonc` | Full Arch system, all-blue. |
| `green_chevrons_core.jsonc` | `>>` icons, all-green, core modules. |
| `minimal_plus_monochrome.jsonc` | `+` icons, grey monochrome. |
| `minimal_red_compact.jsonc` | Text-label icons (`OS:`, `K:`), all red. |
| `monochrome_no_icons.jsonc` | Empty icons, all white. |
| `neon_hardware_compact.jsonc` | Nerd Font icons, cyan/magenta hardware set. |
| `rainbow_letters.jsonc` | Rainbow per-module letters. |
| `x_logo_full_system.jsonc` | `logo_path: logos/x_logo.txt`, full system. |
| `x_logo_red_hardware.jsonc` | `logo_path: logos/x_logo.txt`, red hardware subset. |
| `pacman_full_white.jsonc` | Pac-Man, full set, white palette. |
| `pacman_system_compact.jsonc` | Pac-Man, `os`/`kernel`/`memory`. |
| `pacman_abc_shell.jsonc` | Header A/B/C, footer `ABC`, `os`/`shell`. |
| `pacman_minus_uptime.jsonc` | Header `-`, footer `MINUS`, `uptime`. |
| `pacman_numeric_hardware.jsonc` | Header 1/2/3, footer `NUM`, `cpu`/`memory`. |
| `pacman_plus_os.jsonc` | Header `+`, footer `PLUS`, `os`. |
| `pacman_question_packages.jsonc` | Header `?`, footer `QUEST`, `packages`. |
| `pacman_star_uptime.jsonc` | Header star, footer `STAR`, `os`/`uptime`. |
| `pacman_symbols_portable.jsonc` | Portable `!`/`@`/`#`, footer `SYM`, `disk`/`battery`. |
| `pacman_xox_kernel.jsonc` | Header x/o/x, footer `XOX`, `os`/`kernel`. |
| `pacman_palette_dots.jsonc` | Full set, `palette_style: dots`. |
| `pacman_palette_lines.jsonc` | Full set, `palette_style: lines`. |
| `pacman_palette_triangles.jsonc` | Full set, `palette_style: triangles`. |

Gotcha: the two `x_logo_*` files reference `logos/x_logo.txt` relative to the
working/config directory. That file is not shipped under
`xfetch/presets/showcase/`; the only `x` logo art in this repo is
`xfetch/presets/massive-roulette/logos/x.txt`. The user must provide
`~/.config/xfetch/logos/x_logo.txt` or change `logo_path`.

## xfetch/presets/cyberpunk/ (10 configs)

Standalone cyberpunk collection. Uses the `gauges` and `banner` plugins and the
`config-roulette` extension. No custom colors: it inherits xfetch defaults.

| # | File | Layout | Banner text (Japanese) |
| --- | --- | --- | --- |
| 01 | `configs/01-neon-tokyo.jsonc` | `section` | 永遠は朽ち、儚さが記憶を刻む |
| 02 | `configs/02-acid-terminal.jsonc` | `section-box` | コードの精確さと言葉の感性 |
| 03 | `configs/03-chrome-sunset.jsonc` | `compact` | 経験は文学の素材となる |
| 04 | `configs/04-ghost-protocol.jsonc` | `tree` | 状況の重ね合わせ |
| 05 | `configs/05-blood-machine.jsonc` | `default` | 永遠回帰 |
| 06 | `configs/06-violet-grid.jsonc` | `box` | 秘密の距離 |
| 07 | `configs/07-toxic-rain.jsonc` | `minimal` | 存在しない権利 |
| 08 | `configs/08-magenta-overdrive.jsonc` | `pacman` | 運命の前の儚さ |
| 09 | `configs/09-blue-shadow.jsonc` | `side-block` | 廃墟の間の超越 |
| 10 | `configs/10-amber-relic.jsonc` | `custom-x` | 芥子の舞 |

Additional files:

| File | Role |
| --- | --- |
| `config.jsonc` | Full fallback config plus `config_providers` for the roulette. |
| `routes.json` | 10 routes, `_name` + `~/.config/xfetch/fetchs/cyberpunk/configs/NN-*.jsonc`. |
| `banners/NN-*.txt` | Banner text loaded through `logo_animation.frames_path`. |
| `README.md` | Install, fonts, roulette, banner provenance. |

Requirements: xfetch 1.0.0+, `gauges` and `banner` plugin binaries, a Nerd Font,
a Japanese/CJK font, and `config-roulette`. Install target:
`~/.config/xfetch/fetchs/cyberpunk/`.

Strict-JSON caveat: the 10 configs are comment-free JSON despite the `.jsonc`
extension, because `config-roulette` parses routes with strict JSON.

## xfetch/presets/massive-roulette/ (149 configs)

Large rotation set for `config-roulette`. All colour palettes are inlined, so
themes are optional.

Composition (verified by file inspection):

| Segment | Files | Count | Notes |
| --- | --- | --- | --- |
| Standard configs | `001-*` to `100-*` | 100 | All layouts, 4-12 modules. |
| Custom-X configs | `101-*` to `130-*` | 30 | `layout: custom-x`, intro effects `decrypt`/`glitch`. |
| Gauges configs | `gauges-01-*` to `gauges-10-*` | 10 | Plus 40 of the numbered configs also use `gauges`, for 50 total. |
| Langs configs | `langs-01-*` to `langs-09-*` | 9 | `langs` plugin, GitHub language breakdown. |
| Total | | 149 | |

Other facts:

- `routes.json` has exactly 149 entries.
- 50 configs use `plugin:gauges` and the `banner` plugin with CJK glyphs
  (`存 换 盘 电` / `存 換 盤 電`).
- 30 use `layout: custom-x`.
- 9 use the `langs` plugin.
- Layouts used across the 149: `custom-x` 30, `compact` 10, `default`/`box`/
  `bottom`/`horizontal`/`section`/`section-box`/`side-block`/`tree` 9 each,
  `pacman`/`minimal` 8 each, `line`/`dots`/`bottom_line` 7 each.
- `logos/` contains 21 files to copy to `~/.config/xfetch/logos/`: `alma`,
  `alpine`, `antix`, `arch`, `archbang`, `arcolinux`, `artix`, `bedrock`,
  `blackarch`, `bodhi`, `cachyos`, `garuda`, `gentoo`, `mx-linux`, `nitrux`,
  `nixos`, `opensuse-leap`, `raspberrypi`, `rhel`, `serpent`, `x`.
- `config.jsonc` is a full fallback config with `config_providers` pointing at
  the 149 routes and `theme: "x"`.
- Many theme names in the configs are not present in the current
  `xfetch-cli/themes` registry (e.g. `basalt`, `bluemoon`, `syrup`, `prairie`).
  This is intentional/acceptable: the inline `colors` block is used when the
  theme cannot be resolved. Do not assume every `theme` value is installable.

Requirements: xfetch 1.0.0+, `config-roulette`, plugins `gauges`, `langs`,
`banner`, optional info plugins (`docker`, `github-stats`, `music-player`,
`temperature`, `timezone`, `user-info`, `display-resolution`,
`theme-detection`, `animate-logo`), effects `decrypt` and `glitch`, and a CJK
font.

## plugins/ examples (5 directories)

These require binaries from `xfetch-cli/plugins` (and in one case the
`config-roulette` extension from `xfetch-cli/extensions`). Plugin binaries are
normally installed as `xfetch-plugin-<name>` and referenced in configs by bare
`<name>` in `info_plugins` / `"plugin": "<name>"`.

| Directory | Config file | Required binaries / addons |
| --- | --- | --- |
| `animate-logo/` | `presets/xfetch_pacman_animate.jsonc` | `animate-logo`; logo file at the `ascii` path. |
| `full-stack/` | `presets/full_stack.jsonc` | `animate-logo`, `docker`, `github-stats`. |
| `new-modules/` | `presets/enhanced_modules.jsonc` | `music-player`, `weather`, `timezone`, `user-info`, `display-resolution`, `theme-detection`. |
| `roulette-animation-example/` | `xfetch/config.jsonc` + 6 figure configs | `animate-logo` plugin + `config-roulette` extension. |
| `wasm-guests/` | `presets/wasm_showcase.jsonc`, `presets/wasm_minimal.jsonc` | `wasm-crypto`, `wasm-ip-geo`, `wasm-pacman`, `wasm-proc`; effects `wasm-matrix`, `wasm-python-pulse`. |

### animate-logo

- File: `plugins/animate-logo/presets/xfetch_pacman_animate.jsonc`.
- Pac-Man layout, full module set, `ascii:
  ~/.config/xfetch/logos/xfetch_logo.txt`, `logo_animation` with
  `plugin: animate-logo`, `fps: 12`, `duration_ms: 1200`, `loop: false`.
- Runs only in a TTY.

### full-stack

- File: `plugins/full-stack/presets/full_stack.jsonc`.
- Section layout with animated logo (`style: frame`, `fps: 60`,
  `duration_ms: 7500`, `loop: true`, `frames_path:
  ~/.config/xfetch/logos/decryptfull.txt`), `logo_path:
  ~/.config/xfetch/logos/x.txt`, and info plugins `github-stats` (user
  `xscriptor`, `max-lines: 4`) and `docker`.
- The `frames_path` and `logo_path` files are not bundled.

### new-modules

- File: `plugins/new-modules/presets/enhanced_modules.jsonc`.
- Section groups System/User/Hardware/Media, exercising the six Phase 6 info
  plugins; `weather` uses `location: auto`.

### roulette-animation-example

Complete configuration tree copied into `~/.config/xfetch/`:

| Source | Destination |
| --- | --- |
| `xfetch/config.jsonc` | `~/.config/xfetch/config.jsonc` |
| `xfetch/routes-anim.json` | `~/.config/xfetch/routes-anim.json` |
| `xfetch/fetchs/animations/<figure>/config.jsonc` | `~/.config/xfetch/fetchs/animations/<figure>/config.jsonc` |
| `xfetch/logos/animations/*.txt` | `~/.config/xfetch/logos/animations/` |

Six figures: `fox` (pacman), `cat` (box), `kitty` (section), `blackhole` (tree),
`matrix` (compact), `decrypt` (side-block). Each has `<figure>.txt` (frames
separated by `===`) and `<figure>_static.txt`. All configs set `"daemon": true`,
so the animation is pinned and loops without blocking the prompt; stop with
`xfetch --daemon-stop`. Tall figures (`cat` ~29 rows, `matrix` ~24 rows) need a
tall terminal.

Portability gotcha: `routes-anim.json` contains absolute paths under
`~/.config/...`, which contradicts the repository rule against
machine-specific paths. Rewrite them to `~/.config/...` before use.

### wasm-guests

- `presets/wasm_showcase.jsonc`: all four wasm plugins plus both wasm effects.
- `presets/wasm_minimal.jsonc`: C `/proc` guest (`wasm-proc`) plus
  `wasm-matrix`.
- Install commands documented in `plugins/wasm-guests/README.md` assume a
  checkout with sibling `plugins/` and `effects/` repos:

  ```bash
  xfetch plugin install ./plugins/plugins/wasm-crypto
  xfetch plugin install ./plugins/plugins/wasm-ip-geo
  xfetch plugin install ./plugins/plugins/wasm-pacman
  xfetch plugin install ./plugins/plugins/wasm-proc
  xfetch effects install ./effects/effects/wasm-matrix
  xfetch effects install ./effects/effects/wasm-python-pulse
  ```

- Building guests from source needs `wasm32-wasip1`, `componentize-py`, Go 1.24+
  and clang/wasm-ld depending on the guest.

## Naming conventions

| Location | Pattern | Example |
| --- | --- | --- |
| `xfetch/presets/layouts/` | `layout_<name>[_full].jsonc` | `layout_box_full.jsonc` |
| `xfetch/presets/showcase/` | `<intent>.jsonc` | `pacman_star_uptime.jsonc` |
| `xfetch/presets/cyberpunk/configs/` | `NN-<slug>.jsonc` | `04-ghost-protocol.jsonc` |
| `xfetch/presets/massive-roulette/configs/` | `NNN-<palette>-<layout>.jsonc` | `042-amethyst-dots.jsonc` |
| massive custom-x | `NNN-<name>-customx.jsonc` | `119-catppuccin-latte-customx.jsonc` |
| massive gauges/langs | `<group>-NN-<name>-<layout>.jsonc` | `gauges-05-catppuccin-frappe-box.jsonc` |
| `plugins/<name>/presets/` | `<description>.jsonc` | `wasm_showcase.jsonc` |
| Plugin module key | `plugin:<binary-name>` | `plugin:github-stats` |
| Plugin registration | `"info_plugins": [{ "plugin": "<name>", "args": {...} }]` | see above |

## Key paths

| Path | Meaning |
| --- | --- |
| `xfetch/defaults/config.jsonc` | Reference default config. |
| `xfetch/presets/layouts/` | Layout-only presets. |
| `xfetch/presets/showcase/` | Showcase presets. |
| `xfetch/presets/cyberpunk/config.jsonc` | Roulette-enabled fallback for cyberpunk. |
| `xfetch/presets/massive-roulette/config.jsonc` | Roulette-enabled fallback for massive-roulette. |
| `plugins/wasm-guests/README.md` | Exact wasm install commands and toolchains. |
| `plugins/roulette-animation-example/README.md` | File-to-destination map. |
| `scripts/validate.py` | JSONC + glyph validator. |

## Verified facts

- `scripts/validate.py .` passes: `OK: 206 JSON/JSONC files valid (no broken
  glyphs, no emoji).`
- `routes.json` counts: cyberpunk 10, massive-roulette 149.
- No `.github/workflows/` exists.
- `plugins/CHANGELOG.md` is the per-folder changelog linked from the root
  `CHANGELOG.md`.
