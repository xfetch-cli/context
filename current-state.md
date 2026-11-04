# Current state — 2026-11-03 (post-merge, v1.0.3 prepared)

Snapshot after merging the v1.0.2 and API v0.3.0 PRs and publishing API 0.3.0
to crates.io. Re-verify with `git` and `gh` before acting; this file is
context, not a source of truth. See [crates-io/](crates-io/README.md) for
publishing.

## Core (`xfetch`)

| Item | State |
| --- | --- |
| `main` | `5fb26a316ecbd3f170369ee63a8f1864aa6e7398` — merge of PR #198 |
| Tag | `v1.0.2` (annotated, published) |
| `Cargo.toml` on `main` | `1.0.2`, SDK `0.2.0` |
| `release/v1.0.3` | `dde16cb`; PR #200 open, not merged or tagged |
| `release/v1.0.3` contents | SDK `0.3.0`, crate metadata, version/installer bumps |
| Release workflow v1.0.2 | completed successfully; six assets, `SHA256SUMS` and cosign bundle published |
| crates.io `xfetch-cli` | `1.0.1` (1.0.2 and 1.0.3 are not published to crates.io) |
| Release branches kept | `release/v090`, `release/v1.0.0`, `release/v1.0.1`, `release/v1.0.2`, `release/v1.0.3` |
| Tests | 256 unit (1 ignored) + 6 wasm e2e; audit clean |

v1.0.2 includes the security/dependency update, addon catalog tooling,
xpm/xp/xpm-aur package counts, `--list-layouts`, layout help fixes and
installer/version updates.

v1.0.3 candidate (PR #200) is the SDK 0.3.0 adoption plus crates.io metadata.
It keeps the JSON protocol at `1` and the WIT at `0.1.0`.

## Addon repositories

| Repo | `main` | Release branch | Catalog on `main` |
| --- | --- | --- | --- |
| plugins | `56379ba` (PR #17) | `release/v1.0.2` kept | `index.json` (20 entries) |
| effects | `e76d700` (PR #7) | `release/v1.0.2` kept | `index.json` (4 entries) |
| extensions | `3ed8e76` (PR #10) | `release/v1.0.2` kept | `index.json` (5 entries) |

The changelogs are marked with the `v1.0.2` release train. Remote
`search`/`available` works from the core because the catalogs are on each
repo's `main`. Addons still pin SDK `0.2.0`; they can adopt 0.3.0 later.

## API (`api`)

| Item | State |
| --- | --- |
| `main` | `cb70984` — merge of PR #12 |
| Release branch | `release/v0.3.0` at `de5590d` (kept) |
| Version | all five crates `0.3.0` (`xfetch-api-common` included) |
| Wire / WIT | JSON protocol `1`; WIT `xfetch:runtime@0.1.0` unchanged |
| crates.io | 0.3.0 published for all five crates, common first |
| Local gates | fmt, clippy, tests, examples, audit, packaging clean |
| Compatibility | xfetch 1.0.2 and addon release branches tested against 0.3.0 |
| Adoption | xfetch `release/v1.0.3` (PR #200); addons still on 0.2.0 |

## Logos

- `main` = `ead5f70` (PR #4): real X logo, not the placeholder public-key art.
- `fix/x-logo-placeholder` deleted after merge.

## Branch hygiene

- Kept branches only: `main` and `release/v*`.
  - xfetch: `main`, `release/v090`, `release/v1.0.0`, `release/v1.0.1`,
    `release/v1.0.2`, `release/v1.0.3`.
  - plugins/effects/extensions: `main`, `release/v1.0.2`.
  - api: `main`, `release/v0.3.0`.
  - logos: `main`.
- Dependabot PRs closed and branches removed.
- Open PR: xfetch #200 (`release/v1.0.3`), ready for review; not merged.

## Pending

1. Review and merge xfetch PR #200; tag and publish `v1.0.3` when approved.
2. Publish `xfetch-cli` to crates.io. `1.0.2` and `1.0.3` are both unpublished
   there; publishing 1.0.3 after the tag gives the definitive metadata.
3. Update `web` from `1.0.1` to the released version and refresh docs/changelog.
4. Optionally adopt SDK 0.3.0 in the addon repos.
5. Optional: enforce `python3 scripts/generate_catalog.py --check` in addon CI.
6. Optional: real Windows runtime validation of the plugins branch.

## Known inconsistencies / risks

- `web` still references `1.0.1`; `homebrew-tap` is on 1.0.2.
- crates.io `xfetch-cli` is `1.0.1`; GitHub Releases have `v1.0.2` and the
  `v1.0.3` candidate is only a branch/PR.
- Addons still consume SDK `0.2.0`; `release/v1.0.3` adopts `0.3.0`.
- AUR packages are updated to 1.0.2: `xfetch-core-bin` 1.0.2-1 and
  `xfetch-git` r163.5fb26a3-1. The next bump follows the next release.
