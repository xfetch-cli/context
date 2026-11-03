# xfetch architecture

This document maps the core binary from process start to printed output and
explains the shared infrastructure that addons and probes reuse.

## 1. Startup and command dispatch

`src/main.rs` is intentionally thin:

1. `Cli::parse()` (`src/cli.rs`) parses global flags and an optional
   subcommand.
2. `run(cli)` handles early-exit flags in this order:
   `--list-layouts` -> `--clean-cache` -> `--daemon-stop` ->
   `--daemon-live-stop` -> `--gen-config`.
3. Otherwise it dispatches `plugin`, `extension`, `theme`, `effects`, `wasm`
   or `update`, falling back to the normal fetch.
4. The normal fetch loads the config, optionally applies config providers
   (extensions), collects `Info`, renders, and plays effects/animations.
5. Every error returns `XFetchError` and funnels through `exit_on_error`,
   which prints `Error: ...` (or a feature-specific prefix) and exits 1.

Key files: `src/main.rs`, `src/cli.rs`, `src/error.rs`.

## 2. Configuration model

`src/config.rs` defines the `Config` struct and the JSONC loader.

- Config search order: `$XDG_CONFIG_HOME/xfetch/config.jsonc`, then the
  platform config dir (`~/.config/xfetch/config.jsonc` on Linux,
  `~/Library/Application Support/xfetch/...` on macOS, `%APPDATA%\xfetch\...`
  on Windows). `--config <path>` overrides.
- JSONC is parsed with `json_comments`; missing/unparseable config falls back
  to `Config::default()`.
- Theme resolution: `defaults -> config.jsonc -> <themes>/<name>.jsonc`
  (theme wins, since v0.2.0). Themes are partial JSONC files.
- Extensions (config providers) run after the theme merge, in declaration
  order. Each receives the full resolved config as JSON and returns a modified
  config. Provider failures are skipped; an invalid final config discards the
  provider chain.
- `generate_config()` writes the embedded `src/templates/config.jsonc`,
  optionally swaps the layout, and best-effort embeds a distro logo fetched
  from `xfetch-cli/logos`.

Important fields: `modules`, `layout`, `theme`, `ascii`/`logo_path`,
`show_colors`, `icons`, `colors`, `labels`, `formats`, `logo_*`,
`info_plugins`, `config_providers`, `effects`, `daemon_live*`, `custom_x`,
`disable_cache`, `disable_ip_fetching`, `os_wsl_style`.

## 3. Info collection pipeline

`src/info/mod.rs` builds `Info::with_config(config, benchmark)`.

1. Flatten configured module keys (including group modules) into a set.
2. Start only the expensive containers that are needed, concurrently:
   - `sysinfo::System` only for CPU/memory/swap;
   - `Disks` only for disk;
   - `Networks` only for network/IP modules.
3. Read the package and public-IP caches (300 s TTL) when caching is enabled.
4. Run independent probes in a second `thread::scope`: GPU, battery,
   datetime, packages, public IP, info plugins.
5. Assemble the `Info` struct, using `unknown()` / `N/A` when a probe is not
   requested or fails.
6. Return `bench_lines` when `--benchmark` is set.

The `Info` struct (`src/info/mod.rs`) is the single renderer input: `os`,
`kernel`, `host_name`, `shell`, `terminal`, `cpu`, `gpu`, `memory`, `swap`,
`disks`, `battery`, `uptime`, `packages`, `packages_breakdown`, `desktop`,
`user`, `datetime`, `local_ip`, `local_ip_v6`, `public_ip`,
`network_interfaces`, `plugin_info`.

Platform implementations live under `src/info/platform/`:

- `linux/`: battery, datetime, gpu, live, network, os_release, packages,
  wsl, and `xpm` (package-manager counts).
- `macos/`: battery, datetime, gpu, live, network, packages, software,
  version.
- `windows/`: battery, cpu, datetime, gpu, live, network, packages, process,
  shell, software, version.
