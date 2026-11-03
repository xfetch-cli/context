# API protocols

This file is the exact contract reference. The canonical human prose lives in
`docs/protocol.md`; the canonical component contract is
`wit/xfetch-runtime.wit`. Protocol version is `1` everywhere.

## Surfaces at a glance

| Addon family | Kind string | Transport | SDK crate | Host bridge |
| --- | --- | --- | --- | --- |
| Info provider plugin | `info_provider` | JSON stdin/stdout | `xfetch-plugin-api` | JSON `host_call` (core wasm) |
| Logo animation plugin | `logo_animation` | JSON stdin/stdout | `xfetch-plugin-api` | JSON `host_call` (core wasm) |
| Effect | `effect` | JSON stdin/stdout | `xfetch-effect-api` | JSON `host_call` (core wasm) |
| Config provider extension | `config_provider` | JSON stdin/stdout | `xfetch-extension-api` | JSON `host_call` (core wasm) |
| Component plugin/effect/extension | same kind strings | WIT `run(string) -> result<string,string>` | none (bindings from WIT) | Typed WIT `host` imports |

For native processes and `wasm32-wasip1` core modules the transport is
identical: exactly one JSON document is written to stdin, stdin is closed, and
exactly one JSON document must be written to stdout. Anything else on stdout
(such as `println!` debug noise) makes the response unparseable. Diagnostics
must go to stderr and failures must exit non-zero.

## Common JSON rules

- `version` is the number `1`.
- `kind` is one of the strings above.
- Responses do not repeat `version` or `kind`; they are bare result objects.
- JSON is compact on the wire (`serde_json::to_vec`), with no required trailing
  newline. Consumers parse the whole stdout buffer.
- `serde` ignores unknown fields by default, but missing non-`Option` fields are
  an error unless the struct has `#[serde(default)]`.
- `Option<T>` fields may be omitted and deserialize as `None`; the request
  structs that are not `#[serde(default)]` still require their non-Option
  fields.

## Plugin contracts (`xfetch-plugin-api`)

Constants in `crates/plugin-api/src/protocol.rs`:

```rust
pub const PROTOCOL_VERSION: u32 = 1;
pub const KIND_LOGO_ANIMATION: &str = "logo_animation";
pub const KIND_INFO_PROVIDER: &str = "info_provider";
```

### Logo animation

Request:

```json
{
  "version": 1,
  "kind": "logo_animation",
  "lines": ["line 1", "line 2"],
  "frames": [["frame 1"], ["frame 2"]],
  "args": {
    "fps": 12,
    "duration_ms": 1200,
    "loop": true,
    "style": "wave"
  }
}
```

| Field | Rust type | Required | Notes |
| --- | --- | --- | --- |
| `version` | `u32` | yes | Must equal `1` |
| `kind` | `String` | yes | Must equal `logo_animation` |
| `lines` | `Vec<String>` | yes | Pre-rendered logo lines |
| `frames` | `Option<Vec<Vec<String>>>` | no | Previously loaded frame data, if any |
| `args` | `LogoAnimationArgs` | yes | Object; individual fields are optional |
| `args.fps` | `Option<u64>` | no | Frames per second |
| `args.duration_ms` | `Option<u64>` | no | Total duration |
| `args.loop` | `Option<bool>` | no | Rust field is `loop_enabled`, wire name is `loop` |
| `args.style` | `Option<String>` | no | Style selector |

Response:

```json
{
  "frames": [
    { "delay_ms": 80, "lines": ["rendered line 1", "rendered line 2"] }
  ]
}
```

`AnimationFrame` is `{ "delay_ms": u64, "lines": [String] }`.
`LogoAnimationResponse::validate()` rejects an empty `frames` list with
`PluginApiError::EmptyAnimationFrames`. The core calls this validation.

### Info provider

Request:

```json
{
  "version": 1,
  "kind": "info_provider",
  "args": { "username": "example", "max_lines": 5 }
}
```

