# plugins

Official plugin repository for the XFetch ecosystem.

## Role

`xfetch-cli/plugins` owns the official plugin implementations, their source
manifests, the shared authoring guide (`docs/README.md`), the platform matrix
(`docs/compatibility.md`) and the plugin-facing CI scripts. The runtime that
spawns plugins, resolves their binaries, enforces timeouts and installs them
lives in the core repository `xfetch-cli/xfetch`; the wire types live in
`xfetch-cli/api`.

A plugin is a standalone executable (native) or a WebAssembly guest. It reads
exactly one JSON request from stdin and writes exactly one JSON response to
stdout. There is no in-process linkage and no shared library ABI.

## Location

| Item | Value |
| --- | --- |
| Local clone | `~/repos/xfetch-cli/plugins` |
| Remote (fetch/push) | `https://github.com/xfetch-cli/plugins.git` |
| Default branch | `main` (`origin/HEAD -> origin/main`) |
| HEAD at verification | `56379bae0319a828d3bfbac02fe8876a2c4ea141` |
| HEAD date | 2026-11-03 |
| HEAD subject | `Merge pull request #17 from xfetch-cli/release/v1.0.2` |
| Tags | none |
| License | MIT (`LICENSE`, Copyright 2026 xscriptor) |

The `release/v1.0.2` branch (`4e02244`) was merged into `main` and is kept
for history. `main` now contains the root `index.json` catalog and
`scripts/generate_catalog.py`; see [operations.md](operations.md#catalog).

`main` is the install source for `xfetch plugin install <name>` (the core
clones the repository default branch) and for the hardcoded catalog URL
`https://raw.githubusercontent.com/xfetch-cli/plugins/main/index.json`.

## Toolchain

| Concern | Requirement |
| --- | --- |
| Rust | Stable toolchain; edition 2024; workspace `resolver = "2"` |
| MSRV | Not pinned (no `rust-version` in any `Cargo.toml`) |
| Workspace | `~/repos/xfetch-cli/plugins/Cargo.toml` |
| Shared Rust crates | `xfetch-plugin-api 0.2.0`, `xfetch-guest-api 0.2.0` (crates.io) |
| Rust wasm target | `wasm32-wasip1` (legacy `wasm32-wasi` paths are also probed at development time) |
| Python component | Python 3 and `componentize-py >= 0.25` (`plugins/wasm-ip-geo/requirements.txt`) |
| Go wasm | Go 1.24+ (`plugins/wasm-pacman/go.mod`; `GOOS=wasip1 GOARCH=wasm`) |
| C wasm | `clang` with the WebAssembly backend and `llvm` `wasm-ld` (`plugins/wasm-proc/build.sh`) |
| Git | Required by remote installs (`git clone --depth 1 --`) |

Verified locally on 2026-11-03 with rustc/cargo 1.98.1, Python 3.14.9,
Go 1.27.3 and clang 23.1.4. `componentize-py` was not installed in the
verification environment, so the Python component build was not executed.

## Plugin inventory

Twenty plugin directories exist. Seventeen are Cargo workspace members
(including `wasm-crypto`); the remaining three are non-Cargo WebAssembly
examples (`wasm-ip-geo`, `wasm-pacman`, `wasm-proc`). `banner`, `gauges` and
`langs` are documented in `main` after the v1.0.2 merge.

Versions below are from `main` at `56379ba`: `gauges` and `theme-manager` are
`0.2.1` (with `sysinfo 0.40.1` and `dirs 7` respectively); every other crate
is unchanged.

| Plugin | Version | Kind | Runtime | Language / build | Platform | Summary |
| --- | --- | --- | --- | --- | --- | --- |
| `animate-logo` | 0.2.0 | `logo_animation` | native | Rust | Linux, macOS, Windows | Sweep, wave, rainbow, sparkle, breathing and frame animations over the logo lines. |
| `banner` | 0.2.0 | `logo_animation` | native | Rust | Linux, macOS, Windows | Appends or prepends a centred Chinese/Japanese proverb banner; needs a CJK-capable TTY font. |
| `banner-text` | 0.1.0 | `info_provider` | native | Rust | Linux, macOS, Windows | Emits text lines from a file or literal; works with image logos. |
| `chocolatey` | 0.2.0 | `info_provider` | native | Rust | Windows only | Counts packages with `choco list -r`; graceful fallback elsewhere. |
| `display-resolution` | 0.2.0 | `info_provider` | native | Rust | Linux, macOS, Windows | Monitor resolution/refresh via `xrandr`, `wlr-randr`, `xdpyinfo`, `system_profiler`, PowerShell. |
| `docker` | 0.2.0 | `info_provider` | native | Rust | Linux, macOS, Windows | Container totals/running/paused/stopped via the Docker CLI. |
| `gauges` | 0.2.0 (0.2.1 on release) | `info_provider` | native | Rust + `sysinfo 0.39` | Linux, macOS, Windows | Memory, swap, disk and battery progress bars; battery is Linux-only. |
| `github-stats` | 0.2.0 | `info_provider` | native | Rust + `curl` | Linux, macOS, Windows | Stars, repos, PRs, issues and followers from the GitHub API. |
| `langs` | 0.2.0 | `info_provider` | native | Rust + `curl` | Linux, macOS, Windows | GitHub repository language bytes or owner language aggregation, with brand colours. |
| `music-player` | 0.2.0 | `info_provider` | native | Rust | Linux, macOS | MPD (`mpc`) and Spotify (`playerctl`); Windows reports no active player. |
| `temperature` | 0.2.0 | `info_provider` | native | Rust | Linux, Windows (partial) | Linux thermal zones; Windows WMI ACPI zones; macOS unsupported. |
| `theme-detection` | 0.2.0 | `info_provider` | native | Rust | Linux, Windows | GTK/KDE settings on Linux; Windows registry light/dark plus accent. |
| `theme-manager` | 0.2.0 (0.2.1 on release) | `info_provider` | native | Rust + `curl` | Linux, macOS, Windows | List/search/info/install themes from the `xfetch-cli/themes` registry. |
| `timezone` | 0.2.0 | `info_provider` | native | Rust | Linux, macOS, Windows | Local time, timezone id and UTC offset; `format` is ignored on Windows. |
| `user-info` | 0.2.0 | `info_provider` | native | Rust | Linux, macOS, Windows | User name, UID/GID, home, shell and optional groups. |
| `weather` | 0.2.0 | `info_provider` | native | Rust + `curl` | Linux, macOS, Windows | Current conditions from wttr.in. |
| `wasm-crypto` | 0.2.0 | `info_provider` | wasm (`core`) | Rust -> `wasm32-wasip1` | Linux, macOS, Windows (wasm-enabled) | BTC/ETH/SOL spot prices through allowlisted Coinbase HTTP. |
| `wasm-ip-geo` | 0.2.0 | `info_provider` | wasm (`component`) | Python + `componentize-py` | Linux, macOS, Windows (wasm-enabled) | Public IP, location, network and timezone from ipapi.co. |
| `wasm-pacman` | 0.2.0 | `info_provider` | wasm (`core`) | Go -> `wasip1` | Arch-family Linux | Repository vs AUR/foreign package counts via allowlisted `pacman` exec. |
| `wasm-proc` | 0.2.0 | `info_provider` | wasm (`core`) | freestanding C -> `wasm32-wasip1` | Linux, WSL | Load average, uptime and memory from a read-only `/proc` preopen. |

Native binaries are named `xfetch-plugin-<name>` (`xfetch-plugin-<name>.exe`
on Windows). Installed wasm artifacts are named
`xfetch-plugin-<name>.wasm` with the sidecar `xfetch-plugin-<name>.json`.

## Quick start

End user:

```bash
# Install by name from the default repository.
# Native installs clone and build, so they need git + a Rust toolchain.
xfetch plugin install weather

# Inspect what is installed and whether it is enabled.
xfetch plugin list --verbose
xfetch plugin info weather

# Catalog search (index.json is on main).
xfetch plugin search crypto
xfetch plugin available --json
```

Enabling an info provider requires both an `info_plugins` entry and the
`plugin:<name>` module key:

```jsonc
{
  "info_plugins": [
    { "plugin": "weather", "args": { "location": "London" }, "timeout_secs": 20 }
  ],
  "modules": ["os", "kernel", "plugin:weather", "shell", "cpu"],
  "colors": { "plugin:weather": "Cyan" },
  "icons":  { "plugin:weather": "" }
}
```

Enabling a logo animation uses `logo_animation`:

```jsonc
{
  "logo_animation": {
    "plugin": "animate-logo",
    "style": "sweep",
    "fps": 12,
    "duration_ms": 1200,
    "loop": false
  }
}
```

The installer prints the exact snippet and the declared wasm capabilities and
limits. A native plugin has no sidecar manifest, so `info` shows the fallback
kind and a "Native process" capability line.

Contributor:

```bash
cd ~/repos/xfetch-cli/plugins

# Build and test all Rust workspace members.
cargo test --workspace
cargo clippy --all-targets -- -D warnings

# Local CI (fmt, clippy, tests).
bash scripts/ci.sh

# Full plugin-standard CI: tests plus the with_timeout grep.
bash ci/unix.sh

# Build the wasm examples with whichever toolchains are installed.
bash scripts/ci-wasm.sh
```

## Document index

- [architecture.md](architecture.md) - directory layout, native vs wasm,
  discovery, install and runtime flow, key files, cross-repo links.
- [protocol.md](protocol.md) - JSON wire protocol, manifests, capabilities,
  limits, wasm guest contract and examples.
- [operations.md](operations.md) - build/test/CI, catalog generation,
  release, adding a plugin, lockfile/catalog gotchas and troubleshooting.

Repository-local references: `README.md`, `docs/README.md`,
`docs/compatibility.md`, `CONTRIBUTING.md`, `CHANGELOG.md`, `ROADMAP.md`,
`SECURITY.md`, `SUPPORT.md`.

Last verified: 2026-11-03 against `plugins@56379bae0319a828d3bfbac02fe8876a2c4ea141`
(`main`); `release/v1.0.2` is at `4e02244` and was merged into `main`.