- `shared/`: command runner re-export, GPU helpers, package parsers.
- `mod.rs`: selects the active platform and exposes `detect_os_ids`,
  `logo_category`.

`src/info/hardware.rs` and `src/info/software.rs` contain cross-platform
formatting helpers; `src/info/format.rs` handles byte/GiB and row formatting.

## 4. Renderer and UI

`src/ui.rs` is the public UI facade; `src/ui/` holds the implementation.

- `nodes.rs` converts `Info` + config into `RenderNode`s (key/value/icon,
  colors, labels, templates).
- `layout.rs` selects one of the built-in layouts:
  `default`, `side-block`, `tree`, `section`, `section-box`, `custom-x`,
  `compact`, `minimal`, and the classic variants `pacman`, `box`, `line`,
  `dots`, `bottom_line`, plus the vertical hints `horizontal`, `bottom`.
  `LAYOUT_NAMES` is the canonical list exposed by `--list-layouts`.
- `renders.rs` implements the classic/section/compact/minimal renderers;
  `custom_x.rs` implements the `custom-x` layout.
- `logo.rs` loads ASCII or image logos, applies colors/padding/scaling, and
  supports kitty image output.
- `print.rs` is the terminal writer: frame geometry, truncation, stacked
  output, animation frames, daemon rows, and terminal restore.
- `frames.rs` defines `AnimationFrame`; `effects` playback and logo animation
  both feed frames through the same printer.
- `daemon.rs` implements the animated-logo daemon; `live.rs` implements the
  live-stats daemon (top-pinned refresh, hot reload); `win_daemon.rs` is the
  Windows variant.
- `x.rs` contains the custom-X renderer state/palette helpers.

Effects are resolved and executed by `src/effects/`; the UI only plays the
returned frames. Effects are skipped when stdout is not a TTY or when an image
logo is being printed.

## 5. Addon subsystems

There are three addon kinds with a shared metadata/install layer:

| Kind | Runtime dir | Binary prefix | Config key | Purpose |
| --- | --- | --- | --- | --- |
| plugin | `~/.config/xfetch/plugins` | `xfetch-plugin-` | `info_plugins`, `logo_animation` | info lines / logo frames |
| effect | `~/.config/xfetch/effects` | `xfetch-effect-` | `effects` | intro animation frames |
| extension | `~/.config/xfetch/extensions` | `xfetch-extension-` | `config_providers` | config mutation |

Shared code under `src/addons/`:

- `model.rs`: `AddonKind`, `AddonRuntime`, `AddonEntry`, capability/limit
  formatting, config snippets.
- `install.rs`: direct wasm/URL installs, `--name`/`--manifest` handling,
  sidecar writing, name inference.
- `catalog.rs`: remote catalog fetch/search/JSON output for
  `search`/`available`; env overrides `XFETCH_PLUGIN_INDEX`,
  `XFETCH_EFFECT_INDEX`, `XFETCH_EXTENSION_INDEX`.
- `report.rs`: human-readable list/info output.

Per-kind code:

- `src/plugins/`: discovery (`PATH` then config dirs), native/wasm execution,
  install from local dir/repo/direct wasm, list/remove.
- `src/effects/`: discovery (config dirs then `PATH`), execution returning
  frames, install/list/remove.
- `src/extensions/`: discovery, config-provider runner, install/list/remove.

Native addons are unsandboxed processes. Wasm addons run under wasmtime with
a manifest-derived policy. The installed sidecar manifest is authoritative for
wasm capabilities/limits.

Remote installs use `src/install_support.rs` for path/name validation, unique
temp work dirs (`0700` on Unix), and `git clone --depth 1 --` with option
termination. Installed files are written atomically via `src/atomic_fs.rs`.

## 6. WebAssembly runtime

`src/wasm/` implements both guest styles:

- `manifest.rs`: manifest schema (version, kind, runtime, entry,
  capabilities, limits, artifact/artifact_url/build, sha256).
