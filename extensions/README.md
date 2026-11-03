# extensions

Official extension repository for the XFetch ecosystem.

## Role

`xfetch-cli/extensions` owns the official config-provider extensions. An
extension runs at **config load time**, after the default/file/theme merge:
it receives the fully resolved config as JSON on stdin and returns a modified
config as JSON on stdout. It can change layouts, modules, colours, icons,
labels, footer text, `show_colors`, `palette_style` or any other config field.
It cannot make the core re-run theme resolution.

The core runtime (discovery, install, ordering, timeouts) lives in
`xfetch-cli/xfetch`; the wire types live in `xfetch-cli/api` as
`xfetch-extension-api 0.2.0`. Native extensions are standalone executables;
wasm extensions are sandboxed guests.

## Location

| Item | Value |
| --- | --- |
| Local clone | `~/repos/xfetch-cli/extensions` |
| Remote (fetch/push) | `https://github.com/xfetch-cli/extensions.git` |
| Default branch | `main` (`origin/HEAD -> origin/main`) |
| HEAD at verification | `3ed8e76a52298c2ddd6d6a3c5374764001b8b941` |
| HEAD date | 2026-11-03 |
| HEAD subject | `Merge pull request #10 from xfetch-cli/release/v1.0.2` |
| Tags | none |
| License | MIT (`LICENSE`, Copyright 2026 xscriptor) |

The `release/v1.0.2` branch (`5bf664f`) was merged into `main` and is kept.
`main` contains the root `index.json` catalog and
`scripts/generate_catalog.py`; the catalog URL used by
`xfetch extension search`
(`https://raw.githubusercontent.com/xfetch-cli/extensions/main/index.json`)
returns HTTP 200.

## Toolchain

| Concern | Requirement |
| --- | --- |
| Rust | Stable toolchain; edition 2024; workspace `resolver = "2"` |
| MSRV | Not pinned (no `rust-version` in any `Cargo.toml`) |
| Workspace | `~/repos/xfetch-cli/extensions/Cargo.toml` |
| Shared Rust crate | `xfetch-extension-api 0.2.0` (crates.io) |
| Rust wasm target | `wasm32-wasip1` (legacy `wasm32-wasi` paths are probed for development builds) |
| Python component | Python 3 and `componentize-py >= 0.25` (`extensions/wasm-lang-labels/requirements.txt`) |
| Go wasm | Go 1.24+ (`extensions/wasm-updates-footer/go.mod`; `GOOS=wasip1 GOARCH=wasm`) |
| Git | Required by remote installs (`git clone --depth 1 --`) |

Verified locally on 2026-11-03 with rustc/cargo 1.98.1, Python 3.14.9 and
Go 1.27.3. `componentize-py` was not installed, so the Python component
build was not executed.

## Extension inventory

Five extension directories exist. Three are Cargo workspace members
(`config-roulette`, `layout-override`, `wasm-night-mode`); two are non-Cargo
wasm guests (`wasm-lang-labels`, `wasm-updates-footer`). All crate and
manifest versions are `0.2.0`.

| Extension | Version | Kind | Runtime | Language / build | Platform | Summary |
| --- | --- | --- | --- | --- | --- | --- |
| `config-roulette` | 0.2.0 | `config_provider` | native | Rust | Linux, macOS, Windows | Picks a config from a routes file with `random` or `daily` strategy. |
| `layout-override` | 0.2.0 | `config_provider` | native | Rust | Linux, macOS, Windows | Forces `layout` and/or `modules` after the theme merge. |
| `wasm-night-mode` | 0.2.0 | `config_provider` | wasm (`core`) | Rust -> `wasm32-wasip1` | Linux, macOS, Windows (wasm-enabled) | Disables colours and optionally changes `palette_style` outside a daytime window. |
| `wasm-updates-footer` | 0.2.0 | `config_provider` | wasm (`core`) | Go -> `wasip1` | Arch-family Linux | Appends the pending package-update count to `footer_text`. |
| `wasm-lang-labels` | 0.2.0 | `config_provider` | wasm (`component`) | Python + `componentize-py` | Linux, macOS, Windows (wasm-enabled) | Fills `labels` from `LANG`/`LC_ALL`/`LC_MESSAGES` without overwriting custom labels. |

Native binaries are named `xfetch-extension-<name>` (`.exe` on Windows).
Wasm artifacts are `xfetch-extension-<name>.wasm` with the sidecar
`xfetch-extension-<name>.json`. Installed files live in
`<config>/xfetch/extensions/`: `~/.config/xfetch/extensions/` on Linux/macOS
and `%APPDATA%/xfetch/extensions/` on Windows.

## Quick start

End user:

```bash
# Install by name from the default repository.
# Native installs clone and build, so they need git + a Rust toolchain.
xfetch extension install config-roulette

# Or install a local wasm source.
xfetch extension install ./extensions/wasm-night-mode

# List, inspect and remove.
xfetch extension list --verbose
xfetch extension info config-roulette
xfetch extension remove config-roulette

# Catalog search (requires index.json on the catalog branch).
xfetch extension search night
xfetch extension available --json
```

Activate an extension with `config_providers`. Entries run in declaration
order, after the theme merge:

```jsonc
{
  "config_providers": [
    {
      "extension": "config-roulette",
      "args": {
        "routes": "~/.config/xfetch/routes.json",
        "strategy": "daily"
      },
      "timeout_secs": 5
    },
    {
      "extension": "wasm-night-mode",
      "args": {
        "night_start": 22,
        "night_end": 7,
        "utc_offset": 2,
        "dim_colors": true
      }
    }
  ]
}
```

The installer prints the `config_providers` snippet and, for wasm, the
declared capabilities and limits. A native extension has no sidecar
manifest, so `info` reports the fallback kind `config_provider`, no version,
and the "Native process: runs unsandboxed" capability line.

Contributor:

```bash
cd ~/repos/xfetch-cli/extensions

cargo test --workspace
cargo clippy --all-targets -- -D warnings
cargo fmt --all --check

bash scripts/ci.sh       # fmt + clippy + test + with_timeout grep
bash scripts/ci-wasm.sh  # builds wasm examples when toolchains exist
```

## Document index

- [architecture.md](architecture.md) - workspace layout, config-provider
  lifecycle, native vs wasm, discovery/install, key files, cross-repo links.
- [protocol.md](protocol.md) - config-provider request/response, ordering,
  manifests, capabilities, limits, wasm contract and examples.
- [operations.md](operations.md) - build/test/CI, catalog generation,
  release, adding an extension, troubleshooting.

Repository-local references: `README.md`, `docs/compatibility.md`,
`CONTRIBUTING.md`, `CHANGELOG.md`, `SECURITY.md`, `SUPPORT.md`.

Last verified: 2026-11-03 against
`extensions@9251eecff5a48c403955ae3b457093f9088df80f` (`main`); the catalog
and dependency updates were additionally inspected on
`origin/release/v1.0.2@e32990dd3dd64e585b7feb8dd56de8b05aac3df0`.
