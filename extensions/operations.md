# extensions operations

Build, test, catalog, release and troubleshooting procedures for
`xfetch-cli/extensions`. Commands assume the repository root unless stated
otherwise:

```bash
cd ~/repos/xfetch-cli/extensions
```

## Prerequisites

| Tool | Needed for |
| --- | --- |
| Rust stable (edition 2024) | The three Cargo workspace members |
| `wasm32-wasip1` target | `wasm-night-mode` wasm build |
| Python 3.11+ | `scripts/generate_catalog.py` (`tomllib`) |
| `componentize-py >= 0.25` | `wasm-lang-labels` |
| Go 1.24+ | `wasm-updates-footer` |
| git | Remote installs performed by the core |
| `cargo-audit 0.22.2` | Reproducing the security-audit workflow locally |

## Build

Workspace:

```bash
cargo build --workspace
cargo build --release --workspace
```

Individual native extension:

```bash
cargo build --release -p xfetch-extension-layout-override
cargo build --release -p xfetch-extension-config-roulette
# artifacts: target/release/xfetch-extension-layout-override,
#            target/release/xfetch-extension-config-roulette
```

Rust wasm extension:

```bash
rustup target add wasm32-wasip1
cargo build --release --target wasm32-wasip1 -p xfetch-extension-wasm-night-mode
# artifact: target/wasm32-wasip1/release/xfetch-extension-wasm-night-mode.wasm
```

Python component:

```bash
cd extensions/wasm-lang-labels
python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
mkdir -p dist
componentize-py --quiet -d ../../../api/wit -w extension \
  componentize app -p . -o dist/wasm-lang-labels.wasm
```

The `../../../api/wit` path assumes the multi-repository checkout layout
(`xfetch-cli/api` next to `xfetch-cli/extensions`).

Go core module:

```bash
cd extensions/wasm-updates-footer
mkdir -p dist
GOOS=wasip1 GOARCH=wasm go build -o dist/wasm-updates-footer.wasm .
```

`scripts/ci-wasm.sh` builds all three when the toolchains are installed.

## Test, format, clippy

```bash
cargo test --workspace
cargo fmt --all --check
cargo clippy --all-targets -- -D warnings
```

Verified on 2026-11-03 against the main HEAD:

```text
cargo test --workspace --offline                         PASS
cargo fmt --all --check                                  PASS
cargo clippy --workspace --all-targets --offline -D warnings  PASS
```

Only `wasm-night-mode` has unit tests (2: night window wraps midnight and a
plain window within one day). The other crates have no tests; their logic is
file I/O or JSON transforms. `wasm-lang-labels` and `wasm-updates-footer`
are not Cargo members and are not tested by `cargo test --workspace`.

## CI scripts and what they enforce

There is **no build/test GitHub Actions workflow**. The only workflow is the
security audit. Local scripts:

| Script | What it runs | What it does not do |
| --- | --- | --- |
| `scripts/ci.sh` | `cargo fmt --all --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test`, then `grep -q with_timeout ../extensions/*/src/main.rs` | No wasm builds; skips non-Rust wasm dirs |
| `scripts/ci.ps1` | Same checks on Windows PowerShell | Same limitations |
| `scripts/ci-windows.sh` | `cargo check` and `cargo clippy` for `x86_64-pc-windows-gnu` | No tests (need a Windows host), no linking |
| `scripts/ci-wasm.sh` | Builds all three wasm extensions if their toolchains exist; fails only when a present toolchain fails | Does not test the guests |
| `.github/workflows/security-audit.yml` | `cargo audit --deny warnings` on push, PR and weekly (Mondays 04:17 UTC), `cargo-audit 0.22.2`, actions pinned by SHA | No build, test, fmt or clippy |

The `scripts/ci.sh` `with_timeout` loop covers
`../extensions/*/src/main.rs`, i.e. the two native Rust extensions and
`wasm-night-mode`. It does not cover `wasm-lang-labels/app.py` or
`wasm-updates-footer/main.go`; those rely on the host manifest
`timeout_ms` instead.

`scripts/ci-windows.sh` requires the target first:

```bash
rustup target add x86_64-pc-windows-gnu
bash scripts/ci-windows.sh
```

## Catalog

### Status at the verified HEAD

| Item | `main` (current) |
| --- | --- |
| `index.json` | present, 5 entries, `catalog_version` `1.0.2` |
| `scripts/generate_catalog.py` | present |
| Raw URL `https://raw.githubusercontent.com/xfetch-cli/extensions/main/index.json` | HTTP 200 |
| `xfetch extension search` against the default URL | works |

