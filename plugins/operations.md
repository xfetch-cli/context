# plugins operations

Build, test, catalog, release and troubleshooting procedures for
`xfetch-cli/plugins`. Commands assume the repository root unless stated
otherwise:

```bash
cd ~/repos/xfetch-cli/plugins
```

## Prerequisites

| Tool | Needed for |
| --- | --- |
| Rust stable (edition 2024) | All 17 Cargo workspace members |
| `wasm32-wasip1` target | `wasm-crypto`, `wasm-night-mode`-style Rust guests |
| Python 3.11+ | `scripts/generate_catalog.py` (`tomllib` is standard library from 3.11) |
| `componentize-py >= 0.25` | `wasm-ip-geo` |
| Go 1.24+ | `wasm-pacman` |
| clang + `wasm-ld` | `wasm-proc` |
| git | Remote installs performed by the core, not by this repo's build |
| `cargo-audit 0.22.2` | Reproducing the security-audit workflow locally |

## Build

Workspace:

```bash
cargo build --workspace
cargo build --release --workspace
```

Individual native plugin:

```bash
cargo build --release -p xfetch-plugin-weather
# artifact: target/release/xfetch-plugin-weather
```

Individual wasm plugin:

```bash
rustup target add wasm32-wasip1
cargo build --release --target wasm32-wasip1 -p xfetch-plugin-wasm-crypto
# artifact: target/wasm32-wasip1/release/xfetch-plugin-wasm-crypto.wasm
```

Python component:

```bash
cd plugins/wasm-ip-geo
python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
mkdir -p dist
componentize-py --quiet -d ../../../api/wit -w plugin \
  componentize app -p . -o dist/wasm-ip-geo.wasm
```

The `../../../api/wit` path assumes the multi-repository checkout layout
(`xfetch-cli/api` next to `xfetch-cli/plugins`).

Go core module:

```bash
cd plugins/wasm-pacman
mkdir -p dist
GOOS=wasip1 GOARCH=wasm go build -o dist/wasm-pacman.wasm .
```

Freestanding C core module:

```bash
cd plugins/wasm-proc
sh build.sh
# writes dist/wasm-proc.wasm
```

All four wasm examples are also built by `scripts/ci-wasm.sh`, which skips
any toolchain it cannot find.

## Test, format, clippy

```bash
# Workspace unit tests. Non-Cargo wasm directories are not included.
cargo test --workspace

# Formatting.
cargo fmt --all --check

# Lints; CI treats warnings as errors.
cargo clippy --all-targets -- -D warnings
```

Verified on 2026-11-03 against the main HEAD:

```text
cargo test --workspace --offline                         PASS
cargo fmt --all --check                                  PASS
cargo clippy --workspace --all-targets --offline -D warnings  PASS
```

Unit tests are concentrated in `banner` (5), `banner-text` (3), `gauges`
(6), `langs` (8) and `wasm-crypto` (2); `animate-logo` and the probe plugins
have no tests because their logic is OS/network I/O. The workspace test run
is still the baseline gate.

## CI scripts and what they actually enforce

There is **no build/test GitHub Actions workflow** in this repository. The
only workflow is the security audit. The checked-in CI is a set of local
scripts; run the one that matches your platform before opening a PR.

| Script | What it runs | What it does not do |
| --- | --- | --- |
| `scripts/ci.sh` | `cargo fmt --all --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test` | Does not enforce `with_timeout` |
| `scripts/ci.ps1` | Same three checks on Windows PowerShell | Does not enforce `with_timeout` |
| `ci/unix.sh` | `cargo test --workspace`, then `grep -q with_timeout plugins/*/src/main.rs` for every Rust plugin | No fmt/clippy; skips non-Rust wasm dirs |
| `ci/windows.ps1` | `cargo test --workspace`, then the same `with_timeout` grep via `Select-String` | No fmt/clippy |
| `scripts/ci-windows.sh` | `cargo check` and `cargo clippy` for `x86_64-pc-windows-gnu` | No tests (need a Windows host), no linking |
| `scripts/ci-wasm.sh` | Builds each wasm example if its toolchain exists; fails only when a present toolchain fails | Does not test the guests |
| `.github/workflows/security-audit.yml` | `cargo audit --deny warnings` on push, PR and weekly (Mondays 04:17 UTC), `cargo-audit 0.22.2`, actions pinned by SHA | No build, test, fmt or clippy |

