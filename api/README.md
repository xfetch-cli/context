# xfetch-cli/api

Canonical SDK and wire-contract repository for the xfetch ecosystem.

## Role in the ecosystem

`xfetch-cli/api` owns the public contracts that every xfetch addon is built
against:

- the JSON stdin/stdout wire protocol used by native plugins, effects and
  extensions;
- the canonical WebAssembly component contract (`wit/xfetch-runtime.wit`);
- the Rust SDK crates that addon authors compile against;
- the host-call helper crate used by WebAssembly core-module guests.

It does not ship user configs, effect implementations, plugins or the core
binary. Protocol changes are supposed to land here first, then be adopted by
`xfetch-cli/xfetch`, `xfetch-cli/plugins`, `xfetch-cli/effects` and
`xfetch-cli/extensions`.

## Repository identity

| Item | Value |
| --- | --- |
| Local clone | `~/repos/xfetch-cli/api` |
| Remote | `https://github.com/xfetch-cli/api.git` (fetch and push) |
| Default branch | `main` |
| Working branch | `main` (PR #12 merged); `release/v0.3.0` kept at `de5590d` |
| HEAD | `cb70984f001d03ebcab3ef033660c8ba6d8fea6f` |
| HEAD date | 2026-11-03 |
| HEAD subject | `Merge pull request #12 from xfetch-cli/release/v0.3.0` |
| Tags in local clone | none (`git tag -l` is empty) |
| License | MIT |

## Toolchain and workspace

- Rust edition 2024 in every crate; no `rust-version` field and no
  `rust-toolchain.toml`. The effects README advertises Rust 1.97, but this
  repository itself only requires a stable toolchain new enough for edition
  2024 (Rust 1.85 or newer).
- Cargo workspace resolver 2.
- `Cargo.lock` is deliberately not committed (`.gitignore` contains
  `Cargo.lock`); the security workflow generates a temporary lockfile for
  auditing.
- All five crates are version `0.3.0` on `main`; 0.3.0 is published to crates.io.
- Runtime dependencies are `serde` and `serde_json`; `xfetch-guest-api`
  additionally uses `base64` 0.23.

## Quick facts

- Wire protocol version is `1`.
- Plugin kinds are `logo_animation` and `info_provider`; effect kind is
  `effect`; extension kind is `config_provider`.
- Five crates: `xfetch-api-common`, `xfetch-plugin-api`,
  `xfetch-extension-api`, `xfetch-effect-api`, `xfetch-guest-api`.
- The WIT package is `xfetch:runtime@0.1.0` with worlds `plugin`, `effect` and
  `extension`.
- Native addons use JSON on stdin/stdout. WebAssembly core modules use the same
  JSON plus an `xfetch.host_call` import. Component guests use typed WIT
  imports instead of the JSON host bridge.
- CI workflows: `rust-tests.yml` (fmt, clippy, tests, examples) and
  `security-audit.yml` (cargo audit). Local scripts mirror the Rust gate.

## Quick start

Use a published crate:

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
xfetch-effect-api = "0.3"
```

During multi-repo development, use a path dependency:

```toml
[dependencies]
xfetch-effect-api = { path = "../api/crates/effect-api" }
```

Minimal effect binary:

```rust
use xfetch_effect_api::{EffectFrame, read_effect_request, write_effect_frames};

fn main() -> xfetch_effect_api::Result<()> {
    let request = read_effect_request()?;
    let frames = vec![EffectFrame::new(80, request.lines)];
    write_effect_frames(frames)
}
```

Validate the repository:

```bash
cargo test --workspace
cargo fmt --all --check
cargo clippy --all-targets -- -D warnings
bash scripts/ci.sh
```

## Document index

- [architecture.md](architecture.md): workspace and crate map, data flow,
  dependencies, key files, versioning rules, including `xfetch-api-common`.
- [protocols.md](protocols.md): exact JSON contracts, WIT worlds, guest ABI,
  capabilities, limits, compatibility and consumers.
- [operations.md](operations.md): build, test, lint, CI, publishing, release
  steps, common tasks and troubleshooting.

Last verified: 2026-11-03 against `api@cb70984f001d03ebcab3ef033660c8ba6d8fea6f` (`main`); `release/v0.3.0` is at `de5590d` and was merged.