| Field | Rust type | Required | Notes |
| --- | --- | --- | --- |
| `version` | `u32` | yes | Must equal `1` |
| `kind` | `String` | yes | Must equal `info_provider` |
| `args` | `Option<serde_json::Value>` | no | Free-form; parsed by the plugin |

Response:

```json
{ "lines": ["line one", "line two"] }
```

`InfoPluginResponse` is just `{ "lines": Vec<String> }`; the API does not
validate its contents beyond successful deserialization.

### Plugin SDK entrypoints and validation

- `read_logo_animation_request()` reads stdin, deserializes and calls
  `validate()`.
- `read_info_plugin_request::<T>()` reads stdin, validates and parses `args`
  into `T`, returning `InfoPluginInput { request, args }`.
- `read_info_plugin_args_or_default::<T>()` returns `T::default()` when `args`
  is absent.
- `write_logo_animation_frames(frames)` validates non-empty and writes stdout.
- `write_info_lines(lines)` writes stdout with no extra validation.
- `PluginApiError` variants: `Io`, `Serialize`, `Deserialize`,
  `InvalidProtocolVersion { expected, found }`,
  `InvalidPluginKind { expected, found }`, `InvalidArgs`, `EmptyAnimationFrames`.

## Effect contract (`xfetch-effect-api`)

Constants in `crates/effect-api/src/protocol.rs`:

```rust
pub const PROTOCOL_VERSION: u32 = 1;
pub const KIND_EFFECT: &str = "effect";
```

Request:

```json
{
  "version": 1,
  "kind": "effect",
  "lines": ["\u001b[32m os: Arch \u001b[0m", "\u001b[32m cpu: 3.2 GHz \u001b[0m"],
  "args": {
    "style": "glitch",
    "duration_ms": 800,
    "fps": 30,
    "args": { "glyphs": "abc" }
  }
}
```

| Field | Rust type | Required | Notes |
| --- | --- | --- | --- |
| `version` | `u32` | yes | Must equal `1` |
| `kind` | `String` | yes | Must equal `effect` |
| `lines` | `Vec<String>` | yes | Rendered content lines, ANSI intact |
| `args` | `EffectArgs` | yes | Object; fields optional |
| `args.style` | `Option<String>` | no | Effect-specific style |
| `args.duration_ms` | `Option<u64>` | no | Animation length |
| `args.fps` | `Option<u64>` | no | Frames per second |
| `args.args` | `Option<serde_json::Value>` | no | Free-form effect parameters |

Response:

```json
{
  "frames": [
    { "delay_ms": 33, "lines": ["@peo@f8i*xo&23#", "1qnx4w9@&k6c"] },
    { "delay_ms": 33, "lines": ["os: Arch", "cpu: 3.2 GHz"] }
  ]
}
```

- `EffectFrame` is `{ "delay_ms": u64, "lines": Vec<String> }`.
- `EffectResponse::validate()` rejects empty `frames`.
- The last frame should reach the final, unmodified content. The core
  additionally re-renders the true final content after playback, so a slightly
  off last frame is repaired visually but should still be correct.
- `EffectArgs::parse_args::<T>()` and `parse_args_or_default::<T>()` are the
  typed accessors for `args.args`.
- `EffectApiError` variants: `Io`, `Serialize`, `Deserialize`,
  `InvalidProtocolVersion { expected, found }`, `InvalidEffectKind { found }`,
  `InvalidArgs`, `EmptyEffectFrames`.
- Entrypoints: `read_effect_request()` (reads stdin, validates) and
  `write_effect_frames(frames)` (validates non-empty, writes stdout).

See the companion `effects` documentation for the full effect-side behavior,
timing, ANSI rules and catalog format.

## Extension contract (`xfetch-extension-api`)

Constants in `crates/extension-api/src/types.rs`:

```rust
pub const EXTENSION_PROTOCOL_VERSION: u32 = 1;
pub const KIND_CONFIG_PROVIDER: &str = "config_provider";
```

