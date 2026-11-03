# xfetch operations

Build, test, release and install workflows for the core repository.

## Toolchain

- Rust edition 2024, stable toolchain. Release CI pins Rust `1.98.1`.
- `Cargo.lock` is committed; builds use `--locked`.
- Default features: `wasm`, `update`. Minimal build:
  `cargo build --release --no-default-features`.
- C toolchain is required to link the Rust binary (and `zstd-sys` when
  cross-checking).

## Local CI

```bash
bash scripts/ci.sh          # fmt --check, clippy -D warnings, cargo test
bash scripts/ci-windows.sh  # cross-target check + clippy (needs mingw for zstd)
cargo audit --deny warnings
```

`scripts/ci.sh` mirrors `rust-tests.yml`. `scripts/ci-windows.sh` uses
`x86_64-pc-windows-gnu`; `cargo check/clippy` normally do not link, but
`zstd-sys`'s build script compiles C and therefore needs `x86_64-w64-mingw32-gcc`
on Linux. Tests still require a real Windows host.

Test inventory on `main` (v1.0.2): ~256 unit tests (1 ignored) and 6 wasm e2e
tests. `cargo test --locked` is the release gate.

## GitHub Actions

| Workflow | Trigger | What it does |
| --- | --- | --- |
| `rust-tests.yml` | push to `main`, PR, manual | `cargo test --locked` on Ubuntu. |
| `security-audit.yml` | push to `main`, PR, weekly cron | installs cargo-audit 0.22.2, runs `cargo audit --deny warnings`. |
| `release.yml` | tag `v*` | gate + 6-target build + checksums + cosign + publish. |
| `roadmap-sync.yml` | manual | `python .github/scripts/sync_roadmap.py --roadmap ROADMAP.md`. |

Release branch note: push triggers are limited to `main`; release branches are
validated through their PRs (tests + audit).

## Release process (`release.yml`)

1. Tag `vX.Y.Z` on a commit that is an ancestor of `origin/main`.
2. Gate job:
   - verifies the tag commit is on `main`;
   - verifies the tag matches `Cargo.toml`'s `[package] version`;
   - runs `cargo test --locked`;
   - runs `cargo audit --deny warnings`.
3. Build job matrix:
   - `x86_64-unknown-linux-gnu` (ubuntu-22.04)
   - `aarch64-unknown-linux-gnu` (ubuntu-24.04-arm)
   - `x86_64-unknown-linux-musl` (ubuntu-latest, musl-tools)
   - `x86_64-apple-darwin` (macos-15-intel)
   - `aarch64-apple-darwin` (macos-latest)
   - `x86_64-pc-windows-msvc` (windows-latest)
4. Package:
   - Unix: `xfetch-<version>-<target>.tar.gz` containing `xfetch`, `LICENSE`,
     `README.md`.
   - Windows: `xfetch-<version>-<target>.zip` via 7-Zip (Git Bash tar cannot
     write zip correctly).
5. `checksums` job validates archives and writes `SHA256SUMS`.
6. `sign` job (OIDC keyless):
   - `cosign sign-blob --bundle SHA256SUMS.bundle SHA256SUMS`;
   - `actions/attest-build-provenance` for archives;
   - uploads the signature bundle.
7. `release` job:
   - verifies the cosign bundle identity/issuer and `sha256sum -c`;
   - publishes the GitHub release with archives, `SHA256SUMS`, and
     `SHA256SUMS.bundle`;
   - requires the `release` environment (configure required reviewers in repo
     settings; YAML cannot enforce it).

Version policy: the wasm signature policy threshold remains v1.0.1; v1.0.2
bumps crate/installer versions and closes the wasmtime/zstd advisories.

## Installers

| Script | Behavior |
| --- | --- |
| `install.sh` | Clones `main` (or `--local` from the current checkout), builds `--release --locked`, installs to `~/.local/bin`, generates config if missing, adds PATH. |
| `install-prebuilt.sh` | Downloads the latest release asset for the target, verifies `SHA256SUMS` (and cosign bundle from `SIGNED_MIN_VERSION=1.0.1`), installs to `~/.local/bin`. `--version X.Y.Z` selects a release. |
| `install.ps1` / `install-prebuilt.ps1` | Windows equivalents. |
| `uninstall.sh` / `uninstall.ps1` | Removes the binary and PATH entry; `--purge` also removes config/data. |

Clean install paths:

- Binary: `~/.local/bin/xfetch`, `~/.cargo/bin/xfetch`, `/usr/local/bin/xfetch`.
- Config and installed addons: `~/.config/xfetch` (or `$XDG_CONFIG_HOME/xfetch`).
- Cache: `~/.cache/xfetch` (or `$XDG_CACHE_HOME/xfetch`).
- Data: `~/.local/share/xfetch` (installer does not create it; uninstaller
  removes it with `--purge`).
- macOS symlink: `~/Library/Application Support/xfetch`.

v1.0.2 is tagged and merged on `main`. Once the release workflow publishes the
assets, `install.sh` and `install-prebuilt.sh` install 1.0.2. To build from the
release branch explicitly:

```bash
git clone --branch release/v1.0.2 https://github.com/xfetch-cli/xfetch.git
cd xfetch
bash install.sh --local --yes
hash -r
xfetch --version   # 1.0.2
```

Or:

```bash
cargo build --release --locked
install -Dm755 target/release/xfetch ~/.local/bin/xfetch
```

## Distribution channels

- GitHub Releases (canonical prebuilt assets).
- `xfetch-cli/homebrew-tap`: `brew tap xfetch-cli/tap && brew install xfetch`
  (macOS prebuilt; formula must be bumped after each tag).
- AUR: `xfetch-core-bin` (precompiled) and `xfetch-git` (source).
- `cargo install xfetch-cli` (crates.io).
- `install.sh --local` for development builds.

## Versioning and branches

- `Cargo.toml` version is the source of truth (`1.0.2` on the release branch).
- Release branches: `release/vX.Y.Z`; PRs carry the release; merge to `main`,
  then tag.
- The release workflow refuses tags not on `main` or whose version differs
  from `Cargo.toml`.
- Addon repos use their own `release/vX.Y.Z` branches and catalogs; keep the
  core version and catalog `catalog_version` aligned with the release train.

## Adding a system probe / module

1. Add the field to `Info` (`src/info/mod.rs`) and fill it in
   `Info::with_config`.
2. Add the platform function to `linux/`, `macos/`, `windows/` as needed.
3. Add the renderer key in `src/ui/nodes.rs` and the module table in
   `docs/CONFIGURATION.md`.
4. Gate expensive work with `collect_module_keys` / `needs_*` helpers.
5. Add unit tests next to the probe and run `scripts/ci.sh`.

## Troubleshooting

- Prebuilt installer 404 / old version: the `v1.0.2` release assets may still
  be building or the installed binary is stale; check the GitHub release and
  `type -a xfetch`, then build from `main`/the release branch if needed.
- Old binary after reinstall: check `type -a xfetch`; remove
  `~/.cargo/bin/xfetch` or fix PATH order; `hash -r`.
- `cargo audit` warnings: update the lockfile/dependencies; the release gate
  uses `--deny warnings`.
- Windows cross-check fails in `zstd-sys`: install mingw (`x86_64-w64-mingw32-gcc`);
  check/clippy need it because the build script compiles C.
- Addon `search` 404: the catalogs are on `main` after PRs #17/#7/#10. A 404
  now usually means a stale `XFETCH_*_INDEX` override or an older checkout.
- Wasm addon denied/blocked: inspect the installed sidecar manifest and the
  `Policy` decisions; `xfetch wasm inspect` does not execute.