Important consequences:

- `CONTRIBUTING.md` says every plugin must use `with_timeout`, but only
  `ci/unix.sh` and `ci/windows.ps1` enforce it, and only for
  `plugins/*/src/main.rs`. The Python, Go and C wasm guests have no
  `src/main.rs` and are not checked. `scripts/ci.sh` passes even if a Rust
  plugin lacks `with_timeout`.
- `scripts/ci.sh` changes directory to `scripts/` before invoking Cargo.
  Cargo finds the workspace by walking up, so this works, but the relative
  paths in the script are relative to `scripts/`.
- `scripts/ci-windows.sh` requires the target to be installed first:

  ```bash
  rustup target add x86_64-pc-windows-gnu
  bash scripts/ci-windows.sh
  ```

  `cargo check`/`clippy` do not link, so MinGW is not required.

## Catalog

### Status at the verified HEAD

| Item | `main` (current) |
| --- | --- |
| `index.json` | present, 20 entries, `catalog_version` `1.0.2` |
| `scripts/generate_catalog.py` | present |
| Raw URL `https://raw.githubusercontent.com/xfetch-cli/plugins/main/index.json` | HTTP 200 |
| `xfetch plugin search` against the default URL | works |

The core hardcodes the catalog source as
`https://raw.githubusercontent.com/xfetch-cli/plugins/main/index.json`
(`xfetch/src/addons/catalog.rs`). The v1.0.2 release branch was merged into
`main`, so the default catalog URL works. Installation does not use the
catalog; `xfetch plugin install <name>` clones the repository default branch
and reads the crate/manifest.

To verify the live state:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' \
  https://raw.githubusercontent.com/xfetch-cli/plugins/main/index.json
# 200

