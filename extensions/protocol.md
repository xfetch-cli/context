# extensions protocol

Wire protocol, manifest schema and wasm contract for
`xfetch-cli/extensions`. The canonical types live in
`xfetch-cli/api` (`crates/extension-api/src/types.rs`); the core sends them
from `xfetch/src/extensions/runner.rs`.

## Version and framing

- Protocol version: `1` (`EXTENSION_PROTOCOL_VERSION`).
- Kind: `config_provider` (`KIND_CONFIG_PROVIDER`).
- Transport: one JSON object on stdin, one JSON object on stdout.
- The core writes the request and closes/writes stdin, then waits for the
  process to exit (or the deadline to fire).
- The response must be exactly one JSON object on stdout. Anything else
  (logs, banners, multiple objects) makes the core fail to parse the
  response; that provider is skipped.
- Diagnostics go to stderr. The core does not surface them in normal
  operation; a failed provider is silently ignored.
- Unknown request fields are ignored by Serde. The `config` field is carried
  as `serde_json::Value`, so a provider can preserve fields it does not
  understand. Typing the config into a narrower struct would drop fields;
  the official providers do not do that.

There is no handshake and no version negotiation. The core always sends
version 1 and kind `config_provider`; most providers do not validate either
field. The Rust reference providers check `kind` and exit non-zero on a
mismatch.

## Request

```json
{
  "version": 1,
  "kind": "config_provider",
  "config": { ... fully resolved config ... },
  "args": { ... extension-specific ... }
}
```

Rust type:

```rust
pub struct ConfigProviderRequest {
    pub version: u32,
    pub kind: String,
    pub config: serde_json::Value,
    pub args: Option<serde_json::Value>,
}
```

| Field | Type | Meaning |
| --- | --- | --- |
| `version` | number | `1`. |
| `kind` | string | `config_provider`. |
| `config` | object | The current effective config after defaults, config file and theme merge. |
| `args` | object or null | The `args` object from the `config_providers` entry. |

The `config` value is the serialized `Config` struct, so it contains defaults
for unset fields, not just the raw file. It includes the `config_providers`
array itself.

## Response

```json
{
  "config": { ... modified config ... }
}
```

Rust type:

```rust
pub struct ConfigProviderResponse {
    pub config: serde_json::Value,
}
```

The response does not echo `version` or `kind`. The core reads only
`config` and deserializes it back into `Config`. Unchanged fields must be
preserved in the returned object; the core does not merge the response with
the request. Official providers clone the request `config` and mutate the
fields they care about.

## Ordering and failure semantics

From `xfetch/src/config.rs`:

1. Defaults -> config file -> theme merge produce the initial config.
2. The `config_providers` array is cloned into a local list.
3. Providers run in declaration order. Provider N receives the output of
   provider N-1.
4. A provider that errors (missing binary, non-zero exit, timeout, invalid
   JSON) is skipped; the previous config continues to the next provider.
5. After the last provider, the accumulated value is deserialized into
   `Config`. If that fails, the entire provider chain is discarded and the
   pre-provider (theme-merged) config is used.

Additional constraints:

- Providers only run if a config file exists and parses. No config file means
  `Config::default()` is returned before the provider loop.
- Extensions run after theme resolution. They cannot make the core re-merge a
  theme; they can only change fields.
- An extension cannot add or remove providers for the current run: the list
  was cloned before iteration. Editing `config_providers` in the returned
  config only affects a future `xfetch` invocation (if the value is written
  back to disk by the user, which the core does not do).
- Provider errors are silent. There is no user-facing warning or fallback
  line.

## Extension-specific `args`

