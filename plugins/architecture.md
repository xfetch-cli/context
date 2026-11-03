# plugins architecture

Deep structural notes for `xfetch-cli/plugins`. Everything here was checked
against the working tree at `76f94dd39bab21ceb0b3295e78db4cb5077145ed` and
against the core implementation in the sibling `xfetch` clone.

## Position in the ecosystem

```text
xfetch-cli/xfetch  (core binary, version 1.0.2)
        |
        | spawns native process or runs wasm guest
        v
xfetch-cli/plugins (this repo, 20 plugin directories)
        |
        | uses protocol types and guest helpers from
        v
xfetch-cli/api (xfetch-plugin-api 0.2.0, xfetch-guest-api 0.2.0,
                wit/xfetch-runtime.wit)
```

Other sibling repositories: `xfetch-cli/effects` (intro effects),
`xfetch-cli/extensions` (config providers), `xfetch-cli/themes`
(theme registry consumed by `theme-manager`), `xfetch-cli/assets` (banner
SVG). The core installs, lists and removes all three addon families through
the same `src/addons/` code; execution lives in `src/plugins/`,
`src/effects/` and `src/extensions/`.

## Repository tree

```text
plugins/
|-- Cargo.toml                 workspace: 17 members, resolver = 2
|-- Cargo.lock                 committed lockfile (version 4)
|-- .cargo/config.toml         git-fetch-with-cli = true
|-- README.md                  user-facing catalog table (17 of 20 plugins on main)
|-- CHANGELOG.md               repo-level changes
|-- ROADMAP.md
|-- CONTRIBUTING.md            contribution workflow and with_timeout standard
|-- CODE_OF_CONDUCT.md, LICENSE, SECURITY.md, SUPPORT.md
|-- docs/
|   |-- README.md              authoritative authoring + protocol guide
|   `-- compatibility.md       Linux/macOS/Windows matrix (17 of 20 on main)
|-- plugins/
|   |-- animate-logo/          Cargo member; assets/, configs/
|   |-- banner/                Cargo member
|   |-- banner-text/           Cargo member
|   |-- chocolatey/            Cargo member
|   |-- display-resolution/    Cargo member
|   |-- docker/                Cargo member
|   |-- gauges/                Cargo member
|   |-- github-stats/          Cargo member
|   |-- langs/                 Cargo member
|   |-- music-player/          Cargo member
|   |-- temperature/           Cargo member
|   |-- theme-detection/       Cargo member
|   |-- theme-manager/         Cargo member
|   |-- timezone/              Cargo member
|   |-- user-info/             Cargo member
|   |-- wasm-crypto/           Cargo member; xfetch-plugin.json
|   |-- weather/               Cargo member
|   |-- wasm-ip-geo/           Python component; not a Cargo member
|   |-- wasm-pacman/           Go core module; not a Cargo member
|   `-- wasm-proc/             freestanding C core module; not a Cargo member
|-- scripts/
|   |-- ci.sh                  fmt + clippy + test
|   |-- ci.ps1                 Windows equivalent
|   |-- ci-windows.sh          cross-check/clippy for x86_64-pc-windows-gnu
|   `-- ci-wasm.sh             builds every wasm example it has toolchains for
|-- ci/
|   |-- unix.sh                test + with_timeout grep (Linux/macOS)
|   `-- windows.ps1            test + with_timeout grep (Windows)
`-- .github/
    |-- workflows/security-audit.yml
    `-- dependabot.yml
```

On `origin/release/v1.0.2` the tree additionally contains `index.json` and
`scripts/generate_catalog.py`. They are absent from `main` at the verified
HEAD.

## Workspace layout

`Cargo.toml`:

```toml
[workspace]
members = [
  "plugins/animate-logo",
  "plugins/banner",
  "plugins/banner-text",
  "plugins/chocolatey",
  "plugins/display-resolution",
  "plugins/docker",
  "plugins/gauges",
  "plugins/github-stats",
  "plugins/langs",
  "plugins/music-player",
  "plugins/theme-detection",
  "plugins/theme-manager",
  "plugins/timezone",
  "plugins/user-info",
  "plugins/weather",
  "plugins/temperature",
  "plugins/wasm-crypto",
]
resolver = "2"

[workspace.dependencies]
xfetch-plugin-api = { version = "0.2.0", package = "xfetch-plugin-api" }
xfetch-guest-api  = { version = "0.2.0", package = "xfetch-guest-api" }
```

