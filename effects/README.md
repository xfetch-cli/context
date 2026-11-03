# xfetch-cli/effects

Official intro effects for xfetch, plus the shared ANSI-safe helper crate.

## Role in the ecosystem

Effects are opt-in intro animations. The xfetch core renders the information
lines, hands them to an effect binary (`xfetch-effect-<name>`) over the
`xfetch-effect-api` JSON protocol, and plays the returned frames before
settling on the real content. This repository contains:

- the reference native effects (`decrypt`, `glitch`);
- the WebAssembly examples (`wasm-matrix`, a Rust core module, and
  `wasm-python-pulse`, a Python component);
- `xfetch-effects-lib`, the shared ANSI-safe tokenizer/reveal helpers;
- the root `index.json` catalog on `main`, consumed by
  `xfetch effects search` / `available`.

Effects are not part of the xfetch binary. A missing or broken effect is
skipped and the fetch renders normally.

## Repository identity

| Item | Value |
| --- | --- |
| Local clone | `~/repos/xfetch-cli/effects` |
| Remote | `https://github.com/xfetch-cli/effects.git` (fetch and push) |
| Default branch | `main` |
| HEAD | `e76d700765dd01c465bc2a32c3c14ebc1fe03623` |
| HEAD date | 2026-11-03 |
| HEAD subject | `Merge pull request #7 from xfetch-cli/release/v1.0.2` |
| Release branch | `release/v1.0.2` at `4038034` (kept) |
| Tags in local clone | none (`git tag -l` is empty) |
| License | MIT |

`main` contains `index.json` and `scripts/generate_catalog.py` after the
v1.0.2 merge; see [operations.md](operations.md).

## Toolchain

- Rust stable, edition 2024. The README badge advertises Rust 1.97. There is no
  `rust-version` field and no `rust-toolchain.toml`.
- `Cargo.lock` is committed in this repository (unlike `api`).
- Rust wasm effects need the `wasm32-wasip1` target.
- The Python component needs Python 3, `componentize-py >= 0.25`
  (`effects/wasm-python-pulse/requirements.txt`) and a sibling `api` checkout
  for the WIT path.

## Available effects

| Effect | Crate | Runtime | Default duration | Default fps | Description |
| --- | --- | --- | --- | --- | --- |
| `decrypt` | `xfetch-effect-decrypt` | native | 1500 ms | 30 | Reveals each line from scrambled glyphs, smooth decode |
| `glitch` | `xfetch-effect-glitch` | native | 800 ms | 30 | Stuttery flicker, corruption bursts, horizontal slices, dropped rows |
| `wasm-matrix` | `xfetch-effect-wasm-matrix` | wasm `wasm32-wasip1` core module | 900 ms | 30 | Matrix-style deterministic glyph reveal |
| `wasm-python-pulse` | no Cargo crate; manifest `name` is `wasm-python-pulse` | wasm component, world `effect` | 800 ms | 30 | Line-by-line reveal with a block-gradient cursor |

All effect crate versions are `0.2.0`. The Python example has no Cargo crate;
its identity comes from `effects/wasm-python-pulse/xfetch-effect.json`.

## Quick start

Install from the official repository:

```bash
xfetch effects install decrypt
xfetch effects install glitch
```

Install from a local checkout:

```bash
xfetch effects install ./effects/decrypt
xfetch effects install ./effects/wasm-matrix
xfetch effects install ./effects/wasm-python-pulse
```

List and remove installed effects:

```bash
xfetch effects list
xfetch effects list --verbose
xfetch effects info decrypt
xfetch effects remove decrypt
```

Enable one or more effects in `config.jsonc`:

```jsonc
{
  "effects": [
    { "plugin": "glitch", "duration_ms": 700, "fps": 30 },
    { "plugin": "decrypt", "duration_ms": 1500, "fps": 30 }
  ]
}
```

A single object is also accepted. A list plays in order. Effects only run when
stdout is a terminal and no image logo is being printed.

Manual run (native binary):

```bash
cargo build --release -p xfetch-effect-decrypt
printf '%s' '{"version":1,"kind":"effect","lines":["os: Arch"],"args":{"duration_ms":300,"fps":30}}' \
  | ./target/release/xfetch-effect-decrypt
```

Useful environment overrides while testing forks:

```bash
export XFETCH_EFFECT_REPO=/path/to/effects
export XFETCH_EFFECT_INDEX=/path/to/effects/index.json
```

## Document index

- [architecture.md](architecture.md): workspace layout, crate map, core
  invocation path, effect lifecycle, timing, key files, cross-repo links.
- [protocol.md](protocol.md): stdin/stdout protocol, frame format, delay
  semantics, limits/capabilities, examples, compatibility.
- [operations.md](operations.md): build, test, wasm builds, lint, catalog
  generation and `--check`, CI, release, adding an effect, troubleshooting.

Per-effect docs live in `effects/<name>/README.md`; per-effect changelogs live
in `effects/<name>/CHANGELOG.md`.

Last verified: 2026-11-03 against `effects@370a91dfc13da7a8b3b21dab04f524409defff69` (main); catalog state checked on `origin/release/v1.0.2` at `963b3930a34e8608bc5be14c0d3a07b6769f35d9`.