| Extension | Args | Defaults |
| --- | --- | --- |
| `config-roulette` | `routes` (string path to a JSON routes file), `strategy` (`"random"` or `"daily"`). | `routes` `~/.config/xfetch/routes.json`; `strategy` `"daily"`. |
| `layout-override` | `layout` (string), `modules` (string[]). | Both absent: original values preserved. |
| `wasm-night-mode` | `night_start` (hour 0-23), `night_end` (hour 0-23), `utc_offset` (hours from UTC), `night_palette` (string), `dim_colors` (bool). | 22, 7, 0, unset, true. |
| `wasm-updates-footer` | `program` (allowlisted program), `args` (string[]), `prefix` (string), `up_to_date` (string). | `checkupdates`, `[]`, `" · "`, `"up to date"`. |
| `wasm-lang-labels` | `language` (string; `es`, `en`, `de` included). | Detect from `LC_ALL`, `LC_MESSAGES`, `LANG`; fallback `en`. |

### config-roulette details

The routes file is a JSON array:

```json
[
  { "_name": "default-full", "path": "~/.config/xfetch/fetchs/001/config.jsonc" },
  { "_name": "tree-view", "path": "~/.config/xfetch/fetchs/004/config.jsonc" }
]
```

`_name` is optional and unused. The chosen route's file is read, parsed as
JSON (JSONC comments are not stripped by the extension itself), and returned
as the entire config. `strategy: "random"` uses `SystemTime` sub-second
nanoseconds modulo the route count; `"daily"` uses
`unix_seconds / 86400` modulo the count. `~` expands via `HOME`, with a
`USERPROFILE` fallback on Windows. Missing or invalid routes/config files
exit non-zero (and are silently skipped by the core).

### layout-override details

Only the fields present in `args` are replaced. `layout` is inserted as a
string; `modules` replaces the array with a new array of strings. The rest of
the config is preserved. This runs after theme merge, so it wins over both
the file and the theme.

### wasm-night-mode details

The extension computes the local hour as
`(unix_seconds + utc_offset * 3600) / 3600 % 24` with no timezone database.
The night window is `[night_start, night_end)`; when `night_start >
night_end` it wraps midnight. At night it sets `show_colors: false` when
`dim_colors` is true, and sets `palette_style` when `night_palette` is
provided. During the day the config is returned untouched.

### wasm-updates-footer details

Runs `checkupdates` by default with `timeout_ms: 20000`. If that fails and
the program is still `checkupdates`, it falls back to `pacman -Qu`. It counts
non-empty stdout lines; exit codes 0, 1 and 2 are accepted because the tools
disagree about "up to date". It appends `<prefix><N> updates` (or
`<prefix><up_to_date>`) to `footer_text`. It is idempotent: if `footer_text`
already contains `"updates"` or the `up_to_date` label, it returns the config
unchanged. A missing tool is silent (config returned unchanged).

### wasm-lang-labels details

Reads `LC_ALL`, then `LC_MESSAGES`, then `LANG` (or the explicit
`language` arg), takes the first two letters lowercased, and fills
`config.labels` with translations for `es`, `en` or `de`. Existing labels
are preserved, including intentionally empty strings, via `setdefault`.

## Wasm manifest

Native extensions have no manifest. Their `Cargo.toml` metadata is used for
building and for the catalog; the core treats every non-wasm artifact as an
unsandboxed native process.

A wasm extension source contains `xfetch-extension.json` (or
`extension.json`). The installer copies it to the extensions directory as
`xfetch-extension-<name>.json`. Runtime resolution order:

1. sidecar `xfetch-extension-<name>.json`;
2. embedded custom section `xfetch:manifest`;
3. restrictive defaults.

Schema and capability semantics are the same as plugins. The short version:

```json
{
  "manifest_version": 1,
  "name": "wasm-updates-footer",
  "version": "0.2.0",
  "description": "Appends the pending package-update count to the footer (Go WebAssembly)",
  "kind": "config_provider",
  "runtime": "core",
  "build": "GOOS=wasip1 GOARCH=wasm go build -o dist/wasm-updates-footer.wasm .",
  "artifact": "dist/wasm-updates-footer.wasm",
  "capabilities": {
    "exec": { "allow": ["checkupdates", "pacman"] }
  },
  "limits": {
    "timeout_ms": 30000,
    "memory_mb": 256,
    "output_kb": 512,
    "host_call_kb": 4096
  }
}
```