git ls-remote origin refs/heads/main refs/heads/release/v1.0.2
# 56379ba... refs/heads/main
# 4e02244... refs/heads/release/v1.0.2
```

### Generate the catalog

The generator is on `main` after the v1.0.2 merge:

```bash
python3 scripts/generate_catalog.py
python3 scripts/generate_catalog.py --check
```

Behavior:

- Scans `plugins/*/` and ignores names starting with `.`.
- A directory with `xfetch-plugin.json` becomes a wasm entry.
- Otherwise, a directory with a `Cargo.toml` whose `[package].name` starts
  with `xfetch-plugin-` becomes a native entry.
- Native entries require `[package].name == "xfetch-plugin-<directory>"`,
  a non-empty `version` and a non-empty `description`.
- Wasm entries require `name == <directory>`, non-empty `kind`, `version`,
  `description`, a `capabilities` object and `runtime` in
  `{"core", "component"}`.
- Kind for native entries is `logo_animation` for the hardcoded set
  `{"animate-logo", "banner"}` and `info_provider` otherwise. A new logo
  plugin must be added to `LOGO_ANIMATION_PLUGINS` or it will be catalogued
  as an info provider.
- Entries are sorted by name; duplicate names are an error; an empty result
  is an error.
- `catalog_version` is hardcoded to `"1.0.2"` in the script. Bump it with the
  core release.
- Output is `json.dumps(..., indent=2, ensure_ascii=False)` plus a trailing
  newline. The generator is deterministic; whitespace or ordering drift is a
  `--check` failure.
- `--check` exits 0 and prints `index.json is up to date (N entries).` when
  identical, otherwise prints `error: index.json is stale; run python3
  scripts/generate_catalog.py` or `error: index.json is missing`.

The generator does **not** compare a wasm `Cargo.toml` version with the
manifest version. For Rust wasm plugins, keep both in sync manually. (The
extensions generator does perform this check.)

### Catalog format

Root document:

```json
{
  "schema_version": 1,
  "catalog_version": "1.0.2",
  "entries": []
}
```

Entry fields consumed by the core (`CatalogEntry`):

| Field | Required | Meaning |
| --- | --- | --- |
| `name` | yes | Plugin name shown in search. |
| `description` | no | Free text. |
| `version` | no | Display only. |
| `kind` | no | `info_provider`, `logo_animation`, ... |
| `runtime` | no | `native` or `wasm`. |
| `path` | no | Repository path, e.g. `plugins/weather`. |
| `install` | no | Name passed to `xfetch plugin install`; defaults to `name`. |
| `guest_runtime` | no | Wasm only: `core` or `component`. |
| `capabilities` | no | Wasm capability object, informational here. |

The core rejects `schema_version != 1`, limits remote catalogs to 2 MiB and
uses a 15 s timeout. `XFETCH_PLUGIN_INDEX` may be a URL or a local file path.
Search matches case-insensitively against name, description, kind and runtime;
`--json` prints the filtered entries.

Catalog entries are discovery metadata only. They cannot grant capabilities;
installation always re-reads the source manifest. Do not put credentials,
absolute local paths or executable commands in `index.json`.

### Local catalog override

To use the release branch catalog without merging:

```bash
XFETCH_PLUGIN_INDEX=https://raw.githubusercontent.com/xfetch-cli/plugins/release/v1.0.2/index.json \
  xfetch plugin search crypto
```

Or point at a checked-out file:

```bash
XFETCH_PLUGIN_INDEX=~/repos/xfetch-cli/plugins/index.json \
  xfetch plugin search crypto
```

## Release

Observed state and conventions:

- There are no tags. Releases are branch-based
  (`release/v1.0.1`, `release/v1.0.2`) and merged through pull requests.
  `main` is the merge of `release/v1.0.2` (`56379ba`, PR #17); the release
  branch (`4e02244`) is kept for history.
- There is no release workflow and no binary publishing in this repository.
  Users install plugins from source (`xfetch plugin install` clones and
  builds), or from a direct `.wasm` URL. The repository publishes source,
  manifests and the catalog on `main`.
- `catalog_version` tracks the core release (`1.0.2`).
- `CHANGELOG.md` records changes by date, not by crate version. Crate
  versions live in each `Cargo.toml`.

Release checklist:

1. Ensure `main` and `release/vX.Y.Z` contain the intended plugin versions.
2. Update each changed plugin's `Cargo.toml` version and `CHANGELOG.md`.
3. Update `plugins/<name>/CHANGELOG.md` when the plugin ships its own file.
4. Update `README.md` and `docs/compatibility.md` for new or changed plugins
   (`banner`, `gauges` and `langs` are listed on `main` for v1.0.2).
5. On the release branch, run `python3 scripts/generate_catalog.py` and
   commit `index.json`; then `--check`.
6. Run `cargo test --workspace`, `cargo clippy --all-targets -- -D warnings`,
   `cargo fmt --all --check`, `bash ci/unix.sh` and, where available,
   `bash scripts/ci-wasm.sh`.
7. Run the security audit (`cargo install cargo-audit --version 0.22.2
   --locked` then `cargo audit --deny warnings`).
8. Merge the release branch to `main`. For v1.0.2 this is done (PR #17).
9. Verify from a clean shell:

   ```bash
   xfetch plugin search weather
   xfetch plugin install weather
   xfetch plugin list --verbose
   ```

## Adding a plugin

Native Rust plugin:

1. Create `plugins/<name>/`.
2. Add `Cargo.toml`:

   ```toml
   [package]
   name = "xfetch-plugin-<name>"
   version = "0.1.0"
   edition = "2024"
   license = "MIT"
   description = "One-line description used by the catalog"

   [dependencies]
   serde = { version = "1", features = ["derive"] }
   serde_json = "1"
   xfetch-plugin-api = { workspace = true }
   ```

3. Add `plugins/<name>` to `[workspace].members` in the root `Cargo.toml`.
   A crate inside the workspace root that is not a member fails to build
   with a "package believes it's in a workspace" error.
4. Implement `src/main.rs` with a `const BUDGET`, `with_timeout`, and the
   correct entrypoint:

   ```rust
   use std::time::Duration;
   use xfetch_plugin_api::{
       read_info_plugin_args_or_default, with_timeout, write_info_lines,
   };

   #[derive(Debug, Default, serde::Deserialize)]
   struct PluginArgs {}

   const BUDGET: Duration = Duration::from_secs(2);

   fn main() {
       let lines = with_timeout(BUDGET, || {
           let _args = match read_info_plugin_args_or_default::<PluginArgs>() {
               Ok(value) => value,
               Err(err) => {
                   eprintln!("{}", err);
                   std::process::exit(1);
               }
           };
           vec!["hello".to_string()]
       })
       .unwrap_or_else(|_| vec!["MyPlugin: timed out".to_string()]);

       if let Err(err) = write_info_lines(lines) {
           eprintln!("{}", err);
           std::process::exit(1);
       }
   }
   ```

5. Add `README.md` with args, platform support and a config example.
6. Add a row to `README.md` and `docs/compatibility.md`.
7. If it is a logo animation, append it to `LOGO_ANIMATION_PLUGINS` in
   `scripts/generate_catalog.py` on the release branch.
8. Run the CI commands above, then regenerate the catalog on the release
   branch.

Wasm plugin:

1. Create the source directory and `xfetch-plugin.json` with
   `manifest_version`, `name`, `version`, `description`, `kind`, `runtime`,
   `build`/`artifact`, `capabilities` and `limits`.
2. Add the build command or a prebuilt artifact. The installer checks the
   artifact first, then `artifact_url`, then runs `build`.
3. Keep source and installed manifests in mind: editing the source manifest
   after installation does not update
   `~/.config/xfetch/plugins/xfetch-plugin-<name>.json`; reinstall or edit
   the sidecar.
4. For Rust wasm, add the crate to the workspace. For Python/Go/C, no
   workspace change; extend `scripts/ci-wasm.sh` if the build differs from
   the existing four examples.
5. Verify with:

   ```bash
   xfetch wasm inspect ./plugins/<name>/xfetch-plugin.json
   xfetch plugin install ./plugins/<name>
   xfetch plugin info <name>
   xfetch plugin list --verbose
   ```

## Updating a plugin

1. Change the code and bump `[package].version` (native) or both the
   manifest and any `Cargo.toml` version (wasm).
2. Update the plugin `CHANGELOG.md` and the repository `CHANGELOG.md` when
   the change is user-visible.
3. If behavior or platform support changed, update `docs/compatibility.md`
   and the README table.
4. Run `cargo test --workspace`, clippy and fmt.
5. On the release branch, run `python3 scripts/generate_catalog.py` and
   confirm `--check` passes.
6. If the plugin is installed locally, reinstall it to refresh the copy and
   any wasm sidecar:

   ```bash
   xfetch plugin remove <name>
   xfetch plugin install <name>
   ```

## Lockfile and catalog gotchas

- `Cargo.lock` is committed (lockfile version 4). The workspace resolves
  `xfetch-plugin-api` and `xfetch-guest-api` from crates.io at `0.2.0`; there
  are no `[patch.crates-io]` sections in this repository. A local API change
  must be published or patched in a temporary checkout.
- Cargo's git fetch is forced through the CLI by `.cargo/config.toml` and by
  the core installer's `CARGO_NET_GIT_FETCH_WITH_CLI=true`.
- A new Rust plugin must be a workspace member. Being physically under
  `plugins/` is not enough.
- A new wasm-only directory is invisible to `cargo test --workspace` and to
  the `ci/unix.sh` `with_timeout` loop.
- The plugin catalog generator and `index.json` are on `main` after the v1.0.2
  merge; older checkouts or stale branches can still 404.
- The generator's `CATALOG_VERSION` is hardcoded. Forgetting to bump it
  produces an index whose `catalog_version` does not match the release.
- Native catalog kind is decided by the hardcoded logo plugin set, not by
  reading code or a manifest. A new logo plugin is catalogued as an info
  provider until added to the set.
- Native package naming is enforced by the generator but not by the core
  installer; a mismatched package name can still build and install under the
  directory name, then be missing from the catalog.
- `--check` compares exact bytes. Running the generator with a different
  Python version can change JSON escaping only if non-ASCII strings are
  present; keep `ensure_ascii=False` output as committed.
- Catalog entries do not carry capabilities for native plugins and are not
  authoritative for wasm either. The installed sidecar is authoritative.
- Dependabot groups minor/patch Cargo and GitHub Actions updates weekly
  (Mondays 04:00 UTC, limit 5 each). Security audit runs weekly at 04:17 UTC.

## Troubleshooting

| Symptom | Likely cause | Action |
| --- | --- | --- |
| `xfetch plugin search` fails with HTTP 404 | Stale `XFETCH_PLUGIN_INDEX` or an older checkout | Check the override; `main` now has `index.json` |
| `Failed to fetch catalog ... HTTP 404` with a custom URL | Wrong branch/path or a private fork | Check the URL with `curl -I`; the core accepts a local file path too |
| `Invalid catalog ... Unsupported catalog schema version` | Not `schema_version: 1` | Regenerate with the matching generator |
| `Plugin '<name>' not found` at runtime | Discovery miss | Check `PATH`, `~/.config/xfetch/plugins/`, `XFETCH_PLUGIN_DEV_DIR`, and whether the name contains a path separator |
| `Built binary not found ... target/release` | Package name does not match `xfetch-plugin-<dir>`, or the build went elsewhere | Fix `[package].name`; check the workspace `target/release/` and ancestors |
| `xfetch plugin install ./target/release/xfetch-plugin-<name>` fails with an invalid guest name | A bare native executable is not an install source | Install from the source directory, or copy the binary into `~/.config/xfetch/plugins/` |
| Cargo error "package believes it's in a workspace" | New crate not listed in `[workspace].members` | Add it to the root `Cargo.toml` |
| Plugin installs but does not render | Info plugin enabled requires both `info_plugins` and `modules: ["plugin:<name>"]` | Add the module key; check `plugin list --verbose` for `enabled` |
| `info` shows no version/capabilities for a native plugin | Native plugins have no sidecar manifest | Expected; wasm plugins carry metadata |
| `--sha256 cannot be enforced for a native plugin install` | `--sha256` was passed for a native source | Remove `--sha256` or install a wasm artifact |
| Direct URL ending in `xfetch-plugin-<name>.wasm` installs under a prefixed name | The URL install path does not strip `xfetch-plugin-` | Pass `--name <name>`, or use a URL whose final segment is already `<name>.wasm` |
| Wasm install succeeds but capabilities are denied | Manifest lacks the allowlist entry, or the installed sidecar is stale | Edit the source manifest and reinstall; verify with `xfetch wasm inspect` |
| Wasm guest times out immediately with `timeout_secs: 0` | Wasm treats `0` as a zero-length deadline, unlike native where `0` disables the cap | Remove `timeout_secs` or set a positive value |
| `Wasm guest ... does not export _start` | Core module built as a library | Build a `wasm32-wasip1` binary command |
| `Invalid plugin world` / component fails | Built with the wrong WIT world | Rebuild with `-w plugin` |
| `host returned no buffer` | Guest lacks `xfetch_alloc` / guest-api | Use `xfetch-guest-api` or implement the allocator export |
| `ci/unix.sh` fails on `with_timeout` but `scripts/ci.sh` passed | `scripts/ci.sh` does not run the grep | Run `bash ci/unix.sh` before the PR |
| Python/Go/C wasm guest has no timeout and CI passes | The grep only covers `src/main.rs` | The host manifest `timeout_ms` still protects the fetch, but add an in-guest budget where the language allows it |
| `cargo audit` fails | New advisory or a warning treated as error | Run `cargo audit --deny warnings`; update the dependency or document the exception |
| Windows cross-check cannot find target | `x86_64-pc-windows-gnu` not installed | `rustup target add x86_64-pc-windows-gnu` |

Last verified: 2026-11-03 against
`plugins@56379bae0319a828d3bfbac02fe8876a2c4ea141` (`main`) and
`release/v1.0.2@4e02244` (kept).
