# logos operations

Validation, CI, release and testing procedures for the logo catalog.

Last verified: 2026-11-03 against `logos@801e2014f9c53176fbd6fec0cc249ad83ad71e02` (branch `fix/x-logo-placeholder`; default branch `main`).

## Prerequisites

| Tool | Needed for |
| --- | --- |
| Python 3 | `scripts/validate.py` |
| Bash or PowerShell | CI wrappers |
| `curl` | `xfetch --gen-config` fetches the index and art with curl |
| `xfetch` | End-to-end testing |
| A local HTTP server (optional) | Testing `XFETCH_LOGOS_URL` against a working tree |

## scripts/validate.py

The only maintained validation. It:

1. Resolves the repository root from its own location.
2. Requires `logos.json` and the top-level keys `schema_version`, `defaults`,
   `categories`, `families`, `logos`.
3. Iterates `logos[]` and checks every non-absolute `file` path (without `..`)
   exists on disk.
4. Iterates `defaults` and checks every `.txt` value exists.
5. Prints `OK: logos.json valid, N referenced files present.` and exits 0, or
   prints the first problems and exits 1.

It does not validate per-entry fields (id, category, family, aliases, colors,
ansi, url), unique ids, or art encoding. Those are review responsibilities; see
`catalog.md` for the full contract.

Current result at HEAD:

```text
OK: logos.json valid, 128 referenced files present.
```

The count is entries, not files: 128 art files referenced by `logos[]`; the
three `defaults/*/default.txt` files are validated separately and are not
counted in that number.

## CI scripts

| Script | Behavior |
| --- | --- |
| `scripts/ci.sh` | `cd`s to `scripts/`, runs `python3 validate.py ..`, prints `CI OK`. |
| `scripts/ci.ps1` | Same with `python validate.py ..` on Windows. |

Run locally before every commit:

```bash
cd ~/repos/xfetch-cli/logos
bash scripts/ci.sh
```

There is **no** `.github/workflows/` in this repository at HEAD. Nothing runs
automatically on push or PR; validation is local and reviewer-enforced.

## Release and distribution

- There are no tags, GitHub Releases, packages or build artifacts.
- The catalog is served straight from `main` over
  `https://raw.githubusercontent.com/xfetch-cli/logos/main/...`.
- `xfetch` fetches `<base>/logos.json` and `<base>/<file>` at runtime, where
  `<base>` is the `main` raw URL unless `XFETCH_LOGOS_URL` overrides it.
- Merging to `main` is the release. The branch name in the local clone
  (`fix/x-logo-placeholder`) is not a release channel.
- CHANGELOG entries are the version history; `schema_version` is `1.0.0` and has
  not changed since the initial catalog.

## Testing with XFETCH_LOGOS_URL

`xfetch` reads the base URL from `XFETCH_LOGOS_URL` (default
`https://raw.githubusercontent.com/xfetch-cli/logos/main`). Use it to test a
fork, a branch or a local working tree before merging.

### Test a fork or branch

```bash
XFETCH_LOGOS_URL=https://raw.githubusercontent.com/<user>/logos/<branch> \
  xfetch --gen-config --logo arch
```

### Test a local working tree

Serve the repository root (the directory containing `logos.json` and
`defaults/`) and point xfetch at it:

```bash
cd ~/repos/xfetch-cli/logos
python3 -m http.server 8099 &
XFETCH_LOGOS_URL=http://localhost:8099 xfetch --gen-config --logo x
kill %1
```

The consumer appends `/logos.json` and `/<file>`, so the server root must be the
repository root.

### Test detection instead of override

`--logo` skips ID_LIKE resolution. To test the real detection chain, run on the
target distro (or in a container) without `--logo`:

```bash
XFETCH_LOGOS_URL=http://localhost:8099 xfetch --gen-config
```

### Verify the generated result

```bash
xfetch --gen-config --logo arch
grep -n '"ascii"\|"logo_path"' ~/.config/xfetch/config.jsonc
```

