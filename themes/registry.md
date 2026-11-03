# themes registry

Registry format, theme file format, merge behavior and the relationship with
the `theme-manager` plugin.

Last verified: 2026-11-03 against `themes@8c2a16cc64a268b955f13cbf0f2c7419cb42f4d3` (`main`).

## index.json

Top-level keys:

| Key | Type | Required by schema | Purpose |
| --- | --- | --- | --- |
| `$schema` | string (URI) | No (not in schema) | Points at `https://raw.githubusercontent.com/xfetch-cli/themes/main/schema.json`; used by editors. |
| `registry` | string (URI) | Yes | Base URL for theme downloads: `https://raw.githubusercontent.com/xfetch-cli/themes/main`. |
| `themes` | array | Yes | One object per theme. |

At HEAD, `index.json` has exactly 83 entries and `colors/` has exactly 83
matching `.jsonc` files.

### Theme index entry

Schema location: `schema.json` (draft-07). Fields:

| Field | Type | Required | Constraints / notes |
| --- | --- | --- | --- |
| `name` | string | Yes | Theme identifier used in `"theme": "<name>"`; matches `colors/<name>.jsonc`. |
| `author` | string | Yes | Usually `xscriptor`. |
| `version` | string | Yes | Semver pattern `^\d+\.\d+\.\d+$`; all current entries are `1.0.0`. |
| `description` | string | Yes | One-line listing text. |
| `source` | string (URI) | Yes | Raw URL of the theme file. Convention: `https://raw.githubusercontent.com/xfetch-cli/themes/main/colors/<name>.jsonc`. |
| `layout` | string | No | Enum: `default`, `section`, `pacman`, `side-block`, `tree`, `box`, `line`, `dots`, `bottom_line`, `compact`, `minimal`, `horizontal`, `bottom`. |
| `palette_style` | string | No | Enum: `squares`, `circles`, `triangles`, `lines`, `dots`. |
| `tags` | array of strings | No | Discovery/search tags. Convention: include `dark` or `light`, plus descriptive tags (`minimal`, `popular`, `catppuccin`, ...). |
| `screenshot` | string (URI) | No | Optional preview URL. No current entry uses it. |

Index statistics at HEAD:

| Dimension | Values |
| --- | --- |
| Themes | 83 |
| Versions | `1.0.0` x83 |
| `layout` present | 17 of 83 (66 omit it; the theme file itself may still set it) |
| `layout` values | `section` x7, `default` x2, `pacman`/`tree`/`bottom`/`line`/`minimal`/`compact`/`box`/`horizontal` x1 each |
| `palette_style` present | 83 of 83: `circles` x71, `squares` x6, `dots` x2, `lines` x2, `triangles` x2 |
| Top tags | `dark` x76, `popular` x67, `light` x8, `night` x8, `warm` x6 |

Note: the schema's `layout` enum includes `bottom_line`, but the current index
uses `bottom`; both are valid layout names in core xfetch. The index does not
have to mirror every field in the theme file.

## Theme file format

A theme is a JSONC document under `colors/<name>.jsonc` containing only visual
fields. It must not contain `modules`, `info_plugins`, `icons` or any other
non-visual key.

```jsonc
{
    "layout": "section",
    "show_colors": true,
    "palette_style": "circles",
    "logo_color": "Magenta",
    "colors": {
        "os": "Magenta",
        "kernel": "White",
        "cpu": "Red",
        "memory": "Yellow",
        "shell": "Green"
    }
}
```

Supported fields (from `xfetch/docs/THEMES.md` and `CONTRIBUTING.md`):

| Field | Type | Meaning |
| --- | --- | --- |
| `layout` | string or null | Layout name (`default`, `section`, `pacman`, `side-block`, `tree`, `box`, `line`, `dots`, `bottom_line`, `compact`, `minimal`, `horizontal`, `bottom`, `section-box`, `custom-x`). |
| `colors` | object | Per-module color map. Keys are module names or `plugin:<name>`; values are color names or supported color strings. |
| `palette_style` | string or null | `squares`, `circles`, `triangles`, `lines` (core also accepts `dots`). |
| `show_colors` | boolean | Inline ANSI color swatches next to modules. |
| `logo_path` | string or null | Path to an ASCII/PNG/SVG logo. |
| `logo_color` | string or null | Color for ASCII logo rows. |
| `logo_colors` | array or null | Per-row logo colors; row `i` uses `logo_colors[i % len]`, takes precedence over `logo_color`. |
| `header_icons` | array or null | Pac-Man layout header icons. |
| `footer_text` | string or null | Pac-Man layout footer text. |

Rules:

- No `icons` block. Icons are a per-user font choice; core fills them from
  defaults. All 83 current files comply.
- Only declare what changes. Empty objects `{}` are a no-op during merge; empty
  strings do not override non-empty values.
- Color values are the supported names (`Black`-`White`, `Grey`/`Gray`, dark
  variants) or accepted color strings. Hex values are used by configs, but the
  registry themes use names.

Actual field usage at HEAD:

