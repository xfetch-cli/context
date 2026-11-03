# xfetch (core)

The main binary and runtime of the xfetch ecosystem. It gathers system
information, renders it with a configurable layout/logo/theme, and hosts the
plugin, effect, extension and WebAssembly addon machinery.

Last verified: 2026-11-03 against `xfetch@5fb26a316ecbd3f170369ee63a8f1864aa6e7398`
(`main`, tag `v1.0.2`). `release/v1.0.2` is at `22cd5ef` and was merged.

## Repository identity

| Item | Value |
| --- | --- |
| Local clone | `~/repos/xfetch-cli/xfetch` |
| Remote | `https://github.com/xfetch-cli/xfetch.git` |
| Default branch | `main` |
| Working branch | `main` (tag `v1.0.2`); `release/v1.0.2` kept |
| Package | `xfetch-cli` 1.0.2, binary `xfetch` |
| Edition | Rust 2024 |
| License | MIT |

## Role in the ecosystem

- Owns the `xfetch` binary, the info pipeline, the renderer, the config model,
  the installers/updater, and the addon runtime.
- Consumes the SDK crates from `xfetch-cli/api`:
  `xfetch-plugin-api 0.2.0`, `xfetch-effect-api 0.2.0`,
  `xfetch-extension-api 0.2.0`. API 0.3.0 (including `xfetch-api-common`) is
  merged and published to crates.io; `release/v1.0.3` (PR #200) adopts it.
  Core 1.0.2 was validated against 0.3.0 locally.
- Consumes the catalogs from `xfetch-cli/plugins`, `effects` and `extensions`
  for `search`/`available`.
- Consumes the ASCII art catalog from `xfetch-cli/logos` for `--gen-config`.
- Distributes prebuilt binaries through GitHub Releases and
  `xfetch-cli/homebrew-tap`; installers live in this repo.

## Quick start

```bash
cargo build --release --locked
./target/release/xfetch
./target/release/xfetch --version      # xfetch-cli 1.0.2 on this branch
```

Local CI mirrors the release gate:

```bash
bash scripts/ci.sh                     # fmt + clippy -D warnings + tests
bash scripts/ci-windows.sh             # cross-target check + clippy
cargo audit --deny warnings
```

## Features and defaults

`Cargo.toml` has two default features:

- `wasm`: wasmtime/wasmtime-wasi + anyhow/base64/url; enables wasm addons.
- `update`: flate2/semver/tar; enables the self-updater.

Build `--no-default-features` for a minimal binary without wasm or update.

## Documentation index

- [architecture.md](architecture.md): module map, startup flow, data flow,
  platform layers, renderer, update/cache/subprocess infrastructure.
- [cli.md](cli.md): every command and flag, aliases, addon commands,
  `--list-layouts`, wasm tooling, update.
- [info-modules.md](info-modules.md): `Info` fields, module keys, collection
  pipeline, per-platform probes, package counting (including xpm/xp/xpm-aur),
  caching and formatting.
- [addons.md](addons.md): plugin/effect/extension model, directories,
  manifests, native vs wasm, install/list/info/search/remove, catalogs,
  security boundaries.
- [wasm.md](wasm.md): WIT contract, manifest/policy, engine, host calls,
  runners, limits, wasm install and tests.
- [operations.md](operations.md): build/test/audit, CI workflows, release
  process, installers, versioning, branch/release state and gotchas.

## Key paths

| Path | Purpose |
| --- | --- |
| `src/main.rs` | CLI entry, command dispatch, top-level run flow. |
| `src/cli.rs` | clap definitions for every command/flag. |
| `src/config.rs` | Config model, JSONC loading, theme merge, `--gen-config`. |
| `src/info/` | Info struct and all platform probes. |
| `src/ui/` | Layouts, rendering, logos, daemons, effects playback. |
| `src/addons/` | Shared addon model, installs, catalogs, reports. |
| `src/plugins/`, `src/effects/`, `src/extensions/` | Per-kind runtimes. |
| `src/wasm/` | Manifest/policy/engine/host/runner implementation. |
| `src/update/` | Self-updater and release asset verification. |
| `src/subprocess.rs` | Timeout/pipe/process-tree machinery shared by probes and addons. |
| `wit/xfetch-runtime.wit` | Vendored copy of the canonical api WIT. |
| `tests/wasm_e2e.rs` | End-to-end wasm tests. |
| `docs/` | User-facing reference (installation, config, layouts, addons, wasm). |

## Current state (2026-11-03)

- PR #198 was merged into `main` at `5fb26a3`; tag `v1.0.2` points at the
  merge commit. The release workflow publishes the six target assets.
- v1.0.2 includes the security/dependency update, addon catalog search/info
  tooling, the xpm/xp/xpm-aur package counts, `--list-layouts`, and layout
  help fixes.
- The addon catalogs (`index.json`) are on `main` after PRs #17/#7/#10, so
  remote `search` works.
- `release/v1.0.3` (PR #200, `dde16cb`) adopts SDK 0.3.0 and adds crates.io
  metadata. It is not merged or tagged yet.
- crates.io `xfetch-cli` is still 1.0.1; GitHub Releases have v1.0.2. The
  definitive crates.io publish should use the v1.0.3 release.