The core hardcodes
`https://raw.githubusercontent.com/xfetch-cli/extensions/main/index.json`
(`xfetch/src/addons/catalog.rs`). Installation does not use the catalog;
`xfetch extension install <name>` clones the default branch and reads the
crate/manifest, so installation does not depend on the catalog.

Verify the live state:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' \
  https://raw.githubusercontent.com/xfetch-cli/extensions/main/index.json
# 404 at verification time

git ls-remote origin refs/heads/main refs/heads/release/v1.0.2
# 9251eec... refs/heads/main
# e32990d... refs/heads/release/v1.0.2
```

### Generate the catalog

The generator exists on `release/v1.0.2`:

```bash
git switch release/v1.0.2
python3 scripts/generate_catalog.py
python3 scripts/generate_catalog.py --check
```

Behavior (`scripts/generate_catalog.py` on `release/v1.0.2`):

- Scans `extensions/*/`, skipping dot directories.
- If `xfetch-extension.json` exists, produce a wasm entry; otherwise if
  `Cargo.toml` exists, produce a native entry; otherwise fail with
  `extensions/<name> has neither xfetch-extension.json nor Cargo.toml`.
- Wasm entry:
  - `name` must equal the directory name;
  - `runtime` must be `core` or `component`;
  - `version` and `description` are required;
  - `capabilities` defaults to `{}` and must be an object;
  - `kind` defaults to `config_provider`;
  - if a `Cargo.toml` also exists, its `[package].version` must equal the
    manifest `version` (this catches `wasm-night-mode` drift).
- Native entry:
  - `version` and `description` come from `[package]`;
  - `kind` is always `config_provider`;
  - `runtime` is `native`;
  - `capabilities` is always `{}`.
- Entries are sorted by name; duplicate names are an error.
- `catalog_version` is hardcoded to `"1.0.2"`.
- Output is `json.dumps(..., indent=2, ensure_ascii=False)` plus a trailing
  newline.
- `--check` prints `index.json is up to date` and exits 0 when identical.
  Otherwise it writes a unified diff to stderr and exits 1 with
  `index.json is stale; run python3 scripts/generate_catalog.py`, or
  `index.json is missing`.

Unlike the plugins generator, the extensions generator validates the wasm
manifest version against an adjacent `Cargo.toml`, and native entries always
carry `"capabilities": {}`.

### Catalog format

```json
{
  "schema_version": 1,
  "catalog_version": "1.0.2",
  "entries": [
    {
      "name": "config-roulette",
      "description": "Xfetch extension that randomly cycles a config value from a list on each run",
      "version": "0.2.0",
      "kind": "config_provider",
      "runtime": "native",
      "path": "extensions/config-roulette",
      "install": "config-roulette",
      "capabilities": {}
    }
  ]
}
```

Wasm entries add `guest_runtime` (`core` or `component`) and the manifest
capabilities. Native entries omit `guest_runtime`. Fields consumed by the
core are `name`, `description`, `version`, `kind`, `runtime`, `path`,
`install`, `guest_runtime` and `capabilities`; only `name` is required.
Unknown fields are ignored. `schema_version` must be `1`; remote catalogs are
limited to 2 MiB and fetched with a 15 s timeout.

`XFETCH_EXTENSION_INDEX` may be a URL or a local file path. Search matches
name, description, kind and runtime case-insensitively. `--json` prints the
filtered entries.

To use the release catalog without merging:

```bash
XFETCH_EXTENSION_INDEX=https://raw.githubusercontent.com/xfetch-cli/extensions/release/v1.0.2/index.json \
  xfetch extension search night
```

Catalog entries are discovery metadata only; they cannot grant capabilities.

## Release

Observed state and conventions:

- No tags. Releases are branch-based (`release/v1.0.0`, `release/v1.0.1`,
  `release/v1.0.2`) and merged through pull requests. `main` is the merge of
  `release/v1.0.2` (`3ed8e76`, PR #10); the release branch (`5bf664f`) is
  kept for history.
- There is no release workflow and no binary publishing. Users install from
  source or from a direct `.wasm` URL; the repository publishes source,
  manifests and the catalog on `main`.
- `catalog_version` tracks the core release (`1.0.2`).
- All current extension crates and manifests are `0.2.0`.

Release checklist:

1. Ensure the release branch contains the intended extension versions.
2. Bump `Cargo.toml` versions and, for wasm, the matching
   `xfetch-extension.json` version. Keep the two equal.
3. Update `CHANGELOG.md` and each extension's `CHANGELOG.md`.
4. Update `docs/compatibility.md` for platform changes.
5. Run `python3 scripts/generate_catalog.py` and commit `index.json`; then
   `--check`.
6. Run `cargo test --workspace`, `cargo clippy --all-targets -- -D warnings`,
   `cargo fmt --all --check`, `bash scripts/ci.sh` and (when available)
   `bash scripts/ci-wasm.sh`.
7. Run the security audit (`cargo install cargo-audit --version 0.22.2
   --locked` then `cargo audit --deny warnings`).
8. Merge the release branch to `main`; until then the default catalog URL
   404s.
9. Verify:

   ```bash
   xfetch extension search night
   xfetch extension install layout-override
   xfetch extension list --verbose
   ```

## Adding an extension

Native Rust extension:

1. Create `extensions/<name>/`.
2. Add `Cargo.toml`:

   ```toml
   [package]
   name = "xfetch-extension-<name>"
   version = "0.1.0"
   edition = "2024"
   license = "MIT"
   description = "One-line description used by the catalog"

   [dependencies]
   serde = { version = "1", features = ["derive"] }
   serde_json = "1"
   xfetch-extension-api = { workspace = true }
   ```

3. Add `extensions/<name>` to `[workspace].members` in the root
   `Cargo.toml`. A crate under the workspace root that is not a member fails
   to build with a "package believes it's in a workspace" error.
4. Implement `src/main.rs`:

   ```rust
   use std::io::{Read, Write};
   use std::time::Duration;
   use xfetch_extension_api::{
       ConfigProviderRequest, ConfigProviderResponse, KIND_CONFIG_PROVIDER, with_timeout,
   };

   const BUDGET: Duration = Duration::from_secs(2);

   fn main() {
       let result = with_timeout(BUDGET, || {
           let mut input = String::new();
           std::io::stdin().read_to_string(&mut input).unwrap();
           let request: ConfigProviderRequest = serde_json::from_str(&input).unwrap();
           if request.kind != KIND_CONFIG_PROVIDER {
               eprintln!("Unsupported kind: {}", request.kind);
               std::process::exit(1);
           }
           let mut config = request.config;
           config["layout"] = serde_json::json!("tree");
           config
       });

       match result {
           Ok(config) => {
               let body = serde_json::to_vec(&ConfigProviderResponse { config }).unwrap();
               let mut out = std::io::stdout();
               out.write_all(&body).unwrap();
               out.flush().unwrap();
           }
           Err(_) => {
               eprintln!("<name>: timed out");
               std::process::exit(1);
           }
       }
   }
   ```

5. Preserve unknown fields: keep `config` as `serde_json::Value`, clone it,
   mutate only owned keys, and return the whole object.
6. Add `README.md` with args and a `config_providers` example.
7. Add a row to `docs/compatibility.md`.
8. The catalog generator assigns `kind: config_provider` and
   `capabilities: {}` to every native extension; no extra metadata is
   needed.
9. Run the CI commands, then regenerate the catalog on the release branch.

Wasm extension:

1. Create `xfetch-extension.json` with `manifest_version`, `name`,
   `version`, `description`, `kind: "config_provider"`, `runtime`
   (`core`/`component`), `build`/`artifact`, `capabilities` and `limits`.
2. Add the build command or a prebuilt artifact; the installer checks the
   artifact first, then `artifact_url`, then runs `build`.
3. For Rust, add the crate to the workspace and keep `Cargo.toml` and
   manifest versions equal (enforced by the catalog generator). For
   Python/Go, no workspace change; extend `scripts/ci-wasm.sh` if the build
   differs.
4. Build with the correct component world: `-w extension`.
5. Verify:

   ```bash
   xfetch wasm inspect ./extensions/<name>/xfetch-extension.json
   xfetch extension install ./extensions/<name>
   xfetch extension info <name>
   xfetch extension list --verbose
   ```

## Updating an extension

1. Change the code; bump `Cargo.toml` and, for wasm, the manifest version.
2. Update the extension `CHANGELOG.md` and the repository `CHANGELOG.md` for
   user-visible changes.
3. Update `docs/compatibility.md` if platform support changes.
4. Run tests, clippy and fmt.
5. On the release branch regenerate `index.json` and run `--check`.
6. Reinstall locally to refresh the installed copy and sidecar:

   ```bash
   xfetch extension remove <name>
   xfetch extension install <name>
   ```

Editing a source manifest does not update the installed sidecar at
`~/.config/xfetch/extensions/xfetch-extension-<name>.json`; reinstall or edit
the sidecar.

## Lockfile, install and catalog gotchas

- `Cargo.lock` is committed (version 4). `xfetch-extension-api` resolves from
  crates.io at `0.2.0`; there are no path patches.
- A new Rust extension must be a workspace member.
- Non-Cargo wasm directories are invisible to `cargo test --workspace` and to
  the `with_timeout` grep.
- The catalog generator and `index.json` exist only on `release/v1.0.2` at
  the verified HEAD. The default `main` URL 404s.
- The generator's `CATALOG_VERSION` is hardcoded; bump it with the core
  release.
- Native entries always publish `capabilities: {}` because the core has no
  native manifest. Do not expect native capability metadata in the catalog.
- The catalog does not include a repository URL or a build command; install
  still uses `--repo`/default clone and the source manifest.
- There is no `XFETCH_EXTENSION_REPO` environment variable. Only `--repo`
  changes the install source; `XFETCH_EXTENSION_INDEX` changes the search
  catalog.
- The catalog's `install` field defaults to `name`; if they diverge, search
  prints the `install` value.
- Dependabot updates Cargo and GitHub Actions weekly (Mondays 04:00 UTC,
  limit 5 each). The security audit runs weekly at 04:17 UTC.
- `config-roulette` parses the selected config file with `serde_json`, not
  the core's JSONC parser. Despite the `.jsonc` paths in its README, the
  selected file must be strict JSON; comments or trailing commas make the
  extension exit non-zero and be silently skipped.
- `layout-override` replaces `modules` with an array of strings. If the
  original config uses grouped modules (`{title, modules}`), the override
  flattens that structure.
- `wasm-updates-footer` treats any `footer_text` containing the substring
  `"updates"` as already annotated. A custom footer containing that word
  disables the append.

## Troubleshooting

| Symptom | Likely cause | Action |
| --- | --- | --- |
| `xfetch extension search` fails with HTTP 404 | Stale `XFETCH_EXTENSION_INDEX` or an older checkout | Check the override; `main` now has `index.json` |
| Extension seems to do nothing | No config file exists, or the config failed to parse; providers are skipped | Create/repair `~/.config/xfetch/config.jsonc` |
| Extension errors are invisible | The core ignores provider failures by design | Test manually with a stdin pipe; check the provider's stderr |
| All provider changes are lost | One provider returned a config that cannot deserialize into `Config` | Make each provider return the full valid config; check the last one first |
| A provider cannot add another provider | The provider list is cloned before iteration | Use a static `config_providers` list |
| Extension is not found | Discovery miss | Check `<config>/xfetch/extensions/`, `<config>/xfetch/plugins/` (legacy), `PATH`; native builds in `target/release` are not searched |
| `xfetch extension install ./target/release/xfetch-extension-<name>` fails with an invalid guest name | A bare native executable is not an install source | Install from the source directory, or copy the binary into `~/.config/xfetch/extensions/` |
| Native `info` shows no version | Native extensions have no manifest | Expected |
| Direct URL ending in `xfetch-extension-<name>.wasm` installs under a prefixed name | The URL install path does not strip `xfetch-extension-` | Pass `--name <name>`, or use a URL whose final segment is already `<name>.wasm` |
| Wasm guest times out immediately with `timeout_secs: 0` | Wasm treats `0` as a zero-length deadline; native `0` disables the cap | Omit `timeout_secs` or set a positive value |
| `Invalid extension world` | Component built with the wrong world | Rebuild with `-w extension` |
| `Failed to parse extension output` | Extra stdout, logs on stdout, or a non-JSON response | Write diagnostics to stderr; emit exactly `{"config": ...}` |
| `config-roulette` exits 1 on a `.jsonc` route | The target file contains JSONC comments/trailing commas | Convert the target to strict JSON |
| `config-roulette` picks the same config all day | Default strategy is `daily` | Set `"strategy": "random"` |
| `wasm-updates-footer` does not append | `footer_text` already contains `updates`, or neither `checkupdates` nor `pacman` is available | Check the tool and the footer content |
| `wasm-lang-labels` does nothing | Locale is not `es`, `en` or `de`, or labels already exist | Set `args.language`; existing labels intentionally win |
| Cargo error "package believes it's in a workspace" | New crate not in `[workspace].members` | Add it to the root `Cargo.toml` |
| `cargo audit` fails | New advisory or warning treated as error | Update the dependency or document the exception |
| Windows cross-check cannot find target | `x86_64-pc-windows-gnu` not installed | `rustup target add x86_64-pc-windows-gnu` |

Last verified: 2026-11-03 against
`extensions@3ed8e76a52298c2ddd6d6a3c5374764001b8b941` (`main`) and
`release/v1.0.2@5bf664f` (kept).
