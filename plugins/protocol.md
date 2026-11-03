# plugins protocol

Wire protocol, manifest schema and WebAssembly guest contract for
`xfetch-cli/plugins`. The canonical protocol types live in
`xfetch-cli/api` (`crates/plugin-api/src/protocol.rs`); the core consumes
them in `xfetch/src/plugins/run.rs`.

## Version and framing

- Protocol version: `1` (`PROTOCOL_VERSION` in `xfetch-plugin-api`).
- Transport: one JSON object on stdin, one JSON object on stdout.
- The plugin reads stdin to EOF, processes, writes one response, and exits.
- Diagnostics go to stderr. A non-zero exit is a failure; the core reports
  the trimmed stderr, or `Plugin exited with error` if stderr is empty.
- Serde ignores unknown JSON fields. For Rust plugins that use
  `read_info_plugin_args_or_default` / `read_logo_animation_request`, a
  wrong value type for a known arg fails typed parsing and the plugin exits
  1; non-Rust guests parse the request with their own decoder and may be
  more permissive.
- `args` is free-form and plugin-specific. Each plugin defines its own
  `#[derive(Deserialize)] struct PluginArgs` with optional/default fields.

There is no handshake. The first byte the plugin sees is the request JSON.

## Common request fields

Every request has:

| Field | Type | Meaning |
| --- | --- | --- |
| `version` | number | Must be `1`. |
| `kind` | string | `info_provider` or `logo_animation` for plugins. |
| `args` | object or null | Plugin-specific configuration. |

Rust plugins using `xfetch-plugin-api` validate both `version` and `kind`
before the plugin logic runs:

```rust
pub const PROTOCOL_VERSION: u32 = 1;
pub const KIND_LOGO_ANIMATION: &str = "logo_animation";
pub const KIND_INFO_PROVIDER: &str = "info_provider";
```

A wrong version produces
`Unsupported protocol version: expected 1, found N`; a wrong kind produces
`Unexpected plugin kind: expected '<expected>', found '<found>'`. The
Python, Go and C examples read the JSON directly and do not enforce these
checks.

## info_provider

### Request

```json
{
  "version": 1,
  "kind": "info_provider",
  "args": {
    "location": "London",
    "format": "%C|%t|%w|%h|%p"
  }
}
```

`args` may be `null` or omitted for plugins that take no arguments. The
provided `InfoPluginRequest` type serializes `args` as `Option<Value>`.

### Response

```json
{
  "lines": [
    "+15°C Clear",
    "  Humidity: 60%",
    "  Wind: 15 km/h",
    "  Precipitation: 0%"
  ]
}
```

`InfoPluginResponse { lines: Vec<String> }`. Each element is one rendered
line. ANSI escape codes are allowed and are preserved by the core. The core
renders the lines as rows of the module key `plugin:<name>`.

An empty `lines` array is valid at the protocol level. Individual plugins use
fallback lines such as `Weather: timed out` or `Chocolatey: not installed`
so the user sees why data is missing.

## logo_animation

### Request

