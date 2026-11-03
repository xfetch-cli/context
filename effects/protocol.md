# Effect protocol

Effects speak `xfetch-effect-api` version 1. The canonical types are in
`xfetch-cli/api/crates/effect-api`; this file documents the effect-side
behavior and the extra invariants enforced by the xfetch core and
`xfetch-effects-lib`.

## Transport

- One JSON request is written to stdin. The core closes stdin after writing, so
  `read_effect_request()` (which uses `read_to_string`) can consume the whole
  document.
- Exactly one JSON response is written to stdout. Nothing else may go to stdout.
- Errors go to stderr; the process must exit non-zero on failure.
- The core reads the entire stdout and parses it as one `EffectResponse`.
- The protocol is identical for native binaries and `wasm32-wasip1` core
  modules. Component effects receive the same JSON string through the WIT
  `run(request: string) -> result<string, string>` export.

## Request

```json
{
  "version": 1,
  "kind": "effect",
  "lines": [
    "\u001b[32m os: Arch \u001b[0m",
    "\u001b[32m cpu: 3.2 GHz \u001b[0m"
  ],
  "args": {
    "style": "glitch",
    "duration_ms": 800,
    "fps": 30,
    "args": { "glyphs": "abc" }
  }
}
```

| Field | Type | Required | Meaning |
| --- | --- | --- | --- |
| `version` | number | yes | Must be `1` |
| `kind` | string | yes | Must be `effect` |
| `lines` | array of strings | yes | Final rendered content lines, ANSI sequences intact |
| `args` | object | yes | Tuning object; individual fields are optional |
| `args.style` | string | no | Effect-specific style selector |
| `args.duration_ms` | number | no | Requested animation length |
| `args.fps` | number | no | Requested frames per second |
| `args.args` | any JSON | no | Effect-specific free-form parameters |

Rust type: `EffectRequest { version: u32, kind: String, lines: Vec<String>, args: EffectArgs }`.
`EffectArgs` has `#[serde(default)]`, so missing fields inside `args` are fine;
the `args` object itself is required because it is not `Option`.

The core always constructs the request with `EffectRequest::new`, which sets
`version = 1` and `kind = "effect"`. `read_effect_request()` calls
`validate()` and rejects any other version or kind with
`EffectApiError::InvalidProtocolVersion` / `InvalidEffectKind`.

## Response

```json
{
  "frames": [
    { "delay_ms": 33, "lines": ["@peo@f8i*xo&23#", "1qnx4w9@&k6c"] },
    { "delay_ms": 33, "lines": ["os: Arch", "cpu: 3.2 GHz"] }
  ]
}
```

| Field | Type | Required | Meaning |
| --- | --- | --- | --- |
| `frames` | array of frames | yes | Must be non-empty |
| `frames[].delay_ms` | number | yes | Hold time for this frame; the core clamps the effective value to a minimum of 1 ms |
| `frames[].lines` | array of strings | yes | Complete content column for this frame |

`EffectFrame` is `{ delay_ms: u64, lines: Vec<String> }`.
`write_effect_frames`/`EffectResponse::validate()` rejects an empty array with
`EffectApiError::EmptyEffectFrames`.

### Frame semantics

- A frame's `lines` replace the content column for that frame; the core renders
  the current logo frame alongside them. Effects do not receive or control the
  logo.
- The frame count and line count are not required to match the request at every
  frame. Effects may return fewer lines (for example `wasm-python-pulse`
  reveals line by line). The core renders whatever lines are present.
- The final frame should equal the original `lines` (unmodified). The core also
  re-renders the true final content after playback, so the fetch always settles
  correctly, but a correct last frame keeps chained effects seamless.
- Effects play in the order configured. Each effect receives the same original
  content lines, not the previous effect's last frame.
- Each effect generates all frames up front and writes one response; the core
  does the animation, not the effect process.

## Timing and delay semantics

All shipped effects compute:

```text
frame_count = max(1, duration_ms * fps / 1000)
frame_delay = 1000 / fps
frames      = 0..=frame_count
```

and set the last frame's delay to `1` ms so the core holds the settled state
briefly before the final re-render. The core sleeps `max(1, delay_ms)` for every
frame and then re-renders `final_content`.

