# Common workflows

Task-oriented recipes across the ecosystem. All paths assume the sibling clone
layout in [repositories.md](repositories.md).

## Build and test the core

```bash
cd ~/repos/xfetch-cli/xfetch
cargo build --release --locked
bash scripts/ci.sh
cargo audit --deny warnings
```

Minimal binary without wasm/update:

```bash
cargo build --release --locked --no-default-features
```

Windows cross-check (needs `x86_64-w64-mingw32-gcc` for `zstd-sys`):

```bash
bash scripts/ci-windows.sh
```

## Run the core against a throwaway config

```bash
tmp=$(mktemp -d)
XDG_CONFIG_HOME="$tmp/config" XDG_CACHE_HOME="$tmp/cache" \
    /path/to/xfetch --gen-config
XDG_CONFIG_HOME="$tmp/config" XDG_CACHE_HOME="$tmp/cache" \
    /path/to/xfetch
```

List layouts: `xfetch --list-layouts`. Generate with a specific layout:
`xfetch --gen-config --layout tree`.

## Install a development build on this machine

```bash
cd ~/repos/xfetch-cli/xfetch
git checkout release/v1.0.2
git pull --ff-only
bash install.sh --local --yes
hash -r
xfetch --version
```

Manual alternative:

```bash
cargo build --release --locked
install -Dm755 target/release/xfetch ~/.local/bin/xfetch
```

Check for stale binaries: `type -a xfetch`; remove
`~/.cargo/bin/xfetch` if it shadows `~/.local/bin/xfetch`.

## Add or change a system module/probe

1. Edit `src/info/mod.rs` (`Info` field + fill logic).
2. Add/update the platform function under
   `src/info/platform/{linux,macos,windows}/`.
3. Add the renderer key/label handling in `src/ui/nodes.rs`.
4. Update `docs/CONFIGURATION.md` (module table/format).
5. Add tests next to the probe; run `scripts/ci.sh`.

## Add a plugin

Repo: `~/repos/xfetch-cli/plugins`.

1. Add `plugins/<name>/` crate with `Cargo.toml` and `src/main.rs`.
2. Use `xfetch-plugin-api 0.2.0`; read one JSON request from stdin, write one
   JSON response. Info plugins return `{"lines":[...]}`; logo plugins return
   frames.
3. Add the crate to the workspace members if needed.
4. Update `plugins/index.json` with `scripts/generate_catalog.py`, then run
   `--check`.
5. Run `bash scripts/ci.sh` (fmt/clippy/test) and `scripts/ci-windows.sh`.
6. Document the plugin in `README.md` and `docs/compatibility.md`.
7. Release via the addon `release/vX.Y.Z` branch; merge catalogs before the
   core release.

Install locally for testing:

```bash
xfetch plugin install ~/repos/xfetch-cli/plugins/plugins/<name>
xfetch plugin list --verbose
```

## Add an effect

Repo: `~/repos/xfetch-cli/effects`.

1. Add `effects/<name>/` using `xfetch-effect-api 0.2.0`.
2. Read `EffectRequest`, return `EffectResponse { frames }`; honor
   `duration_ms`/`fps`/`style`/`args`.
3. Regenerate `effects/index.json` (`scripts/generate_catalog.py`), run
   `--check`.
4. Run `bash scripts/ci.sh`; build the wasm target if applicable.
5. Document in `README.md`; release with the addon branch.

## Add an extension

Repo: `~/repos/xfetch-cli/extensions`.

1. Add `extensions/<name>/` using `xfetch-extension-api 0.2.0`.
2. Read the resolved config JSON from stdin, return `{"config": ...}`.
3. Wrap work in `with_timeout`; `scripts/ci.sh` enforces this for Rust
   `extensions/*/src/main.rs`.
4. Regenerate `extensions/index.json`, run `--check`.
5. Document, test, release with the addon branch.

## Add or fix a theme

Repo: `~/repos/xfetch-cli/themes`.

1. Add `colors/<name>.jsonc` (partial config: colors, layout, palette style,
   logo colors; no icons).
2. Add the registry entry to `index.json` with metadata + raw `source` URL.
3. Run `bash scripts/ci.sh` (validate.py).
4. Test by copying to `~/.config/xfetch/themes/` or via the `theme-manager`
   plugin.

## Add or fix a logo

Repo: `~/repos/xfetch-cli/logos`.

1. Add/replace `defaults/<category>/<id>.txt` (UTF-8, ~12 rows, <=200
   chars/line, no control characters).
2. Ensure `logos.json` maps the id/aliases to the file; keep colors/ansi.
3. Run `bash scripts/ci.sh` (`python3 scripts/validate.py ..`).
4. Test before merging:

```bash
XFETCH_LOGOS_URL=https://raw.githubusercontent.com/xfetch-cli/logos/<branch> \
    xfetch --gen-config --logo <id> --config /tmp/test-xfetch.jsonc
```

Known state: `x` and `equis` art are identical on the fix branch; the fix is
PR #4, not on `main` yet.

## Add or update a preset

Repo: `~/repos/xfetch-cli/configs`.

1. Add JSONC under `xfetch/defaults/`, `xfetch/presets/<category>/`, or
   `plugins/<plugin>/`.
2. Validate JSONC syntax and keep names descriptive.
3. Check required plugin/effect/theme dependencies; document them in the
   README if non-obvious.
4. Test with `xfetch --config <path>`.

## Update the website/docs

Repo: `~/repos/xfetch-cli/web`.

1. Edit `docs/<lang>/*.md` for en/es/de; keep `SUMMARY.md` ordering.
2. Update the hardcoded version (`src/...` `VERSION = "1.0.1"` today) and
   `docs/<lang>/changelog.md` per release.
3. `npm install`, `npm run dev`, `npm run lint`, `npm run build`.
4. Static export/basePath only when `GITHUB_ACTIONS=true`; local `npm run
   build` is a server build.
5. Deploy via the repo's Pages workflow.

## Bump Homebrew after a release

Repo: `~/repos/xfetch-cli/homebrew-tap`.

1. Wait for the `vX.Y.Z` GitHub release and assets.
2. Update `Formula/xfetch.rb`: `version`, both macOS asset URLs, and SHA-256
   values from `SHA256SUMS`.
3. `brew audit --strict` / `brew install --build-from-source` locally if
   possible; commit to `main`.

## Prepare and tag a release

1. Merge addon catalog PRs first (so remote `search` works).
2. Merge the core `release/vX.Y.Z` PR.
3. Tag `vX.Y.Z` on `main` (must match `Cargo.toml`).
4. `release.yml` gates on tests/audit, builds 6 targets, signs checksums,
   attests provenance, and publishes the release.
5. Close superseded Dependabot PRs.
6. Bump Homebrew, AUR, web docs/version, and any catalog `catalog_version`.

## Debug a failing probe or addon

- Probe: run the underlying command manually (`pacman -Qn`, `xpm query`,
  `brew list --formula`, etc.); check `src/info/mod.rs` gating and
  `collect_module_keys`.
- Addon: `xfetch plugin list --verbose`, `xfetch plugin info <name>`,
  `xfetch wasm inspect <file>`, run the binary manually with a JSON request.
- Wasm denial: inspect the installed sidecar manifest; `Policy` is
  deny-by-default.
- Catalog: verify the raw `index.json` URL; use `XFETCH_*_INDEX` to point at a
  branch/local file.
- Install: check `type -a xfetch`, PATH order, and whether the binary version
  matches the branch.
