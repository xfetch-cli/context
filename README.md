# xfetch-cli context

Durable, deep context for the xfetch ecosystem, written for AI agents and
maintainers who need to work across the `xfetch-cli` repositories over the
medium/long term.

This is the public developer and AI context repository for the xfetch
ecosystem. It is documentation only: it does not build, run or modify the
xfetch source repositories.

Start with [AGENTS.md](AGENTS.md) for agent instructions and
[llms.txt](llms.txt) for the short machine-readable entrypoint. Reusable
agent roles and task skills live under [resources/](resources/README.md).

Last verified: 2026-11-03. Commit-level details are in
[repositories.md](repositories.md) and [current-state.md](current-state.md).

## How to use this repository

1. Start here for the map and the release state.
2. Read [ecosystem.md](ecosystem.md) for the architecture and contracts.
3. Open the repository folder for the area you are changing:
   [xfetch/](xfetch/README.md), [api/](api/README.md),
   [plugins/](plugins/README.md), [effects/](effects/README.md),
   [extensions/](extensions/README.md), [web/](web/README.md),
   [configs/](configs/README.md), [themes/](themes/README.md),
   [logos/](logos/README.md), [assets/](assets/README.md),
   [homebrew-tap/](homebrew-tap/README.md), [.github/](.github/README.md).
4. Use [workflows.md](workflows.md) for task recipes and
   [conventions.md](conventions.md) for invariants before editing.
5. Check [current-state.md](current-state.md) before assuming a branch, PR or
   catalog is merged.
6. Use [resources/](resources/README.md) for agent roles and task skills.

## Repository index

| Folder | Repo | One-line role |
| --- | --- | --- |
| [xfetch/](xfetch/README.md) | `xfetch-cli/xfetch` | Core binary, probes, UI, addon runtime, installers, releases. |
| [api/](api/README.md) | `xfetch-cli/api` | SDK crates, JSON protocols, canonical WIT. |
| [plugins/](plugins/README.md) | `xfetch-cli/plugins` | 20 official plugins + catalog. |
| [effects/](effects/README.md) | `xfetch-cli/effects` | Intro effects + catalog. |
| [extensions/](extensions/README.md) | `xfetch-cli/extensions` | Config-provider extensions + catalog. |
| [web/](web/README.md) | `xfetch-cli/web` | Next.js landing + multilingual docs. |
| [configs/](configs/README.md) | `xfetch-cli/configs` | Defaults, presets, plugin example configs. |
| [themes/](themes/README.md) | `xfetch-cli/themes` | Theme registry (`index.json`, 83 themes). |
| [logos/](logos/README.md) | `xfetch-cli/logos` | ASCII logo catalog (128 files). |
| [assets/](assets/README.md) | `xfetch-cli/assets` | Shared brand assets (banner SVG). |
| [homebrew-tap/](homebrew-tap/README.md) | `xfetch-cli/homebrew-tap` | macOS Homebrew formula. |
| [.github/](.github/README.md) | `xfetch-cli/.github` | Organization profile. |

## Cross-cutting contexts

| Path | Role |
| --- | --- |
| [crates-io/](crates-io/README.md) | crates.io profile, published inventory and publishing operations. |

## Local paths

All clones are siblings under:

```text
~/repos/xfetch-cli/
```

The context repository is `~/repos/xfetch-cli/context/`. Use the
`~/repos/xfetch-cli/<repo>` form in commands so they work from any working
directory.

## The 30-second mental model

- `xfetch` is the only user-facing binary.
- `api` owns the contracts; addons are separate native/wasm processes.
- Addon implementations live in `plugins`, `effects` and `extensions`; each
  repo publishes an `index.json` catalog on `main` (the v1.0.2 catalogs were
  merged).
- `themes`, `logos`, `configs` and `web` are data/content consumers.
- `assets`, `homebrew-tap` and `.github` are distribution/branding.
- Releases are tag-driven from `main` after a release PR; the tag must match
  `Cargo.toml`; assets are checksummed and cosign-signed.

## Core commands

```bash
# Build/test the core
cd ../xfetch && cargo build --release --locked && bash scripts/ci.sh

# List layouts / generate a config
xfetch --list-layouts
xfetch --gen-config --layout tree

# Addon discovery (catalog must be on main; use env overrides on branches)
xfetch plugin search theme
xfetch effects search glitch
xfetch extension info wasm-night-mode

# Wasm tooling
xfetch wasm inspect ./foo.wasm
xfetch wasm run ./foo.wasm --request '{"version":1,"kind":"info_provider"}'
```

## Context files

- [ecosystem.md](ecosystem.md): cross-repo architecture, protocols, catalogs,
  version matrix, security boundaries, release train.
- [repositories.md](repositories.md): every repo with local path, remote,
  branch, HEAD and authoritative sources.
- [workflows.md](workflows.md): build, install, add module/addon/theme/logo/
  preset, update docs, release, debug.
- [conventions.md](conventions.md): versioning, naming, manifests, catalogs,
  security, CI, data formats, known inconsistencies.
- [current-state.md](current-state.md): branch/PR/tag state as of 2026-11-03.
- [crates-io/](crates-io/README.md): crates.io ownership, inventory, metadata
  and publish runbook.
- [resources/](resources/README.md): agents and skills for core, addons,
  content, release and context maintenance.
- [AGENTS.md](AGENTS.md): instructions for AI coding agents.
- [llms.txt](llms.txt): short entrypoint for language models.

## License

Documentation and resources are licensed under CC BY 4.0; see [LICENSE](LICENSE).
Attribution: xscriptor (https://github.com/xscriptor).

## Scope notes

- Every repository folder has a `README.md` plus focused documents
  (`architecture.md`, `protocol(s).md`, `operations.md`, etc.).
- Documents state the commit they were verified against; treat any branch
  state as a snapshot, not a permanent invariant.
- If a fact here conflicts with the code, the code wins. Update this tree when
  the architecture or contracts change.
