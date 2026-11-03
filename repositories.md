# Repository map

Every repository in the xfetch ecosystem, with its local clone path, remote,
role, default branch and the commit observed during verification on
2026-11-03.

## Core and contracts

| Repo | Local path | Remote | Branch | HEAD | Role |
| --- | --- | --- | --- | --- | --- |
| `xfetch` | `../xfetch` | `github.com/xfetch-cli/xfetch` | `main`, `release/v1.0.2`, `release/v1.0.3`, older releases | `5fb26a3` (main, tag `v1.0.2`), `dde16cb` (`release/v1.0.3`, PR #200) | Core binary, probes, renderer, addon runtime, installers, release CI. |
| `api` | `../api` | `github.com/xfetch-cli/api` | `main` + `release/v0.3.0` | `cb70984` (main), `de5590d` (release) | SDK crates, JSON protocols, canonical WIT contract, shared `xfetch-api-common`. |

## Official addons

| Repo | Local path | Remote | Branch | HEAD | Role |
| --- | --- | --- | --- | --- | --- |
| `plugins` | `../plugins` | `github.com/xfetch-cli/plugins` | `main` + `release/v1.0.2` | `56379ba` (main), `4e02244` (release) | 20 official plugins, authoring docs, catalog generator. |
| `effects` | `../effects` | `github.com/xfetch-cli/effects` | `main` + `release/v1.0.2` | `e76d700` (main), `4038034` (release) | Intro effects (decrypt, glitch, wasm-matrix, python-pulse), catalog. |
| `extensions` | `../extensions` | `github.com/xfetch-cli/extensions` | `main` + `release/v1.0.2` | `3ed8e76` (main), `5bf664f` (release) | Config-provider extensions, catalog. |

## Content, assets and distribution

| Repo | Local path | Remote | Branch | HEAD | Role |
| --- | --- | --- | --- | --- | --- |
| `web` | `../web` | `github.com/xfetch-cli/web` | `main` | `09782e2` | Next.js landing + multilingual docs site at `xfetch-cli.github.io/web`. |
| `configs` | `../configs` | `github.com/xfetch-cli/configs` | `main` | `31ffba7` | Presets/defaults and plugin-specific example configs. |
| `themes` | `../themes` | `github.com/xfetch-cli/themes` | `main` | `8c2a16c` | Theme registry: `index.json`, schema, 83 `colors/*.jsonc`. |
| `logos` | `../logos` | `github.com/xfetch-cli/logos` | `main` | `ead5f70` | ASCII logo catalog: `logos.json`, 128 art files; real X art. |
| `assets` | `../assets` | `github.com/xfetch-cli/assets` | `main` | `d1863fe` | Shared brand assets, primarily `logo/banner/xfetch.svg`. |
| `homebrew-tap` | `../homebrew-tap` | `github.com/xfetch-cli/homebrew-tap` | `main` + `release/v1.0.2` | `7b6d3bd` | Homebrew formula for macOS prebuilt binaries. |
| `.github` | `../.github` | `github.com/xfetch-cli/.github` | `main` | `9522504` | Organization profile (`profile/README.md`). |

## Context repository

| Repo | Local path | Remote | Branch | HEAD | Role |
| --- | --- | --- | --- | --- | --- |
| `context` | `.` (this repo) | `github.com/xfetch-cli/context` | `main` | this commit | Public developer and AI context for the xfetch ecosystem. |

## Local path convention

All clones are siblings:

```text
~/repos/xfetch-cli/
├── xfetch/
├── api/
├── plugins/
├── effects/
├── extensions/
├── web/
├── configs/
├── themes/
├── logos/
├── assets/
├── homebrew-tap/
├── .github/
└── context/
```

The context repository is a sibling clone: `~/repos/xfetch-cli/context/`.
From the context root, source-repo clones are `../<repo>`; from nested docs,
use the appropriate `../` depth or the absolute form
`~/repos/xfetch-cli/<repo>`. Prefer the `~/repos/xfetch-cli/...` form in
commands so they work from any working directory.

## Branch conventions

- `xfetch` has `main` at v1.0.2 and keeps `release/v1.0.2` plus the prepared
  `release/v1.0.3` (PR #200). Older `release/v*` branches are intentionally
  kept. The addon repos keep `release/v1.0.2`.
- `api` has `main` at 0.3.0, all five crates published to crates.io, and keeps `release/v0.3.0`.
- `logos` is `main`-only; the X placeholder fix is merged.
- Dependabot branches and `fix/x-logo-placeholder` were removed. Only `main`
  and `release/v*` remain.
- `web`, `configs`, `themes`, `assets`, `homebrew-tap`, `.github` are
  `main`-only.

## Authoritative sources per concern

| Concern | Source of truth |
| --- | --- |
| Binary behavior | `xfetch` source + `docs/` |
| Wire protocols / WIT | `api` crates + `api/wit/xfetch-runtime.wit` |
| Plugin inventory | `plugins/index.json` (main) and crate manifests |
| Effect inventory | `effects/index.json` (main) |
| Extension inventory | `extensions/index.json` (main) |
| User-facing docs | `web/docs/en|es|de` and `xfetch/docs/` |
| Theme registry | `themes/index.json` |
| Logo registry | `logos/logos.json` |
| Presets | `configs/xfetch/` and `configs/plugins/` |
| Release assets / signatures | GitHub Releases of `xfetch` |
| macOS distribution | `homebrew-tap/Formula/xfetch.rb` |