- `policy.rs`: pure capability decisions: HTTP glob allow, exec basename
  allow + env allowlist, WASI env allowlist, filesystem preopens (RO/RW),
  args exposure, host-call byte cap.
- `engine.rs`: process-wide wasmtime engine with epoch interruption, a 10 ms
  ticker, and an on-disk compilation cache at
  `~/.cache/xfetch/wasmtime`.
- `detect.rs`, `inspect.rs`, `install.rs`: artifact kind detection, manifest
  discovery, sha256 verification, sidecar install.
- `host/http.rs`, `host/exec.rs`, `host/mod.rs`: host-call implementations.
- `runner/component_guest.rs`: typed WIT component guests.
- `runner/core_guest.rs`: core modules using the JSON `xfetch.host_call`
  bridge (`host_call(op_ptr, op_len, args_ptr, args_len) -> packed ptr/len`).
- `runner/mod.rs`: shared runner dispatch.
- `tests/wasm_e2e.rs`: real wasm end-to-end tests (limits, timeout, host
  denial, inspect errors).

The canonical WIT lives in `xfetch-cli/api`; `wit/xfetch-runtime.wit` is a
vendored copy and a core test checks they match when the sibling clone is
present.

## 7. Update, cache and shared infrastructure

- `src/update/`: GitHub release client (`github.rs`), asset selection by
  target, checksum parsing, prebuilt install (`install.rs`), provenance
  classification (`provenance.rs`), and the CLI flow (`mod.rs`). Release assets
  are `xfetch-<version>-<target>.tar.gz` / `.zip` plus `SHA256SUMS` and a
  cosign bundle.
- `src/cache.rs`: JSON cache at `$XDG_CACHE_HOME/xfetch/cache.json`; used for
  package counts and public IP, 300 s TTL; `--clean-cache` removes it.
- `src/subprocess.rs`: shared spawn/timeout/pipe machinery. Drains pipes on
  detached threads, kills the process tree on deadline, can feed stdin for the
  addon JSON protocol, and skips unreachable PATH entries on WSL Windows
  mounts.
- `src/atomic_fs.rs`: atomic copy/write via same-directory temp + rename.
- `src/hash.rs`: SHA-256 hex helpers used by update and wasm installs.
- `src/logos.rs`: distro logo fetch/resolve/validate for `--gen-config`
  (64 KiB, 200 chars/line, control-character rejection).
- `src/themes/mod.rs`: local theme list/set/remove/export; edits the
  `"theme"` key in JSONC without reformatting the rest.
- `src/install_support.rs`: shared remote-install safety helpers.

## 8. Security boundaries

- Native addons: unsandboxed, user privileges; treat as trusted code.
- Wasm addons: deny-by-default policy from the manifest; HTTP/exec/fs/env/args
  must be explicitly granted; time/memory/output/host-call limits apply.
- Remote installs: validate guest names and repository strings, shallow clone
  with `--`, unique `0700` temp dirs, atomic destination writes.
- Wasm artifacts: optional/required SHA-256 verification on every install path;
  a checked-in artifact never falls back to an unverified URL alternative.
- Logo catalog: size/line/control-character validation before printing raw art.
- Subprocesses: bounded timeouts and process-tree kill; Windows pipe
  grandchild handling is explicit.

## 9. Test and build layout

- Unit tests live next to most modules; there are ~256 unit tests on the
  release branch.
- `tests/wasm_e2e.rs` runs six end-to-end wasm scenarios.
- `scripts/ci.sh` = fmt + clippy + tests.
- `scripts/ci-windows.sh` = `cargo check/clippy --target
  x86_64-pc-windows-gnu` (tests need a Windows host).
- CI workflows: `rust-tests.yml` (test), `security-audit.yml` (audit),
  `release.yml` (tag gate + 6-target build + checksums + cosign + publish),
  `roadmap-sync.yml` (manual ROADMAP/issue sync).