### Capabilities

Deny-by-default. For extensions the useful capabilities are:

| Capability | Field | Extension use |
| --- | --- | --- |
| Environment | `env: ["LANG", ...]` | `wasm-lang-labels` reads the locale variables. `["*"]` is allowed but discouraged. |
| Processes | `exec.allow: ["checkupdates", "pacman"]` | `wasm-updates-footer` runs package queries. `exec.env` optionally forwards variables; argv is verbatim and no shell is used. |
| Files | `fs` | Read or write config/theme directories. Strings mount read-only; objects set `host`, `guest`, `mode` (`ro`/`rw`). |
| HTTP | `http.allow` | Fetch remote data. Glob patterns match the full URL; redirects are rechecked per hop. |
| Args | `args: true` | Expose the guest name as `argv[0]`. |

`log` and `version` host operations are always available.

### Limits

| Field | Default | Meaning |
| --- | --- | --- |
| `timeout_ms` | 30000 | Whole-invocation deadline; a config `timeout_secs` overrides it. |
| `memory_mb` | 256 | Linear memory cap per memory. |
| `output_kb` | 4096 | Response stdout cap. |
| `host_call_kb` | 4096 | One host-call response cap. |

### Official manifests

| Extension | Runtime | Build command | Capabilities | Limits (ms / MiB / KiB / KiB) |
| --- | --- | --- | --- | --- |
| `wasm-night-mode` | `core` | `cargo build --release --target wasm32-wasip1` | `{}` | 5000 / 64 / 512 / 64 |
| `wasm-updates-footer` | `core` | `GOOS=wasip1 GOARCH=wasm go build -o dist/wasm-updates-footer.wasm .` | `exec.allow: ["checkupdates", "pacman"]` | 30000 / 256 / 512 / 4096 |
| `wasm-lang-labels` | `component` | `componentize-py -d ../../../api/wit -w extension componentize app -p . -o dist/wasm-lang-labels.wasm` | `env: ["LANG", "LC_ALL", "LC_MESSAGES"]` | 20000 / 256 / 512 / 64 |

Artifacts:

```text
wasm-night-mode       ../../target/wasm32-wasip1/release/xfetch-extension-wasm-night-mode.wasm
wasm-updates-footer   dist/wasm-updates-footer.wasm
wasm-lang-labels      dist/wasm-lang-labels.wasm
```

## Core-module guest contract

Target `wasm32-wasip1`, exported `_start`. Request bytes on WASI stdin,
response bytes on WASI stdout. The response is the same
`{"config": ...}` object. stderr is forwarded to the terminal but capped at
256 KiB; it is not part of the protocol.

Core modules import the `xfetch.host_call` bridge. The ABI is identical to
plugins:

```wat
(func "host_call" (param i32 i32 i32 i32) (result i64))
```

The guest exports `xfetch_alloc(size) -> ptr`; the host writes the response
there and returns `(length << 32) | pointer`. Host responses are

```json
{ "ok": true,  "value": { } }
{ "ok": false, "error": { "kind": "denied", "message": "..." } }
```

with error kinds `denied`, `failed`, `timeout`, `too_large`, `unsupported`.
Operations are `http`, `exec`, `log`, `version`; see the plugins protocol
book for exact argument and result shapes.

`wasm-updates-footer` demonstrates the Go bridge:

```go
//go:wasmimport xfetch host_call
func hostCall(opPtr, opLen, argsPtr, argsLen uint32) uint64

//go:wasmexport xfetch_alloc
func xfetchAlloc(size uint32) uint32
```

and `decodeBase64` in `base64.go` decodes the `stdout_base64` field. The
manifest allows only `checkupdates` and `pacman`.

## Component guest contract

Target: component model, world `extension` from
`api/wit/xfetch-runtime.wit`:

