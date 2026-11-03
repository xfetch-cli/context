# Effects architecture

## Workspace layout

```text
effects/
├── Cargo.toml                         # workspace + workspace.dependencies
├── Cargo.lock                         # committed
├── README.md
├── CHANGELOG.md
├── index.json                         # only on release/v1.0.2 (catalog)
├── effects/
│   ├── effects-lib/                   # xfetch-effects-lib (shared, not installable)
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── decrypt/                       # xfetch-effect-decrypt
│   │   ├── Cargo.toml
│   │   ├── README.md
│   │   ├── CHANGELOG.md
│   │   ├── configs/example.jsonc
│   │   └── src/main.rs
│   ├── glitch/                        # xfetch-effect-glitch
│   │   ├── Cargo.toml
│   │   ├── README.md
│   │   ├── CHANGELOG.md
│   │   ├── configs/example.jsonc
│   │   └── src/main.rs
│   ├── wasm-matrix/                   # xfetch-effect-wasm-matrix, core module
│   │   ├── Cargo.toml
│   │   ├── README.md
│   │   ├── CHANGELOG.md
│   │   ├── xfetch-effect.json
│   │   └── src/main.rs
│   └── wasm-python-pulse/             # Python component, no Cargo crate
│       ├── README.md
│       ├── CHANGELOG.md
│       ├── app.py
│       ├── requirements.txt
│       └── xfetch-effect.json
├── scripts/
│   ├── ci.sh
│   ├── ci.ps1
│   ├── ci-windows.sh
│   ├── ci-wasm.sh
│   └── generate_catalog.py            # only on release/v1.0.2
└── .github/workflows/security-audit.yml
```

The workspace members are `effects/effects-lib`, `effects/decrypt`,
`effects/glitch` and `effects/wasm-matrix`. `wasm-python-pulse` is not a Cargo
member; it is a Python source tree compiled by `componentize-py`.

## Crate map

| Path | Package | Purpose | Dependencies |
| --- | --- | --- | --- |
| `effects/effects-lib` | `xfetch-effects-lib` | ANSI-safe tokenizer, visible-index extraction, rebuild transform, deterministic LCG, `reveal` | none |
| `effects/decrypt` | `xfetch-effect-decrypt` | Smooth scrambled-glyph reveal | `xfetch-effect-api` + `xfetch-effects-lib` |
| `effects/glitch` | `xfetch-effect-glitch` | Aggressive flicker/corruption reveal | `xfetch-effect-api` + `xfetch-effects-lib` |
| `effects/wasm-matrix` | `xfetch-effect-wasm-matrix` | Matrix-style reveal, also built to `wasm32-wasip1` | `xfetch-effect-api` + `xfetch-effects-lib` |
| `effects/wasm-python-pulse` | none | Python component reveal | `componentize-py`, WIT `world effect` |

Workspace dependency declarations:

```toml
[workspace.dependencies]
xfetch-effect-api = { version = "0.2.0", package = "xfetch-effect-api" }
xfetch-effects-lib = { version = "0.2.0", path = "effects/effects-lib", package = "xfetch-effects-lib" }
```

`xfetch-effect-api` comes from crates.io, not a local path. To test uncommitted
API changes, add a `[patch.crates-io]` override locally; do not commit it as the
normal build configuration.

## `xfetch-effects-lib` public surface

`effects/effects-lib/src/lib.rs`:

- `SCRAMBLE_GLYPHS: &[u8]` - lowercase alphanumerics plus `#@%&*$+=`.
- `lcg(state: &mut u64) -> u64` - deterministic 64-bit LCG (multiplier
  `6364136223846793005`, increment `1442695040888963407`, returns `state >> 33`).
- `enum Seg { Esc(String), Ch(char) }` - tokenized line unit.
- `tokenize(line: &str) -> Vec<Seg>` - consumes a whole CSI escape sequence
  (`ESC [` parameters until a final byte in `0x40..=0x7E`) as one `Esc`, and
  every other `char` as `Ch`.
- `visible_indices(&[Seg]) -> Vec<usize>` - indexes of `Ch` segments.
- `rebuild(segs, apply)` - rebuilds the line, applying `apply(index, char)` to
  visible characters only.
- `reveal(line, progress, seed) -> String` - keeps a prefix of visible
  characters and scrambles the rest with deterministic glyphs.

Invariant: escape sequences must stay byte-identical. The tests cover full
reveal, escape integrity under many seeds, glyph alphabet membership and rebuild
order.