`--gen-config` embeds the art (or a path, depending on the version) in the
generated config. Confirm the art is the expected file and renders.

## Common tasks

### Add a logo

1. Create `defaults/<category>/<id>.txt` (UTF-8, LF, no control chars, ~12 rows,
   padded width up to ~60).
2. Add the entry to `logos.json` with `id`, `name`, `category`, `family`,
   `aliases` (include the os-release `ID` and `ID_LIKE` tokens), `file`,
   `colors`, `url`, `ansi`.
3. Add a `CHANGELOG.md` entry.
4. `bash scripts/ci.sh`.
5. Test with `--logo` and with detection, then with `XFETCH_LOGOS_URL`
   pointing at your fork/branch.
6. PR.

### Fix detection for an existing distro

1. Confirm `/etc/os-release` `ID` and `ID_LIKE`.
2. Add the real `ID` (and useful variants) to the matching entry's `aliases`.
   Example: Linux Mint's real id is `linuxmint`; the file is `linux-mint.txt`.
3. If no existing logo fits, add a new file + entry instead.
4. Test `xfetch --gen-config` without `--logo`.

### Fix art

1. Edit the `.txt` only; keep the file path and id stable.
2. Preserve UTF-8/LF and remove any control characters.
3. Keep total size well under 64 KiB and lines under 200 chars (the consumer
   rejects violations and falls back).
4. Test `xfetch --gen-config --logo <id>`.
5. Add a changelog note if the visible art changes.

### Deprecate or rename

Do not delete or rename ids casually: user configs and `--logo` values depend on
them. Prefer keeping the id, updating `name`/aliases, and adding a new id only
when the art is genuinely different.

## Gotchas

1. **Clone is not on `main`.** HEAD is `fix/x-logo-placeholder`
   (`801e201`). The raw `main` URL may not contain the X logo fix yet. Check
   `git log origin/main -- defaults/linux/x.txt` before claiming published
   behavior.
2. **x and equis are identical at HEAD.** Both art files are byte-identical
   (23 rows, 2771 bytes). The CHANGELOG describes `equis` as alternative art,
   but the files do not differ today.
3. **The ~12-row rule is not enforced.** 111 of 128 files have 12 rows, but
   `x`/`equis` have 23 and two files have 4. Do not assume rectangular or
   12-row input when writing consumers.
4. **Ragged files exist.** 117 of 128 files have lines of varying lengths, even
   though CONTRIBUTING asks for padding. Do not assume fixed width.
5. **No CI.** `scripts/ci.sh` is local-only; a PR can be green by review while
   `validate.py` was never run.
6. **Validator does not check entry semantics.** It only checks referenced files.
   Missing aliases/categories/colors or duplicate ids pass validation.
7. **`--logo` has no ID_LIKE fallback.** An explicit override that does not
   match an id/alias falls back to the generic category logo with a warning.
8. **Aliases are the detection contract.** The real os-release `ID` must be in
   `aliases`; the filename/id do not have to match it (`linux-mint` vs
   `linuxmint`).
9. **`curl` is a runtime dependency.** Offline machines and minimal containers
   fall back to the built-in logo; `--gen-config` does not error.
10. **Defaults are not in `logos[]`.** The three `defaults/*/default.txt` files
    are referenced from the `defaults` map only. `validate.py` counts 128
    entries but 131 art files exist.
11. **Consumer-side limits are stricter than the repo.** `validate_art` in
    `xfetch/src/logos.rs` caps at 64 KiB and 200 columns and rejects all control
    characters including ESC. A file that merely "looks fine" can still be
    rejected at runtime.
12. **Art is embedded/printed raw.** Treat the catalog as a security boundary:
    do not add escape sequences, and review art changes for control bytes.
13. **Case-insensitive matching.** `entry_matches` lowercases both sides, so
    `ID` case differences are fine, but alias content still has to exist.
14. **Raw URLs are the public contract.** `logos.json` and `defaults/...` paths
    on `main` are fetched by released binaries; moving them breaks user
    `--gen-config` runs.
