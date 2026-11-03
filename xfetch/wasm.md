# WebAssembly runtime

xfetch can run wasm addons in two styles: typed **components** and legacy
**core modules**. Both are executed with wasmtime 50.0.1 and governed by a
manifest-derived policy.

## Canonical contract

- Canonical WIT: `xfetch-cli/api/wit/xfetch-runtime.wit`, package
  `xfetch:runtime@0.1.0`, worlds `plugin`, `effect`, `extension`.
- Vendored copy: `xfetch/wit/xfetch-runtime.wit`. A core test fails if the two
  copies drift when both repos are checked out together.
- SDK crates: `xfetch-plugin-api 0.2.0`, `xfetch-effect-api 0.2.0`,
  `xfetch-extension-api 0.2.0`, `xfetch-guest-api 0.2.0`.
- JSON protocol version is `1` for native addons and core-module guests.
  Components use typed WIT imports instead of JSON host calls.

## Module map (`src/wasm/`)

| File | Responsibility |
| --- | --- |
| `detect.rs` | Detect artifact kind (component vs core module) and entry. |
| `manifest.rs` | Manifest schema: version, kind, runtime, entry, capabilities, limits, artifact/build/URL, sha256. |
| `policy.rs` | Pure capability decisions: HTTP globs, exec basename + env, WASI env, fs preopens, args, host-call cap. |
| `engine.rs` | Process-wide engine, epoch interrupt ticker (10 ms), on-disk compilation cache. |
| `host/http.rs` | `http` host call (allowlisted URLs). |
| `host/exec.rs` | `exec` host call (allowlisted program, allowlisted env). |
| `host/mod.rs` | Host-call dispatch and JSON response envelopes. |
| `install.rs` | Wasm install paths, sha256 verification, sidecar handling. |
| `inspect.rs` | `xfetch wasm inspect` report. |
| `runner/component_guest.rs` | Typed component runner. |
| `runner/core_guest.rs` | Core-module runner and `xfetch.host_call` bridge. |
| `runner/mod.rs` | Runner dispatch. |

## Manifest

```jsonc
{
  "manifest_version": 1,
  "name": "example",
  "version": "0.2.0",
  "description": "Example addon",
  "kind": "info_provider",          // logo_animation | info_provider | effect | config_provider
  "runtime": "wasm",
  "entry": "run",
  "capabilities": {
    "http": { "allow": ["https://api.example.com/*"] },
    "exec": { "allow": ["curl"], "env": ["HTTP_PROXY"] },
    "fs": [
      "~/.config/xfetch",
      { "host": "~/.cache/xfetch", "guest": "/data", "mode": "rw" }
    ],
    "env": ["HOME"],
    "args": false
  },
  "limits": {
    "timeout_ms": 2000,
    "memory_mb": 64,
    "output_kb": 512,
    "host_call_kb": 64
  },
  "artifact": "example.wasm",        // checked-in artifact
  "artifact_url": "https://...",    // optional download
  "sha256": "64 hex chars"           // optional integrity pin
}
```

- `manifest_version` must be `1`.
- `kind` must match the command family (`plugin`, `effect`, `extension`).
- `runtime` selects native/wasm handling; wasm manifests use `wasm`.
- `entry` is the exported function for core modules/components.
- `sha256` is an integrity pin on the same origin as the artifact; a
  `--sha256` flag from a trusted separate channel is the real provenance pin.
- A checked-in artifact never falls back to an unverified `artifact_url`.

## Policy (`policy.rs`)

The policy is the single authority for host decisions; the runtime never
inspects the environment directly.

- `http.allow`: glob patterns matched against the full URL, e.g.
  `https://wttr.in/*`, `https://*.example.com/api/*`.
- `exec.allow`: glob patterns matched against the program **basename**
  (`curl` allows `/usr/bin/curl`).
- `exec.env`: environment variables forwarded to the child process.
- `env`: WASI environment variables exposed to the guest (`*` allows all).
- `fs`: `Path` entries are read-only preopens; `Mount { host, guest, mode }`
  chooses `ro`/`rw` and guest path. Leading `~` is expanded.
- `args`: whether `wasi:cli` arguments beyond `argv[0]` are exposed.
- `host_call_kb`: cap for a single host-call response.

## Engine and timeouts

- One `Engine` per process (`OnceLock`), with epoch interruption enabled.
- A detached ticker increments the epoch every 10 ms.
- `deadline_ticks()` rounds the timeout up; a store traps when the epoch
  catches up. This interrupts runaway guests deterministically.
- Native code is cached by wasmtime under `~/.cache/xfetch/wasmtime`, keyed by
  wasmtime version + module hash. If the cache cannot be set up, the engine
  falls back to a clean configuration.

## Guest bridge

### Core modules (`core_guest.rs`)

- Import: `xfetch.host_call(op_ptr, op_len, args_ptr, args_len)`.
- Return: packed `(len << 32) | ptr`.
- Host allocates via the guest's exported `xfetch_alloc`; guest frees with
  `xfetch_free`.
- Operations: `http`, `exec`, `log`, `version`.
- Envelope: `{"ok":true,"value":...}` or
  `{"ok":false,"error":{"kind","message"}}`.
- Binary payloads use padded standard base64.

### Components (`component_guest.rs`)

- Typed WIT imports from the `plugin`/`effect`/`extension` worlds.
- Binary payloads use raw `list<u8>`.
- `xfetch wasm run --kind plugin|effect|extension` selects the world.

## Limits and failure behavior

- `timeout_ms`: wall-clock cap; `0` becomes an immediate zero-length deadline
  for wasm (known inconsistency with native `0` = disable).
- `memory_mb`: wasmtime store memory cap.
- `output_kb`: max response size.
- `host_call_kb`: max single host-call response.
- Missing/invalid manifest, unreadable artifact, wrong kind, policy denial,
  trap, timeout, or oversized output are reported as errors by the CLI wasm
  tooling; inside the normal fetch, addon failures are non-fatal.

## Install and integrity

- Direct wasm installs copy the artifact and write `xfetch-<kind>-<name>.json`
  sidecar.
- Repo/source installs build the wasm target (`wasm32-wasip1` is the expected
  target) and install the resulting artifact + manifest.
- `--sha256` is validated (`hash.rs`) and enforced; it takes precedence over
  the manifest digest. Supplying it for a native source is a hard error.
- `install.rs` validates guest names and uses the shared atomic/install
  helpers.

## Inspect and run

```bash
xfetch wasm inspect ./foo.wasm
xfetch wasm inspect ./foo.wasm --json
xfetch wasm run ./foo.wasm --request '{"version":1,"kind":"info_provider"}'
xfetch wasm run ./foo.wasm --request-file req.json --timeout 2 --kind effect
xfetch wasm wit
```

`inspect` reports resolved kind, manifest, capabilities and limits without
executing. `run` is the author/CI path for raw request/response testing.

## Tests

- `tests/wasm_e2e.rs` covers: core module writes JSON, manifest limits, host
  call denial, non-wasm rejection, missing export error, infinite-loop timeout.
- Unit tests cover manifest parsing, policy globs, engine deadline rounding,
  host HTTP/exec, and install verification.
- `cargo test --locked` is the release gate; wasmtime is built with default
  features disabled except the required runtime/compiler bits.

## Known contract mismatches

- SDK crates are `0.2.0`; JSON protocol is `1`; WIT package is `0.1.0`;
  manifest schema is `1`.
- Effect catalog schema is `1`; `catalog_version` tracks the xfetch release
  train (`1.0.2`).
- Explicit wasm `timeout_secs: 0` currently means zero deadline, not
  "unlimited"; native `0` disables the cap.
