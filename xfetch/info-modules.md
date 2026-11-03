# Info modules and probes

This document covers how xfetch turns a `modules` list into the `Info` struct
and where each probe lives. It is the map to use before changing any system
information output.

## Module keys

The public module table lives in `docs/CONFIGURATION.md`. Keys accepted by the
renderer:

`os`, `kernel`, `hostname`/`host`, `user`, `uptime`, `datetime`, `packages`,
`packages:<manager>`, `shell`, `terminal`, `wm`, `cpu`, `gpu`, `memory`,
`swap`, `disk`, `battery`, `local_ip`, `local_ip:v6`, `public_ip`,
`interfaces`, `palette`, plus plugin keys (`plugin:<name>`) and group entries
in `modules`.

`collect_module_keys()` in `src/info/mod.rs` flattens groups and nested module
entries into a `HashSet` so probes only run when needed.

## Collection pipeline

`Info::with_config(config, benchmark)` (`src/info/mod.rs`):

1. Determine needed keys.
2. Initialize only required sysinfo containers concurrently:
   - `System::new_with_specifics` for CPU/memory/swap;
   - `Disks::new_with_refreshed_list` for disk;
   - `Networks::new_with_refreshed_list` for network/IP.
3. Load the 300-second package and public-IP caches unless disabled.
4. Run independent probes in parallel: GPU, battery, datetime, packages,
   public IP, info plugins.
5. Fill `Info`; probe failures become `unknown()` / `N/A` / empty strings.
6. Emit `bench_lines` for `--benchmark`.

Caching is `src/cache.rs` (`$XDG_CACHE_HOME/xfetch/cache.json`). Package counts
and public IP use a 300 s TTL; `--clean-cache` removes the file. Disable with
`"disable_cache": true`; disable public IP with `"disable_ip_fetching": true`.

## Info fields and sources

| Field | Source |
| --- | --- |
| `os` | `platform::get_os_info()`; Linux applies `wsl::decorate_os(..., os_wsl_style)`. |
| `kernel` | `get_kernel_info()` (`sysinfo`). |
| `host_name` | `get_host_name()`. |
| `shell` | `software::get_shell_info()`. |
| `terminal` | `software::get_terminal_info()`. |
| `cpu` | `hardware::get_cpu_info(&System)`. |
| `gpu` | `platform::get_gpu_info()` (list). |
| `memory` / `swap` | `hardware::get_memory_info` / `get_swap_info(&System)`. |
| `disks` | `hardware::get_disk_info(&Disks)`. |
| `battery` | `platform::get_battery_info()`. |
| `uptime` | `get_uptime_info()`. |
| `packages` / `packages_breakdown` | `platform::get_packages_breakdown()`. |
| `desktop` | `software::get_desktop_info()`. |
| `user` | `software::get_user_info()`. |
| `datetime` | `platform::get_datetime_info()`. |
| `local_ip`, `local_ip_v6`, `network_interfaces` | `system::*` with `Networks`. |
| `public_ip` | `system::get_public_ip_info(true)` with cache. |
| `plugin_info` | `load_plugin_info(&config.info_plugins)` (parallel subprocesses). |

Formatting helpers: `src/info/format.rs` (bytes/GiB, row text),
`src/info/hardware.rs`, `src/info/software.rs`, `src/info/system.rs`.

## Platform layers

The contract is: each platform folder exposes the same function names, and
`src/info/platform/mod.rs` re-exports the active one.

### Linux (`src/info/platform/linux/`)

- `battery.rs`, `datetime.rs`, `gpu.rs`, `network.rs`, `os_release.rs`,
  `live.rs`, `wsl.rs`.
- `packages.rs`: subprocess probes + DB readers.
- `xpm.rs`: xpm / xp / xpm-aur package counts (new in v1.0.2).

### macOS (`src/info/platform/macos/`)

- `battery.rs`, `datetime.rs`, `gpu.rs`, `network.rs`, `packages.rs`,
  `software.rs`, `version.rs`, `live.rs`.
- Packages: `brew list --formula`, parsed by `count_brew_output`.

### Windows (`src/info/platform/windows/`)

