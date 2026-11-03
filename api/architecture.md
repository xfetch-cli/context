# API architecture

## Workspace layout

```text
api/
├── Cargo.toml                       # workspace: five crates, resolver 2
├── crates/
│   ├── common/                      # xfetch-api-common
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs               # re-exports
│   │       ├── protocol.rs          # JSON_PROTOCOL_VERSION + kind constants
│   │       ├── frame.rs             # shared Frame { delay_ms, lines }
│   │       ├── io.rs                # JSON stdin/stdout helpers + IoError
│   │       └── timeout.rs           # with_timeout / TimedOut
│   ├── plugin-api/                  # xfetch-plugin-api
│   │   ├── Cargo.toml
│   │   ├── examples/
│   │   │   ├── info-plugin.rs
│   │   │   └── logo-animation-plugin.rs
│   │   └── src/
│   │       ├── lib.rs               # re-exports
│   │       ├── protocol.rs          # constants + request/response types
│   │       ├── entrypoints.rs       # read_* / write_* helpers
│   │       ├── io.rs                # wrappers over xfetch-api-common
│   │       ├── error.rs             # PluginApiError
│   │       └── timeout.rs           # re-export from xfetch-api-common
│   ├── extension-api/               # xfetch-extension-api
│   │   ├── Cargo.toml
│   │   ├── examples/
│   │   │   └── simple-extension.rs
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── types.rs             # config-provider request/response
│   │       ├── entrypoints.rs       # read_* / write_* helpers
│   │       ├── io.rs                # wrappers over xfetch-api-common
│   │       ├── error.rs             # ExtensionApiError
│   │       └── timeout.rs           # re-export from xfetch-api-common
│   ├── effect-api/                  # xfetch-effect-api
│   │   ├── Cargo.toml
│   │   ├── examples/
│   │   │   └── simple-effect.rs
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── protocol.rs
│   │       ├── entrypoints.rs
│   │       ├── io.rs
│   │       ├── error.rs
│   │       └── timeout.rs
│   └── guest-api/                   # xfetch-guest-api
│       ├── Cargo.toml
│       └── src/
│           ├── lib.rs               # public re-exports
│           ├── host.rs              # raw host_call ABI + protocol_version
│           ├── ops.rs               # http_request / exec / log wrappers
│           ├── alloc.rs             # xfetch_alloc / xfetch_free exports
│           └── error.rs             # HostCallError / HostErrorKind
├── wit/
│   └── xfetch-runtime.wit           # canonical component contract
├── docs/                            # authored documentation (see below)
├── scripts/                         # local CI scripts
└── .github/workflows/
    ├── rust-tests.yml               # fmt, clippy, tests, examples
    └── security-audit.yml           # cargo audit
There is no `src/` at the repository root and no binary target. Every deliverable
is a library crate plus the WIT file.

## Crate map

| Crate | Path | Purpose | Key public surface |
| --- | --- | --- | --- |
| `xfetch-api-common` | `crates/common` | Shared primitives for all addon SDKs | `JSON_PROTOCOL_VERSION`, `KIND_*`, `Frame`, `IoError`, `with_timeout`, `TimedOut`, `is_current_version` |
| `xfetch-plugin-api` | `crates/plugin-api` | Info providers and logo animations | `PROTOCOL_VERSION`, `KIND_*`, `PluginKind`, `LogoAnimationRequest/Response`, `AnimationFrame`, `LogoAnimationArgs`, `InfoPluginRequest/Response`, `EmptyArgs`, read/write entrypoints, `with_timeout`, `TimedOut`, `PluginApiError`, shared `Frame` conversions |
| `xfetch-extension-api` | `crates/extension-api` | Config-provider extensions | `ConfigProviderRequest/Response`, `ExtensionKind`, `PROTOCOL_VERSION`, `EXTENSION_PROTOCOL_VERSION`, `KIND_CONFIG_PROVIDER`, `ExtensionApiError`, read/write entrypoints, `io`, `with_timeout`, `TimedOut` |
| `xfetch-effect-api` | `crates/effect-api` | Intro effects over rendered content lines | `PROTOCOL_VERSION`, `KIND_EFFECT`, `EffectKind`, `EffectRequest`, `EffectArgs`, `EffectFrame`, `EffectResponse`, read/write entrypoints, `with_timeout`, `TimedOut`, `EffectApiError`, shared `Frame` conversions |
| `xfetch-guest-api` | `crates/guest-api` | Host calls for WebAssembly core-module guests | `host_call`, `http_request`, `exec`, `log`, `protocol_version`, `HttpResponse`, `ExecResult`, `HostCallError`, `HostErrorKind`, `HOST_PROTOCOL_VERSION`, `Value`, `json` |

All three addon SDKs now provide full `read_*`/`write_*` entrypoints,
validation and a typed error. `xfetch-api-common` owns the shared JSON IO,
timeout, frame and protocol constants; the per-kind crates keep their own
request/response types. `guest-api` is guest-side only: the xfetch core does
not depend on it.

## Public dependency graph

```text
xfetch core ───────────────► xfetch-plugin-api ──► xfetch-api-common
                             xfetch-extension-api ─► xfetch-api-common
                             xfetch-effect-api ────► xfetch-api-common

