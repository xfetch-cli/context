# Skill: addon development

## Choose the kind

- Plugin: adds information lines or logo animation frames.
- Effect: transforms the rendered lines into intro frames.
- Extension: mutates the resolved config before rendering.

Do not blur these roles.

## Implement

1. Read the matching repo `README.md` and `protocol.md`.
2. Use the published SDK crate (`0.3.0` current):
   - `xfetch-plugin-api`
   - `xfetch-effect-api`
   - `xfetch-extension-api`
3. Read one JSON request from stdin; write one JSON response to stdout.
   Diagnostics go to stderr.
4. Keep the request/response wire shape compatible with protocol `1`.
5. For native addons, respect the timeout budget; native is unsandboxed.
6. For wasm addons, declare capabilities and limits in the manifest; the
   policy is deny-by-default.

## Catalog

```bash
cd ~/repos/xfetch-cli/<repo>
python3 scripts/generate_catalog.py
python3 scripts/generate_catalog.py --check
```

The catalog `index.json` is on `main` for the v1.0.2 train. Keep
`catalog_version` aligned with the release train.

## Test

```bash
cargo fmt --all --check
cargo clippy --all-targets -- -D warnings
cargo test --workspace
bash scripts/ci.sh
python3 scripts/generate_catalog.py --check
cargo audit --deny warnings
```

Install from the source directory and inspect:

```bash
xfetch plugin install ~/repos/xfetch-cli/plugins/plugins/<name>
xfetch plugin list --verbose
xfetch plugin info <name>
```

Use the matching `effects`/`extension` command for other kinds.

## Update docs

Update the repo README, `docs/compatibility.md`, the addon README and the root
CHANGELOG. If behaviour or contract changed, update the context repository.
