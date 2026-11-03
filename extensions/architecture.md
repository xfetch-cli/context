# extensions architecture

Deep structural notes for `xfetch-cli/extensions`. Checked against the working
tree at `9251eecff5a48c403955ae3b457093f9088df80f` and the core
implementation in the sibling `xfetch` clone.

## Position in the ecosystem

```text
xfetch-cli/xfetch (core: config loader -> extensions -> renderer)
        |
        | stdin config JSON -> stdout modified config JSON
        v
xfetch-cli/extensions (this repo: 5 config providers)
        |
        | uses
        v
xfetch-cli/api (xfetch-extension-api 0.2.0)
```

The core loads plugins, effects and extensions through the same
`src/addons/` layer for install/list/info/remove, but execution is separate:
plugins run during rendering, effects run over rendered lines, extensions run
during `load_config` before rendering.

## Repository tree

```text
extensions/
|-- Cargo.toml                 workspace: 3 members, resolver = 2
|-- Cargo.lock                 committed lockfile (version 4)
|-- README.md
|-- CHANGELOG.md
|-- CONTRIBUTING.md
|-- CODE_OF_CONDUCT.md, LICENSE, SECURITY.md, SUPPORT.md
|-- docs/
|   `-- compatibility.md
|-- extensions/
|   |-- config-roulette/       Cargo member
|   |-- layout-override/       Cargo member
|   |-- wasm-night-mode/       Cargo member + xfetch-extension.json
|   |-- wasm-lang-labels/      Python component; not a Cargo member
|   `-- wasm-updates-footer/   Go core module; not a Cargo member
|-- scripts/
|   |-- ci.sh                  fmt + clippy + test + with_timeout grep
|   |-- ci.ps1                 Windows equivalent
|   |-- ci-windows.sh          cross check/clippy for x86_64-pc-windows-gnu
|   `-- ci-wasm.sh             builds all three wasm examples when possible
`-- .github/
    |-- workflows/security-audit.yml
    `-- dependabot.yml
```

On `origin/release/v1.0.2` the tree additionally contains `index.json` and
`scripts/generate_catalog.py`. They are absent from `main` at the verified
HEAD.

## Workspace layout

```toml
[workspace]
members = [
  "extensions/layout-override",
  "extensions/config-roulette",
  "extensions/wasm-night-mode",
]
resolver = "2"

[workspace.dependencies]
xfetch-extension-api = { version = "0.2.0", package = "xfetch-extension-api" }
```

- All three Rust crates share `extensions/target/`, and each uses
  `xfetch-extension-api` from crates.io. There is no `[patch.crates-io]`.
- `wasm-night-mode` is a normal Cargo crate that can build natively for
  debugging and as a `wasm32-wasip1` guest. Its source uses the same
  `with_timeout` and protocol types in both builds; the wasm build runs the
  task inline and the host enforces the epoch deadline.
- `wasm-lang-labels` (Python) and `wasm-updates-footer` (Go) are not Cargo
  members and are not covered by `cargo test --workspace` or the
  `with_timeout` grep in `scripts/ci.sh`.

## Extension directory conventions

Native Rust extension:

```text
extensions/<name>/
|-- Cargo.toml        [package] name = "xfetch-extension-<name>", edition = "2024"
|-- src/main.rs       reads one request from stdin, writes one response to stdout
|-- README.md
`-- CHANGELOG.md      optional; present in all five official extensions
```

There is no sidecar manifest for native extensions. The core therefore cannot
report a version, description, kind or capabilities for them; `info` falls
back to `config_provider` and the native-process capability line.

WebAssembly extension:

