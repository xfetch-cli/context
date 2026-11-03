# Effects operations

## Prerequisites

- Stable Rust with edition 2024 support. The repository README advertises Rust
  1.97; there is no `rust-version` or `rust-toolchain.toml`.
- For the Windows cross-check: `rustup target add x86_64-pc-windows-gnu`.
- For wasm core-module builds: `rustup target add wasm32-wasip1`.
- For the Python component: Python 3, `pip install -r
  effects/wasm-python-pulse/requirements.txt` (`componentize-py >= 0.25`) and a
  sibling `api` checkout (the build command references
  `../../../api/wit` relative to `effects/wasm-python-pulse`, which resolves to
  `<parent>/api/wit`).
- `cargo fmt` and `clippy` components.

All commands below are run from the repository root unless noted.

## Build

Native effects (workspace members):

```bash
cargo build --workspace
cargo build --release --workspace
```

Individual native effect:

```bash
cargo build --release -p xfetch-effect-decrypt
cargo build --release -p xfetch-effect-glitch
```

Because this is a Cargo workspace, the output goes to the workspace target
directory:

```text
target/release/xfetch-effect-decrypt
target/release/xfetch-effect-glitch
```

The per-effect READMEs say `effects/<name>/target/release/...`; that path is
stale for a workspace checkout. Use `target/release/...` from the repository
root, or check `cargo metadata --no-deps --format-version 1` for
`target_directory`.

Rust wasm core module:

```bash
rustup target add wasm32-wasip1
cargo build --release --target wasm32-wasip1 -p xfetch-effect-wasm-matrix
```

Artifact:

```text
target/wasm32-wasip1/release/xfetch-effect-wasm-matrix.wasm
```

This matches the manifest's `artifact` value
`../../target/wasm32-wasip1/release/xfetch-effect-wasm-matrix.wasm` relative to
`effects/wasm-matrix`.

Python component:

```bash
cd effects/wasm-python-pulse
python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
componentize-py -d ../../../api/wit -w effect componentize app -p . \
  -o dist/wasm-python-pulse.wasm
```

`dist/` is git-ignored. The `-w effect` world is mandatory; using
`plugin`/`extension` produces a component the core will reject.

Convenience script for both wasm effects:

```bash
bash scripts/ci-wasm.sh
```

The script skips missing toolchains with a notice and only fails when an
available toolchain cannot build its example.

## Test and lint

```bash
cargo test --workspace
cargo fmt --all --check
cargo clippy --all-targets -- -D warnings
```

Local CI wrappers:

```bash
bash scripts/ci.sh          # fmt, clippy, test
./scripts/ci.ps1            # Windows PowerShell: same checks
bash scripts/ci-windows.sh  # Windows target check + clippy
bash scripts/ci-wasm.sh     # wasm-matrix + Python component, if toolchains present
```

`scripts/ci-windows.sh` requires the `x86_64-pc-windows-gnu` target and runs
`cargo check`/`clippy` only (no tests). The Python component is not exercised by
`cargo test`; only `scripts/ci-wasm.sh` builds it.

Current test coverage is inline in each crate:

- `effects-lib`: tokenizer, escape preservation, glyph alphabet, rebuild, reveal.
- `decrypt`: no inline tests; relies on `effects-lib`.
- `glitch`: final-frame equality, escape preservation, glyph alphabet, block
  glyphs present.
- `wasm-matrix`: final-frame equality, escape preservation, glyph alphabet.

## Catalog (`index.json`) generation and `--check`

`main` now contains `index.json` and `scripts/generate_catalog.py` after the
v1.0.2 merge. The core's `xfetch effects search`/`available` commands use the
catalog at the repository root.

From `main`:

```bash
python3 scripts/generate_catalog.py
python3 scripts/generate_catalog.py --check
```

The script is standard-library only (`argparse`, `difflib`, `json`, `tomllib`,
`pathlib`) and is deterministic.

What it scans (`ROOT = parent of scripts/`):

- every immediate directory under `effects/`;
- skips dot-directories and `effects-lib` (`EXCLUDED_DIRS`);
- if the directory has `xfetch-effect.json`, it is a wasm entry;
- otherwise, if it has `Cargo.toml`, it is a native entry;
- otherwise it fails ("neither xfetch-effect.json nor Cargo.toml").

