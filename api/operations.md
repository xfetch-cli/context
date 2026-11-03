# API operations

## Prerequisites

- Stable Rust with edition 2024 support (`rustc` 1.85 or newer). The repository
  has no `rust-version` and no `rust-toolchain.toml`; the ecosystem README uses
  Rust 1.97 in its badge.
- `cargo`, `rustfmt` and `clippy` components.
- For the Windows cross-target check: `rustup target add x86_64-pc-windows-gnu`.
- For publishing: a crates.io account and `cargo login`.
- No system libraries are required; the crates are pure Rust.

## Build, test and lint

Run from the repository root:

```bash
cargo build --workspace
cargo test --workspace
cargo fmt --all --check
cargo clippy --all-targets -- -D warnings
```

The local CI wrappers are the expected pre-PR gate:

```bash
bash scripts/ci.sh          # Linux/macOS: fmt, clippy, test
./scripts/ci.ps1            # Windows PowerShell: same checks
bash scripts/ci-windows.sh  # Windows target check + clippy from Linux/macOS
```

`scripts/ci-windows.sh` refuses to run until the `x86_64-pc-windows-gnu` target
is installed. It runs `cargo check` and `cargo clippy` only; it cannot run tests
because they need a Windows runtime.

There is no committed `Cargo.lock`; builds resolve dependencies from
`Cargo.toml`. Use `cargo generate-lockfile` if you need a local lockfile for
auditing, and do not commit it.

### Examples

Compile the two plugin examples as part of `--all-targets`:

```bash
cargo check -p xfetch-plugin-api --examples
```

They read JSON from stdin, so a manual run looks like:

```bash
cargo build -p xfetch-plugin-api --example info-plugin
printf '%s' '{"version":1,"kind":"info_provider","args":{"label":"hi"}}' \
  | ./target/debug/examples/info-plugin
```

Expected output shape: `{"lines":["plugin says: hi"]}`.

## CI workflows

The only workflow is `.github/workflows/security-audit.yml`:

- Triggers: every `push`, every `pull_request`, and a weekly schedule
  (`17 4 * * 1`).
- Permissions: `contents: read`; concurrency group per workflow/ref.
- Steps: checkout with `persist-credentials: false`, install stable Rust,
  `cargo install cargo-audit --version 0.22.2 --locked`, run
  `cargo generate-lockfile` (because no lockfile is committed), then
  `cargo audit --deny warnings`.
- Actions are pinned by commit SHA.

`rust-tests.yml` runs `cargo fmt --check`, `cargo clippy -D warnings`, workspace
tests and example builds. `security-audit.yml` runs `cargo audit`. There is no
release workflow; publishing is manual.

`.github/dependabot.yml` opens weekly grouped minor/patch updates for Cargo
dependencies and GitHub Actions (limit 5 open PRs each).

## Packaging and publishing

The five crates are published to crates.io under their package names:

| Crate | Package | Current version |
| --- | --- | --- |
| `crates/common` | `xfetch-api-common` | `0.3.0` |
| `crates/plugin-api` | `xfetch-plugin-api` | `0.3.0` |
| `crates/extension-api` | `xfetch-extension-api` | `0.3.0` |
| `crates/effect-api` | `xfetch-effect-api` | `0.3.0` |
| `crates/guest-api` | `xfetch-guest-api` | `0.3.0` |

Packaging check (does not upload):

```bash
cargo package -p xfetch-api-common
cargo package -p xfetch-plugin-api
cargo package -p xfetch-extension-api
cargo package -p xfetch-effect-api
cargo package -p xfetch-guest-api
```

Publish, in a deliberate order:

```bash
cargo publish -p xfetch-api-common
cargo publish -p xfetch-plugin-api
cargo publish -p xfetch-extension-api
cargo publish -p xfetch-effect-api
cargo publish -p xfetch-guest-api
```

`xfetch-api-common` must be published first: the plugin, extension and effect
crates depend on it by version and path. `guest-api` is independent. Publish all
crates that changed, keep their versions in lockstep when the change is
coordinated, and remember that crates.io versions are immutable. There is no
automated publisher or tag in this repository.

