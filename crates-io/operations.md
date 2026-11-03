# crates.io operations

How to publish and verify the xfetch crates safely.

## Authentication

```bash
cargo login
```

The token is stored in `~/.cargo/credentials.toml` (mode `0600`). Do not commit
that file or paste the token into a chat/log. CI would use a scoped
`CARGO_REGISTRY_TOKEN` secret, but the current process is manual.

Check that the session works without printing the token:

```bash
test -f ~/.cargo/credentials.toml && echo "credentials present"
cargo publish --dry-run -p <crate>
```

## Pre-publish checklist

For every crate:

1. The version in `Cargo.toml` is the intended final version.
2. `CHANGELOG.md` has a dated entry for that version.
3. Package metadata is complete: `license`, `description`, `repository`,
   `homepage`, `documentation`, `readme`, `keywords`, `categories`.
4. The crate's README/docs are current.
5. `Cargo.toml` has no unpublished path dependency without a matching
   `version` (path + version is required for packagability).
6. The git tree is clean (commit first; `--allow-dirty` is only for review).
7. Local gates pass:

```bash
cargo fmt --all --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
cargo audit --deny warnings
cargo package --workspace --allow-dirty --no-verify
cargo publish --dry-run -p <crate>
```

8. For `xfetch-cli`, also run `cargo publish --dry-run --allow-dirty -p
   xfetch-cli` and check the packaged file list:

```bash
cargo package -p xfetch-cli --allow-dirty --list
```

## Publish order

API:

1. `xfetch-api-common`
2. `xfetch-plugin-api`
3. `xfetch-extension-api`
4. `xfetch-effect-api`
5. `xfetch-guest-api`

`xfetch-api-common` must exist in the index before the three protocol crates
can be published. `cargo publish` waits for availability; if a dependent
publish races the index, wait a few seconds and retry.

Core:

1. Merge/tag the core release.
2. Verify the GitHub release assets.
3. `cargo publish -p xfetch-cli`.

## Commands

```bash
# Dry runs first
cargo publish --dry-run -p xfetch-api-common
cargo publish --dry-run -p xfetch-plugin-api

# Real publish
cargo publish -p xfetch-api-common
cargo publish -p xfetch-plugin-api
cargo publish -p xfetch-extension-api
cargo publish -p xfetch-effect-api
cargo publish -p xfetch-guest-api

# Core
cargo publish -p xfetch-cli
```

## Verify

```bash
cargo search xfetch-cli
cargo info xfetch-cli
cargo search xfetch-api-common
cargo info xfetch-plugin-api
cargo install xfetch-cli --version <version>
```

crates.io API:

```bash
curl -sS -A 'xfetch-publish-check' \
  https://crates.io/api/v1/crates/xfetch-api-common
```

## Versioning rules

- Published versions are immutable. To fix a mistake, publish a new patch
  version; use `cargo yank` only to stop new resolutions, not to erase
  history.
- `xfetch-api-common` and the protocol crates should stay in lockstep for a
  coordinated release.
- For 0.x SDKs, additive changes can be a patch/minor; a wire-protocol change
  is a breaking change for every addon and needs a deliberate release.
- `xfetch-cli` follows the core release version; the GitHub tag and
  `Cargo.toml` must match before publishing.

## xfetch-cli 1.0.3 release path

1. Review/merge PR #200 (`release/v1.0.3`).
2. Tag `v1.0.3` on `main`.
3. Confirm the release workflow produced the six assets, `SHA256SUMS` and the
   cosign bundle.
4. `cargo publish --dry-run -p xfetch-cli`.
5. `cargo publish -p xfetch-cli`.
6. Verify `cargo install xfetch-cli --version 1.0.3`.
7. Bump Homebrew/web/AUR to 1.0.3.

Do not publish `1.0.2` to crates.io; 1.0.3 is the definitive crates.io
release candidate.

## Troubleshooting

| Symptom | Cause | Action |
| --- | --- | --- |
| `no token found` / 401 | Not logged in or expired token | `cargo login` again |
| OTP prompt | crates.io 2FA enabled | Complete the browser/OTP flow |
| `crate version already uploaded` | Version already exists | Bump the version; never reuse |
| Dependency not found | Common crate not indexed yet | Wait and retry |
| `path dependency ... without version` | Manifest missing a version for a path dependency | Add `version = "..."` next to `path` |
| `dirty working directory` | Uncommitted files | Commit first, or use `--allow-dirty` only for dry runs |
| Invalid category/keyword | crates.io rejects unknown category | Use a valid crates.io category (`command-line-utilities`) |
| Missing README on crates.io | No `readme` field and no README in package | Add `readme = "README.md"` and package it |

## Security

- Keep tokens out of git, shell history where possible, and chat.
- Use a scoped token for automation; do not reuse a broad personal token.
- Review the packaged file list before publishing.
- Verify checksums/signatures for the GitHub release assets; crates.io
  publishing is separate from the GitHub release signature.
