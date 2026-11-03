# Addon system (plugins, effects, extensions)

xfetch has three installable addon kinds. They share metadata, install and
listing infrastructure but run in different parts of the lifecycle.

## Kinds and paths

| Kind | CLI | Install dir | Native prefix | Wasm name | Config key |
| --- | --- | --- | --- | --- | --- |
| Plugin | `xfetch plugin` (alias `plugins`) | `~/.config/xfetch/plugins` | `xfetch-plugin-` | `xfetch-plugin-<name>.wasm` | `info_plugins`, `logo_animation` |
| Effect | `xfetch effects` (alias `effect`) | `~/.config/xfetch/effects` | `xfetch-effect-` | `xfetch-effect-<name>.wasm` | `effects` |
| Extension | `xfetch extension` (alias `extensions`) | `~/.config/xfetch/extensions` | `xfetch-extension-` | `xfetch-extension-<name>.wasm` | `config_providers` |

Sidecar manifests are `<binary-name>.json` next to wasm artifacts. Native
addons are bare executables (`.exe` on Windows).

## Shared model (`src/addons/`)

- `model.rs`: `AddonKind` (label, plural, prefix, directory, binary/wasm/
  manifest names, installed path), `AddonRuntime` (`native`/`wasm`),
  `AddonEntry` (loads manifest, reports version/description/capabilities/
  limits/enabled state/config snippet).
- `install.rs`: name inference, direct wasm/URL installs, `--name` and
  `--manifest` handling, sidecar writing, prefix stripping.
- `catalog.rs`: remote catalog fetch/search/JSON printing. Catalog sources:
  - `https://raw.githubusercontent.com/xfetch-cli/plugins/main/index.json`
  - `https://raw.githubusercontent.com/xfetch-cli/effects/main/index.json`
  - `https://raw.githubusercontent.com/xfetch-cli/extensions/main/index.json`
  Environment overrides: `XFETCH_PLUGIN_INDEX`, `XFETCH_EFFECT_INDEX`,
  `XFETCH_EXTENSION_INDEX`. Catalog schema version is validated.
- `report.rs`: list/info text, including capability/limit lines.

## Plugin runtime (`src/plugins/`)

- Discovery: `find_plugin_binary` checks `PATH` first, then config directories
  (including a development `target/release` lookup for source installs).
- Execution: `run_info_plugin` / `run_logo_animation_plugin` send one JSON
  request on stdin and read one JSON response from stdout.
- Install: from a local source directory, a repository (`--repo`), a direct
  `.wasm` file, or a URL. Native installs build with `cargo build --release`
  and copy the artifact; wasm installs copy the `.wasm` and its sidecar.
- List/remove: `src/plugins/manage.rs`.

## Effect runtime (`src/effects/`)

- Discovery: config dirs first, then `PATH`.
- Execution: `run_effect` sends `EffectRequest { lines, style, duration_ms,
  fps, args }`, parses `EffectResponse { frames }`, and returns frames to the
  UI. Effects are played only on a TTY and not with image logos.
- Failure is non-fatal: missing binary, non-zero exit, invalid JSON, empty
  frames, or timeout are reported and skipped; the fetch still renders.
- Native effects run unsandboxed; wasm effects use the wasm policy.

## Extension runtime (`src/extensions/`)

- Discovery: config dirs first, then `PATH`.
- Lifecycle: config providers run after the default/file/theme merge, in
  declaration order, but only if a config exists and parses.
- Protocol: full resolved config JSON on stdin -> modified config JSON on
  stdout (`{"config": ...}`).
- Provider failures are silent; an invalid final config discards the provider
  chain. The provider list is cloned before iteration, so an extension cannot
  add providers for the current run.
- There is no development `target/release` lookup for native extensions; bare
  native binaries are not accepted by `install` (use the source directory or
  copy manually).

## Wasm addons

Wasm addons are described by a manifest (`src/wasm/manifest.rs`):

