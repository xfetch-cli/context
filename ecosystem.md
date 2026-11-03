# Ecosystem architecture

This document describes how the repos fit together: the core, contracts,
addon implementations, content catalogs and distribution.

## High-level diagram

```text
                    xfetch-cli/api
             SDK crates + WIT + JSON protocols
                       |
        +--------------+--------------+
        |              |              |
   xfetch-cli/xfetch  plugins      effects/extensions
   (runtime, UI,      (official     (official
    probes, install)   addons)       addons)
        |              |              |
        +------ reads catalogs ------+
                       |
          themes / logos / configs / web
                 (content + docs)
                       |
              assets / homebrew-tap / .github
                 (distribution + branding)
```

## The core (`xfetch`)

The core is the only executable that end users run directly. It:

1. Loads JSONC config from `~/.config/xfetch/config.jsonc` (or `--config`).
2. Merges the active theme, then runs extension config providers.
3. Collects `Info` from platform probes.
4. Renders `Info` with a layout/logo/theme.
5. Plays effects and logo animations.
6. Provides addon install/list/info/search/remove, theme management, wasm
   tooling and self-update.

Addons are not linked into the binary. They are separate native executables or
wasm guests, spawned with a JSON stdin/stdout protocol.

## Contracts

### JSON protocol (version 1)

Used by native addons and wasm core modules.

- Request kinds: `logo_animation`, `info_provider`, `effect`,
  `config_provider`.
- Requests carry `version: 1` and `kind`; responses do not echo them.
- Plugins:
  - info provider -> `{"lines": ["...", ...]}`
  - logo animation -> non-empty `{"frames":[{"delay_ms": N, "lines": [...]}]}`
- Effects -> `{"frames":[{"delay_ms": N, "lines": [...]}, ...]}`.
- Extensions -> `{"config": { ...modified config... }}`.

### WIT (version 0.1.0)

Canonical contract in `api/wit/xfetch-runtime.wit`, package
`xfetch:runtime@0.1.0`, worlds `plugin`, `effect`, `extension`. The core
vendors a copy and a test detects drift.

### Manifest schema (version 1)

Wasm addons ship a manifest with kind/runtime/entry, capabilities
(http/exec/fs/env/args), limits (timeout_ms, memory_mb, output_kb,
host_call_kb), artifact/artifact_url/build, and an optional sha256.

### Catalogs (schema version 1)

`plugins/index.json`, `effects/index.json`, `extensions/index.json` share:

```jsonc
{
  "schema_version": 1,
  "catalog_version": "1.0.2",
  "entries": [
    {
      "name": "...",
      "description": "...",
      "version": "...",
      "kind": "...",
      "runtime": "native" /* or "wasm" */,
      "path": "...",
      "install": "..."
    }
  ]
}
```

- `catalog_version` tracks the xfetch release train, not addon crate versions.
- `path`/`install` tell the core where the source lives and what to install.
- Wasm entries may carry manifest capabilities/limits.
- The core fetches from `raw.githubusercontent.com/<repo>/main/index.json`;
  `XFETCH_*_INDEX` override for testing/branches.

## Addon runtime model

```text
xfetch plugin install <name/source>
  -> resolve source (catalog/dir/repo/wasm/url)
  -> build native or validate wasm + manifest + sha256
  -> atomically install binary/wasm + sidecar

xfetch run
  -> plugin info: spawn native/wasm, JSON request/response, collect lines
  -> effect: spawn, send rendered lines, receive frames, play frames
  -> extension: spawn, send config, receive modified config (before render)
```

- Plugins are discovered in `PATH` then config dirs.
- Effects/extension discovery is config dirs then `PATH`.
- Native addons are unsandboxed; wasm addons are deny-by-default with
  manifest-granted capabilities and resource limits.
- Extension providers run after the theme merge and in declaration order.
- Failures are generally non-fatal for the fetch (effects/plugins skipped).

## Content and documentation flow

- `xfetch/docs/` is the core reference (installation, configuration, layouts,
  addons, wasm, daemon, themes).
- `web/docs/<lang>/` renders the public documentation (en/es/de) and the
  landing page; it hardcodes the xfetch version and is updated per release.
- `configs/` provides defaults and presets that reference core modules,
  plugins, effects and themes.
- `themes/` is a registry of partial JSONC themes consumed by the core and the
  `theme-manager` plugin.
- `logos/` is the ASCII art catalog consumed by `--gen-config`.
- `assets/` holds shared brand files (the banner is hotlinked by READMEs and
  the web site).

## Distribution and release train

```text
xfetch tag vX.Y.Z
  -> release.yml gate (main ancestry, version match, tests, audit)
  -> 6-target build
  -> archive + SHA256SUMS + cosign bundle + provenance attestation
  -> GitHub Release
     -> install-prebuilt.sh/ps1
     -> homebrew-tap formula bump
     -> AUR package bump
     -> web/docs version bump (manual)
```

Addon repos release separately using their `release/vX.Y.Z` branches and
catalog `catalog_version` aligned with the core release train. The core's
catalog URLs point at each addon repo's `main`, so catalogs must be merged
before remote `search` works.

## Security boundaries by repo

| Repo | Trust boundary |
| --- | --- |
| `xfetch` | Native probes/addons run with user privileges; wasm sandbox for wasm addons. |
| `api` | Contract only; no runtime. Changes affect all consumers. |
| `plugins`/`effects`/`extensions` | Addon code; native is trusted, wasm is sandboxed. |
| `logos` | Art is validated (size/line/control chars) before printing raw. |
| `themes`/`configs` | JSONC data; no code execution. |
| `web` | Static content; no secrets. |
| `homebrew-tap` | Formula pins URLs + SHA-256; must match release assets. |

## Version matrix (2026-11-03)

| Component | Version |
| --- | --- |
| `xfetch-cli` | 1.0.2 (main, tag `v1.0.2`); 1.0.3 candidate on `release/v1.0.3`; crates.io 1.0.1 |
| `xfetch-api-common` | 0.3.0 (main, published to crates.io) |
| `xfetch-plugin-api` | 0.3.0 (main, published); core 1.0.2 consumes 0.2.0, `release/v1.0.3` adopts 0.3.0 |
| `xfetch-effect-api` | 0.3.0 (main, published); core 1.0.2 consumes 0.2.0, `release/v1.0.3` adopts 0.3.0 |
| `xfetch-extension-api` | 0.3.0 (main, published); core 1.0.2 consumes 0.2.0, `release/v1.0.3` adopts 0.3.0 |
| JSON protocol | 1 |
| WIT package | 0.1.0 |
| Manifest schema | 1 |
| Catalog schema | 1 |
| Catalog version | 1.0.2 (main) |
| Homebrew formula | 1.0.1 (needs bump to the released version) |
| Web docs version | 1.0.1 (needs bump to the released version) |
| Theme registry | 83 themes |
| Logo catalog | 128 art files |