```text
extensions/<name>/
|-- xfetch-extension.json   source manifest, copied next to the installed wasm
|-- src/main.rs             Rust core module (wasm-night-mode)
|-- app.py                  Python component (wasm-lang-labels)
|-- main.go, go.mod         Go core module (wasm-updates-footer)
|-- dist/                   build output, gitignored
`-- README.md, CHANGELOG.md
```

The installer recognizes `xfetch-extension.json` and the generic
`extension.json`. A file named `<name>.json` is not a source manifest.

Manifest identity invariants (checked by the extensions catalog generator on
the release branch):

- `name` must equal the directory name.
- `runtime` must be `core` or `component`.
- `version`, `description` and `capabilities` must be present (`capabilities`
  defaults to `{}` in the generator, but the official manifests always
  include it).
- `kind` defaults to `config_provider` if omitted.
- For `wasm-night-mode`, where both `Cargo.toml` and
  `xfetch-extension.json` exist, the two versions must match.

## Native vs wasm

| Aspect | Native extension | Wasm extension |
| --- | --- | --- |
| Artifact | executable | `.wasm` core module or component |
| Detection | not wasm magic | first 8 bytes (`\0asm` + version/layer) |
| Execution | child process, unsandboxed | wasmtime 50.0.1 sandbox |
| Config access | whatever the process can read | only manifest capabilities (`env`, `fs`, `http`, `exec`, `args`) |
| Input/output | stdin/stdout JSON | WASI stdin/stdout for core; typed `run` for components |
| Timeout | `with_timeout` plus core `timeout_secs` (default 30 s, `0` disables) | guest `with_timeout` plus host epoch deadline (`timeout_secs` or manifest `timeout_ms`) |
| Installed names | `xfetch-extension-<name>` / `.exe` | `xfetch-extension-<name>.wasm` + `.json` |

Wasm capability and limit handling is identical to plugins because it is the
same `xfetch/src/wasm/` code. See the plugins protocol book for the full
manifest schema.

## Config-provider lifecycle

`load_config` in `xfetch/src/config.rs` is the only caller of
`run_config_provider`. Order of operations:

1. Determine the config path: explicit `--config`, otherwise
   `<config>/xfetch/config.jsonc`.
2. **If the file does not exist, return `Config::default()` immediately.**
   Extensions do not run at all in this case.
3. Parse the JSONC (comments stripped). On parse failure, return
   `Config::default()`; extensions do not run.
4. Deserialize into `Config`. Unknown or invalid top-level shape falls back to
   defaults (`unwrap_or_default`).
5. If `theme` is set and the theme file resolves, build `merged` by deep
   merging, in this order:
   `Config::default()` -> raw config file value -> theme file value. The
   theme has the highest priority.
6. Clone `config.config_providers` into a local list.
7. Serialize the merged `Config` to a JSON value (`current`).
8. For each provider in declaration order:
   - build `ConfigProviderRequest { version: 1, kind: "config_provider",
     config: current, args: provider.args }`;
   - resolve the extension binary and run it with the deadline;
   - on `Ok(modified)`, set `current = modified`;
   - on any error (not found, spawn failure, timeout, non-zero exit, invalid
     JSON), **ignore it and keep the previous `current`**; processing
     continues with the next provider.
9. Deserialize `current` back into `Config`. If that fails, keep the original
   pre-provider `config` (the theme-merged config), discarding the whole
   provider chain result.
10. Return the final `Config`.

Consequences and edge cases:

- Providers run after theme merge. Setting `theme` from an extension changes
  the field for later consumers but does not re-run theme resolution.
- Providers see the serialized, default-filled config, not the raw partial
  JSON file.
- The provider list is cloned before iteration. An extension that edits
  `config_providers` cannot add, remove or reorder the providers that run in
  the current invocation; the final returned config may still contain the
  edited field.
- Provider errors are silent. There is no stderr forwarding to the user and
  no warning line; a missing or broken extension is a no-op.
- If any provider returns a JSON value that cannot deserialize into `Config`,
  all provider changes are discarded, not just that provider's changes.
- Extensions execute in declaration order and each receives the previous
  provider's output, so a later provider can override an earlier one.
- Extensions only exist for `kind: config_provider`; the API enum currently
  has exactly one kind.

## Discovery

`xfetch/src/extensions/mod.rs` resolves the extension name from each
`config_providers` entry. Exact order for `find_extension_binary(name)`:

1. Each config search directory (`XDG_CONFIG_HOME` when absolute, then the
   platform config dir):
   - `<config>/xfetch/extensions/xfetch-extension-<name>`
   - `<config>/xfetch/extensions/xfetch-extension-<name>.wasm`
   - `<config>/xfetch/plugins/xfetch-extension-<name>`
   - `<config>/xfetch/plugins/xfetch-extension-<name>.wasm` (legacy
     fallback)
2. `PATH`, looking for the native name and the `.wasm` name.
3. Development-time wasm targets, walking up from the current working
   directory: `extensions/target/wasm32-wasip1/release/<wasm>`,
   `extensions/target/wasm32-wasi/release/<wasm>`,
   `extensions/extensions/target/wasm32-wasip1/release/<wasm>` and the
   `wasm32-wasi` equivalent, under each ancestor.

Notable differences from plugins:

- Config directories are searched before `PATH`.
- There is no native `target/release` development lookup. A native extension
  built in the workspace is not discovered from `target/`; install it into
  the extensions directory or put it on `PATH`.
- `xfetch extension info <name>` first checks the default install directory
  (`AddonKind::installed_path`) and only then falls back to
  `find_extension_binary`.

## Install flow

`xfetch extension install <source>` (`xfetch/src/extensions/install.rs`)
dispatches in this order:

1. **HTTP(S) URL**: prebuilt wasm download. The name is the last URL
   segment with query/fragment and `.wasm` removed, validated with
   `validate_guest_name` (`[A-Za-z0-9._-]`). This URL path does **not**
   strip the `xfetch-extension-` prefix; pass `--name <name>` to override
   (which switches to `addons::install::install_direct` and its
   prefix-stripping `infer_name`). `--sha256` is supported.
2. **Existing local file**: wasm header -> install as wasm. Non-wasm file
   plus `--sha256` -> hard error.
3. **Local directory**: resolve `path`, `cwd/path`, `cwd/extensions/path`,
   `cwd/extensions/extensions/path`, `parent/extensions/path`,
   `parent/extensions/extensions/path`.
4. **Remote repository**: clone `--repo <url>` or the default
   `https://github.com/xfetch-cli/extensions.git` with
   `git clone --depth 1 -- <repo> <temp>/repo`. The extension directory is
   looked up at `<clone>/<name>`, `<clone>/extensions/<name>` and
   `<clone>/extensions/extensions/<name>`.