| Field | Files |
| --- | --- |
| `colors` | 83 |
| `logo_color` | 82 |
| `layout` | 17 |
| `show_colors` | 17 (16 true, 1 false) |
| `palette_style` | 17 |
| `header_icons` | 1 (`retro-pacman`) |
| `footer_text` | 1 (`retro-pacman`) |
| `logo_colors` | 0 |
| `logo_path` | 0 |
| `icons` | 0 |

The 66 themes without `layout` rely on the user's config layout. All 83 set
`colors`, so every registry theme changes the palette even when it changes
nothing else.

## Source URLs

Every index entry's `source` must point at the theme file in this repository:

```text
https://raw.githubusercontent.com/xfetch-cli/themes/main/colors/<name>.jsonc
```

Verified at HEAD: all 83 `source` values match exactly that pattern, and every
referenced file exists. `theme-manager` uses `source` verbatim to download the
theme, so a wrong URL is a broken install even if `index.json` is valid JSON.

## How the core merges themes

Implementation: `xfetch/src/config.rs` (`resolve_theme_path`, `deep_merge`) and
`xfetch/src/themes/mod.rs`.

Resolution order for `"theme": "<value>"`:

1. If `<value>` is an existing file path (contains `/` or is a direct path),
   load it directly.
2. Otherwise look for `<name>.jsonc` in `~/.config/xfetch/themes/`
   (`default_themes_dir()` = platform config dir + `xfetch/themes`).
3. If not found, the config loads as-is; no error.

Merge order (last wins):

```text
core defaults (hardcoded in config.rs)
        |
        v
user config.jsonc
        |
        v
theme/<name>.jsonc        <- highest priority since v0.2.0
        |
        v
final Config
```

`deep_merge` semantics:

- Objects merge recursively, key by key.
- Strings, numbers and booleans in the overlay replace the base.
- Empty strings do not override non-empty base values.
- Empty objects are a no-op.
- In v0.1.x the order was reversed (config won over theme); upgrading users may
  need to move visual overrides out of `config.jsonc`.

The theme wins over the user config, so visual fields left in `config.jsonc`
will be overwritten by the theme. Module lists and plugin configuration always
come from `config.jsonc` because themes cannot contain them.

## CLI

```bash
xfetch theme list                 # scan ~/.config/xfetch/themes/*.jsonc
xfetch theme set <name>           # set "theme" in config.jsonc (preserves comments)
xfetch theme remove <name>        # delete a theme from the themes directory only
xfetch theme export <name>        # write current visual state as a theme file
```

`theme set` does a line-oriented edit of `config.jsonc`, preserving comments,
formatting and trailing content. `theme export` writes only `layout`, `colors`,
`palette_style`, `header_icons`, `footer_text`, `logo_path` and `show_colors`
when present; it intentionally omits icons.

## theme-manager plugin relationship

The plugin lives in `xfetch-cli/plugins/plugins/theme-manager` and is a
`info_provider` binary (`xfetch-plugin-theme-manager`). It is the only official
network path from the binary to this registry.

| Property | Value |
| --- | --- |
| Default registry | `https://raw.githubusercontent.com/xfetch-cli/themes/main/index.json` |
| Override | `args.registry` (URL, `/absolute/path`, or `~/path`) |
| Actions | `list` (default), `search`, `info`, `install` |
| `list` | Reads the index and prints every theme. |
| `search` | Case-insensitive match on name, description, author and tags. |
| `info` | Shows author, version, layout, palette, tags, installed status and source. |
| `install` | Downloads `source` to `~/.config/xfetch/themes/<name>.jsonc`; local sources are copied. |
| Dependencies | `curl` in `PATH`; network for remote actions. |
| Timeouts | curl `--max-time 15` for the index, `30` for a download; plugin budget 20 s. |

Example config:

```jsonc
{
    "info_plugins": [
        { "plugin": "theme-manager", "args": { "action": "install", "name": "dracula" } }
    ],
    "modules": ["plugin:theme-manager"]
}
```

The plugin does not edit `config.jsonc`; it only installs the file. Activation
is still `xfetch theme set <name>` or a manual `"theme": "<name>"`.

## Adding a theme

1. Create `colors/<name>.jsonc` with only visual fields. Use an existing theme
   as a template (`dracula.jsonc` is a full palette; `berlin.jsonc` is a minimal
   one).
2. Add an entry to `index.json` with `name`, `author`, `version`, `description`,
   optional `layout`/`palette_style`/`tags`/`screenshot`, and a `source` URL that
   points at the new file on `main`.
3. Tag it `dark` or `light` so `theme-manager search` is useful.
4. Add a note to `CHANGELOG.md`.
5. Run `bash scripts/ci.sh`.
6. Test end to end: copy the file into `~/.config/xfetch/themes/`, run
   `xfetch theme set <name>` (or install it with `theme-manager`) and render.
7. Open a PR. PRs that fail validation are rejected.

## Validation

`scripts/validate.py` strips JSONC comments and trailing commas, then parses
every `*.jsonc` and `*.json` in the repository. It does not:

- validate against `schema.json`,
- check that index entries have matching files,
- check that `source` URLs resolve,
- reject `icons` in theme files.

Those invariants are currently maintained by hand and by review. The validator
passes 85 files at HEAD. If you need a stronger check, add it to
`scripts/validate.py` and note it here.