Consequences:

- All 17 Rust members share one `target/` directory at the repository root
  (`plugins/target/release/`), not per-crate `target/` directories. The core
  installer walks ancestor `target/release/` directories, which is why the
  workspace build is discoverable.
- `wasm-crypto` is a workspace member and builds natively too. Its native
  build compiles, but host calls return `Unsupported`; the fallback lines are
  emitted. The installer sees its `xfetch-plugin.json` and treats the
  directory as a wasm source, so installing it runs the manifest `build`
  command unless an artifact already exists.
- The three non-Cargo wasm directories are invisible to `cargo test
  --workspace` and to the `ci/unix.sh` `with_timeout` loop.
- `.cargo/config.toml` sets `[net] git-fetch-with-cli = true`, matching the
  core installer's `CARGO_NET_GIT_FETCH_WITH_CLI=true`.

## Plugin directory conventions

Native Rust plugin:

```text
plugins/<name>/
|-- Cargo.toml        [package] name = "xfetch-plugin-<name>", edition = "2024"
|-- src/main.rs       reads stdin, writes stdout, exits non-zero on error
|-- README.md         features, args, platform, example config
|-- CHANGELOG.md      optional; present in most official plugins
|-- assets/           optional static assets (animate-logo)
`-- configs/          optional example presets (animate-logo)
```

The package name must be exactly `xfetch-plugin-<name>`; the catalog
generator enforces this for native entries, and the installer derives the
installed binary name from the directory name, not from `[package].name`.

WebAssembly plugin:

```text
plugins/<name>/
|-- xfetch-plugin.json    source manifest (copied to the install dir)
|-- src/main.rs           Rust core module (wasm-crypto)
|-- app.py                Python component (wasm-ip-geo)
|-- main.go, go.mod       Go core module (wasm-pacman)
|-- main.c, build.sh      freestanding C core module (wasm-proc)
|-- dist/                 build output, gitignored
`-- README.md, CHANGELOG.md
```

Only the explicit file name `xfetch-plugin.json` is recognized as a source
manifest by the installer. It looks for `xfetch-plugin.json` or
`plugin.json` (the generic `<label>.json` fallback); an unrelated
`<name>.json` is ignored.

Manifest identity invariants (checked by `generate_catalog.py --check` on the
release branch, not by the core at install time):

- `name` must equal the directory name.
- `runtime` must be `core` or `component`.
- `version`, `description`, `kind` and `capabilities` must be present.
- `capabilities` must be a JSON object; an empty object means no
  capabilities.

## Native vs wasm

| Aspect | Native plugin | Wasm plugin |
| --- | --- | --- |
| Artifact | ELF/PE/Mach-O executable | `.wasm` core module or component |
| Detection | not wasm magic bytes | first 8 bytes: core version `1` or component version `13`, layer `1` |
| Execution | ordinary child process | wasmtime 50.0.1 (core feature `wasm`, enabled by default) |
| Sandbox | none; full user privileges | deny-by-default capabilities from the manifest |
| Timeout control | plugin's `with_timeout` plus core `timeout_secs` (default 30 s, `0` disables) | guest `with_timeout` on Rust wasm plus wasmtime epoch deadline from `timeout_secs` or manifest `timeout_ms` |
| Capabilities | whatever the OS user can do | `http`, `exec`, `fs`, `env`, `args` allowlists |
| Manifest | none (sidecar ignored) | sidecar `xfetch-plugin-<name>.json`, embedded `xfetch:manifest` section, or defaults |
| Installed names | `xfetch-plugin-<name>` / `.exe` | `xfetch-plugin-<name>.wasm` + `xfetch-plugin-<name>.json` |

Wasm detection is content-based, never extension-based. Renaming a native
binary to `.wasm` does not make it a wasm guest; the header check fails and
the native path is used. Conversely, a wasm artifact with no `.wasm` suffix
is still detected.

Building xfetch with `--no-default-features` removes the `wasm` feature. Such
a binary still detects wasm headers but returns a clear error instead of
spawning the guest.

## Discovery