Practical consequences:

- Total playback is approximately `(frame_count + 1) * frame_delay`, subject to
  integer truncation, the 1 ms minimum and process/scheduler overhead.
- A very high `fps` yields `frame_delay = 0` after integer division, but the
  core still sleeps 1 ms per frame.
- A very large `duration_ms * fps` creates a very large response; effects bound
  work with `with_timeout` and should reject or clamp absurd values.
- The core applies a per-effect process deadline as a safety net (see below).
  An effect that times out is skipped; the fetch continues.

Defaults used when the config omits the fields:

| Effect | `duration_ms` default | `fps` default | Notes |
| --- | --- | --- | --- |
| `decrypt` | `1500` | `30` | Ignores `style` and `args.args` |
| `glitch` | `800` | `30` | Ignores `style` and `args.args` |
| `wasm-matrix` | `900` | `30` | Ignores `style` and `args.args` |
| `wasm-python-pulse` | `800` | `30` | Ignores `style` and `args.args` |

## ANSI invariants

The rendered `lines` carry colors and icon glyphs as ANSI escape sequences.
Effects must never split or reorder those sequences.

- Use `xfetch-effects-lib`'s `tokenize`/`rebuild` (or `reveal`) and transform
  only `Seg::Ch` visible characters.
- `tokenize` consumes complete CSI sequences (`ESC [ ... final`) as one `Esc`.
  Only CSI is handled. OSC, DCS and other two-byte escape families are outside
  the helper's guarantee; do not introduce them into effect input.
- A partial escape can make the terminal interpret subsequent bytes as a
  device query or control sequence, which is a security issue documented in
  `SECURITY.md`. CI/unit tests in `effects-lib` enforce escape integrity for the
  shipped effects.
- Scrambling should use a deterministic glyph alphabet and seed. Avoid
  characters whose display width differs wildly from the original text if you
  care about layout; the shipped effects deliberately use ASCII plus block
  glyphs.

## Errors and failure behavior

- Print diagnostics to stderr and exit non-zero.
- The core treats a non-zero exit, an unparseable response or an empty frame
  list as a skipped effect and prints a warning. The fetch output is unaffected.
- A missing effect binary is also a skip, not a fatal error.
- `with_timeout` returning `Err(TimedOut)` should make the effect emit a
  fallback (for example the unmodified lines as one frame) or exit non-zero.
  Never hang.

## Limits and capabilities

### Native effects

Native effects are ordinary unsandboxed processes running with the invoking
user's privileges. They inherit the parent environment. There is no manifest and
no capability enforcement. The only runtime cap is the core's `timeout_secs`:

- absent -> 30 seconds;
- `0` -> no cap;
- positive -> that many seconds.

On expiry the core kills the whole process tree. Each shipped native effect also
uses its own `with_timeout` budget of 10 seconds for frame generation.

### Wasm effects

Wasm effects run in wasmtime with a manifest. Capabilities are deny-by-default;
effects need none. Manifests live in the source tree as `xfetch-effect.json` and
are copied next to the installed artifact as
`xfetch-effect-<name>.json`. The installer may also embed a manifest in the
`xfetch:manifest` custom section; a sidecar takes precedence.

Manifest fields relevant to effects:

```json
{
  "manifest_version": 1,
  "name": "wasm-matrix",
  "version": "0.2.0",
  "description": "Matrix-style reveal effect compiled to WebAssembly",
  "kind": "effect",
  "runtime": "core",
  "build": "cargo build --release --target wasm32-wasip1",
  "artifact": "../../target/wasm32-wasip1/release/xfetch-effect-wasm-matrix.wasm",
  "capabilities": {},
  "limits": {
    "timeout_ms": 10000,
    "memory_mb": 64,
    "output_kb": 2048,
    "host_call_kb": 64
  }
}
```

- `runtime` is `core` or `component`. It is informational; the core detects the
  shape from the wasm header.
- `capabilities` for effects should be `{}` (no HTTP, exec, filesystem, env or
  args).
- `limits.timeout_ms` applies when the user config omits `timeout_secs`.
- `limits.memory_mb`, `output_kb` and `host_call_kb` bound memory and I/O.
  Python components need more memory than Rust core modules; the Python example
  uses 256 MiB and a 20 s timeout.

