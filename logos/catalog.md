# logos catalog

Schema, resolution rules, file conventions and current X/equis state of the
logo catalog.

Last verified: 2026-11-03 against `logos@801e2014f9c53176fbd6fec0cc249ad83ad71e02` (branch `fix/x-logo-placeholder`; default branch `main`).

## Tree

```text
logos/
|-- logos.json                 # index / contract
|-- defaults/
|   |-- linux/                 # 92 .txt files: 91 referenced entries + default.txt
|   |-- macos/                 # 23 .txt files: 22 referenced entries + default.txt
|   `-- windows/               # 16 .txt files: 15 referenced entries + default.txt
|-- scripts/
|   |-- ci.sh
|   |-- ci.ps1
|   `-- validate.py
|-- README.md
|-- CONTRIBUTING.md
|-- CHANGELOG.md
|-- SECURITY.md
|-- SUPPORT.md
|-- CODE_OF_CONDUCT.md
`-- LICENSE (MIT)
```

Totals: 128 catalog entries; 131 art files on disk (128 entries + 3 category
defaults, which are referenced from `defaults`, not from `logos[]`).

## logos.json top-level schema

There is no separate `schema.json`. The contract is the file itself plus
`scripts/validate.py`.

| Key | Type | Required by validator | Value at HEAD |
| --- | --- | --- | --- |
| `schema_version` | string | Yes | `"1.0.0"` |
| `name` | string | No | `"Operating system logos"` |
| `description` | string | No | Human-readable description of the catalog. |
| `defaults` | object | Yes | Map of category -> generic art file (see below). |
| `categories` | array | Yes | 3 category objects. |
| `families` | array | Yes | 7 family objects. |
| `logos` | array | Yes | 128 logo entries. |

### defaults

| Category | Fallback file |
| --- | --- |
| `linux` | `defaults/linux/default.txt` |
| `macos` | `defaults/macos/default.txt` |
| `windows` | `defaults/windows/default.txt` |

The fallback is selected by `crate::info::platform::logo_category()` on the
consumer side, not by the catalog.

### categories

| `id` | `name` | `directory` | Entries |
| --- | --- | --- | --- |
| `linux` | Linux | `defaults/linux` | 91 |
| `macos` | macOS | `defaults/macos` | 22 |
| `windows` | Windows | `defaults/windows` | 15 |

Each category also has a `description`.

### families

| `id` | `name` | `base` | Notes | Entries |
| --- | --- | --- | --- | --- |
| `debian` | Debian / Ubuntu | `debian` | apt / dpkg (.deb) | 42 |
| `arch` | Arch Linux | `arch` | pacman / AUR | 16 |
| `fedora` | Fedora / RHEL | `fedora` | dnf / rpm | 11 |
| `opensuse` | openSUSE | `opensuse` | zypper / rpm | 4 |
| `independent` | Independent | `null` | Not derived from another distribution | 18 |
| `macos` | Apple macOS | `macos` | Darwin / XNU kernel | 22 |
| `windows` | Microsoft Windows | `windows` | NT kernel | 15 |

### Logo entry

Every entry at HEAD has all of these fields:

| Field | Type | Purpose |
| --- | --- | --- |
| `id` | string | Stable catalog id (`arch`, `linux-mint`, `windows-11`, `x`). |
| `name` | string | Display name. |
| `category` | string | `linux`, `macos` or `windows`. |
| `family` | string | One of the family ids above. |
| `aliases` | array of strings | Alternate names used for detection; must include the real `/etc/os-release` `ID` and ideally the `ID_LIKE` tokens. |
| `file` | string | Relative path to the art file, e.g. `defaults/linux/arch.txt`. |
| `colors` | object | `primary` and `secondary` hex colors for the logo. |
| `url` | string | Project URL. |
| `ansi` | object | `primary` and `secondary` ANSI 256-color indices. |

Example:

```jsonc
{
    "id": "x",
    "name": "X",
    "category": "linux",
    "family": "arch",
    "aliases": ["x", "x linux", "x-linux", "x os"],
    "file": "defaults/linux/x.txt",
    "colors": { "primary": "#00FFFF", "secondary": "#00A3C4" },
    "url": "https://xscriptor.io/x",
    "ansi": { "primary": 51, "secondary": 38 }
}
```

## Resolution chain (consumer side)

`xfetch/src/logos.rs` implements the lookup:

```text
detect OS IDs -> execos-release ID + ID_LIKE tokens
        |
        v
1. exact match on entry.id or entry.aliases (case-insensitive)
        |
        v
2. for each ID_LIKE token in order, match id or aliases (case-insensitive)
        |
        v
3. index.defaults[logo_category()]  (generic Linux/macOS/Windows art)
        |
        v
fetch art from <base>/<file> and validate it
```

Key details:

- `--logo <id>` bypasses ID_LIKE entirely: only an exact id/alias match is
  attempted. If nothing matches, xfetch warns and uses the generic category
  logo.
- Without `--logo`, detection uses the platform module (`detect_os_ids()`), then
  the chain above.
- The index is fetched from `<base>/logos.json`; the art from `<base>/<file>`.
- `<base>` is `https://raw.githubusercontent.com/xfetch-cli/logos/main` unless
  `XFETCH_LOGOS_URL` is set (for forks, branches and local mirrors).
- Fetching uses `curl -fsS --max-time 10` with a 10-second process timeout.
- On any failure (network, parse, bad art), `--gen-config` falls back to the
  default behavior; the catalog is best-effort.