```json
{
  "version": 1,
  "kind": "logo_animation",
  "lines": ["__  __", " \\ \\/ /"],
  "frames": [
    ["frame 1 line 1", "frame 1 line 2"],
    ["frame 2 line 1", "frame 2 line 2"]
  ],
  "args": {
    "fps": 12,
    "duration_ms": 1200,
    "loop": false,
    "style": "sweep"
  }
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `lines` | string[] | The current ASCII logo, split into lines. |
| `frames` | string[][] or null | Only populated when the core loaded `frames_path` (for `style: "frame"` and banner plugins). |
| `args.fps` | number or null | Requested frames per second. |
| `args.duration_ms` | number or null | Requested total duration. Ignored by the core in daemon mode. |
| `args.loop` | boolean or null | Advisory only. The core daemon repeats frames; plugins do not have to loop. |
| `args.style` | string or null | Plugin-specific style name. |

`frames_path` parsing is done by the core, not the plugin: a single file is
split on lines containing exactly `===`, or an array of files supplies one
frame each. The resulting frame sets arrive in `frames`.

### Response

```json
{
  "frames": [
    {
      "delay_ms": 83,
      "lines": ["\u001b[31m_\u001b[0m\u001b[32m_\u001b[0m"]
    }
  ]
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `frames` | Frame[] | Required and must be non-empty. |
| `frames[].delay_ms` | number | Milliseconds before advancing. |
| `frames[].lines` | string[] | Rendered frame; ANSI allowed. |

`LogoAnimationResponse::validate()` rejects an empty `frames` array with
`Animation plugins must return at least one frame`.

## Plugin-specific `args`

Current argument structs, from the source. All fields are optional unless
noted.

| Plugin | Args |
| --- | --- |
| `animate-logo` | `fps` (u64, clamped 1-60, default 12), `duration_ms` (u64, default 1200 for non-frame styles), `style` (`sweep` default, `wave`, `rainbow`, `sparkle`, `breathing`, `frame`, `none`), `loop` ignored. |
| `banner` | `style` (`banner` default, `banner-above`, `banner-static`, `banner-N`). `frames_path` (core-side) supplies custom banner text. |
| `banner-text` | `path` (string, `~`/`~/` expanded), `text` (string fallback), `blank_line` (bool, default true). |
| `chocolatey` | none. |
| `display-resolution` | none. |
| `docker` | none. |
| `gauges` | `style` (`auto`/`slashes`/`hash`/`blocks`), `width` (3-40, default 10), `metrics` (subset of `memory`, `swap`, `disk`, `battery`), `include_tmpfs` (bool, default false), `glyphs` (`nerd`/`cjk`/`jp`). |
| `github-stats` | `username` (falls back to `GITHUB_USER`), `token`, `max_lines` (1-7). |
| `langs` | `repo`, `owner`, `limit` (default 5), `color` (bool default true), `include_forks` (bool default false), `info` (bool default true), `token` (also `GITHUB_TOKEN`, `GH_TOKEN`, then `gh auth token`). |
| `music-player` | none. |
| `temperature` | `unit` (`celsius` default or `fahrenheit`). |
| `theme-detection` | none. |
| `theme-manager` | `action` (`list` default, `search`, `info`, `install`), `name`, `query`, `registry`. |
| `timezone` | `format` (GNU `date` format, default `%Z %z`; ignored on Windows). |
| `user-info` | `show_groups` (bool, default false). |
| `weather` | `location`, `format` (default `%C|%t|%w|%h|%p`). |
| `wasm-crypto` | `assets` (string[], default `["BTC","ETH","SOL"]`, max 5 quoted), `currency` (default `USD`). |
| `wasm-ip-geo` | `fields` (subset of `ip`, `location`, `org`, `timezone`). |
| `wasm-pacman` | `samples` (number, default 3). |
| `wasm-proc` | none. |

## Errors and timeouts

Each plugin declares a runtime budget with
`xfetch_plugin_api::with_timeout(budget, task)`:

```rust
const BUDGET: Duration = Duration::from_secs(10);

let lines = match with_timeout(BUDGET, || {
    // read stdin, do the work, return Vec<String>
}) {
    Ok(lines) => lines,
    Err(_) => vec!["MyPlugin: timed out".to_string()],
};
```

The native helper runs the task on a worker thread and returns
`Err(TimedOut)` after the budget. The thread keeps running until the process
exits, which is immediate because the plugin writes its fallback response and
terminates. On `wasm32`, the helper runs the task inline and the host enforces
the deadline through wasmtime epochs.

Budgets in this repository:

| Plugin | Budget | Notes |
| --- | --- | --- |
| `animate-logo` | 10 s | CPU-bound; frames can be thousands. |
| `banner` | 3 s | Pure computation. |
| `banner-text` | 2 s | Local file/string. |
| `chocolatey` | 10 s | First `choco list` can be slow. |
| `display-resolution` | 2 s | Local probes. |
| `docker` | 3 s | Local daemon. |
| `gauges` | 3 s | `sysinfo` probes. |
| `github-stats` | 25 s | Four API calls through `curl`. |
| `langs` | 20 s | Two API calls. |
| `music-player` | 2 s | `mpc`/`playerctl`. |
| `temperature` | 2 s | Sysfs/WMI. |
| `theme-detection` | 2 s | `gsettings`/config files/registry. |
| `theme-manager` | 20 s | Registry fetch with `curl --max-time 15`. |
| `timezone` | 2 s | `date`/PowerShell. |
| `user-info` | 2 s | `id`/`getent`/`groups`. |
| `weather` | 15 s | `curl --max-time 10`. |
| `wasm-crypto` | 12 s | Per-request host timeout 4000 ms. |
| `wasm-ip-geo` | host 20 s | Python; no in-guest `with_timeout`; manifest `timeout_ms`. |
| `wasm-pacman` | host 20 s | Go; no in-guest `with_timeout`; manifest `timeout_ms`. |
| `wasm-proc` | host 3 s | C; no in-guest `with_timeout`; manifest `timeout_ms`. |

The core adds a second, independent safety net: `timeout_secs` on the config
entry. For native plugins, unset means 30 s and `0` disables the cap. For
wasm, unset uses the manifest `timeout_ms`, and `0` becomes an immediate
deadline (it does **not** disable the cap).

## Wasm source manifest

Native plugins have no manifest. Their `Cargo.toml` metadata is used for
building and for the catalog, not at runtime; the core treats every
non-wasm artifact as an unsandboxed native process.

A wasm plugin directory contains `xfetch-plugin.json` (or the generic
`plugin.json`). The installer copies it to the config directory as
`xfetch-plugin-<name>.json`. At runtime the manifest is resolved in this
order:

1. Sidecar JSON next to the artifact:
   `xfetch-plugin-<name>.wasm` -> `xfetch-plugin-<name>.json`.
2. Custom section embedded in the wasm binary under `xfetch:manifest`.
3. Restrictive defaults (no capabilities, safe limits).

A malformed sidecar is a hard error; a missing manifest is valid and grants
nothing.

### Schema

```json
{
  "manifest_version": 1,
  "name": "wasm-crypto",
  "version": "0.2.0",
  "description": "Crypto spot prices (Coinbase) fetched through the sandboxed HTTP host call",
  "kind": "info_provider",
  "runtime": "core",
  "entry": "run",
  "build": "cargo build --release --target wasm32-wasip1",
  "artifact": "../../target/wasm32-wasip1/release/xfetch-plugin-wasm-crypto.wasm",
  "artifact_url": "https://example.com/release/plugin.wasm",
  "sha256": "ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad",
  "capabilities": {
    "http": { "allow": ["https://api.coinbase.com/*"] },
    "exec": { "allow": ["pacman"], "env": ["PATH"] },
    "fs": [
      "~/.config/xfetch",
      { "host": "~/.cache/xfetch", "guest": "/data", "mode": "rw" }
    ],
    "env": ["HOME", "USER"],
    "args": false
  },
  "limits": {
    "timeout_ms": 15000,
    "memory_mb": 64,
    "output_kb": 64,
    "host_call_kb": 256
  }
}
```

| Field | Type | Default | Meaning |
| --- | --- | --- | --- |
| `manifest_version` | number | `1` | Schema version; `0` is normalized to `1`. |
| `name` | string | artifact stem | Guest name in diagnostics and logs. |
| `version` | string | none | Informational. |
| `description` | string | none | Informational. |
| `kind` | string | none | `info_provider`, `logo_animation`, `effect`, `config_provider`. |
| `runtime` | string | none | `core` or `component`; informational for execution (header detection wins), used by the catalog. |
| `entry` | string | `run` | Component export name; empty is normalized to `run`. |
| `artifact` | string | none | Relative path to a prebuilt artifact in the source directory. |
| `artifact_url` | string | none | Absolute URL of a prebuilt artifact for remote installs. |
| `build` | string | none | Host command run in the source directory when no artifact exists. |
| `sha256` | string | none | 64 hex chars, case-insensitive; checked before writing. |
| `capabilities` | object | empty | Deny-by-default allowlists. |
| `limits` | object | see below | Runtime bounds. |

Zeroed limit fields are normalized to the defaults. Unknown manifest fields
are ignored by Serde.

### Capabilities

| Capability | Field | Semantics |
| --- | --- | --- |
| HTTP | `http.allow` | Glob patterns (`*`, `?`) matched against the full URL. Redirects are followed manually and every hop is re-checked. |
| Processes | `exec.allow` | Patterns matched against the program file name. argv is passed verbatim; no shell. `exec.env` allowlists which variables may be forwarded to the child. |
| Filesystem | `fs` | A string mounts the expanded host path read-only at the same guest path. An object sets `host`, `guest`, `mode` (`ro` default, `rw`). `~` is expanded. |
| Environment | `env` | Variable names exposed to the guest. `["*"]` forwards everything (discouraged). |
| Arguments | `args` | Boolean, default false. When true, `argv[0]` is the guest name; otherwise argv is empty. |

`log` and `version` host operations are always available and need no
capability entry.

### Limits

| Field | Default | Meaning |
| --- | --- | --- |
| `timeout_ms` | `30000` | Wall-clock deadline for the whole invocation. A config `timeout_secs` overrides it. |
| `memory_mb` | `256` | Linear memory cap per memory. |
| `output_kb` | `4096` | Cap for the JSON response written to stdout. |
| `host_call_kb` | `4096` | Cap for one host-call response (HTTP body, exec stdout/stderr). |

### Manifests in this repository

| Plugin | Runtime | Kind | Build command | Capabilities | Limits (ms / MiB / KiB / KiB) |
| --- | --- | --- | --- | --- | --- |
| `wasm-crypto` | `core` | `info_provider` | `cargo build --release --target wasm32-wasip1` | `http.allow: ["https://api.coinbase.com/*"]` | 15000 / 64 / 64 / 256 |
| `wasm-ip-geo` | `component` | `info_provider` | `componentize-py -d ../../../api/wit -w plugin componentize app -p . -o dist/wasm-ip-geo.wasm` | `http.allow: ["https://ipapi.co/*"]` | 20000 / 256 / 128 / 256 |
| `wasm-pacman` | `core` | `info_provider` | `GOOS=wasip1 GOARCH=wasm go build -o dist/wasm-pacman.wasm .` | `exec.allow: ["pacman"]` | 20000 / 256 / 64 / 2048 |
| `wasm-proc` | `core` | `info_provider` | `sh build.sh` | `fs: [{host:"/proc", guest:"/proc", mode:"ro"}]` | 3000 / 16 / 16 / 16 |

The `artifact` fields are:

```text
wasm-crypto    ../../target/wasm32-wasip1/release/xfetch-plugin-wasm-crypto.wasm
wasm-ip-geo    dist/wasm-ip-geo.wasm
wasm-pacman    dist/wasm-pacman.wasm
wasm-proc      dist/wasm-proc.wasm
```

## Core-module guest contract

Target: `wasm32-wasip1` command. The artifact must export `_start`.

- Request bytes arrive on WASI stdin.
- Response bytes must be written to WASI stdout.
- Stderr is forwarded to the terminal (capped at 256 KiB) and is not part of
  the protocol.
- Filesystem, environment, sockets and processes are denied unless the
  manifest allows them.
- A `proc_exit(0)` (common in Go) is treated as success. A non-zero exit
  becomes `Wasm guest '<name>' exited with code N`.
- Empty stdout becomes `Wasm guest '<name>' produced no output`.

### `xfetch.host_call` ABI

Core modules import one function:

```wat
(func "host_call" (param i32 i32 i32 i32) (result i64))
```

The guest places the operation name and JSON arguments in linear memory and
must export:

```wat
xfetch_alloc(size: i32) -> i32
```

The host allocates the response there. The returned `i64` packs
`(length << 32) | pointer`. A return of `0` means allocation failed.

Every host response is one of:

```json
{ "ok": true,  "value": { } }
{ "ok": false, "error": { "kind": "denied", "message": "..." } }
```

Error kinds: `denied`, `failed`, `timeout`, `too_large`, `unsupported`.

| Operation | Arguments | Result |
| --- | --- | --- |
| `http` | `{ method, url, headers, body_base64?, timeout_ms? }` | `{ status, headers: [[name, value], ...], body_base64 }` |
| `exec` | `{ program, args, stdin_base64?, env?, timeout_ms? }` | `{ code, stdout_base64, stderr_base64 }` |
| `log` | `{ level, message }` | `{ }` |
| `version` | `{ }` | `{ runtime, protocol, xfetch, guest_kind }` |

Rust guests should use `xfetch-guest-api 0.2.0` (`http_request`, `exec`,
`log`, `protocol_version`); it implements the ABI and allocator exports. The
core does not depend on it.

The Go examples implement the bridge directly:

```go
//go:wasmimport xfetch host_call
func hostCall(opPtr, opLen, argsPtr, argsLen uint32) uint64

//go:wasmexport xfetch_alloc
func xfetchAlloc(size uint32) uint32
```

The C example declares the WASI preview 1 imports it uses
(`path_open`, `fd_read`, `fd_close`, `fd_write`) directly and exports
`_start`.

### Guest logs

`log` writes through the host with the guest name and level:

```text
[wasm-crypto] [info] fetching prices
```

By default only `warn` and `error` are shown. Set `XFETCH_WASM_LOG_LEVEL` to
`off`, `error`, `warn`, `info` or `debug` to change the threshold. Raw
stderr writes and panics are always forwarded; only structured host logs are
filtered.

## Component guest contract

Target: component model. The canonical contract is
`api/wit/xfetch-runtime.wit`, also embedded in the core and printable with
`xfetch wasm wit`.

Three worlds exist, all with the same shape:

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

The `run` request is the same JSON document described above. Returning
`err(string)` fails the invocation; the string reaches the core error path.

The imported `xfetch:runtime/host` interface exposes:

```wit
fetch: func(req: http-request) -> result<http-response, host-error>;
exec: func(req: exec-request) -> result<exec-result, host-error>;
log: func(level: string, message: string);
protocol-version: func() -> u32;
```

`http-request` carries `method`, `url`, `headers`, optional `body` bytes and
optional `timeout-ms`; `exec-request` carries `program`, `args`, optional
`stdin`, allowlisted `env` and `timeout-ms`. The host error variants are
`denied`, `failed`, `timeout`, `too-large`, `unsupported`.

WASI preview 2 is linked, so componentized Python and JavaScript runtimes
get clocks, randomness and preopened filesystems. Build with the matching
world flag: `-w plugin` for a plugin component (for example
`wasm-ip-geo`).

## Examples

### Weather info request/response

Request:

```json
{"version":1,"kind":"info_provider","args":{"location":"London"}}
```

Response:

```json
{"lines":["\u{e30d} +15°C Clear","  Humidity: 60%","  Wind: 15 km/h","  Precipitation: 0%"]}
```

### animate-logo request/response

Request:

```json
{
  "version": 1,
  "kind": "logo_animation",
  "lines": ["__  __", " \\ \\/ /"],
  "frames": null,
  "args": {"fps": 12, "duration_ms": 1200, "loop": false, "style": "sweep"}
}
```

Response (abridged):

```json
{
  "frames": [
    {"delay_ms": 83, "lines": ["\u001b[31m_\u001b[0m\u001b[32m_\u001b[0m", "\u001b[33m \u001b[0m..."]},
    {"delay_ms": 83, "lines": ["\u001b[32m_\u001b[0m\u001b[33m_\u001b[0m", "..."]}
  ]
}
```

### wasm-crypto manifest and host call

Manifest as shipped:

```json
{
  "manifest_version": 1,
  "name": "wasm-crypto",
  "version": "0.2.0",
  "description": "Crypto spot prices (Coinbase) fetched through the sandboxed HTTP host call",
  "kind": "info_provider",
  "runtime": "core",
  "build": "cargo build --release --target wasm32-wasip1",
  "artifact": "../../target/wasm32-wasip1/release/xfetch-plugin-wasm-crypto.wasm",
  "capabilities": {
    "http": { "allow": ["https://api.coinbase.com/*"] }
  },
  "limits": {
    "timeout_ms": 15000,
    "memory_mb": 64,
    "output_kb": 64,
    "host_call_kb": 256
  }
}
```

The Rust guest issues:

```rust
let response = http_request("GET", &url, &[], None, Some(4_000))?;
```

The host performs the request only if the URL matches
`https://api.coinbase.com/*`; any other origin returns `denied`.

## Compatibility

- Protocol version 1 has been stable through `xfetch-plugin-api 0.2.0` and
  core `1.0.2`.
- The wasm manifest schema is version 1; unknown fields are ignored and a
  missing manifest is valid with deny-by-default capabilities.
- A core built without the `wasm` feature (non-default build) still detects
  wasm artifacts but refuses to run them with an explicit error.
- Wasm compilation is cached under the platform cache directory
  (`~/.cache/xfetch/wasmtime` on Linux), keyed by wasmtime version and
  module hash. `WASMTIME_CACHE_DISABLE=1` bypasses it.
- Protocol changes are supposed to land in `xfetch-cli/api` first, then be
  adopted by the core and this repository together. Do not change request or
  response shapes in a plugin without checking the core parser and the API
  version.
- The installed sidecar manifest is authoritative at runtime; the catalog
  entry is not. `xfetch plugin install` re-reads the source manifest and
  copies it next to the artifact.

Last verified: 2026-11-03 against `plugins@76f94dd39bab21ceb0b3295e78db4cb5077145ed` (`main`).