Limitation: `tokenize` only understands CSI sequences beginning with `ESC [`.
Other escape families (OSC `ESC ] ... BEL/ST`, DCS, two-byte escapes) are not
consumed as a unit, so a future effect using them could expose broken
sequences. Keep effect input to the CSI colors/icons the core produces.

## How the core invokes effects

Verified against `xfetch-cli/xfetch` checkout `22cd5ef` (`release/v1.0.2`).

1. Config parsing produces `Config.effects: Option<EffectConfigList>`, where
   `EffectConfigList` is `Single(EffectConfig)` or `Multiple(Vec<EffectConfig>)`
   (`xfetch/src/config.rs`). `effects_list()` normalizes both to a slice.
2. After rendering the logo and content lines, `xfetch/src/ui.rs` checks:
   - `stdout().is_terminal()` is true;
   - `image_printed` is false (image logos skip effects);
   - the effects list is non-empty.
3. For each configured effect in order, the core calls
   `crate::effects::run_effect(effect_cfg, &content_lines)`.
4. `run_effect` (`xfetch/src/effects/mod.rs`):
   - requires a `plugin` name;
   - resolves the binary with `find_effect_binary(name)`;
   - builds `EffectRequest::new(lines, EffectArgs { style, duration_ms, fps, args })`;
   - serializes with `to_json_vec` (compact JSON).
5. If the path is a wasm artifact (`is_wasm_file`), the core calls
   `wasm::run_request(path, payload, timeout, GuestKind::Effect)`, parses the
   stdout as `EffectResponse` and validates non-empty frames. `timeout` is
   `config.timeout_secs.map(Duration::from_secs)`; absent means the manifest
   `timeout_ms` applies.
6. Otherwise it runs the native binary with
   `run_cmd_with_stdin_timeout(path, &[], Some(payload), timeout)`.
   `timeout` comes from `guest_timeout(config.timeout_secs)`:
   absent -> 30 s, `0` -> no cap, otherwise seconds. On expiry the whole process
   tree is killed.
7. Non-zero exit or invalid JSON -> `Err(String)`. In `ui.rs` this is printed as
   `Effect '<name>' skipped: <error>` and the next effect is tried. An effect
   that returns zero frames is also skipped.
8. Successfully returned `Vec<Vec<EffectFrame>>` sequences are passed to
   `ui::print::print_effect_output` together with logo frames, geometry and the
   true `final_content`.

### Binary resolution order

`find_effect_binary` searches for both `xfetch-effect-<name>` and
`xfetch-effect-<name>.wasm`:

1. every `PATH` directory;
2. each config search directory (`config_search_dirs()`), under
   `<dir>/xfetch/effects/` and then `<dir>/xfetch/plugins/`;
3. development fallbacks while walking up from the current working directory:
   `<dir>/effects/target/{wasm32-wasip1,wasm32-wasi}/release/<artifact>` and
   `<dir>/effects/effects/target/...`.

The canonical install directory is `~/.config/xfetch/effects/` (on Windows the
binary name gains `.exe`). `xfetch effects install` builds the native effect
with `cargo build --release` in the effect directory and copies the binary
there; for wasm sources it delegates to the shared wasm installer and installs
`xfetch-effect-<name>.wasm` plus the sidecar manifest
`xfetch-effect-<name>.json`.

## Effect lifecycle

1. `xfetch` loads and merges config.
2. The core renders the final content lines (icon + value, ANSI included).
3. Effect selection happens only for interactive terminal output.
4. The core starts one process/guest per configured effect, in order.
5. Each effect reads the entire request from stdin, generates all frames, writes
   one `EffectResponse` JSON to stdout, exits 0.
6. The core parses and validates the response.
7. `print_effect_output` plays every effect sequence in order:
   - for each frame: move to the frame origin, clear down, render the current
     logo frame plus the effect frame lines, sleep `max(1 ms, delay_ms)`;
   - logo frames cycle independently (`frame_index % logo_frames.len()`).
8. After all sequences, the core re-renders `final_content` exactly, so the
   screen matches a normal fetch even if the last effect frame was imperfect.
9. The terminal cursor is hidden during playback and shown again at the end.

The effect does not receive the logo and does not control layout; it only
returns transformed content lines. The logo is handled by the core.

## Frame timing math

All shipped effects use the same pattern:

```text
duration_ms = request.args.duration_ms.unwrap_or(DEFAULT).max(1)
fps         = request.args.fps.unwrap_or(30).max(1)
frame_count = max(1, duration_ms * fps / 1000)     // integer division
frame_delay = 1000 / fps                            // integer division
frames      = 0..=frame_count                       // frame_count + 1 frames
last frame  = progress 1.0, delay 1 ms
```

