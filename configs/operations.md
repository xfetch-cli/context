# configs operations

Using presets, validating JSONC, adding configs and understanding the
cross-repository dependencies.

Last verified: 2026-11-03 against `configs@31ffba72ad3c5ce3f1c4380d04fd319bcf325f63` (`main`).

## Using a preset

### Directly with `--config`

No installation required. The path can be absolute or relative to the current
directory:

```bash
xfetch --config xfetch/presets/layouts/layout_pacman_full.jsonc
xfetch --config xfetch/presets/showcase/pacman_full_white.jsonc
xfetch --config xfetch/presets/massive-roulette/configs/009-espresso-section.jsonc
```

### As the persistent user config

```bash
mkdir -p ~/.config/xfetch
cp xfetch/defaults/config.jsonc ~/.config/xfetch/config.jsonc
# preferred: generate a config with the detected distro logo
xfetch --gen-config
```

Some presets reference logo/animation files under `~/.config/xfetch/logos/` or
`~/.config/xfetch/fetchs/`; copy those too (see the preset README).

### As a roulette

`cyberpunk` and `massive-roulette` ship a `config.jsonc` with a
`config_providers` block that points `config-roulette` at their `routes.json`.
Either run with that file:

```bash
xfetch --config ~/.config/xfetch/fetchs/massive-roulette/config.jsonc
```

or merge the `config_providers` block into your own
`~/.config/xfetch/config.jsonc`:

```jsonc
{
    "config_providers": [
        {
            "extension": "config-roulette",
            "args": {
                "routes": "~/.config/xfetch/fetchs/massive-roulette/routes.json",
                "strategy": "random"
            },
            "timeout_secs": 5
        }
    ]
}
```

`strategy` is `random` by default; `daily` keeps the same config for the day.
Install the extension first:

```bash
xfetch extension install config-roulette
```

### Installing plugin binaries

Configs that use `plugin:<name>` need the binary installed. The documented way:

```bash
xfetch plugin install animate-logo
xfetch plugin install docker github-stats
xfetch plugin install music-player weather timezone user-info display-resolution theme-detection
xfetch plugin install gauges langs banner
```

Plugins that are not published yet (for example `gauges`, `langs`, `banner` at
the time of writing) are built from a `xfetch-cli/plugins` checkout:

```bash
cargo build --release -p xfetch-plugin-gauges -p xfetch-plugin-langs -p xfetch-plugin-banner
mkdir -p ~/.config/xfetch/plugins
cp target/release/xfetch-plugin-{gauges,langs,banner} ~/.config/xfetch/plugins/
```

Effects and extensions come from their own repositories / install commands:

```bash
xfetch effects install decrypt
xfetch effects install glitch
xfetch extension install config-roulette
```

## Validating JSONC

The repository validates with Python only; there is no Node/Rust build.

```bash
# Whole repository (CI wrapper)
bash scripts/ci.sh          # Linux/macOS
./scripts/ci.ps1            # Windows

# Just the validator (any platform)
python3 scripts/validate.py .
```

`scripts/validate.py`:

1. Recursively finds every `*.jsonc` and `*.json` (skipping `.git`).
2. Strips `//` line comments and `/* ... */` block comments with a string-aware
   scanner.
3. Removes trailing commas.
4. Parses the result with `json.loads`.
5. Rejects, in the raw text:
   - U+FFFD replacement characters (mangled glyphs),
   - control characters other than `\n` and `\t`,
   - emoji codepoints (a curated `EMOJI_RANGES` table).

Rules enforced indirectly: use Nerd Font glyphs (private use area) or plain
text for icons, never emoji; keep the files parseable JSONC.

Current result at HEAD: `OK: 206 JSON/JSONC files valid (no broken glyphs, no
emoji).`

## Adding a preset

1. Pick the correct directory:
   - `xfetch/presets/layouts/` — a layout-only preset; follow
     `layout_<name>[_full].jsonc`.
   - `xfetch/presets/showcase/` — a self-contained showcase; descriptive name.
   - `plugins/<plugin-name>/` — a config that requires a plugin binary; include
     the plugin in `info_plugins`.
2. Keep it portable: no usernames, no absolute `/home/...` paths, no
   machine-specific identifiers. Use `~/.config/xfetch/...` when referring to
   installed user files.
3. Use only keys the installed xfetch version supports. The canonical schema is
   `xfetch-cli/xfetch/docs/CONFIGURATION.md`.
