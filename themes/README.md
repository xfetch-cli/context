# themes

Official theme registry for `xfetch`. Themes are partial JSONC configs merged
over the user's config to change colors, layout, palette style and logo
coloring without touching module selection.

Last verified: 2026-11-03 against `themes@8c2a16cc64a268b955f13cbf0f2c7419cb42f4d3` (`main`).

## Role

`xfetch-cli/themes` is the remote registry consumed by the `theme-manager`
plugin and by users who copy a theme into `~/.config/xfetch/themes/`. It holds:

- `index.json` — registry metadata for every theme (name, author, version,
  description, layout, palette style, tags, download URL).
- `schema.json` — JSON Schema for `index.json`.
- `colors/<name>.jsonc` — the theme files themselves.
- `scripts/validate.py` — JSONC validator used by local CI.
- `README.md`, `CONTRIBUTING.md`, `CHANGELOG.md`, `SUPPORT.md`,
  `CODE_OF_CONDUCT.md`, `LICENSE` (MIT).

The core binary does not fetch this repository automatically. `xfetch` only
resolves a theme that is already on disk (see `registry.md`). The network path
is the `theme-manager` plugin (`list`, `search`, `info`, `install`).

## Location

| Item | Value |
| --- | --- |
| Local clone | `~/repos/xfetch-cli/themes` |
| Remote (fetch/push) | `https://github.com/xfetch-cli/themes.git` |
| Default branch | `main` (`origin/HEAD -> origin/main`) |
| HEAD | `8c2a16cc64a268b955f13cbf0f2c7419cb42f4d3` |
| HEAD date | 2026-09-20 07:48:04 +0200 |
| HEAD subject | `upload common palettes based on popular themes (#2)` |
| Registry base URL | `https://raw.githubusercontent.com/xfetch-cli/themes/main` |
| Index URL | `https://raw.githubusercontent.com/xfetch-cli/themes/main/index.json` |
| Themes | 83 files / 83 index entries, all version `1.0.0` |
| CI | Local scripts only; no `.github/workflows/` |

## Repository tree

```text
themes/
|-- index.json             # 83 theme entries
|-- schema.json            # draft-07 schema for index.json
|-- colors/
|   |-- dracula.jsonc
|   |-- nord.jsonc
|   |-- catppuccin-mocha.jsonc
|   |-- retro-pacman.jsonc
|   |-- berlin.jsonc
|   |-- ...                # 83 total
|-- scripts/
|   |-- ci.sh
|   |-- ci.ps1
|   `-- validate.py
|-- README.md
|-- CONTRIBUTING.md
|-- CHANGELOG.md
|-- SUPPORT.md
|-- CODE_OF_CONDUCT.md
|-- LICENSE
`-- .gitattributes         # * text=auto eol=lf; linguist hints
```

There are no theme subdirectories; `colors/` is flat.

## Quick start

Install a theme locally:

```bash
# copy the file into the core themes directory
mkdir -p ~/.config/xfetch/themes
cp colors/dracula.jsonc ~/.config/xfetch/themes/

# or fetch from the registry with the plugin
xfetch plugin install theme-manager
# configure info_plugins/modules as shown in plugins/theme-manager/README.md

# activate
xfetch theme set dracula
# or add "theme": "dracula" to ~/.config/xfetch/config.jsonc
```

Inspect the registry:

```bash
python3 - <<'PY'
import json
d = json.load(open('index.json'))
print(len(d['themes']))
for t in d['themes'][:5]:
    print(t['name'], t['layout'], t['palette_style'])
PY
```

Validate before committing:

```bash
bash scripts/ci.sh          # Linux/macOS
./scripts/ci.ps1            # Windows
```

Current result: `OK: 85 JSON/JSONC files valid.` (83 theme files +
`index.json` + `schema.json`).

## Documentation index

| File | Contents |
| --- | --- |
| `README.md` | This file: role, location, quick start, doc index. |
| `registry.md` | `index.json` schema, theme file format, source URLs, core merge, theme-manager relationship, adding and validating themes. |
| `operations.md` | Scripts, validation, release/versioning, common tasks, gotchas. |

## Related repositories

| Repository | Relationship |
| --- | --- |
| `xfetch-cli/xfetch` | Resolves and merges themes (`config.rs`), runs `xfetch theme list/set/remove/export`. |
| `xfetch-cli/plugins` | `theme-manager` plugin fetches this registry and installs themes. |
| `xfetch-cli/web` | Documents themes in `docs/<lang>/themes.md` and `theme-manager.md`. |
| `xfetch-cli/configs` | Some presets set `"theme": "<name>"`; themes are optional and not bundled there. |
| `xfetch-cli/logos` | Logo art used by `--gen-config`; `logo_color`/`logo_colors` refer to the ASCII logo, not to this repo. |
