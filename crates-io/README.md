# crates.io context

Publishing context for the xfetch crates. Covers ownership, metadata, current
versions, publish order and gotchas.

Last verified: 2026-11-03.

## Profile and ownership

| Item | Value |
| --- | --- |
| crates.io user | `xscriptor` (id `439389`) |
| Profile | `https://crates.io/users/xscriptor` |
| GitHub match | yes (`https://github.com/xscriptor`) |
| Owner of `xfetch-cli` | `xscriptor` |
| Owner of the SDK crates | `xscriptor` |

No ownership transfer or team setup is needed for the current release line.

## Published versions

| Crate | crates.io version | Status |
| --- | --- | --- |
| `xfetch-cli` | `1.0.1` | Published; GitHub `v1.0.2` and `v1.0.3` candidate are not on crates.io |
| `xfetch-api-common` | `0.3.0` | Published 2026-11-03 |
| `xfetch-plugin-api` | `0.3.0` | Published 2026-11-03 |
| `xfetch-extension-api` | `0.3.0` | Published 2026-11-03 |
| `xfetch-effect-api` | `0.3.0` | Published 2026-11-03 |
| `xfetch-guest-api` | `0.3.0` | Published 2026-11-03 |

The definitive `xfetch-cli` crates.io release is expected to be `1.0.3`, after
PR #200 is merged and tagged. Do not publish `1.0.2` there: `1.0.3` is the
candidate with SDK 0.3.0 and corrected metadata.

## Why `xfetch-cli` looked different

`xfetch-cli` 1.0.1 was published without `repository`, `homepage`,
`documentation`, `readme`, `keywords` or `categories`. The SDK crates at least
declared `repository`. The fix is on `release/v1.0.3` (PR #200):

```toml
repository = "https://github.com/xfetch-cli/xfetch"
homepage = "https://xfetch-cli.github.io/web/"
documentation = "https://docs.rs/xfetch-cli"
readme = "README.md"
keywords = ["system-information", "fetch", "cli", "terminal", "neofetch"]
categories = ["command-line-utilities"]
```

The published API 0.3.0 crates still lack homepage/docs/keywords/categories;
that is cosmetic and can be addressed in a future metadata-only patch.

## Documents

- [inventory.md](inventory.md): crate-by-crate versions, paths, metadata and
  consumers.
- [operations.md](operations.md): login, pre-publish checks, publish order,
  verification and troubleshooting.

## Quick facts

- crates.io versions are immutable; a mistake requires a new version (or a
  yank, which does not delete the version).
- Publish `xfetch-api-common` before plugin/extension/effect; `guest-api` is
  independent.
- `cargo publish --dry-run` and a clean git tree are part of the checklist.
- Never commit `~/.cargo/credentials.toml` or a token.
- `cargo install xfetch-cli` uses crates.io, so it remains on 1.0.1 until the
  next publish.