4. Use only supported color names (`Black`-`White`, `Grey`/`Gray`, dark
   variants) or hex values where the field allows them; icons may be any text
   including Nerd Font glyphs, but not emoji.
5. Add a README section for plugin-dependent presets: required binaries, fonts,
   effects, extensions and install commands.
6. Run `bash scripts/ci.sh`.
7. Test end to end with `xfetch --config <path>`.
8. Add an entry to `CHANGELOG.md` (and `plugins/CHANGELOG.md` for plugin
   examples).
9. Open a PR. PRs that fail validation are rejected.

For a roulette route, add the file and a matching `_name`/`path` entry to the
adjacent `routes.json` (and keep the count in the README accurate).

## Cross-repo dependencies

| Dependency | Where used | Contract |
| --- | --- | --- |
| `xfetch-cli/xfetch` | Every file | Config schema, layout names, `--config`, `--gen-config`, plugin/effect wiring, daemon. |
| `xfetch-cli/plugins` | `plugins/*`, cyberpunk, massive-roulette | Plugin binaries (`animate-logo`, `docker`, `github-stats`, `gauges`, `banner`, `langs`, ...), JSON-over-stdio protocol. |
| `xfetch-cli/extensions` | cyberpunk, massive-roulette, roulette-animation | `config-roulette` config provider. |
| `xfetch-cli/effects` | massive-roulette custom-x, wasm-guests | `decrypt`, `glitch`, `wasm-matrix`, `wasm-python-pulse`. |
| `xfetch-cli/themes` | Some configs set `"theme": "<name>"` | Optional. If the theme is absent, `xfetch` ignores it and keeps inline `colors`. |
| `xfetch-cli/logos` | `--gen-config`, huge copy of logo art in massive-roulette | ASCII art files; configs usually reference `~/.config/xfetch/logos/<name>.txt`. |
| `xfetch-cli/web` | Documentation | `docs/<lang>/presets.md` describes this repository. |

## Gotchas

1. **JSONC is not universal.** Core xfetch accepts JSONC. `config-roulette`
   parses route targets with strict JSON, so the cyberpunk configs are
   comment-free despite the `.jsonc` extension. If you add comments there, the
   roulette will fail to load that route.
2. **No bundled theme registry.** Configs may reference theme names that do not
   exist in `xfetch-cli/themes` (very common in massive-roulette). Inline colors
   are the source of truth; the theme is a bonus.
3. **Relative logo paths.** Showcase `x_logo_*` configs point at
   `logos/x_logo.txt`, which is not in this repository. Either provide the file
   under `~/.config/xfetch/logos/` or change `logo_path`.
4. **Absolute paths in one example.** `plugins/roulette-animation-example/xfetch/
   routes-anim.json` hardcodes `~/...`. Rewrite to `~` before use; this
   also violates the repo's own portability rule.
5. **Fetch path conventions.** `cyberpunk` and `massive-roulette` expect to be
   copied to `~/.config/xfetch/fetchs/<name>/` because their routes and banner
   paths hardcode that location.
6. **TTY-only features.** `animate-logo`, `banner`, daemons and effects need a
   real TTY. Piping xfetch output disables animation/banner behavior.
7. **CJK fonts.** Cyberpunk banners and the 50 gauge configs print Japanese or
   Chinese characters; install `noto-fonts-cjk` (or equivalent) or they render
   as tofu.
8. **Daemon semantics.** The roulette-animation configs use `"daemon": true`;
   `duration_ms`/`loop` are ignored while pinned. Stop with
   `xfetch --daemon-stop`.
9. **`config-roulette` can repeat.** Consecutive runs in the same second can
   select the same config because the extension seeds from sub-second time.
10. **Validator is not a schema check.** `scripts/validate.py` proves parseability
    and glyph safety, not that keys exist in the current xfetch. A valid JSONC
    file can still use an obsolete key.
11. **No GitHub Actions.** There is no remote CI gate in this repository; the
    only checks are the local scripts and reviewer tests.
12. **Plugin names versus binaries.** Configs use `"plugin": "github-stats"`,
    while the installed file is `xfetch-plugin-github-stats`. Use the bare name
    in `info_plugins` and `plugin:<name>` in `modules`.
13. **`config.jsonc` in a preset is a full config, not a theme.** In `cyberpunk`
    and `massive-roulette` it is the roulette entry point plus fallback; do not
    merge it blindly over an existing config without reviewing modules.
