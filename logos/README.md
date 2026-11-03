# logos

ASCII logo catalog for Linux distributions, macOS and Windows, consumed by
`xfetch --gen-config` over HTTPS.

Last verified: 2026-11-03 against `logos@801e2014f9c53176fbd6fec0cc249ad83ad71e02` (branch `fix/x-logo-placeholder`; default branch `main`).

## Role

`xfetch-cli/logos` is a data-only repository:

- `logos.json` — the index and contract: categories, families, defaults and one
  entry per logo (id, name, aliases, file, colors, ANSI codes, project URL).
- `defaults/<category>/<id>.txt` — the ASCII art files.
- `scripts/validate.py` — checks that the index is parseable and every
  referenced art file exists.

The repository has no build step, package, release or binary. `xfetch` fetches
`logos.json` and the raw art at `--gen-config` time and embeds the art in the
generated `config.jsonc`.

## Location

| Item | Value |
| --- | --- |
| Local clone | `~/repos/xfetch-cli/logos` |
| Remote (fetch/push) | `https://github.com/xfetch-cli/logos.git` |
| Default branch | `main` (`origin/HEAD -> origin/main`) |
| Current checkout | `fix/x-logo-placeholder` |
| HEAD | `801e2014f9c53176fbd6fec0cc249ad83ad71e02` |
| HEAD date | 2026-11-03 19:17:21 +0100 |
| HEAD subject | `fix: use the real X logo for the x entry` |
| Catalog base URL | `https://raw.githubusercontent.com/xfetch-cli/logos/main` |
| Entries | 128 (Linux 91, macOS 22, Windows 15) |
| CI | Local scripts only; no `.github/workflows/` |

Heads-up: the local clone is on a feature branch, not `main`. The committed
"real X logo" change is not necessarily on `origin/main` yet. When documenting
or testing "current published behavior", compare with
`git log origin/main` / the raw `main` URLs.

## Quick start

```bash
# Embed the detected distro logo into a generated config
xfetch --gen-config

# Force a specific catalog id
xfetch --gen-config --logo arch
xfetch --gen-config --logo windows-11
xfetch --gen-config --logo x

# Test against a fork/branch/local mirror
XFETCH_LOGOS_URL=https://raw.githubusercontent.com/<user>/logos/main xfetch --gen-config
```

Validate the catalog:

```bash
cd ~/repos/xfetch-cli/logos
bash scripts/ci.sh          # Linux/macOS
./scripts/ci.ps1            # Windows
```

Current result: `OK: logos.json valid, 128 referenced files present.`

## Documentation index

| File | Contents |
| --- | --- |
| `README.md` | This file: role, location, quick start, doc index. |
| `catalog.md` | `logos.json` schema, defaults map, resolution chain, file format/size conventions, colors/ANSI, X/equis entries, adding/fixing, placeholder notes. |
| `operations.md` | `scripts/validate.py`, CI, release, testing with `XFETCH_LOGOS_URL`, gotchas. |

## Related repositories

| Repository | Relationship |
| --- | --- |
| `xfetch-cli/xfetch` | Consumer: `src/logos.rs` fetches the index and art for `--gen-config`; OS detection comes from `src/info/platform`. |
| `xfetch-cli/configs` | Bundles a subset of logo art for presets (`massive-roulette/logos/`) and references files under `~/.config/xfetch/logos/`. |
| `xfetch-cli/themes` | Themes set `logo_color`/`logo_colors`; they color logos, they do not contain art. |
| `xfetch-cli/web` | Documents logo behavior under `gen-config` and `customization`. |