Native entry:

- `name`: directory name;
- `description` and `version`: required non-empty strings from the
  `[package]` table;
- `kind`: `effect`;
- `runtime`: `native`;
- `path`: `effects/<name>`;
- `install`: `<name>`;
- `capabilities`: `{}`.

Wasm entry:

- `name`: directory name; the manifest `name` must match it exactly;
- `description`, `version`: required from the manifest;
- `runtime`: `wasm`;
- `guest_runtime`: manifest `runtime`, which must be `core` or `component`;
- `kind`: manifest `kind`, default `effect`;
- `path`: `effects/<name>`;
- `install`: `<name>`;
- `capabilities`: manifest `capabilities`, which must be an object;
- if the directory also has `Cargo.toml`, its `[package].version` must equal the
  manifest `version`.

The document is:

```json
{
  "schema_version": 1,
  "catalog_version": "1.0.2",
  "entries": [ ... ]
}
```

Entries are sorted by `name`, duplicates are rejected, and the output is
`json.dumps(..., indent=2, ensure_ascii=False)` plus a trailing newline.

`--check` reads `index.json`, compares it byte-for-byte with the generated
text, prints a unified diff to stderr when stale, and exits 1. It also exits 1
if `index.json` is missing. It prints `index.json is up to date` on success.
This was verified on the `release/v1.0.2` tree (`git archive` copy): the check
passes.

`CATALOG_VERSION = "1.0.2"` is hardcoded in the script. It tracks the xfetch
release train, not the per-effect crate version (all effects remain `0.2.0`).
Bump it and regenerate whenever the catalog is released with a new xfetch
version.

The core consumes the catalog from
`https://raw.githubusercontent.com/xfetch-cli/effects/main/index.json` by
default. `XFETCH_EFFECT_INDEX` can point at a local file or another URL, and
`XFETCH_EFFECT_REPO` overrides the install repository. The catalog only
advertises metadata; installation still clones/builds the source and reads the
manifest, so the catalog cannot grant capabilities.

## CI

The only GitHub Actions workflow is `.github/workflows/security-audit.yml`:

- triggers on every push, every pull request and a weekly schedule
  (`17 4 * * 1`);
- `contents: read`, pinned action SHAs, `persist-credentials: false`;
- installs `cargo-audit` `0.22.2` and runs `cargo audit --deny warnings`;
- this repository commits `Cargo.lock`, so no lockfile generation step is
  needed.

There is no CI workflow for build, test, clippy, wasm builds or the catalog
check. `scripts/*` are the local gate. The release notes currently list
enforcing `python3 scripts/generate_catalog.py --check` in addon CI as optional
and still manual.

Dependabot is configured weekly for Cargo and GitHub Actions, grouped
minor/patch, limit 5 open PRs, with `chore(deps)`/`ci(deps)` prefixes.

## Release steps

There is no automated publisher or release workflow and no tags in the local
clone.

1. Make the effect change; keep protocol behavior version 1 compatible.
2. Bump the affected crate version in `effects/<name>/Cargo.toml` and, for wasm
   effects, the matching `version` in `xfetch-effect.json` (the catalog
   generator enforces equality when both exist).
3. Update the effect `README.md`, `configs/example.jsonc` if needed, and
   `effects/<name>/CHANGELOG.md`.
4. Update the root `CHANGELOG.md`.
5. On the release branch, bump `CATALOG_VERSION` in
   `scripts/generate_catalog.py`, run the generator, and commit `index.json`.
6. Run:

   ```bash
   cargo fmt --all --check
   cargo clippy --all-targets -- -D warnings
   cargo test --workspace
   bash scripts/ci-windows.sh
   bash scripts/ci-wasm.sh
   python3 scripts/generate_catalog.py --check
   ```

7. Run `cargo audit --deny warnings` (or rely on the security workflow).
8. Open the release PR from `release/vX.Y.Z`.
9. Merge the catalog before or together with the corresponding xfetch release;
   remote `search`/`available` only works once `index.json` exists on `main`.