Request:

```json
{
  "version": 1,
  "kind": "config_provider",
  "config": { "show_colors": true, "modules": ["os", "cpu"] },
  "args": { "strategy": "random" }
}
```

| Field | Rust type | Required | Notes |
| --- | --- | --- | --- |
| `version` | `u32` | yes | Should equal `1` |
| `kind` | `String` | yes | Should equal `config_provider` |
| `config` | `serde_json::Value` | yes | Fully resolved config (default -> file -> theme merge) |
| `args` | `Option<serde_json::Value>` | no | Provider-specific parameters |

Response:

```json
{ "config": { "show_colors": false, "modules": ["cpu", "memory"] } }
```

Contract caveats:

- `ConfigProviderRequest::new(config, args)` always sets version `1` and kind
  `config_provider`.
- `xfetch-extension-api` has **no** `validate()` and **no** read/write
  entrypoints. The xfetch core (`src/extensions/runner.rs`) deserializes the
  request/response with serde directly and does not check `version` or `kind` on
  the response.
- If an extension fails, the core keeps the previous config.

## Native timeout contract

The same helper exists in `plugin-api`, `effect-api` and `extension-api`:

```rust
pub fn with_timeout<T: Send + 'static>(
    budget: Duration,
    task: impl FnOnce() -> T + Send + 'static,
) -> Result<T, TimedOut>;

pub struct TimedOut;
```

- On native targets the task runs on a worker thread and
  `rx.recv_timeout(budget)` returns `Err(TimedOut)` when the budget elapses.
- On `target_arch = "wasm32"` the task runs inline and the helper always returns
  `Ok(task())`; the host enforces the deadline with wasmtime epochs. The same
  source compiles for both.
- The worker thread is not joined; when `TimedOut` is returned the addon should
  emit fallback output and exit, which terminates the worker with the process.
- The core also has an independent safety kill (`timeout_secs`). Native addons
  default to 30 seconds when unset; `0` disables the core cap. The addon's own
  `with_timeout` budget is the primary control.

## WebAssembly core-module bridge (`xfetch-guest-api`)

This bridge is for core modules only, not components. The host import lives in
module `xfetch`:

```text
host_call(op_ptr: *const u8, op_len: usize,
          args_ptr: *const u8, args_len: usize) -> u64
```

- The return value packs the response: low 32 bits are the pointer, high 32 bits
  are the length (`(len << 32) | ptr`).
- The host allocates response memory in the guest via the exported
  `xfetch_alloc(size) -> *mut u8` and expects the guest to release it with
  `xfetch_free(ptr, size)`.
- A return value of `0` means the host could not produce a buffer.
- `xfetch-guest-api` exports `xfetch_alloc`/`xfetch_free` with alignment 1 on
  wasm32 and provides a native stub where every call returns
  `HostErrorKind::Unsupported`.
- `HOST_PROTOCOL_VERSION` is `1`; `protocol_version()` reads the `protocol`
  field of the `version` operation response.

### Wire envelope

```json
{ "ok": true,  "value": { } }
{ "ok": false, "error": { "kind": "denied", "message": "..." } }
```

Error kinds (stable wire labels): `denied`, `failed`, `timeout`, `too_large`,
`unsupported`. Unknown labels map to `failed`.

### Operations

Operation constants in `crates/guest-api/src/host.rs`: `OP_HTTP = "http"`,
`OP_EXEC = "exec"`, `OP_LOG = "log"`, `OP_VERSION = "version"`.

| Operation | Arguments | `value` result |
| --- | --- | --- |
| `http` | `{ "method", "url", "headers": [[name, value], ...], "body_base64"?, "timeout_ms"? }` | `{ "status", "headers": [[name, value], ...], "body_base64" }` |
| `exec` | `{ "program", "args": [string], "stdin_base64"?, "env"?, "timeout_ms"? }` | `{ "code", "stdout_base64", "stderr_base64" }` |
| `log` | `{ "level", "message" }` | `{}` (best-effort; failures ignored by `log`) |
| `version` | `{}` | `{ "runtime", "protocol", "xfetch", "guest_kind" }` |

