# Conventions and invariants

Cross-repo rules an agent should preserve when changing xfetch.

## Language and style

- Source comments, docs, commit messages, CLI help and user-facing text are
  English.
- Rust edition 2024; `cargo fmt` and `clippy -D warnings` are mandatory.
- No emojis in code, docs or generated context.
- Prefer atomic writes for install paths (`src/atomic_fs.rs`).
- Keep probes fast: gate expensive work on the flattened module key set and
  use DB reads instead of subprocesses where practical.

## Versioning

| Thing | Current | Rule |
| --- | --- | --- |
| Core crate | 1.0.2 on release branch | `Cargo.toml` is the source of truth; tag must match. |
| SDK crates | 0.3.0 on `main`, published to crates.io; core v1.0.2/addons still consume 0.2.0, `release/v1.0.3` adopts 0.3.0 | Independent semver from the core; publish `xfetch-api-common` first. |
| JSON protocol | 1 | Bump only with a compatible reader/writer story. |
| WIT package | `xfetch:runtime@0.1.0` | Canonical in `api`; core vendors a copy. |
| Manifest schema | 1 | Validate before executing. |
| Catalog schema | 1 | `catalog_version` tracks the core release train. |
| Catalog version | 1.0.2 (release branches) | Update with each release. |

Rules:

- A core release tag must be a commit on `main` and match `Cargo.toml`.
- Addon releases use `release/vX.Y.Z` branches; catalogs must land before the
  core release that advertises remote search.
- Do not silently change protocol/WIT shapes; update `api` first and all
  consumers.

## Branches and release flow

- Feature/release work uses `release/vX.Y.Z` branches and PRs.
- Push CI triggers are limited to `main`; release branches are validated via
  PR checks.
- Merge order for the current train: addon catalogs (`plugins`, `effects`,
  `extensions`) -> core `xfetch` -> tag `vX.Y.Z`.
- After a release: bump Homebrew, AUR, web docs/version, catalog versions, and
  close superseded Dependabot PRs.

## Addon naming

- Native binaries: `xfetch-plugin-<name>`, `xfetch-effect-<name>`,
  `xfetch-extension-<name>` (`.exe` on Windows).
- Wasm artifacts: `<native-name>.wasm`.
- Sidecar manifests: `<native-name>.json`.
- Install dirs: `~/.config/xfetch/{plugins,effects,extensions}`.
- Plugin kinds: `logo_animation`, `info_provider`; effect kind: `effect`;
  extension kind: `config_provider`.
- Native package names use lowercase/kebab-case (`theme-manager`,
  `config-roulette`).

## Manifest and catalog invariants

- Manifest `manifest_version` must be 1; `kind` must match the command family.
- Wasm policy is deny-by-default; every capability must be explicit.
- `sha256` is same-origin integrity; a user-supplied `--sha256` is the real
  provenance pin and overrides it.
- A checked-in wasm artifact never falls back to an unverified `artifact_url`.
- Catalog entries need `name`, `description`, `version`, `kind`, `runtime`,
  `path`, `install`; schema version is validated.
- `catalog_version` is the xfetch release train, not addon semver.
- Generate catalogs with `python3 scripts/generate_catalog.py`; commit
  `index.json`; `--check` is byte-for-byte and must pass.

## Security rules

- Validate guest names (`[A-Za-z0-9._-]`, no `.`/`..`/separators) before using
  them in paths.
- Reject repository strings starting with `-` or containing NUL; use
  `git clone --depth 1 -- <repo> <dest>`.
- Create remote-install work dirs uniquely with `0700` on Unix.
- Wasm: enforce HTTP/exec/fs/env/args policy and time/memory/output/host-call
  limits; use epoch interruption for timeouts.
- Logo art: reject >64 KiB, lines >200 chars, and control/escape characters.
- Never log or commit secrets; release assets are checksummed and signed.
- Native addons are trusted code; do not claim they are sandboxed.

## CI and validation

| Repo | CI |
| --- | --- |
| `xfetch` | `rust-tests.yml`, `security-audit.yml`, `release.yml`, `roadmap-sync.yml` |
| `api`, `plugins`, `effects`, `extensions` | `security-audit.yml` only |
| `web` | repo Pages/build workflow (inspect `.github/workflows`) |
| `configs`, `themes`, `logos` | local `scripts/ci.sh` only |
| `assets`, `homebrew-tap`, `.github` | no build CI |

Local checks before committing:

```bash
# Rust repos
bash scripts/ci.sh
cargo audit --deny warnings
# Windows cross-check (xfetch/plugins/extensions/effects where applicable)
bash scripts/ci-windows.sh
# Data repos
bash scripts/ci.sh
```

## Paths and environment variables

| Variable | Meaning |
| --- | --- |
| `XFETCH_PLUGIN_REPO`, `XFETCH_EFFECT_REPO`, `XFETCH_EXTENSION_REPO` | Default install source repo. |
| `XFETCH_PLUGIN_INDEX`, `XFETCH_EFFECT_INDEX`, `XFETCH_EXTENSION_INDEX` | Catalog override (URL or local path). |
| `XFETCH_LOGOS_URL` | Logo catalog base URL override. |
| `XFETCH_PLUGIN_DEV_DIR` | Development plugin directory. |
| `XPM_AUR_STATE` | xpm-aur state file override (package count). |
| `XDG_CONFIG_HOME`, `XDG_CACHE_HOME`, `XDG_STATE_HOME` | Config/cache/state roots. |

## Data format rules

- Config, themes and presets are JSONC; comments are allowed in core config.
- `config-roulette` parses route targets as strict JSON, so presets used with
  it must be comment-free.
- Themes must not contain `icons`; icons are a user font choice.
- Logo files are UTF-8 text, rectangular-ish, ~12 rows, no control chars.
- Web docs must exist in en/es/de; sidebar order comes from `SUMMARY.md`.

## Documentation rules

- Update the core `docs/` when behavior changes.
- Update `web/docs/<lang>/` for user-facing docs; keep versions aligned.
- Update `CHANGELOG.md` and catalog metadata per release.
- Keep this context repository (`xfetch-cli/context`) in sync when architecture
  or contracts change; it is the long-term AI map, not generated output.

## Known cross-repo inconsistencies (2026-11-03, post-merge)

- API `0.3.0` is published to crates.io and merged on `api/main`; xfetch
  `release/v1.0.3` (PR #200) adopts it, while v1.0.2 and the addons still use
  `0.2.0`.
- `web` hardcodes 1.0.1 and lists 17 themes while the registry has 83.
- `--gen-config` template default is `section`; the CLI help now matches, but
  older docs/help snapshots may still say `pacman`.
- `assets` HEAD reverted the social preview directory; the org profile text
  still mentions social preview artwork.
- The `v1.0.2` release workflow may still be building when this snapshot is
  read; check the GitHub release before treating assets as published.