### Art validation (security boundary)

`validate_art` rejects an art file unless all are true:

| Limit | Value |
| --- | --- |
| Total size | <= 64 KiB (`MAX_ART_SIZE = 64 * 1024`) |
| Line width | <= 200 chars (`MAX_LINE_WIDTH`) |
| Control characters | Only `\n` and `\t` allowed; ESC, C0/C1, NUL, DEL and terminal escape sequences are rejected |

This matters because the art is printed raw to the terminal and embedded in the
generated config. A compromised catalog falls back to the built-in default logo
instead of delivering a payload.

## File format and size conventions

Rules from `CONTRIBUTING.md`:

- Plain UTF-8 text, LF line endings.
- No control characters (no tabs, no NUL).
- Roughly 12 rows; pad shorter rows with spaces so every line has the same
  width.
- Width up to about 60 columns.
- ANSI escape codes are allowed for colored logos (but the consumer validation
  rejects ESC, so current art is plain text).

Actual state at HEAD (measured over the 128 referenced files):

| Metric | Value |
| --- | --- |
| Rows | 111 files have 12 rows; 9 have 11; 2 have 23 (`x`, `equis`); 2 have 10; 2 have 4; 1 each has 8 and 9. Range 4-23. |
| Max width | 60 columns. |
| Uniform-width files | 11 of 128; 117 have varying line lengths. |
| Bytes | 90 to 2771; largest are `x` and `equis` at 2771 bytes each. |
| Encoding | UTF-8, LF only; no CRLF and no control characters found in any referenced file. |

The "rectangular block of ~12 rows" rule is a convention, not enforced by the
validator. Many existing files are ragged. New art should aim for the rule even
though the current corpus does not fully follow it.

## colors and ansi

- `colors.primary` / `colors.secondary` are hex strings used for display and
  potential image/theme integration.
- `ansi.primary` / `ansi.secondary` are 256-color indices. They are metadata;
  `--gen-config` embeds the raw art, not ANSI conversion.
- Example families:
  - Arch: `#1793D1` / `#33AADD`, ANSI `32` / `74`
  - Debian: `#A81D33` / `#D70A53`, ANSI `125` / `161`
  - X/equis: `#00FFFF` / `#00A3C4`, ANSI `51` / `38`

## X and equis entries

Both are Linux entries in the `arch` family:

| Field | `x` | `equis` |
| --- | --- | --- |
| `name` | X | X (Equis) |
| `aliases` | `x`, `x linux`, `x-linux`, `x os` | `equis`, `equis linux`, `x equis` |
| `file` | `defaults/linux/x.txt` | `defaults/linux/equis.txt` |
| `colors` | `#00FFFF` / `#00A3C4` | same |
| `ansi` | `51` / `38` | same |
| `url` | `https://xscriptor.io/x` | same |

Current placeholder/duplication notes:

- `defaults/linux/x.txt` and `defaults/linux/equis.txt` are currently
  **byte-identical** (both 23 rows, max width 44, 2771 bytes).
- The CHANGELOG says `equis` was added as "alternative X Linux art" reusing the
  `x` palette and `arch` family. In practice it is a duplicate of `x.txt` at
  HEAD.
- The checked-out branch `fix/x-logo-placeholder` changes `x.txt` to the "real X
  logo" (23 lines, replacing 21). The branch name suggests the placeholder
  cleanup is still in progress, and `equis.txt` was not changed in that commit.
  Confirm the state on `main` before treating `equis` as distinct art.
- Both entries are 23 rows, well outside the ~12-row convention.

## Adding or fixing a logo

1. Add or edit the art file:
   - Linux: `defaults/linux/<id>.txt`
   - macOS: `defaults/macos/<id>.txt`
   - Windows: `defaults/windows/<id>.txt`
   - UTF-8, LF, no tabs/control chars, ~12 rows, padded to a consistent width.
2. Add or update the entry in `logos.json`:
   - `id` should be the lowercase `/etc/os-release` `ID` with dashes
     (`linux-mint`, `opensuse-leap`).
   - `aliases` must include the real `ID` and the `ID_LIKE` tokens; this is what
     makes detection work (e.g. Linux Mint's real id is `linuxmint`).
   - `file` must be the repo-relative path.
   - `colors`, `ansi` and `url` are expected on every entry.
3. Add a `CHANGELOG.md` entry (new logos, fixes, alias changes).
4. Run `bash scripts/ci.sh` (checks the index and that every referenced file
   exists).
5. Test end to end:
   - `xfetch --gen-config --logo <id>` for the explicit path,
   - `xfetch --gen-config` on the distro for the detection/ID_LIKE path,
   - `XFETCH_LOGOS_URL=... xfetch --gen-config` for a fork or branch.
6. Open a PR.

When fixing a detection bug, the usual fix is adding an alias, not adding a new
file. The catalog intentionally maps ID_LIKE families to a representative logo
(e.g. many Ubuntu derivatives point at `ubuntu`).

## Catalog invariants

- Every `logos[].file` must exist; every default must exist. `validate.py`
  enforces both.
- `id` values must be unique (verified: 128 unique ids at HEAD).
- `category` and `family` values must match the `categories`/`families` lists.
- Keep `source`-like URLs stable: the `logos.json` path and `defaults/` paths are
  a public contract consumed through raw URLs.
- Do not add executable content, control characters or oversized art; the
  consumer rejects it and falls back.
- Do not reorder/rename existing IDs: user configs and `--logo` overrides depend
  on them.
