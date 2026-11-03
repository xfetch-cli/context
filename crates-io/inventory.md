# crates.io inventory

Crate-by-crate state for the xfetch ecosystem.

## Published crates

| Crate | Source | Version | Published | License | Repository | Consumers |
| --- | --- | --- | --- | --- | --- | --- |
| `xfetch-cli` | `xfetch/Cargo.toml` | `1.0.1` | 2026-10-25 | MIT | missing in 1.0.1; added on `release/v1.0.3` | End users via `cargo install` |
| `xfetch-api-common` | `api/crates/common` | `0.3.0` | 2026-11-03 | MIT | `https://github.com/xfetch-cli/api` | plugin/effect/extension APIs |
| `xfetch-plugin-api` | `api/crates/plugin-api` | `0.3.0` | 2026-11-03 | MIT | `https://github.com/xfetch-cli/api` | core, plugins |
| `xfetch-extension-api` | `api/crates/extension-api` | `0.3.0` | 2026-11-03 | MIT | `https://github.com/xfetch-cli/api` | core, extensions |
| `xfetch-effect-api` | `api/crates/effect-api` | `0.3.0` | 2026-11-03 | MIT | `https://github.com/xfetch-cli/api` | core, effects |
| `xfetch-guest-api` | `api/crates/guest-api` | `0.3.0` | 2026-11-03 | MIT | `https://github.com/xfetch-cli/api` | wasm guests |

## Pending / unreleased

| Artifact | Version | Where it is | Why not on crates.io |
| --- | --- | --- | --- |
| `xfetch-cli` | `1.0.2` | git tag `v1.0.2`, GitHub release | Superseded by the planned 1.0.3 crates.io release |
| `xfetch-cli` | `1.0.3` | `release/v1.0.3`, PR #200 | Not merged/tagged yet |
| API crates | `0.3.0` | crates.io + `api/main` | Published |

## Metadata matrix

| Field | `xfetch-cli` 1.0.1 | `xfetch-cli` 1.0.3 candidate | API crates 0.3.0 |
| --- | --- | --- | --- |
| `license` | MIT | MIT | MIT |
| `description` | present | present | present |
| `repository` | missing | present | present |
| `homepage` | missing | present | missing |
| `documentation` | missing | present | missing |
| `readme` | auto-detected | explicit | absent for most; `common` has a README |
| `keywords` | missing | present | missing |
| `categories` | missing | present | missing |
| `rust-version` | missing | missing | missing |

The API metadata gaps are non-blocking. A metadata-only `0.3.1` release could
fill them if consistency matters.

## Dependency order

```text
xfetch-api-common
  ├── xfetch-plugin-api
  ├── xfetch-extension-api
  └── xfetch-effect-api

xfetch-guest-api   (independent)
```

`xfetch-cli` consumes the three protocol SDK crates from crates.io. On
`release/v1.0.3` it uses 0.3.0; v1.0.2 uses 0.2.0.

## How to inspect the live state

```bash
cargo search xfetch-cli
cargo info xfetch-cli
cargo search xfetch-plugin-api
cargo info xfetch-plugin-api
```

API equivalents:

```bash
curl -sS -A 'xfetch-publish-check' https://crates.io/api/v1/crates/xfetch-cli
curl -sS -A 'xfetch-publish-check' https://crates.io/api/v1/crates/xfetch-cli/owners
curl -sS -A 'xfetch-publish-check' https://crates.io/api/v1/crates/xfetch-cli/1.0.1
```

The version endpoint shows `license`, `repository`, `homepage`,
`documentation`, `keywords` and `categories`; the crate endpoint may leave some
of those null.

## crates.io user checks

- `xscriptor` is the owner of `xfetch-cli` and the SDK crates.
- `github_username_matches: true`.
- No organization/team owner is configured.

## Related context

- [../api/](../api/README.md): SDK versions, protocols and adoption.
- [../xfetch/](../xfetch/README.md): core version and release branches.