plugins repo ──────────────► xfetch-plugin-api
                             xfetch-guest-api (wasm guests only)

effects repo ──────────────► xfetch-effect-api
                             xfetch-effects-lib (effects repo)

extensions repo ───────────► xfetch-extension-api
                             xfetch-guest-api (wasm guests only)

component guests ──────────► wit/xfetch-runtime.wit
```

The per-kind crates depend on `xfetch-api-common`; `guest-api` is independent.
The dependencies are only `serde`/`serde_json` (and `base64` 0.23 in
`guest-api`), so they stay platform-neutral and compile for native and
`wasm32-wasip1` targets.

## Data flow

### Native addon (plugin, effect or extension)

1. The xfetch core builds a request struct from the SDK and serializes it with
   `serde_json::to_vec` (compact JSON, no trailing newline).
2. The core spawns `xfetch-<kind>-<name>` and writes the payload to the child
   stdin, then closes stdin.
3. The child reads stdin to EOF, validates the request, computes its result and
   writes one JSON response to stdout.
4. The core parses the entire stdout as one JSON document, validates the
   response where required, and uses it. Diagnostics go to stderr; a non-zero
   exit makes the core skip the addon.

### WebAssembly core module

Same JSON contract. The core writes the request into WASI stdin and reads WASI
stdout. The guest may additionally import `xfetch.host_call`; the host
allocates response memory through the guest's exported `xfetch_alloc`, writes
the JSON response there and returns a packed pointer/length. `log` and
`version` are always available; `http`, `exec`, filesystem, environment and
args are deny-by-default and require manifest allowlists.

### WebAssembly component

The component exports `run(request: string) -> result<string, string>` from the
`plugin`, `effect` or `extension` world in `wit/xfetch-runtime.wit`. The request
and response are the same JSON strings as the native protocol. Host operations
are typed WIT imports (`fetch`, `exec`, `log`, `protocol-version`) instead of
the JSON bridge. WASI preview 2 is linked as well.

## Key files and their responsibilities

| File | Responsibility |
| --- | --- |
| `crates/plugin-api/src/protocol.rs` | Protocol version/kind constants, request/response structs, `validate`, typed `args` parsing |
| `crates/plugin-api/src/entrypoints.rs` | `read_logo_animation_request`, `read_info_plugin_request`, `read_info_plugin_args_or_default`, `write_logo_animation_frames`, `write_info_lines` |
| `crates/plugin-api/src/io.rs` | `parse_json_str/slice`, `to_json_vec`, `read_json_from_stdin`, `write_json_to_stdout` |
| `crates/plugin-api/src/timeout.rs` | Re-export of `xfetch-api-common` timeout helper |
| `crates/common/src/protocol.rs` | Shared `JSON_PROTOCOL_VERSION` and kind constants |
| `crates/common/src/frame.rs` | Shared `Frame` shape and JSON roundtrip |
| `crates/common/src/io.rs` | Shared JSON stdin/stdout helpers and `IoError` |
| `crates/common/src/timeout.rs` | Native worker-thread / wasm inline `with_timeout` |
| `crates/extension-api/src/types.rs` | `ConfigProviderRequest/Response`, `ExtensionKind`, `PROTOCOL_VERSION`, `parse_args`, `validate` |
| `crates/extension-api/src/entrypoints.rs` | Extension read/write entrypoints |
| `crates/extension-api/src/io.rs` | Extension IO wrappers over `xfetch-api-common` |
| `crates/extension-api/src/error.rs` | `ExtensionApiError` |
| `crates/effect-api/src/protocol.rs` | `EffectRequest/Args/Frame/Response`, validation |
| `crates/effect-api/src/entrypoints.rs` | `read_effect_request`, `write_effect_frames` |
| `crates/guest-api/src/host.rs` | `xfetch.host_call` ABI, operation constants, packed return decoding |
| `crates/guest-api/src/ops.rs` | Base64-aware HTTP/exec wrappers, best-effort `log` |
| `crates/guest-api/src/alloc.rs` | `#[unsafe(no_mangle)] xfetch_alloc` / `xfetch_free` on wasm32 |
| `wit/xfetch-runtime.wit` | Canonical component-model contract and host interface |
| `docs/protocol.md` | Human-readable wire reference and JSON examples |
| `docs/timeouts.md` | Timeout contract and fallback guidance |

