# web content

What the documentation site contains, where each language lives, how the pages
map to the xfetch CLI docs, and how to change them.

Last verified: 2026-11-03 against `web@09782e2d95582665e212213657652d6ab43c49ca` (`main`).

## Content layout

```text
docs/
|-- en/
|   |-- SUMMARY.md              # title, description, version/license/author/repo, order
|   |-- <22 root pages>.md
|   |-- plugins/                # 17 files (index README + 16 plugin pages)
|   `-- extensions/             # 5 pages
|-- es/                         # exact mirror, Spanish
`-- de/                         # exact mirror, German
```

Verified counts at HEAD:

| Language | Markdown files | Rendered content pages | Notes |
| --- | --- | --- | --- |
| English | 45 | 44 | 45 includes `SUMMARY.md`, which is not a page. |
| Spanish | 45 | 44 | Same slugs as English. |
| German | 45 | 44 | Same slugs as English. |
| Total | 135 | 132 | Plus `/`, `/previews`, `/docs`, 3 language hubs. |

All 132 content pages have an H1. There is no frontmatter anywhere; metadata is
derived at build time by `src/lib/docs.ts` (see `architecture.md`).

## Pages by group

The group is assigned by `GROUP_BY_SLUG` / prefix rules in `src/lib/docs.ts`.
The sidebar and hub cards use these groups; `more` is the fallback.

### Getting Started (`start`)

| Slug | English title |
| --- | --- |
| `getting-started` | Getting Started with xfetch |
| `gen-config` | Config Generation (`--gen-config`) |

### Reference (`reference`)

| Slug | English title |
| --- | --- |
| `configuration` | Configuration |
| `modules` | Modules |
| `layouts` | Layouts |
| `custom-x` | Custom-X Layout |
| `presets` | Presets Reference |
| `themes` | Themes |
| `theme-manager` | Theme Manager Plugin |
| `customization` | Customization |
| `advanced-usage` | Advanced Usage |

### Plugins (`plugins`)

Root pages plus nested `plugins/*` pages. Sidebar order is root pages (except
`wasm`), then nested plugin pages, then `wasm` last.

| Slug | English title |
| --- | --- |
| `plugins` | Plugins |
| `plugins/README` | Plugins Index (routable, not in nav/search) |
| `plugins/animate-logo` | animate-logo |
| `plugins/chocolatey` | chocolatey |
| `plugins/display-resolution` | display-resolution |
| `plugins/docker` | docker |
| `plugins/github-stats` | github-stats |
| `plugins/music-player` | music-player |
| `plugins/temperature` | temperature |
| `plugins/theme-detection` | theme-detection |
| `plugins/theme-manager` | theme-manager |
| `plugins/timezone` | timezone |
| `plugins/user-info` | user-info |
| `plugins/wasm-crypto` | wasm-crypto |
| `plugins/wasm-ip-geo` | wasm-ip-geo |
| `plugins/wasm-pacman` | wasm-pacman |
| `plugins/wasm-proc` | wasm-proc |
| `plugins/weather` | weather |
| `wasm` | WebAssembly Guests |

Note: `theme-manager` exists twice as a topic: `theme-manager.md` (top-level,
in Reference, the plugin overview) and `plugins/theme-manager.md` (the plugin
page). Keep both in sync if the plugin changes.

### Extensions (`extensions`)

| Slug | English title |
| --- | --- |
| `extensions` | Extensions |
| `extensions/config-roulette` | Config Roulette |
| `extensions/layout-override` | Layout Override |
| `extensions/wasm-lang-labels` | wasm-lang-labels |
| `extensions/wasm-night-mode` | wasm-night-mode |
| `extensions/wasm-updates-footer` | wasm-updates-footer |

### Effects (`effects`)

| Slug | English title |
| --- | --- |
| `effects` | Effects |

### Project (`project`)

| Slug | English title |
| --- | --- |
| `contributing` | Contributing |
| `roadmap` | Roadmap |
| `security` | Security Policy |
| `support` | Support |
| `changelog` | Changelog |
| `code-of-conduct` | Code of Conduct |
| `license` | License |

### More (`more`)

Empty at HEAD. Any unlisted root page or unknown path would land here.

## Mapping to the xfetch CLI docs

The site docs are adaptations, not verbatim copies. The canonical prose for the
binary lives in `xfetch-cli/xfetch/docs/`. Current equivalents:

| Web page (all languages) | xfetch CLI source | Notes |
| --- | --- | --- |
| `getting-started` | `xfetch/docs/INSTALLATION.md`, `README.md` install section | Site adds install tabs, AUR/Homebrew/crates, `xfetch update`. |
| `gen-config` | `xfetch/docs/GEN_CONFIG.md` | `--gen-config`, `--logo`, `--layout`, offline fallback. |
| `configuration` | `xfetch/docs/CONFIGURATION.md` | JSONC keys, modules, icons, colors, plugin wiring. |
| `modules` | `xfetch` module registry / `docs/CONFIGURATION.md` | Per-module fields and formats. |
| `layouts` | `xfetch/docs/LAYOUTS.md` | All layout names. |
| `custom-x` | `xfetch/docs/CUSTOM_X.md` | `layout: "custom-x"` box drawing. |
| `presets` | `xfetch-cli/configs` (this book: `configs/inventory.md`) | Site text predates some new presets. |
| `themes` | `xfetch/docs/THEMES.md` + `xfetch-cli/themes` | Merge order, theme format, registry. |
| `theme-manager` | `xfetch/docs/THEMES.md` + `xfetch-cli/plugins` theme-manager | Actions list/search/info/install. |
| `customization` | `xfetch/docs/CONFIGURATION.md`, `THEMES.md` | Logos, images, Nerd Font icons, palette styles. |
| `advanced-usage` | `xfetch/docs/DAEMON.md`, `CONFIGURATION.md` | Benchmark, cache, privacy, performance. |
| `plugins` | `xfetch/docs/PLUGINS.md` + `xfetch-cli/plugins` | JSON wire protocol, discovery, official list. |
| `plugins/*` | `xfetch-cli/plugins/plugins/<name>/README.md` | One page per official plugin. |
| `wasm` | `xfetch/docs/WASM.md` | Host/guest ABI, capabilities, manifests. |
| `extensions` / `extensions/*` | `xfetch/docs/EXTENSIONS.md` + `xfetch-cli/extensions` | Config providers. |
| `effects` | `xfetch/docs/EFFECTS.md` + `xfetch-cli/effects` | Frame effects. |
| `changelog` | `xfetch/CHANGELOG.md` | Site copy is curated and shorter. |
| `contributing` | `xfetch/CONTRIBUTING.md`, `xfetch-cli/*/CONTRIBUTING.md` | Repo-specific workflows. |
| `security` | `xfetch/SECURITY.md` | Report process. |
| `support` | `xfetch/SUPPORT.md` | Help channels. |
| `code-of-conduct`, `license` | `xfetch/CODE_OF_CONDUCT.md`, `LICENSE` | MIT. |
| `roadmap` | `xfetch/ROADMAP.md` | Site roadmap is narrower. |

`configs`, `themes` and `logos` do not have a one-to-one page in every case:

- `presets` is the only page that documents `xfetch-cli/configs`.
- Themes are documented from the core merge code and the registry; the registry
  itself has no site page beyond `themes` and `theme-manager`.
- Logos are documented indirectly under `gen-config` and `customization`; the
  catalog file format lives in the logos repo (this book: `logos/catalog.md`).

## Language content

All three languages use identical slugs and the same directory shape:

```text
docs/en/configuration.md
docs/es/configuration.md
docs/de/configuration.md
```

`src/lib/docs.ts` scans each language independently. There is no fallback to
English if a page is missing: the route simply does not exist for that language
and `DocArticle` renders nothing for it.

Translation quality/coverage notes:

- `CONTRIBUTING.md` requires mirroring every change into all three languages.
- `docs/es/SUMMARY.md` and `docs/de/SUMMARY.md` carry translated titles and
  metadata, but the version/license numbers must match English.
- Language descriptions (`GROUP_DESCRIPTIONS`) and all docs UI strings live in
  `src/lib/docs-ui.ts`, not in the Markdown.

## Update workflow

1. Pick the canonical source repository and confirm the command/key/behaviour
   there first. The web docs must describe real code, not intent.
2. Edit the same page in `docs/en/`, `docs/es/` and `docs/de/`.
3. If the page is new:
   - add the three Markdown files with an H1 and a one-line description,
   - add a numbered entry to all three `SUMMARY.md` files if you want a fixed
     position (otherwise it sorts alphabetically after listed root pages),
   - if it is a nested page under `plugins/` or `extensions/`, it is picked up
     automatically; `plugins/README.md` is the index and is excluded from nav.
4. Update `docs/<lang>/SUMMARY.md` metadata only when the core version,
   license, author or repository changes.
5. Run the local gate:

   ```bash
   bash scripts/ci.sh          # npm run lint && npm run build
   # or: ./scripts/ci.ps1 on Windows
   ```

6. Preview the docs locally and check:
   - the page appears in the right group,
   - relative `.md` links resolve (the renderer rewrites them),
   - the language switcher does not 404,
   - tables are wrapped and code blocks have the copy button,
   - the 404 hint for old `/docs/<slug>` URLs is not needed.
7. Update `CHANGELOG.md` for user-visible documentation changes.
8. If the release version changed, update `src/components/landing/links.ts`
   (`VERSION`) and the `SUMMARY.md` version bullets together.

## Known content drift at HEAD

These are real mismatches to fix, not intentions:

1. **Version**: `src/components/landing/links.ts` sets `VERSION = "1.0.1"` and
   all `SUMMARY.md` files say `1.0.1`, while the xfetch core clone is already at
   `version = "1.0.2"` and its changelog documents 1.0.2 work.
2. **Theme count**: `docs/en/themes.md` (and the ES/DE equivalents) lists 17
   "Built-in Themes" ending at `x`, while `xfetch-cli/themes/index.json` now has
   83 themes. The page is materially incomplete.
3. **Presets paths**: `docs/en/presets.md` uses paths such as
   `configs/plugins/animate-logo/...` (relative to a checkout root). The configs
   repo is used via `xfetch --config <path>`; the page does not mention the
   newer `cyberpunk` and `massive-roulette` presets.
4. **Landing "Build from source" snippet**: `src/components/landing/data.ts`
   still emits `cp configs/config.jsonc ~/.config/xfetch/config.jsonc`. The
   configs repo no longer ships a root `config.jsonc`; the reference file is
   `xfetch/defaults/config.jsonc` and the supported flow is
   `xfetch --gen-config`.
5. **Docs changelog**: `docs/en/changelog.md` currently stops at v1.0.1.

When updating content, prefer fixing a drift item rather than propagating it.

## Content invariants

- No emojis in docs. Use plain text, Nerd Font glyphs or inline code.
- Keep the H1 as the first line; title/description extraction depends on it.
- The first prose line becomes the hub/search description; keep it meaningful.
- Do not add frontmatter; it is not parsed and would surface as body text.
- Keep doc-to-doc links relative (`.md`) so the language-aware rewriter works.
- Never hardcode `/web/...` inside Markdown; the renderer and Next.js handle the
  deployment base path.
- A page that exists in only one language is effectively invisible in the other
  two languages' navigation; mirror it or delete it everywhere.