Binary data is standard base64 with padding. `HttpResponse`, `ExecResult` and
the base64 encode/decode helpers are implemented in `crates/guest-api/src/ops.rs`.
HTTP redirects are followed manually and re-checked against the manifest
allowlist on every hop. `exec` passes argv verbatim, never uses a shell, and
clears the environment except for allowlisted names.

## Component contract (`wit/xfetch-runtime.wit`)

Package: `xfetch:runtime@0.1.0`.

Three worlds, all importing the same `host` interface and exporting the same
entrypoint:

```wit
world plugin {
    import host;
    export run: func(request: string) -> result<string, string>;
}

world effect {
    import host;
    export run: func(request: string) -> result<string, string>;
}

world extension {
    import host;
    export run: func(request: string) -> result<string, string>;
}
```

The `host` interface:

```wit
fetch: func(req: http-request) -> result<http-response, host-error>;
exec: func(req: exec-request) -> result<exec-result, host-error>;
log: func(level: string, message: string);
protocol-version: func() -> u32;
```

Typed records in `interface types`:

- `http-request { method: string, url: string,
  headers: list<tuple<string,string>>, body: option<list<u8>>,
  timeout-ms: option<u64> }`
- `http-response { status: u16, headers: list<tuple<string,string>>,
  body: list<u8> }`
- `exec-request { program: string, args: list<string>,
  stdin: option<list<u8>>, env: list<tuple<string,string>>,
  timeout-ms: option<u64> }`
- `exec-result { code: s32, stdout: list<u8>, stderr: list<u8> }`
- `host-error` variant: `denied(string)`, `failed(string)`, `timeout(string)`,
  `too-large(string)`, `unsupported(string)`.

Unlike the JSON bridge, component bodies are raw `list<u8>` and status/exit
codes are typed integers. The request/response strings are still the same JSON
documents documented above. The xfetch core detects core modules versus
components from the 8-byte wasm header, not the file extension. `xfetch wasm
wit` prints the vendored copy.

## Capabilities and limits

Capabilities are deny-by-default for wasm guests. Native addons run as ordinary
unsandboxed processes with the invoking user's privileges and are not governed
by these manifests.

Manifest schema version is `1`. Resolution order for an installed artifact:

1. sidecar `<stem>.json` (for example
   `xfetch-effect-wasm-matrix.wasm` -> `xfetch-effect-wasm-matrix.json`);
2. custom section `xfetch:manifest` embedded in the wasm binary;
3. empty restrictive default.

Capabilities:

| Field | Semantics |
| --- | --- |
| `http.allow` | Glob patterns matched against the full URL; redirects re-checked per hop |
| `exec.allow` | Program filename patterns; argv verbatim, no shell |
| `exec.env` | Extra environment names the guest may forward to the child |
| `fs` | WASI preopens: a string is a read-only host path; an object has `host`, `guest`, `mode` (`ro`/`rw`) |
| `env` | Environment variable names exposed to the guest; `["*"]` forwards all (discouraged) |
| `args` | When true the guest receives `argv[0]`; default false |

`log` and `version` are always available.

Limits and defaults (`src/wasm/manifest.rs` in the core; same names in the
source manifests in addon repos):

| Limit | Default | Meaning |
| --- | --- | --- |
| `timeout_ms` | `30000` | Wall-clock deadline for the whole invocation; a configured `timeout_secs` overrides it |
| `memory_mb` | `256` | Linear memory cap per memory |
| `output_kb` | `4096` | Cap for the JSON stdout response |
| `host_call_kb` | `4096` | Cap for a single host-call response |

Timeout resolution in the core:

- Native: `timeout_secs` absent -> 30 s; `0` -> no cap; otherwise that many
  seconds. The whole process tree is killed on expiry.