`docs/` currently contains: `README.md`, `getting-started.md`,
`plugin-sdk.md`, `extension-sdk.md`, `effect-sdk.md`, `protocol.md`,
`timeouts.md`, `wasm-guests.md`, `examples.md`.

## Versioning rules

These are the repository's own rules from `CONTRIBUTING.md`, plus the constants
as implemented:

- **Crate semver.** Breaking protocol changes bump the major version of the
  affected crate. Additive changes (new fields, new constants) must stay
  backward compatible with the wire format.
- **Wire version.** `xfetch-api-common::JSON_PROTOCOL_VERSION` is `1`; the
  per-kind `PROTOCOL_VERSION` constants (and `EXTENSION_PROTOCOL_VERSION`) alias
  it. Requests carrying any other version are rejected by the SDK entrypoints.
  Version is validated, not negotiated.
- **Kind constants.** `KIND_LOGO_ANIMATION = "logo_animation"`,
  `KIND_INFO_PROVIDER = "info_provider"`, `KIND_EFFECT = "effect"`,
  `KIND_CONFIG_PROVIDER = "config_provider"`. Never rely on Rust type names on
  the wire.
- **Additive fields.** New request/response fields should be `Option<T>` or
  have `#[serde(default)]` so old peers keep parsing. Unknown JSON fields are
  ignored by serde by default.
- **WIT.** The package version is `xfetch:runtime@0.1.0`. The xfetch core
  vendors a copy at `xfetch/wit/xfetch-runtime.wit`; its test
  `vendored_wit_matches_the_api_repository` fails when the two copies drift in a
  multi-repo checkout. Change the canonical file here first, then copy it to the
  core.
- **Documentation and changelog.** Every public item needs a doc comment. Add
  protocol changes to `docs/` and an entry to `CHANGELOG.md`.
- **Timeouts stay in the API.** `with_timeout`/`TimedOut` are the single
  enforcement point implemented by each addon crate; do not move that logic into
  consumer repositories.

## Tests and examples

- Unit tests live inline in `protocol.rs`, `entrypoints.rs` and `timeout.rs` of
  each crate.
- Doc tests exist in `plugin-api`, `effect-api` and `guest-api`.
- Runnable examples:
  - `crates/plugin-api/examples/info-plugin.rs`
  - `crates/plugin-api/examples/logo-animation-plugin.rs`
  - `crates/effect-api/examples/simple-effect.rs`
  - `crates/extension-api/examples/simple-extension.rs`
- `rust-tests.yml` runs fmt, clippy, workspace tests and example builds.
- There are no benchmark, fuzz or integration-test directories.

Last verified: 2026-11-03 against `api@cb70984f001d03ebcab3ef033660c8ba6d8fea6f` (`main`); `release/v0.3.0` is at `de5590d` and was merged.
