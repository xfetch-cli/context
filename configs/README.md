# configs

Curated presets, layouts and example configurations for `xfetch`, split between
configurations that work with the core binary only and ones that need plugin
binaries.

Last verified: 2026-11-03 against `configs@31ffba72ad3c5ce3f1c4380d04fd319bcf325f63` (`main`).

## Role

`xfetch-cli/configs` is the configuration example repository for the ecosystem:

- `xfetch/defaults/config.jsonc` — the reference default config (the shape of a
  generated `~/.config/xfetch/config.jsonc`).
- `xfetch/presets/layouts/` — small, focused layout presets.
- `xfetch/presets/showcase/` — 23 single-purpose showcase configs.
- `xfetch/presets/cyberpunk/` — a 10-config themed collection.
- `xfetch/presets/massive-roulette/` — 149 configs for the `config-roulette`
  extension.
- `plugins/` — examples that require external plugin binaries, extensions or
  effects.

It is not a package, a binary distribution, or a theme registry. There is no
build step. The repository is consumed by copying files or passing `--config`.

## Location

| Item | Value |
| --- | --- |
| Local clone | `~/repos/xfetch-cli/configs` |
| Remote (fetch/push) | `https://github.com/xfetch-cli/configs.git` |
| Default branch | `main` (`origin/HEAD -> origin/main`) |
| HEAD | `31ffba72ad3c5ce3f1c4380d04fd319bcf325f63` |
| HEAD date | 2026-10-19 14:50:26 +0200 |
| HEAD subject | `upload cyberpunk presets on config repo (#8)` |
| Tracking | Plain JSON/JSONC files, MIT license |

There is no `.github/workflows/`. The only GitHub metadata is the issue-template
directory. Validation is local via `scripts/ci.*`.

## Quick start

Use a preset directly:

```bash
xfetch --config /path/to/configs/xfetch/presets/layouts/layout_pacman_full.jsonc
```

Install the reference config:

```bash
mkdir -p ~/.config/xfetch
cp /path/to/configs/xfetch/defaults/config.jsonc ~/.config/xfetch/config.jsonc
# or, preferred: let the binary generate it
xfetch --gen-config
```

Validate the whole repository before committing:

```bash
cd ~/repos/xfetch-cli/configs
bash scripts/ci.sh          # Linux/macOS
# or: ./scripts/ci.ps1      # Windows
```

The validator currently passes 206 JSON/JSONC files (193 under `xfetch/`, plus
13 plugin example files).

## Documentation index

| File | Contents |
| --- | --- |
| `README.md` | This file: role, location, quick start, doc index. |
| `inventory.md` | Complete inventory of defaults, presets by category, plugin examples, naming and plugin-binary requirements. |
| `operations.md` | Using a preset, validating JSONC, adding a preset, cross-repo dependencies, gotchas. |

## Related repositories

| Repository | Relationship |
| --- | --- |
| `xfetch-cli/xfetch` | Consumes the configs; authoritative config schema in `docs/CONFIGURATION.md`. |
| `xfetch-cli/plugins` | Binaries required by `plugins/` examples and several presets (`animate-logo`, `gauges`, `banner`, `langs`, ...). |
| `xfetch-cli/extensions` | `config-roulette` extension used by `cyberpunk` and `massive-roulette`. |
| `xfetch-cli/effects` | `decrypt`/`glitch` and WebAssembly effects used by some presets. |
| `xfetch-cli/themes` | Theme names referenced by configs; themes are optional and not bundled here. |
| `xfetch-cli/logos` | Source of distro ASCII art; some presets reference logo files under `~/.config/xfetch/logos/`. |
| `xfetch-cli/web` | Documents the presets in `docs/<lang>/presets.md`. |