All five `0.3.0` crates are published to crates.io (common first, 2026-11-03).
The core `main` line still consumes `0.2.0`; `release/v1.0.3` (PR #200) adopts
`0.3.0`. The addons still consume `0.2.0`.

- `xfetch/Cargo.toml` on v1.0.2: `xfetch-plugin-api = "0.2.0"`,
  `xfetch-extension-api = "0.2.0"`, `xfetch-effect-api = "0.2.0"`.
- `xfetch/Cargo.toml` on `release/v1.0.3`: all three at `0.3.0`.
- `effects/Cargo.toml`: `xfetch-effect-api = "0.2.0"` (crates.io).
- `plugins/Cargo.toml`: `xfetch-plugin-api = "0.2.0"`,
  `xfetch-guest-api = "0.2.0"`.
- `extensions/Cargo.toml`: `xfetch-extension-api = "0.2.0"`.

For local multi-repo development, use path dependencies or a
`[patch.crates-io]` section; do not commit those overrides to consumer repos
unless the repository explicitly keeps them.

## Release steps

There is no release automation; the following is the safe manual sequence.

1. Make the protocol change in the affected crate.
2. Update `docs/` (`protocol.md`, matching `*-sdk.md`, `timeouts.md`,
   `wasm-guests.md` if relevant) and every public doc comment.
3. Add a dated entry to `CHANGELOG.md`.
4. Bump the crate version in the affected `crates/<name>/Cargo.toml` files.
   Bump all four together if the release is coordinated; the current state is
   lockstep on the release branch. The current release branch is `0.3.0`.
5. Run the full local gate:

   ```bash
   bash scripts/ci.sh
   bash scripts/ci-windows.sh
   ```

6. Run `cargo package` for each changed crate and inspect the file list.
7. Publish to crates.io.
8. Update dependent repositories (`xfetch`, `plugins`, `effects`,
   `extensions`) to the new version and run their own CI.
9. If WIT changed, copy `wit/xfetch-runtime.wit` into the xfetch core at
   `xfetch/wit/xfetch-runtime.wit`; the core drift test compares them in a
   multi-repo checkout.

## Common tasks

### Add an optional field to an existing message

1. Add `Option<T>` or a `#[serde(default)]` field to the struct.
2. Keep the wire name explicit with `#[serde(rename = "...")]` if the Rust name
   differs.
3. Serialize/deserialize tests for old payloads (without the field) and new
   payloads.
4. Document the field in `docs/protocol.md` and the matching SDK page.
5. Add a changelog entry. This is usually a minor/patch version.

### Add a new addon kind

1. Add a `KIND_*` constant, an enum variant and `from_wire`/`as_str` mapping.
2. Add request/response structs and validation.
3. Add it to the WIT if component guests are supported, with a new world.
4. Document it in `docs/protocol.md`, the SDK page, `docs/README.md` and the
   root README.
5. Update the core dispatcher and addon installers in `xfetch`.
6. This is a breaking/ecosystem-wide change; plan versions across all repos.

### Add a host operation

1. Add the operation constant and typed wrapper in `crates/guest-api`.
2. Add the capability semantics to the manifest policy in the core.
3. Add the operation to the WIT `host` interface and to the core bindings.
4. Document it in `docs/wasm-guests.md`, `docs/protocol.md` and
   `xfetch/docs/WASM.md`.
5. Add allowlist/deny tests; deny-by-default is the invariant.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Unsupported protocol version: expected 1, found N` | Request `version` is not `1` | Set `version: 1`; the protocol is not negotiated |
| `Unexpected plugin kind: expected 'info_provider', found '...'` | Wrong `kind` string or the wrong SDK entrypoint | Match the kind constant to the entrypoint |
| `Failed to parse plugin args: ...` | `args` does not match the plugin's typed struct | Make fields optional/defaulted or correct the config |
| `Animation plugins must return at least one frame` | Empty `frames` response | Always return at least one `AnimationFrame` |
| `Effects must return at least one frame` | Empty `frames` response | Always return at least one `EffectFrame` |
| `Failed to parse JSON: ...` | Stdout is not pure JSON (debug prints, banners, partial writes) | Move diagnostics to stderr; write exactly one JSON document |
| `host calls are only available on wasm32 guests` | `xfetch-guest-api` called from a native build | Expected: native builds return `HostErrorKind::Unsupported`; test around it |
| `host returned no buffer (missing xfetch_alloc export?)` | Guest does not export `xfetch_alloc`/`xfetch_free` | Use `xfetch-guest-api` or add the exports |
| WIT drift test fails in xfetch | `api/wit/xfetch-runtime.wit` and `xfetch/wit/xfetch-runtime.wit` differ | Copy the canonical api file into the core |
| `cargo audit` fails only in CI | Workspace has no lockfile; generated lockfile picked a new dependency | Reproduce with `cargo generate-lockfile && cargo audit --deny warnings` |
| `scripts/ci-windows.sh` exits early | Windows target not installed | `rustup target add x86_64-pc-windows-gnu` |

Last verified: 2026-11-03 against `api@cb70984f001d03ebcab3ef033660c8ba6d8fea6f` (`main`); `release/v0.3.0` is at `de5590d` and was merged.
