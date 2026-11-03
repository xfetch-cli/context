# themes operations

Scripts, validation, versioning, common tasks and gotchas for the theme
registry.

Last verified: 2026-11-03 against `themes@8c2a16cc64a268b955f13cbf0f2c7419cb42f4d3` (`main`).

## Prerequisites

| Tool | Needed for |
| --- | --- |
| Python 3 | `scripts/validate.py` |
| Bash (Linux/macOS) or PowerShell (Windows) | CI wrappers |
| `xfetch` | End-to-end theme testing |
| `curl` | `theme-manager` network operations |
| Git | Normal PR workflow |

No Node, Rust or package manager is involved in this repository.

## Scripts

| Script | Behavior |
| --- | --- |
| `scripts/validate.py` | Recursively parses every `.json`/`.jsonc` (excluding `.git`), after stripping `//`, `/* */` and trailing commas. Exits non-zero on the first broken file. |
| `scripts/ci.sh` | `cd`s to `scripts/`, runs `python3 validate.py ..`, prints `CI OK`. |
| `scripts/ci.ps1` | Same with `python validate.py ..` on Windows. |

There is no `.github/workflows/` in this repository. CI is local-only; a pull
request is expected to include a passing `scripts/ci.sh` run in its description
or reviewer verification.

## Validation

```bash
cd ~/repos/xfetch-cli/themes
bash scripts/ci.sh
# or
python3 scripts/validate.py .
```

Current result:

```text
OK: 85 JSON/JSONC files valid.
```

85 = `index.json` + `schema.json` + 83 `colors/*.jsonc`.

### What validation does not cover

The validator is syntax-only. It does not enforce the registry contract. Before
merging, also check manually (or with a script):

1. Every `index.json` entry has a matching `colors/<name>.jsonc`.
2. Every `colors/<name>.jsonc` is listed in `index.json`.
3. Every `source` equals
   `https://raw.githubusercontent.com/xfetch-cli/themes/main/colors/<name>.jsonc`.
4. No theme contains `icons`, `modules` or `info_plugins`.
5. `layout` and `palette_style` values are in the schema enums.
6. `version` matches `^\d+\.\d+\.\d+$`.

A one-liner used during this audit:

```bash
python3 - <<'PY'
import json, os
d = json.load(open('index.json'))
names = {t['name'] for t in d['themes']}
files = {f[:-len('.jsonc')] for f in os.listdir('colors') if f.endswith('.jsonc')}
print('missing files:', sorted(names - files))
print('missing entries:', sorted(files - names))
bad = [t['name'] for t in d['themes']
       if t['source'] != f"https://raw.githubusercontent.com/xfetch-cli/themes/main/colors/{t['name']}.jsonc"]
print('bad sources:', bad)
PY
```

At HEAD the output is empty for all three checks.

## Release and versioning

- Themes are versioned independently through `index.json` (`version` per entry),
  not through Git tags or GitHub Releases. There are no tags in the clone and no
  release workflow.
- Version history is recorded in `CHANGELOG.md`. Bump `version` in `index.json`
  when a theme changes materially; all current entries are `1.0.0`.
- The registry is served straight from the `main` branch through
  `raw.githubusercontent.com`. There is no build artifact or package to publish.
- `theme-manager` reads `main` by default, so a merge to `main` is effectively a
  release.

## Common tasks

### Add a theme

```bash
cp colors/dracula.jsonc colors/my-theme.jsonc
# edit colors/my-theme.jsonc
# add an entry to index.json (name, author, version, description, layout,
# palette_style, tags, source)
python3 scripts/validate.py .
# manual consistency check (see above)
# test end to end
mkdir -p ~/.config/xfetch/themes
cp colors/my-theme.jsonc ~/.config/xfetch/themes/
xfetch theme set my-theme
xfetch
```

### Update an existing theme

1. Edit `colors/<name>.jsonc`.
2. Bump `version` in `index.json` if the visual result changes.
3. Keep `source` unchanged (it is path-based, not content-addressed).
4. Add a `CHANGELOG.md` entry.
5. Validate and test.

### Install a theme without the plugin

```bash
mkdir -p ~/.config/xfetch/themes
curl -fsSL https://raw.githubusercontent.com/xfetch-cli/themes/main/colors/nord.jsonc \
  -o ~/.config/xfetch/themes/nord.jsonc
xfetch theme set nord
```

### Test the registry locally with theme-manager

`theme-manager` accepts a local registry path through `args.registry`. A local
copy of `index.json` with a `source` pointing at a local file works:

```jsonc
{
    "info_plugins": [
        {
            "plugin": "theme-manager",
            "args": {
                "action": "install",
                "name": "dracula",
                "registry": "~/dev/themes/index.json"
            }
        }
    ],
    "modules": ["plugin:theme-manager"]
}
```

The plugin treats a registry that starts with `/` or `~` as a local file and a
`source` that starts with `/` or `~` as a local copy.

### Check a theme's merge locally

```bash
# after installing the theme
xfetch --config ~/.config/xfetch/config.jsonc
# temporarily test without changing the config
xfetch theme export before
xfetch theme set <name>
xfetch
```

Remember that the theme wins over `config.jsonc` visual fields, so remove or
export any per-user visual overrides you want to keep.

## Gotchas

1. **No remote CI.** Nothing blocks a PR except review; run
   `scripts/ci.sh` yourself.
2. **Validator is syntax-only.** A valid `index.json` can still contain a
   missing file, a broken `source`, or a theme with `icons`.
3. **No tags/releases.** Do not look for a `vX.Y.Z` release; `main` is the
   distribution channel.
4. **Theme overrides config.** Since v0.2.0 the theme is the highest-priority
   merge layer. A theme's `layout` or `colors` can surprise users who also set
   those fields in `config.jsonc`.
5. **`theme set` requires a local file.** It only updates the `theme` key if
   `resolve_theme_path` finds the theme; install it first.
6. **`theme remove` is directory-scoped.** It refuses to delete a theme that
   resolves outside `~/.config/xfetch/themes/`.
7. **`theme export` omits icons and only exports non-empty fields.** It is not a
   full config export.
8. **Index/file drift is possible.** Reviewers must keep the two in sync; the
   current 83/83 state is consistent but not machine-enforced.
9. **Schema is advisory.** `schema.json` is not loaded by the validator and the
   index includes fields (`$schema`) the schema does not define. Treat the
   schema as documentation for editors.
10. **Theme registry is not bundled with xfetch.** Offline users must copy files
    manually; there is no embedded subset.
11. **`theme-manager` needs `curl`.** It shells out to `curl` for both the index
    and the theme download; a missing `curl` breaks every network action.
12. **Names are identifiers.** `"theme": "x"` refers to `colors/x.jsonc`; renaming
    a theme file without updating `index.json` and user configs breaks them.
13. **Docs drift on the web side.** `xfetch-cli/web` currently documents 17
    built-in themes while this registry has 83. The registry is authoritative.