10. The core installs effects by cloning the repository and building from
    source, so do not move or rename `effects/<name>` without considering
    install path resolution and the catalog.

## Adding a new effect

Follow `CONTRIBUTING.md`:

1. Create `effects/<name>/` with a crate whose package/binary is
   `xfetch-effect-<name>` (Cargo's default binary name equals the package name).
2. Add `"effects/<name>"` to the workspace `members` in `Cargo.toml`.
3. Depend on `xfetch-effect-api` and, unless you have a reason not to,
   `xfetch-effects-lib` via the workspace dependency table.
4. Implement `main` with:
   - `read_effect_request()`;
   - all work inside `with_timeout(BUDGET, ...)`;
   - deterministic frame generation;
   - a final frame equal to the original lines;
   - `write_effect_frames(frames)`.
5. Add unit tests for final-frame equality and escape preservation.
6. Add `README.md`, `CHANGELOG.md` and an example config under `configs/`.
7. Update the effect table in the root `README.md`.
8. For a wasm effect, add `xfetch-effect.json` with `manifest_version: 1`,
   `name` equal to the directory, `kind: "effect"`, `runtime` (`core` or
   `component`), build/artifact fields, `"capabilities": {}` and limits.
9. Run all build/test/lint/wasm checks.
10. Run `python3 scripts/generate_catalog.py` and commit the updated
    `index.json` (release branch/current catalog workflow).
11. Open a PR with usage details and required dependencies.

Rules enforced by review:

- Binary naming convention `xfetch-effect-<name>`.
- One focused visual idea per effect.
- stderr for errors, non-zero exit on failure.
- Never break ANSI sequences; use the shared tokenizer.
- Every effect must have a runtime limit via `with_timeout`.
- Deterministic output where possible.
- Last frame reaches the final content.
- Minimal, maintained dependencies.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `Effect 'name' skipped: Effect not found: name` | Binary not installed and not on PATH | `xfetch effects install name`; check `~/.config/xfetch/effects/` and PATH |
| `Effect 'name' skipped: Failed to parse effect output` | Stdout contains non-JSON text or is empty | Move all logging to stderr; write exactly one JSON response |
| `Effects must return at least one frame` | Empty `frames` array | Emit at least one frame, even for a no-op |
| Terminal shows garbage after playback | Partial/unsupported escape sequence produced by the effect | Use `tokenize`/`rebuild`; do not split CSI; do not introduce OSC/DCS |
| Effect never plays | stdout is not a TTY, or an image logo is active, or the binary is missing | Test in a terminal without image mode; install the effect |
| `Effect 'x' exceeded its timeout of 30s` | Native effect exceeded the core safety cap | Keep `with_timeout`; optimize frame generation; raise `timeout_secs` |
| Wasm effect reports timeout immediately | `timeout_secs: 0` on the wasm path becomes a zero deadline | Remove `timeout_secs` so the manifest `timeout_ms` applies |
| Wasm: `does not export _start` | Core module built as a library or for the wrong target | Build a `wasm32-wasip1` command |
| Wasm: `does not export run` / invalid world | Component built with `-w plugin` or `-w extension` | Rebuild with `-w effect` |
| Component build cannot find WIT | `api` checkout missing at `../../../api/wit` | Clone `xfetch-cli/api` next to `effects` or adjust the command |
| `python3 scripts/generate_catalog.py --check` fails | Catalog is stale or missing; script/version changed | Run the generator and commit `index.json` |
| `cargo test --workspace` passes but Python effect is untested | Python is not a Cargo member | Use `scripts/ci-wasm.sh` or build with `componentize-py` |
| Binary expected under `effects/<name>/target` is missing | Workspace target dir is at the repository root | Use `target/release/...` |
| `cargo audit` flags a new advisory | Lockfile dependency | Update with `cargo update`, rerun tests and audit |

Repository hygiene note: `effects/wasm-python-pulse/__pycache__/app.cpython-314.pyc`
is tracked on `main`. Do not add more generated Python bytecode; `dist/` is
already ignored.

Last verified: 2026-11-03 against `effects@e76d700765dd01c465bc2a32c3c14ebc1fe03623` (main); `release/v1.0.2` at `4038034` is kept.