```wit
world extension {
    import host;
    export run: func(request: string) -> result<string, string>;
}
```

`run` receives the same JSON request and must return the JSON response
string. Returning `err(string)` fails the invocation. The imported
`xfetch:runtime/host` interface provides `fetch`, `exec`, `log` and
`protocol-version`; WASI preview 2 is linked, so Python and JavaScript
runtimes have clocks, randomness and preopened filesystems.

`wasm-lang-labels` imports `host` and calls `host.log`; it does not call
`fetch` or `exec`. Build with `-w extension`.

Guest structured logs:

```text
[wasm-lang-labels] [info] applying 'es' labels
```

The default threshold shows only `warn` and `error`. Set
`XFETCH_WASM_LOG_LEVEL=off|error|warn|info|debug` to change it. Raw stderr is
always forwarded; only host logs are filtered.

## Examples

### layout-override

Request:

```json
{
  "version": 1,
  "kind": "config_provider",
  "config": {
    "layout": "pacman",
    "modules": ["os", "kernel", "uptime"],
    "show_colors": true
  },
  "args": {
    "layout": "tree",
    "modules": ["os", "kernel", "uptime", "packages", "shell", "cpu"]
  }
}
```

Response:

```json
{
  "config": {
    "layout": "tree",
    "modules": ["os", "kernel", "uptime", "packages", "shell", "cpu"],
    "show_colors": true
  }
}
```

### config-roulette

Request:

```json
{
  "version": 1,
  "kind": "config_provider",
  "config": {"show_colors": true, "modules": ["os"]},
  "args": {"routes": "~/.config/xfetch/routes.json", "strategy": "random"}
}
```

Response (the entire chosen config file, not a patch):

```json
{
  "config": {
    "layout": "tree",
    "modules": ["os", "kernel", "packages"],
    "theme": "dracula",
    "show_colors": true
  }
}
```

### wasm-night-mode

Request with a night window:

```json
{
  "version": 1,
  "kind": "config_provider",
  "config": {"show_colors": true, "palette_style": "solid", "footer_text": "arch"},
  "args": {"night_start": 22, "night_end": 7, "utc_offset": 2, "night_palette": "dots"}
}
```

At 23:00 local time the response is:

```json
{
  "config": {"show_colors": false, "palette_style": "dots", "footer_text": "arch"}
}
```

During the day the config is returned unchanged.

### Direct testing

```bash
echo '{"version":1,"kind":"config_provider","config":{"layout":"pacman"},"args":{"layout":"tree"}}' \
  | ./target/release/xfetch-extension-layout-override
```

Expected:

```json
{"config":{"layout":"tree"}}
```

For wasm:

```bash
xfetch wasm run ./xfetch-extension-wasm-night-mode.wasm \
  --kind extension \
  --request '{"version":1,"kind":"config_provider","config":{"show_colors":true},"args":{}}'
```

## Compatibility

- Protocol version 1 and kind `config_provider` are stable through
  `xfetch-extension-api 0.2.0` and core `1.0.2`.
- The request `config` is a JSON value, not a fixed schema at the protocol
  layer. Providers that preserve unknown fields keep working when the core
  adds config fields; providers that deserialize into a narrower struct do
  not.
- The core does not validate an extension's response beyond deserializing
  `config` into `Config`. A wrong shape discards the whole provider chain.
- Native providers are unsandboxed and can read/write any user-accessible
  path. Wasm providers are deny-by-default and limited by the manifest.
- `timeout_secs: 0` disables the cap for native providers but produces an
  immediate wasm deadline. Use a positive value or omit the field for wasm.
- A config entry with `timeout_secs` overrides the manifest `timeout_ms`
  rather than being clamped to it.
- Protocol changes should land in `xfetch-cli/api` first, then in the core
  and this repository together.

Last verified: 2026-11-03 against `extensions@9251eecff5a48c403955ae3b457093f9088df80f` (`main`).