`src/plugins/mod.rs` in the core resolves the plugin name from the config
(`"plugin": "<name>"`). Exact order for `find_plugin_binary(name)`:

1. If `name` contains a path separator (`/` or `\`) and `Path::new(name)`
   is an existing file, use it verbatim. A bare file name in the current
   directory is not accepted here.
2. `PATH`, looking for `xfetch-plugin-<name>` and then
   `xfetch-plugin-<name>.wasm`.
3. Each config search directory (`XDG_CONFIG_HOME` when absolute, then the
   platform config dir), looking in `<config>/xfetch/plugins/` for the native
   name and the `.wasm` name.
4. Development directories, rooted at the current working directory. If
   `XFETCH_PLUGIN_DEV_DIR` is set, these are tried first:
   `<dev>/<name>`, `<dev>/plugins/<name>`,
   `<dev>/plugins/plugins/<name>`. Then:
   `<cwd>/<name>`, `<cwd>/plugins/<name>`,
   `<cwd>/plugins/plugins/<name>`, `<parent>/plugins/<name>`,
   `<parent>/plugins/plugins/<name>`.
5. Inside each development directory, for each binary name, check
   `<dir>/target/release/<binary>`, `<dir>/<binary>`, then every ancestor
   `<ancestor>/target/release/<binary>`, plus wasm development targets
   `<dir>/target/wasm32-wasip1/release/<binary>`,
   `<dir>/target/wasm32-wasi/release/<binary>` and the same two under every
   ancestor.

The concrete directory set is deduplicated, but the first hit wins. The
practical effect is that an installed native plugin in the config directory
is not used if a same-named executable is earlier in `PATH`; and a binary in
the workspace `target/release/` is found before a stale dev copy in a plugin
subdirectory.

For an info plugin, "installed" does not mean "enabled": `plugin list
--verbose` reports `enabled` only when the name is in `info_plugins` **and**
the module key `plugin:<name>` is present in `modules` (including nested
module groups), or, for a logo plugin, when
`logo_animation.plugin == name`. An info plugin can therefore be installed
and still show as `not enabled`.

Extensions use a different resolver (see the extensions book), notably
searching config directories before `PATH` and having no native
`target/release` development lookup.

## Install flow

`xfetch plugin install <source>` dispatches in this order
(`xfetch/src/plugins/install.rs`):

1. **HTTP(S) URL**: treated as a prebuilt wasm artifact. The name is the
   last URL path segment with query/fragment and `.wasm` removed;
   `validate_guest_name` enforces `[A-Za-z0-9._-]`. This URL path does
   **not** strip the `xfetch-plugin-` prefix, so
   `.../xfetch-plugin-weather.wasm` installs under
   `xfetch-plugin-weather` unless you pass `--name weather`. Passing
   `--name`/`--manifest` switches to the direct installer
   (`addons::install::install_direct`), whose `infer_name` does strip the
   prefix. The file is downloaded and installed with `--sha256` when
   provided.
2. **Existing local file**: if the header is wasm, install as a wasm
   artifact. If it is not wasm and `--sha256` was supplied, fail with
   `--sha256 cannot be enforced for a native plugin install`.
3. **Local directory**: resolve `path`, `cwd/path`, `cwd/plugins/path`,
   `cwd/plugins/plugins/path`, `parent/plugins/path`,
   `parent/plugins/plugins/path`.
4. **Remote repository**: if no local directory was found, clone
   `--repo <url>` or `XFETCH_PLUGIN_REPO` or the default
   `https://github.com/xfetch-cli/plugins.git` with
   `git clone --depth 1 -- <repo> <temp>/repo`. The plugin directory is
   looked up at `<clone>/<name>`, `<clone>/plugins/<name>` and
   `<clone>/plugins/plugins/<name>`.

A bare native executable is **not** an accepted install source. Passing
`./target/release/xfetch-plugin-weather` falls through directory resolution
and then fails guest-name validation on the remote path. Install native
plugins from their source directory (or copy the binary into
`<config>/xfetch/plugins/` manually).

Local wasm sources go through `wasm::install::plan`:

1. If a manifest `artifact` path exists, use it.
2. Otherwise try conventional paths: `dist/<name>.wasm`, `<name>.wasm`,
   `xfetch-plugin-<name>.wasm`, `xfetch-effect-<name>.wasm`,
   `xfetch-extension-<name>.wasm`,
   `target/wasm32-wasip1/release/<name>.wasm`,
   `target/wasm32-wasip1/release/xfetch-<label>-<name>.wasm`, and the
   `wasm32-wasi` equivalent.
3. Else if the manifest has `artifact_url`, download it.
4. Else if the manifest has `build`, run it in the source directory
   (`sh -c` on Unix, `cmd /C` on Windows), print a warning that the build is
   unsandboxed, then re-plan.
5. Else if a manifest declares a wasm runtime but no artifact, error.
6. Else fall through to the native `cargo build --release` flow.

Native build flow:

```bash
cargo build --release          # in the plugin source directory
# locate target/release/xfetch-plugin-<name> in the crate or any ancestor
# copy atomically to <config>/xfetch/plugins/xfetch-plugin-<name>
```

The copy is atomic (`atomic_fs::copy_atomic`) and creates the destination
directory if needed. No source, README, asset or config is copied. This path
requires `git` (remote clone) and `cargo` on the user's machine; a prebuilt
wasm URL or a checked-in artifact does not.

Wasm install flow:

```text
copy artifact           -> <config>/xfetch/plugins/xfetch-plugin-<name>.wasm
copy source manifest    -> <config>/xfetch/plugins/xfetch-plugin-<name>.json
```

The sidecar is what the runtime resolves first. If `--sha256` was passed, the
normalized digest is also persisted into the installed sidecar. A manifest
`sha256` is verified too, but is same-origin integrity only. When neither
source supplies a digest, a downloaded artifact still installs with an
unverified-integrity warning; a mismatch aborts before any file is written.

After any successful install, the CLI prints the enabling config snippet,
the manifest limits (wasm) and the capability summary. Native plugins print
the two possible snippets (info and logo) because no manifest declares
their kind.

`xfetch plugin remove <name>` deletes all three possible files in
`<config>/xfetch/plugins/`: native binary, `.wasm`, and `.json` sidecar.

## Runtime flow

Info plugin (`xfetch/src/plugins/run.rs`):

```text
config.info_plugins[n] -> run_info_plugin
  build InfoPluginRequest { version: 1, kind: "info_provider", args }
  serialize to JSON
  resolve binary
  if wasm: wasm::run_request(..., GuestKind::Plugin)
  else:   spawn with stdin payload and deadline
  parse InfoPluginResponse { lines }
  return lines for rendering under module `plugin:<name>`
```

Logo animation plugin:

```text
config.logo_animation -> run_logo_animation_plugin
  build LogoAnimationRequest {
    version: 1, kind: "logo_animation",
    lines: current ASCII logo lines,
    frames: optional frames loaded by the core,
    args: { fps, duration_ms, loop, style }
  }
  run as above
  parse LogoAnimationResponse { frames }
  validate frames is non-empty
  play frames
```

`loop` is advisory: `animate-logo` ignores it and returns a finite frame
set; daemon mode in the core repeats the animation. `frames_path` (single
file with `===` separators, or an array of files) is loaded by the core and
passed in `frames`; the plugin does not read files for this.

Native timeout semantics (`guest_timeout`):

- `timeout_secs` unset: 30 s safety cap.
- `timeout_secs = 0`: cap disabled.
- `timeout_secs = N`: N seconds.
- On expiry the whole descendant tree is terminated on Windows
  (`taskkill /T`); on Unix the direct child is killed. Output pipes are
  drained with a 500 ms grace period so grandchildren cannot hang the
  fetch.

Wasm timeout semantics (`wasm::runner::run_request`):

- `timeout_secs` set: exactly that duration, overriding the manifest
  `timeout_ms` (it is not clamped to it).
- `timeout_secs` unset: manifest `limits.timeout_ms`, default 30000 ms.
- Gotcha: wasm passes `Some(Duration::from_secs(0))` when
  `timeout_secs = 0`, which is an immediate deadline. Unlike native, `0` does
  not disable the wasm cap.

Plugin errors: the core returns the trimmed stderr as the error text; an
empty stderr becomes `Plugin exited with error`. A JSON parse failure becomes
`Failed to parse plugin output`. Info plugin failures are non-fatal to the
fetch; the module is omitted/falls back according to the core renderer.

## Key files

| Path | Responsibility |
| --- | --- |
| `Cargo.toml` | Workspace members and shared crate versions |
| `.cargo/config.toml` | Forces git CLI for Cargo git fetches |
| `docs/README.md` | Authoring guide: naming, discovery, protocol, wasm, contribution |
| `docs/compatibility.md` | Platform matrix (stale for `banner`, `gauges`, `langs` on `main`) |
| `CONTRIBUTING.md` | Workflow; requires `with_timeout` with a `const BUDGET` |
| `ci/unix.sh`, `ci/windows.ps1` | Test + `grep with_timeout plugins/*/src/main.rs` |
| `scripts/ci.sh`, `scripts/ci.ps1` | fmt + clippy + test (no timeout grep) |
| `scripts/ci-windows.sh` | Cross-target check/clippy for Windows |
| `scripts/ci-wasm.sh` | Builds wasm-crypto, wasm-ip-geo, wasm-pacman, wasm-proc when toolchains exist |
| `plugins/animate-logo/src/main.rs` | Reference logo animation implementation |
| `plugins/banner-text/src/main.rs` | Reference info provider implementation |
| `plugins/wasm-crypto/` | Reference Rust core-module wasm plugin with HTTP |
| `plugins/wasm-ip-geo/` | Reference Python component with typed HTTP |
| `plugins/wasm-pacman/` | Reference Go `wasmimport`/`wasmexport` host bridge |
| `plugins/wasm-proc/` | Reference freestanding C WASI guest and fs preopen |

## Cross-repo links

| Repository | Local clone | What matters for plugins |
| --- | --- | --- |
| `xfetch-cli/xfetch` | `../xfetch` | `src/plugins/` (install, run, manage, discovery), `src/addons/` (catalog, model, report), `src/wasm/` (manifest, policy, install, runner), `src/subprocess.rs` (timeouts), `docs/PLUGINS.md`, `docs/WASM.md` |
| `xfetch-cli/api` | `../api` | `crates/plugin-api` (`xfetch-plugin-api 0.2.0`), `crates/guest-api`, `wit/xfetch-runtime.wit`, `docs/protocol.md`, `docs/timeouts.md` |
| `xfetch-cli/effects` | `../effects` | Same addon machinery; `xfetch-effect-*` naming and `xfetch-effect.json` manifests |
| `xfetch-cli/extensions` | `../extensions` | Same addon machinery; config providers, `xfetch-extension-*`, `xfetch-extension.json` |
| `xfetch-cli/themes` | `../themes` | Registry consumed by the `theme-manager` plugin (`index.json`) |
| `xfetch-cli/assets` | `../assets` | README banner `logo/banner/xfetch.svg` |

Absolute URLs:

- https://github.com/xfetch-cli/plugins
- https://github.com/xfetch-cli/xfetch
- https://github.com/xfetch-cli/api
- https://github.com/xfetch-cli/effects
- https://github.com/xfetch-cli/extensions
- https://github.com/xfetch-cli/themes
- https://github.com/xfetch-cli/assets

## Invariants

- Every official plugin directory name is also the user-facing plugin name
  (`xfetch plugin install <name>`).
- Native package names are exactly `xfetch-plugin-<name>`; edition 2024.
- Rust plugins read one request and write one response; diagnostics go to
  stderr; non-zero exit on failure.
- Rust plugins wrap all work, including stdin parsing, in `with_timeout`
  with a `const BUDGET`. Local probes use 2-3 s, network calls 15-25 s.
- Info providers return `{"lines":[...]}`; logo animation returns a non-empty
  `{"frames":[...]}`.
- Wasm manifests are deny-by-default; `capabilities: {}` grants nothing.
- Wasm `http` patterns glob the full URL; `exec` patterns match the program
  file name and never invoke a shell.
- Installed wasm always has a sidecar `.json`; native plugins have none.
- `index.json` is a discovery aid only. It cannot grant capabilities; the
  installer re-reads the source manifest.
- Do not hardcode the repository root into a plugin: use config args and the
  documented `~` expansion where a plugin supports it.

Last verified: 2026-11-03 against `plugins@76f94dd39bab21ceb0b3295e78db4cb5077145ed` (`main`).
