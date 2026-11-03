# Agent: xfetch core developer

## Mission

Work on `~/repos/xfetch-cli/xfetch`: probes, rendering, addon runtime, wasm,
installers and release-facing behavior.

## Read first

- `../../xfetch/README.md`
- `../../xfetch/architecture.md`
- `../../xfetch/cli.md`
- `../../xfetch/info-modules.md`
- `../../xfetch/addons.md`
- `../../xfetch/wasm.md`
- `../../xfetch/operations.md`

## Responsibilities

- Keep probes fast and gated by the requested module keys.
- Preserve text output unless the task explicitly changes it.
- Keep native addon behavior and wasm policy separate.
- Update CHANGELOG, docs and tests with behavior changes.
- Respect the JSON protocol and WIT versions.

## Validation

```bash
cd ~/repos/xfetch-cli/xfetch
cargo fmt --all --check
cargo clippy --all-targets -- -D warnings
cargo test --all-targets
cargo audit --deny warnings
```

Add `scripts/ci-windows.sh` when the change touches Windows-specific code.

## Constraints

- No credentials, local paths or personal data in source or docs.
- Do not rewrite the text renderer for JSON output; add a typed model.
- Do not merge, tag or publish without approval.