- Native Win32 probes for CPU/date/battery/GPU; `process.rs` for process-tree
  control; `software.rs`, `version.rs`, `network.rs`, `packages.rs`,
  `live.rs`.
- Packages: `scoop list` (via `scoop.cmd`) and `winget list --source winget`;
  Chocolatey is a plugin, not a core probe.

### Shared (`src/info/platform/shared/`)

- `commands.rs`: backwards-compatible re-export of `crate::subprocess`.
- `gpu.rs`: shared GPU helpers.
- `packages.rs`: `PackageCheck`, timeouts, output parsers
  (`count_scoop_output`, `count_winget_output`, `count_brew_output`),
  `format_package_count`, `run_package_checks`,
  `packages_info_from_breakdown`.

## Package counting

The `packages` module renders `N (manager) + M (manager) + ...`; individual
`packages:<label>` keys select one label.

### Linux subprocess/DB probes

`src/info/platform/linux/packages.rs`:

- CHECKS: `pacman -Qn` (`pacman`), `pacman -Qm` (`aur`), `dpkg
  --get-selections`, `rpm -qa`, `flatpak list --app`, `snap list` (only when
  snapd is running), `apk info`, `nix-env -q`, `xbps-query -l`, `portage`
  (DB-only).
- DB fast paths: dpkg status, apk installed DB, flatpak app dirs, Void
  `/var/db/xbps`, Gentoo `/var/db/pkg`; these avoid spawning and keep counts
  correct on WSL.
- `snap` only runs when `/run/snapd.socket` or `/run/snapd-snap.socket`
  exists.
- Ordering is preserved by iterating `CHECKS` and looking up counts.

### xpm / xp / xpm-aur (`src/info/platform/linux/xpm.rs`)

Reads only local installation state:

- `xpm`: union of `/var/lib/xpm/local` entries (dirs with a `version` file,
  dot entries skipped) and `/var/lib/pacman/local` entries. The pacman side
  accepts both pacman-style `<name>-<version>` dirs (`%NAME%` from `desc`) and
  xpm-style `<name>/version` dirs. This is required because xpm mirrors its
  transactions into pacman's local DB.
- `xp`: installed xpm entries whose `origin` file is the native X repo `x`.
  This is a subset of `xpm`; Arch/PKGBUILD-compat packages have origin
  `core`/`extra`/etc. and do not count as `xp`.
- `xpm-aur`: `$XPM_AUR_STATE` or `$XDG_STATE_HOME/xpm-aur/state.json` (fallback
  `~/.local/state/...`, then `/var/lib/xpm-aur`). Counts installed output
  packages from each record, intersected with the local installed set so stale
  sidecar records do not inflate the count.
- If xpm is not present but `/var/lib/pacman/local` exists, the pacman DB is
  used as a fallback count for the `pacman` label (only when neither xpm nor
  the `pacman` command produced a label).

To change the native repo list, edit `X_NATIVE_REPOS` in `xpm.rs`. To change
the xpm DB path, edit `XPM_DB_DIR`.

### Other platforms

- macOS: `brew list --formula`, parsed by `count_brew_output`.
- Windows: scoop/winget parsers in `shared/packages.rs`.

## Info plugins

`load_plugin_info()` spawns every configured info plugin concurrently, sending
`InfoPluginRequest` and collecting `plugin:<name>` lines into
`Info.plugin_info`. Failures are skipped. The plugin runtime lives in
`src/plugins/run.rs`; the protocol in `xfetch-cli/api`.

## Output templates and labels

- `config.labels` renames/hides the key shown per module.
- `config.formats` provides `{field}` templates; every module exposes
  `{value}`/`{key}`, structured modules add fields (e.g. CPU `{brand}`,
  `{model}`, `{cores}`, `{freq}`). Unknown placeholders render empty; `{{`/`}}`
  escape braces.
- `config.icons`/`colors` are consumed by the renderers in `src/ui/`.

## Benchmarking and debugging

- `xfetch --benchmark` prints parallel probe and total timings.
- `RUST_LOG`/verbose flags are not a general logging system; most probes are
  silent on failure and fall back to `unknown`/`N/A`.
- To debug a probe, add a focused unit test next to the module or run the
  relevant command manually (`pacman -Qn`, `xpm query`, etc.).