- Wasm: `timeout_secs` absent -> manifest `timeout_ms`; when present the raw
  value is converted to `Duration`. Implementation caveat: for wasm,
  `timeout_secs: 0` becomes a zero-length deadline (not a disabled cap) because
  the wasm path uses `timeout_secs.map(Duration::from_secs)`. Omit the field to
  use the manifest limit.

## Consumers and implementation sites

| Contract | Implemented in `api` | Consumed by |
| --- | --- | --- |
| Plugin JSON | `crates/plugin-api` | `xfetch/src/plugins/run.rs`; `xfetch-cli/plugins` crates; wasm guest sources |
| Effect JSON | `crates/effect-api` | `xfetch/src/effects/mod.rs`; `xfetch-cli/effects` crates |
| Extension JSON | `crates/extension-api` | `xfetch/src/extensions/runner.rs`; `xfetch-cli/extensions` crates |
| Guest JSON bridge | `crates/guest-api` + `wit` docs | `xfetch-cli/plugins` (`wasm-crypto`), extensions (`wasm-night-mode`, `wasm-updates-footer`) |
| Component WIT | `wit/xfetch-runtime.wit` | `xfetch/src/wasm/runner/component_guest.rs` (vendored copy at `xfetch/wit/xfetch-runtime.wit`); Python components in plugins/effects/extensions repos |
| Manifest schema | documented here | `xfetch/src/wasm/manifest.rs`; `xfetch-effect.json` / `xfetch-plugin.json` / `xfetch-extension.json` in addon repos |

## Compatibility summary

| Thing | Current value |
| --- | --- |
| Native/core-module JSON protocol | `1` |
| Extension protocol constant | `1` (`EXTENSION_PROTOCOL_VERSION`, `types` module) |
| Guest host protocol | `1` (`HOST_PROTOCOL_VERSION`) |
| WIT package | `xfetch:runtime@0.1.0` |
| Manifest schema | `1` (`MANIFEST_VERSION`, core) |
| SDK crates | `0.3.0` (release/v0.3.0) |

Rules for changing a contract:

1. Change the canonical definition here first, including docs and `CHANGELOG.md`.
2. Keep old peers working: add optional/defaulted fields, do not rename or
   retype existing wire fields.
3. For WIT, update `wit/xfetch-runtime.wit` and then sync the vendored core
   copy; the core test fails on drift.
4. Bump the affected crate version according to semver. A protocol version bump
   is a breaking change for every addon.
5. Publish the crate, then update consumers (`xfetch` and the addon repos). The
   core and official addons still pin `0.2.0` from crates.io until they adopt
   the additive `0.3.0` release; the wire protocol is unchanged.

## Gotchas

- Do not print anything except the response JSON to stdout. Use stderr for
  diagnostics.
- `read_json_from_stdin` uses `read_to_string`, so it waits for EOF; the core
  closes stdin after writing the payload.
- `EffectRequest.args` and `LogoAnimationRequest.args` are optional on the wire
  as of 0.3.0 (`serde(default)`); the core still always includes them.
- The response objects have no `version`/`kind`; do not add them and expect
  consumers to read them.
- The core does not validate extension requests/responses; the 0.3.0 SDK adds
  `ConfigProviderRequest::validate()`, but the core still does not enforce it.
  Extension authors must be defensive.
- `loop` is the wire name for `LogoAnimationArgs::loop_enabled`.
- Base64 in the guest bridge is standard padded base64, not URL-safe.
- The component `fetch` import maps to the JSON operation named `http`, not
  `fetch`.
- `log` in the guest bridge intentionally ignores failures.
- Native addons are unsandboxed. Only wasm guests get capability enforcement.
- The SDK crates are platform-neutral; adding OS-specific code or heavy
  dependencies breaks the wasm guest story.

Last verified: 2026-11-03 against `api@cb70984f001d03ebcab3ef033660c8ba6d8fea6f` (`main`); `release/v0.3.0` is at `de5590d` and was merged.