Unlike plugins, there is **no `XFETCH_EXTENSION_REPO` environment override**.
The repository comes from `--repo` or the constant `DEFAULT_EXTENSION_REPO`.
`XFETCH_EXTENSION_INDEX` only overrides the catalog used by `search`.

A bare native executable is not an accepted install source; pass the source
directory or copy the binary into `<config>/xfetch/extensions/` manually.

Local wasm sources use the same `wasm::install::plan` as plugins:

1. manifest `artifact` if present;
2. conventional `dist/<name>.wasm`, `<name>.wasm`,
   `xfetch-extension-<name>.wasm`, other label variants, and Cargo wasm
   target directories;
3. manifest `artifact_url`;
4. manifest `build` (run unsandboxed after printing the command);
5. error if the manifest declares wasm but no artifact exists;
6. otherwise the native Cargo flow.

Native build flow:

```bash
cargo build --release          # in the extension source directory
# locate target/release/xfetch-extension-<name> in the crate or an ancestor
# copy atomically to <config>/xfetch/extensions/xfetch-extension-<name>
```

Wasm install flow:

```text
copy artifact  -> <config>/xfetch/extensions/xfetch-extension-<name>.wasm
copy manifest  -> <config>/xfetch/extensions/xfetch-extension-<name>.json
```

Native installs require `git` (remote clone) and `cargo` on the user's
machine. Prebuilt wasm URLs and checked-in artifacts do not.

The installed sidecar is authoritative at runtime. A CLI `--sha256` is
persisted into it when supplied. A manifest `sha256` is same-origin
integrity only; a digest obtained from a separate channel is the provenance
pin. Mismatches abort before any file is written.

`xfetch extension remove <name>` deletes the native binary, the `.wasm` and
the `.json` sidecar from the default extensions directory. `list` scans only
that directory and skips `.json` files, so sidecars do not appear as
extensions.

## Runtime flow

```text
load_config
  -> current = serde_json::to_value(config)
  -> run_config_provider(provider, current)
       request = ConfigProviderRequest {
         version: 1,
         kind: "config_provider",
         config: current,
         args: provider.args
       }
       payload = serde_json::to_vec(request)
       path = find_extension_binary(provider.extension)
       if wasm: wasm::run_request(path, payload, timeout, GuestKind::Extension)
       else:    run_cmd_with_stdin_timeout(path, [], payload, timeout)
       parse ConfigProviderResponse { config }
       return response.config
```