Consequences:

- Playback time is approximately `(frame_count + 1) * frame_delay`, not exactly
  `duration_ms`; integer truncation and the final 1 ms settle frame cause drift.
- The core enforces a minimum frame delay of 1 ms, so `delay_ms: 0` is not a
  busy spin.
- The last frame's 1 ms delay is intentional. The visible settling moment is the
  core's final re-render.
- Effects should be deterministic. `decrypt`, `glitch` and `wasm-matrix` seed
  from the frame index (and line index where relevant); the Python effect is
  progress-based.

## Native versus WebAssembly effects

| Aspect | Native | Wasm core module | Wasm component |
| --- | --- | --- | --- |
| Artifact | executable `xfetch-effect-<name>` | `xfetch-effect-<name>.wasm` | `.wasm` component |
| Contract | JSON stdin/stdout | JSON stdin/stdout + `xfetch.host_call` | WIT `run(string) -> result<string,string>` |
| Sandbox | none; user privileges | wasmtime, manifest allowlists | wasmtime, manifest allowlists |
| Limits | core `timeout_secs` only | manifest `limits` + config override | manifest `limits` + config override |
| Manifest | not used | `xfetch-effect.json` copied as sidecar | `xfetch-effect.json` copied as sidecar |
| Detection | non-wasm file | 8-byte wasm header, core module | 8-byte wasm header, component |

Manifest schema is shared with plugins and extensions (`manifest_version: 1`,
`kind`, `runtime`, `capabilities`, `limits`, `artifact`, `build`,
`artifact_url`, `sha256`). Effects need no capabilities because they only
compute text, so `"capabilities": {}` is the correct setting.

## Key files

Inside this repository:

| File | Responsibility |
| --- | --- |
| `Cargo.toml` | Workspace members and shared dependency versions |
| `effects/effects-lib/src/lib.rs` | ANSI-safe primitives and tests |
| `effects/decrypt/src/main.rs` | Decrypt frame generator |
| `effects/glitch/src/main.rs` | Glitch frame generator and unit tests |
| `effects/wasm-matrix/src/main.rs` | Matrix frame generator and unit tests |
| `effects/wasm-matrix/xfetch-effect.json` | Core-module manifest, build command, artifact path, limits |
| `effects/wasm-python-pulse/app.py` | Component `WitWorld.run` implementation |
| `effects/wasm-python-pulse/xfetch-effect.json` | Component manifest and limits |
| `scripts/ci-wasm.sh` | Local wasm builds for both wasm effects |
| `scripts/generate_catalog.py` | Catalog generator/checker (release branch) |
| `index.json` | Machine-readable catalog (release branch) |

In the xfetch core:

| File | Responsibility |
| --- | --- |
| `src/config.rs` | `EffectConfig`, `EffectConfigList`, `effects_list()` |
| `src/ui.rs` | TTY/image gate, runs effects, calls playback |
| `src/effects/mod.rs` | `run_effect`, binary resolution, native/wasm dispatch |
| `src/effects/install.rs` | `xfetch effects install` |
| `src/effects/manage.rs` | `xfetch effects list` / `remove` |
| `src/ui/print.rs` | `print_effect_output`, frame delays, final settle |
| `src/subprocess.rs` | stdin writing, 30 s default deadline, process-tree kill |
| `src/wasm/runner/mod.rs` | Wasm deadline/manifest resolution |
| `src/wasm/manifest.rs` | Manifest schema, capabilities, limit defaults |
| `docs/EFFECTS.md` | User-facing effects guide |

## Cross-repo links

- API contract: `xfetch-cli/api` ->
  `crates/effect-api`, `wit/xfetch-runtime.wit`, `docs/effect-sdk.md`,
  `docs/protocol.md`, `docs/wasm-guests.md`.
- Core runtime: `xfetch-cli/xfetch` -> `src/effects/`, `src/ui/print.rs`,
  `src/wasm/`, `docs/EFFECTS.md`, `docs/WASM.md`.
- Sibling addon repos: `xfetch-cli/plugins`, `xfetch-cli/extensions` use the
  same install/manifest machinery and are useful references for wasm source
  layouts.
- Catalog consumer: `xfetch/src/addons/catalog.rs` fetches
  `https://raw.githubusercontent.com/xfetch-cli/effects/main/index.json` by
  default and honors `XFETCH_EFFECT_INDEX`.

Last verified: 2026-11-03 against `effects@370a91dfc13da7a8b3b21dab04f524409defff69` (main) and core `xfetch@22cd5ef` (`release/v1.0.2`).
