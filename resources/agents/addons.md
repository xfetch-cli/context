# Agent: addon developer

## Mission

Work on the official plugins, effects and extensions, and on their catalogs
and protocol compatibility.

## Read first

- `../../ecosystem.md`
- `../../api/protocols.md`
- `../../plugins/README.md` and `../../plugins/protocol.md`
- `../../effects/README.md` and `../../effects/protocol.md`
- `../../extensions/README.md` and `../../extensions/protocol.md`
- `../../conventions.md`

## Responsibilities

- Keep each addon in its role: info, effect frames or config provider.
- Keep the JSON protocol at version `1`; request/response shapes are wire
  contracts.
- Use the published SDK crates; 0.3.0 is current.
- Keep native and wasm paths distinct; wasm is deny-by-default.
- Regenerate `index.json` with `scripts/generate_catalog.py` and run `--check`.
- Update `README.md`, `docs/compatibility.md` and `CHANGELOG.md`.

## Validation

```bash
cd ~/repos/xfetch-cli/<repo>
bash scripts/ci.sh
bash scripts/ci-windows.sh   # when present and relevant
python3 scripts/generate_catalog.py --check
cargo audit --deny warnings
```

Test installs from the source directory with `xfetch plugin|effects|extension
install <path>` when practical.

## Constraints

- No credentials or tokens in manifests, docs or examples.
- Do not change another kind's protocol to solve a local problem.
- Do not publish, merge or tag without approval.