Native timeout semantics:

- unset `timeout_secs`: 30 s default;
- `0`: cap disabled;
- `N`: N seconds;
- on expiry the whole descendant tree is terminated on Windows; on Unix the
  direct child is killed.

Wasm timeout semantics:

- unset: manifest `limits.timeout_ms` (default 30000);
- set: exactly `timeout_secs`, overriding the manifest (not clamped);
- gotcha: `timeout_secs: 0` yields a zero-length wasm deadline, unlike the
  native path where `0` disables the cap.

Extension errors (missing binary, non-zero exit, timeout, invalid JSON) are
caught by `load_config` and ignored. There is no user-visible diagnostic from
the core for a failed provider. Test extensions with a direct stdin pipe to
see their stderr.

## Key files

| Path | Responsibility |
| --- | --- |
| `Cargo.toml` | Workspace members and `xfetch-extension-api 0.2.0` |
| `docs/compatibility.md` | Linux/macOS/Windows matrix for the five extensions |
| `CONTRIBUTING.md` | Native/wasm contribution rules; `with_timeout` standard |
| `scripts/ci.sh` | fmt + clippy + test + `with_timeout` grep |
| `scripts/ci.ps1` | Windows equivalent |
| `scripts/ci-windows.sh` | Cross-target check/clippy |
| `scripts/ci-wasm.sh` | Builds `wasm-night-mode`, `wasm-lang-labels`, `wasm-updates-footer` when toolchains exist |
| `extensions/config-roulette/src/main.rs` | Reference native config provider with file reads |
| `extensions/layout-override/src/main.rs` | Minimal pure-JSON config transform |
| `extensions/wasm-night-mode/src/main.rs` | Rust core-module extension, native + wasm |
| `extensions/wasm-lang-labels/app.py` | Python component with `env` capability |
| `extensions/wasm-updates-footer/main.go` | Go core module using allowlisted `exec` |

## Cross-repo links

| Repository | Local clone | What matters for extensions |
| --- | --- | --- |
| `xfetch-cli/xfetch` | `../xfetch` | `src/config.rs` (`load_config` ordering), `src/extensions/` (runner/install/manage/discovery), `src/wasm/`, `src/subprocess.rs`, `docs/EXTENSIONS.md`, `docs/WASM.md` |
| `xfetch-cli/api` | `../api` | `crates/extension-api` (`ConfigProviderRequest`, `ConfigProviderResponse`, `with_timeout`), `crates/guest-api`, `wit/xfetch-runtime.wit` |
| `xfetch-cli/plugins` | `../plugins` | Sibling addon family; same wasm machinery and installer conventions |
| `xfetch-cli/effects` | `../effects` | Sibling addon family; same addon model |
| `xfetch-cli/themes` | `../themes` | Theme files merged before providers run |

Absolute URLs:

- https://github.com/xfetch-cli/extensions
- https://github.com/xfetch-cli/xfetch
- https://github.com/xfetch-cli/api
- https://github.com/xfetch-cli/plugins
- https://github.com/xfetch-cli/effects
- https://github.com/xfetch-cli/themes

## Invariants

- Every extension directory name is the user-facing extension name.
- Native package names are exactly `xfetch-extension-<name>`; edition 2024.
- An extension writes exactly one JSON object to stdout and nothing else;
  diagnostics go to stderr. Extra stdout breaks parsing.
- Rust extensions wrap all work, including stdin parsing, in
  `with_timeout`; the native budgets are 2 s for local transforms.
- Providers run in declaration order after the theme merge. The list is
  cloned before iteration.
- Provider failures are non-fatal and silent; the core keeps the previous
  config.
- The final provider output must still deserialize into `Config`, or every
  provider's changes are discarded.
- Native extensions are unsandboxed and can read/write any path the user can.
  Wasm extensions are deny-by-default and only get the manifest allowlists.
- `index.json` is discovery metadata only; it cannot grant capabilities.
- Do not assume a provider can add another provider at runtime. It cannot
  affect the current chain.

Last verified: 2026-11-03 against `extensions@9251eecff5a48c403955ae3b457093f9088df80f` (`main`).