Manifest defaults when a limit is absent:

| Limit | Default |
| --- | --- |
| `timeout_ms` | `30000` |
| `memory_mb` | `256` |
| `output_kb` | `4096` |
| `host_call_kb` | `4096` |

Timeout resolution for wasm: absent config `timeout_secs` -> manifest
`timeout_ms`; present -> that many seconds. Implementation caveat: in the wasm
path the config value is converted with `.map(Duration::from_secs)`, so an
explicit `timeout_secs: 0` produces a zero-length deadline rather than disabling
the cap. Omit the field to use the manifest value.

## Worked examples

### Decrypt

Request:

```json
{
  "version": 1,
  "kind": "effect",
  "lines": ["\uf31c X x86_64", "\uf2db AMD Ryzen 9"],
  "args": { "duration_ms": 1500, "fps": 30 }
}
```

Response (abridged; the real response has `frame_count + 1` frames):

```json
{
  "frames": [
    { "delay_ms": 33, "lines": ["xeri$5ep#9t4", "#1bnf9c#i"] },
    { "delay_ms": 33, "lines": ["\uf31c X x86_64", "\uf2db AMD Ryzen 9"] }
  ]
}
```

### Glitch

The glitch effect additionally inserts leading spaces as a horizontal slice and
occasionally replaces a row with an empty string. The response format is
unchanged; a mid-animation frame can look like:

```json
{
  "frames": [
    { "delay_ms": 33, "lines": ["  ▓╬4#@k9$", ""] },
    { "delay_ms": 1,  "lines": ["\uf31c X x86_64", "\uf2db AMD Ryzen 9"] }
  ]
}
```

### Python component (line-by-line pulse)

`wasm-python-pulse` builds each frame from a prefix of the lines plus a block
cursor from `[" ", "▁", "▂", "▃", "▄", "▅", "▆", "▇", "█"]`:

```json
{
  "frames": [
    { "delay_ms": 33, "lines": ["▂"] },
    { "delay_ms": 33, "lines": ["\uf31c X x86_64", "▄"] },
    { "delay_ms": 1,  "lines": ["\uf31c X x86_64", "\uf2db AMD Ryzen 9"] }
  ]
}
```

The last frame is exactly the original content.

## Compatibility

| Thing | Current value |
| --- | --- |
| Effect protocol | `1` (`PROTOCOL_VERSION`) |
| Effect kind | `effect` (`KIND_EFFECT`) |
| SDK crate | `xfetch-effect-api` `0.2.0` |
| WIT package for components | `xfetch:runtime@0.1.0`, world `effect` |
| Manifest schema | `1` |
| Catalog schema | `1` (`schema_version` in `index.json`) |
| Effect crate versions | `0.2.0` |

Rules:

- Additive request/response fields must be optional or `#[serde(default)]` so
  older effects and older cores keep interoperating.
- The response has no version/kind field; do not add one and expect it to be
  checked.
- A protocol version bump is a breaking change across `api`, `xfetch` and this
  repository.
- `with_timeout` must stay in every effect. `CONTRIBUTING.md` rejects effects
  without a runtime limit.
- The binary name is always `xfetch-effect-<name>` (plus `.exe` on Windows);
  the installed wasm artifact is `xfetch-effect-<name>.wasm`.

## Gotchas

- Any `println!`/`print()` on stdout corrupts the response. Log through stderr.
- Effects are skipped when stdout is not a terminal or when an image logo is
  being printed; this is expected, not a protocol failure.
- The core clamps `delay_ms` to at least 1 ms and always re-renders the final
  content, so the last frame delay does not control how long the final image
  stays.
- Native effects are unsandboxed and inherit the environment; treat untrusted
  native effects as arbitrary code.
- `wasm-python-pulse` must be built with world `effect`, not `plugin` or
  `extension`; the component must export `run`.
- The response can be large at high `duration_ms * fps`; respect `output_kb` for
  wasm guests and keep frame counts sane for native effects.

Last verified: 2026-11-03 against `effects@370a91dfc13da7a8b3b21dab04f524409defff69` (main) and core `xfetch@22cd5ef` (`release/v1.0.2`).
