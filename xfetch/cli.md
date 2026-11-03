# xfetch CLI surface

All commands and flags are defined in `src/cli.rs` and dispatched in
`src/main.rs`. This document is the quick reference for agents that need to
change or extend the CLI.

## Global flags

| Flag | Meaning |
| --- | --- |
| `-c, --config <CONFIG>` | Use a custom JSONC config path. |
| `--gen-config` | Generate a starter config (default location or `--config`). |
| `--logo <LOGO>` | Logo id/alias to embed with `--gen-config`; overrides detection. |
| `--layout <LAYOUT>` | Layout for the generated config; defaults to `section`. |
| `--list-layouts` | Print every accepted layout name and exit. |
| `--clean-cache` | Remove `$XDG_CACHE_HOME/xfetch/cache.json`. |
| `--benchmark` | Print probe timing lines. |
| `--daemon` | Run the animated-logo daemon. |
| `--daemon-stop` | Stop the animated-logo daemon. |
| `--no-daemon-live` | Disable the live-stats daemon from the terminal. |
| `--daemon-live-stop` | Stop the live-stats daemon. |
| `--daemon-live-reload` | Force hot reload in the live-stats daemon. |

Early-exit order in `run()`: `--list-layouts`, `--clean-cache`,
`--daemon-stop`, `--daemon-live-stop`, `--gen-config`, then subcommands, then
the normal fetch.

## Layout names

`xfetch --list-layouts` prints:

```text
default
side-block
tree
section
section-box
custom-x
compact
minimal
pacman
box
line
dots
bottom_line
horizontal
bottom
```

`section` is what `--gen-config` writes by default. Unknown `--layout` values
warn and keep `section`. `demo`/custom layouts are not accepted by the flag;
`custom-x` has its own `custom_x` config block.

## Addon commands

The plugin, effect and extension command families share a shape:

```text
xfetch plugin    install <path|name> [--repo URL] [--sha256 HEX] [--name NAME] [--manifest FILE]
xfetch plugin    list [--verbose]
xfetch plugin    info <name>
xfetch plugin    search [QUERY] [--json]      # alias: available
xfetch plugin    remove <name>

xfetch effects   install|list|info|search|remove ...   # alias: effect
xfetch extension install|list|info|search|remove ...
```

Notes:

- The singular product names are `plugin`, `effects` (alias `effect`),
  `extension` (alias `extensions`).
- `install` accepts a local directory/repo, a direct `.wasm` file, a URL, or a
  catalog name. Direct wasm installs accept `--name` and `--manifest`.
- `--sha256` is only valid for wasm/remote artifacts; using it for native
  sources is a hard error.
- `list --verbose` adds kind, runtime, version, capabilities, limits, path and
  enabled state.
- `info <name>` reports manifest kind/runtime/version/capabilities/limits,
  enabled state, and the config snippet needed to activate the addon.
- `search` hits the remote catalog. Override with
  `XFETCH_PLUGIN_INDEX`, `XFETCH_EFFECT_INDEX`, `XFETCH_EXTENSION_INDEX`.
  `--json` emits the catalog entries.
- Remote installs default to `XFETCH_PLUGIN_REPO`,
  `XFETCH_EFFECT_REPO`, `XFETCH_EXTENSION_REPO` or the official GitHub repos.

## Theme commands

```text
xfetch theme list
xfetch theme set <name>
xfetch theme remove <name>
xfetch theme export <name>
```

- `list` scans `~/.config/xfetch/themes/*.jsonc`.
- `set` rewrites only the `"theme"` value in the active config.
- `remove` only removes themes inside the local themes dir.
- `export` writes the current resolved theme as a reusable file.

Theme files are partial JSONC (colors, layout, palette style, logo colors);
they must not contain icons.

## WebAssembly tooling

```text
xfetch wasm inspect <path> [--json]
xfetch wasm run <path> [--request JSON | --request-file PATH] [--timeout SECS] [--kind plugin|effect|extension]
xfetch wasm wit
```

- `inspect` prints kind, manifest, capabilities, limits without executing.
- `run` executes a guest with a raw JSON request and prints the response; used
  by authors and scripted end-to-end checks. `--kind` selects the component
  world for component guests (default `plugin`).
- `wit` prints the WIT package for componentize-py/componentize-js/wit-bindgen.

## Update

```text
xfetch update [--check] [--prebuilt] [--bin-dir DIR] [--yes]
```

- `--check` exits 1 when a newer release exists (useful in scripts).
- Without `--prebuilt`, the updater chooses the install method (prebuilt
  release or `cargo install`) based on provenance.
- `--prebuilt` forces replacing the binary in place (Unix); `--bin-dir`
  selects the directory.
- Downloads verify `SHA256SUMS` and, from v1.0.1 onward, the cosign bundle.

## Generated config

```bash
xfetch --gen-config
xfetch --gen-config --layout tree
xfetch --gen-config --logo arch
xfetch --gen-config --layout section --logo windows-11 --config /tmp/xfetch.jsonc
```

`--gen-config` writes the embedded template (`src/templates/config.jsonc`),
optionally swaps the layout, and tries to fetch the detected distro logo from
`xfetch-cli/logos`. If the fetch fails, the template is written without an
`ascii` override. `--logo` forces a catalog id/alias and skips `ID_LIKE`
resolution.

## Exit behavior

- Normal fetch and successful commands: exit 0.
- `update --check` with a newer release: exit 1.
- Any `XFetchError`: message on stderr, exit 1.
- Addon install/list/search failures return human-readable errors; most
  non-fatal runtime failures (missing effect binary, bad addon JSON) are
  reported and skipped so the fetch still renders.

## Command-file map

| Concern | File |
| --- | --- |
| clap definitions | `src/cli.rs` |
| dispatch/run order | `src/main.rs` |
| addon commands | `src/addons/`, `src/plugins/`, `src/effects/`, `src/extensions/` |
| theme commands | `src/themes/mod.rs` |
| wasm commands | `src/wasm/` |
| update | `src/update/` |
| config generation | `src/config.rs`, `src/logos.rs` |
| layout listing | `src/ui/layout.rs`, `src/main.rs` |