```jsonc
{
  "manifest_version": 1,
  "name": "foo",
  "version": "0.2.0",
  "description": "...",
  "kind": "info_provider",          // logo_animation | info_provider | effect | config_provider
  "runtime": "wasm",
  "entry": "run",
  "capabilities": {
    "http": { "allow": ["https://api.example.com/*"] },
    "exec": { "allow": ["curl"], "env": ["HTTP_PROXY"] },
    "fs": ["~/.config/xfetch", { "host": "~/.cache/xfetch", "guest": "/data", "mode": "rw" }],
    "env": ["HOME"],
    "args": false
  },
  "limits": {
    "timeout_ms": 2000,
    "memory_mb": 64,
    "output_kb": 512,
    "host_call_kb": 64
  },
  "artifact_url": "...",             // optional
  "sha256": "..."                    // optional integrity pin
}
```

Policy is deny-by-default (`src/wasm/policy.rs`). The installed sidecar is
authoritative. See [wasm.md](wasm.md) for the full runtime.

## Install flows

- **Source directory**: `xfetch plugin install ./plugins/gauges` builds the
  crate and installs the native binary (or wasm artifact + sidecar).
- **Repository**: `--repo <url>` shallow-clones the repo and looks for the
  addon by name; uses the shared temp-dir/`git clone --` safety helpers.
- **Direct wasm**: `xfetch plugin install ./foo.wasm [--name foo]
  [--manifest foo.json]` copies the artifact and optional sidecar.
- **URL**: downloads a wasm artifact; `--sha256` is enforced when provided.
- **Catalog name**: `xfetch plugin install <name>` resolves the official repo
  by default (`XFETCH_PLUGIN_REPO` etc. override).

Remote installs validate guest names (`[A-Za-z0-9._-]`, no `.`/`..`/separators)
and repository strings (no leading `-`, no NUL), create unique `0700` work
dirs on Unix, and use `git clone --depth 1 -- <repo> <dest>`.

## Listing and inspection

- `list` prints installed native/wasm artifacts from the config directory.
- `list --verbose` adds kind, runtime, version, capabilities, limits, path,
  and enabled state.
- `info <name>` loads the manifest and prints the same plus the configuration
  snippet needed to enable the addon.
- `search [query] [--json]` reads the remote catalog; `available` is an alias.

`enabled` for plugins/effects is inferred from the config keys
(`info_plugins`, `logo_animation.plugin`, `effects`); extensions are enabled
when listed in `config_providers`.

## Security model

- Native addons: unsandboxed, user privileges. Treat as trusted code.
- Wasm addons: wasmtime with epoch timeouts, memory cap, output cap,
  host-call cap, and manifest-granted HTTP/exec/fs/env/args.
- Installed sidecar manifests are the authority for wasm capabilities; a
  checked-in manifest cannot silently fall back to an unverified URL.
- `--sha256` on install takes precedence over the manifest digest; supplying
  it for a native source is a hard error.
- Catalog fetches are bounded (15 s, size limit) and schema-version checked.

## Cross-repo links

- `xfetch-cli/api`: SDK crates and WIT contract.
- `xfetch-cli/plugins`, `effects`, `extensions`: implementations, catalogs and
  generators.
- `xfetch-cli/themes`: registry consumed by the `theme-manager` plugin.
- `xfetch-cli/logos`: art catalog consumed by `--gen-config`.

## Gotchas

- `main` of the addon repos lacks `index.json`/`generate_catalog.py` at the
  time of writing; catalogs live on `release/v1.0.2`. Remote `search` 404s
  until those merge.
- Catalog `catalog_version` tracks the xfetch release train (`1.0.2`), not the
  addon crate versions (`0.2.0`).
- The plugins catalog generator hardcodes native kind for
  `{animate-logo, banner}`; extensions validate wasm manifest versions, the
  plugins generator does not.
- Native timeout semantics: unset `timeout_secs` = 30 s, `0` disables the cap.
  Wasm: unset uses manifest `timeout_ms`; explicit `0` currently creates an
  immediate deadline rather than disabling the cap.
