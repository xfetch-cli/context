# web architecture

Architecture of the Next.js 16 App Router site at `xfetch-cli/web`.

Last verified: 2026-11-03 against `web@09782e2d95582665e212213657652d6ab43c49ca` (`main`).

## Directory map

```text
web/
|-- docs/                       # The documentation source (Markdown, 3 languages)
|   |-- en/                     # 44 content pages + SUMMARY.md
|   |-- es/                     # same tree, Spanish
|   |-- de/                     # same tree, German
|-- public/
|   `-- previews/               # demo GIF/WebP captures used by /previews
|-- scripts/
|   |-- ci.sh                   # local CI: lint + build (Linux/macOS)
|   `-- ci.ps1                  # local CI: lint + build (Windows)
|-- src/
|   |-- app/                    # App Router routes and global CSS
|   |   |-- layout.tsx          # Root layout, fonts, nav, background
|   |   |-- page.tsx            # Landing page composition
|   |   |-- globals.css         # Design tokens + site-wide styles
|   |   |-- landing.css         # Landing-page-only styles/animations
|   |   |-- not-found.tsx       # 404, with old-docs-URL redirect hint
|   |   |-- previews/page.tsx   # /previews screenshot gallery
|   |   |-- robots.ts           # /robots.txt (force-static)
|   |   |-- sitemap.ts          # /sitemap.xml (force-static)
|   |   `-- docs/
|   |       |-- page.tsx        # /docs language portal
|   |       `-- [lang]/
|   |           |-- layout.tsx  # Docs shell + search for one language
|   |           |-- page.tsx    # /docs/<lang> group hub
|   |           `-- [...slug]/page.tsx  # /docs/<lang>/<slug...> article
|   |-- components/
|   |   |-- BgPaths.tsx         # Ambient grid/glow SVG background
|   |   |-- Footer.tsx          # Shared footer (VERSION from landing/links)
|   |   |-- Icons.tsx           # Inline SVG icon set + renderModuleIcon()
|   |   |-- Nav.tsx             # Legacy landing-only anchor nav (unused by root)
|   |   |-- SiteNav.tsx         # Real sticky nav, theme toggle, progress bar
|   |   |-- docs/               # DocArticle, DocsShell, DocsSearch, LangFlag
|   |   `-- landing/            # Landing sections and static data
|   `-- lib/
|       |-- docs.ts             # Build-time docs scanner, nav, search index
|       `-- docs-ui.ts          # Languages, groups, UI strings, descriptions
|-- next.config.ts              # Conditional static export + basePath
|-- eslint.config.mjs           # ESLint 9 flat config
|-- postcss.config.mjs          # @tailwindcss/postcss
|-- tsconfig.json               # strict TS, @/* -> src/*
|-- package.json / package-lock.json
|-- CHANGELOG.md / CONTRIBUTING.md / ROADMAP.md / SECURITY.md / SUPPORT.md
`-- .github/
    |-- workflows/deploy.yml        # GitHub Pages build + deploy
    |-- workflows/security-audit.yml# npm audit on push/PR/weekly
    `-- dependabot.yml             # weekly npm and Actions updates
```

## Route map

| URL | File | Rendering | Notes |
| --- | --- | --- | --- |
| `/` | `src/app/page.tsx` | Static | Landing composed of 14 sections (see below). |
| `/previews` | `src/app/previews/page.tsx` | Static | Gallery from `public/previews/`; uses `NEXT_PUBLIC_BASE_PATH`. |
| `/docs` | `src/app/docs/page.tsx` | Static | Language portal; counts pages per language, links popular English pages. |
| `/docs/<lang>` | `src/app/docs/[lang]/page.tsx` | Static (`generateStaticParams` over `en`,`es`,`de`) | Group hub with badge, language chips and per-group cards. |
| `/docs/<lang>/<slug...>` | `src/app/docs/[lang]/[...slug]/page.tsx` | Static, `dynamicParams = false` | Article. `generateStaticParams` enumerates every Markdown page in every language. |
| `/robots.txt` | `src/app/robots.ts` | `force-static` | Sitemap hardcoded to `https://xfetch-cli.github.io/web/sitemap.xml`. |
| `/sitemap.xml` | `src/app/sitemap.ts` | `force-static` | Base hardcoded to `https://xfetch-cli.github.io/web`; one entry per doc page. |
| any other | `src/app/not-found.tsx` | Client component | If the path looks like `/docs/<slug>`, it suggests `/docs/en/<slug>` (pre-i18n URL). |

There is no `/docs/<lang>/<slug>` rewrite table. The slug is the Markdown path
relative to `docs/<lang>/`, minus `.md`. Examples:

| Markdown source | Slug | URL |
| --- | --- | --- |
| `docs/en/configuration.md` | `configuration` | `/docs/en/configuration` |
| `docs/en/plugins/weather.md` | `plugins/weather` | `/docs/en/plugins/weather` |
| `docs/en/plugins/README.md` | `plugins/README` | `/docs/en/plugins/README` |
| `docs/en/extensions/wasm-night-mode.md` | `extensions/wasm-night-mode` | `/docs/en/extensions/wasm-night-mode` |

## Rendering and export mode

`next.config.ts` is environment-dependent:

```ts
const isGhPages = process.env.GITHUB_ACTIONS === "true";

const nextConfig: NextConfig = {
  ...(isGhPages && {
    output: "export",
    basePath: "/web",
    images: { unoptimized: true },
  }),
};
```

Consequences:

- A plain `npm run build` (local, no `GITHUB_ACTIONS`) produces a normal Next.js
  server build (`.next/`), not `out/`.
- In GitHub Actions, `GITHUB_ACTIONS=true` is set by the runner, so the build is
  a static export with `basePath: "/web"`.
- `NEXT_PUBLIC_BASE_PATH` is set to `/web` by the deploy workflow. It is read by
  `src/app/previews/page.tsx` and `src/components/docs/DocsShell.tsx` (the
  latter strips the base path before matching the active nav item). `next/link`
  and `next/font` handle the base path automatically.
- Docs pages are statically generated at build time. `generateStaticParams` in
  the `[lang]` layout and the catch-all page enumerate all languages and pages.
  `dynamicParams = false` makes unknown slugs a 404 instead of an on-demand
  render.
- `robots.ts` and `sitemap.ts` are explicitly `force-static` and hardcode the
  production origin `https://xfetch-cli.github.io/web`.

Because docs are read with `fs` at module scope (`src/lib/docs.ts`), the
documentation lives in the repo and is part of the build. There is no CMS, API
or database.

## Documentation pipeline

```text
docs/<lang>/*.md
        |
        v
src/lib/docs.ts
  - scanDocs()        walk dirs, skip SUMMARY.md, read Markdown
  - parseSummary()    parse docs/<lang>/SUMMARY.md (order + metadata)
  - buildLangData()   group, order, search index, hub metadata
        |
        v
src/app/docs/[lang]/layout.tsx   DocsShell + DocsSearch
src/app/docs/[lang]/page.tsx     group hub
src/app/docs/[lang]/[...slug]/page.tsx  article
        |
        v
src/components/docs/DocArticle.tsx  react-markdown rendering + TOC + pager
```

### From Markdown to page metadata

`src/lib/docs.ts` derives everything from the file, with no frontmatter:

| Field | Source |
| --- | --- |
| `title` | First `# Heading` in the file (`extractTitle`). |
| `description` | First non-heading, non-empty, non-`---` line (`extractDescription`). |
| `navTitle` | The title in `SUMMARY.md` if the slug is listed; otherwise `humanize()` of the last slug segment for nested pages, or the H1 for root pages. |
| `group` | `GROUP_BY_SLUG` map; nested `plugins/*`, `extensions/*`, `effects/*` fall back to the matching group; anything else is `more`. |
| `sourcePath` | `<slug>.md`, shown in the article footer. |
| `isIndex` | True for files named `README.md`. |
| `href` | `/docs/<lang>/<slug>`. |
| `search` | `toPlainText()` strips code fences, inline code, images, links markers, heading markers, HTML tags and markdown punctuation. |

### SUMMARY.md

`docs/<lang>/SUMMARY.md` has two jobs:

1. Hub metadata: the `# Title`, the first prose paragraph, and four bullets
   parsed in order into `version`, `license`, `author`, `repo`.
   Example from `docs/en/SUMMARY.md`:

   ```markdown
   - **Version:** 1.0.1
   - **License:** MIT
   - **Author:** xscriptor
   - **Repository:** github.com/xfetch-cli/xfetch
   ```

2. Ordering: numbered links `1. [Title](slug.md)` (with optional `#anchor`)
   produce the sidebar order and override `navTitle` for the listed slug.
   Nested/dir pages are not listed as numbered entries; they are appended by
   `buildLangData()`.

Pages not listed in `SUMMARY.md` are still scanned and sorted after the listed
root pages. `SUMMARY.md` itself is never rendered as a page.

### Navigation ordering rules (`buildLangData`)

1. Root pages follow `SUMMARY.md` link order, then remaining unlisted root pages
   sorted alphabetically.
2. Pages are bucketed by `GROUP_KEYS`:
   `start`, `reference`, `plugins`, `extensions`, `effects`, `project`, `more`.
3. Plugin group ordering: root-level plugin pages (except `wasm`) first, then
   every nested `plugins/*` page sorted by slug, then `wasm` last.
4. Extension group: root `extensions` page first, then nested `extensions/*`
   pages sorted by slug.
5. `plugins/README` (`isIndex`) is excluded from navigation and search but is
   still routable.

### i18n

`src/lib/docs-ui.ts` is the single source of truth:

- `DOCS_LANGS = ["en", "es", "de"]`
- `LANG_META` display names: English, Español, Deutsch
- `GROUP_KEYS` and `GROUP_LABELS` per language
- `GROUP_DESCRIPTIONS` per language (hub cards)
- `DOCS_UI` strings for search, TOC, pager, copy/source labels

There is no i18n library and no locale routing for the landing page. Only
documentation is multilingual; the landing/nav/footer is English.

### Search

`src/components/docs/DocsSearch.tsx` is a client-side search over the
precomputed `SearchEntry[]` passed from the server layout. It:

- normalizes diacritics (`fold`),
- requires a query of at least 2 characters and 2-character terms,
- scores title (60), slug (25), section (15) plus body term counts,
- returns up to 12 hits and displays 8,
- supports `/` and `Cmd/Ctrl+K` to focus, arrow keys and Enter to navigate.
  There is no external search service.

### Article rendering

`src/components/docs/DocArticle.tsx`:

- strips the leading H1 (already rendered in the header),
- splits the rest into sections on `##` / `###` outside code fences,
- slugifies headings for anchors and a right-hand "On this page" TOC,
- renders each section with `react-markdown` + `remark-gfm`,
- adds a copy button to every `<pre>` block,
- resolves relative Markdown links (`./x.md`, `../plugins/y.md`, `x#anchor`)
  to `/docs/<lang>/<slug>`, with `README` fallback,
- renders breadcrumb, language switcher, previous/next pager and source path.

## Components

### Landing (`src/components/landing/`)

| Component | Section id | Responsibility |
| --- | --- | --- |
| `Hero.tsx` | `hero-title` | Headline, version badge (`VERSION`), animated 3D/terminal visual, CTA. |
| `HeroVisual.tsx` | n/a | Isometric system/terminal illustration, version badge. |
| `InstallSection.tsx` | `install` | Install tabs and commands. |
| `DemoGallery.tsx` | `screens` | Screenshot/demo gallery. |
| `FeaturesSection.tsx` | `features` | Feature grid from `FEATURES`. |
| `TerminalDemo.tsx` | `terminal` | Animated typewriter terminal transcript. |
| `SystemMap.tsx` | `architecture` | Interactive architecture/system map. |
| `ModuleExplorer.tsx` | `modules` | Searchable/filterable module table from `MODULES`. |
| `LayoutExplorer.tsx` | `layouts` | Layout picker/preview from `LAYOUT_EXAMPLES`. |
| `ConfigBuilder.tsx` | `config` | Interactive `config.jsonc` generator (modules + layout). |
| `PluginSection.tsx` | `plugins` | Plugin cards from `PLUGINS` + `PluginSandbox`. |
| `PluginSandbox.tsx` | n/a | Decorative plugin JSON sandbox. |
| `UsageSection.tsx` | `usage` | Usage snippets. |
| `RoadmapSection.tsx` | `roadmap` | Phases from `PHASES`. |
| `UninstallSection.tsx` | `uninstall` | Uninstall commands. |
| `FinalCta.tsx` | `cta-title` | Closing call to action. |
| `CommandBlock.tsx` | n/a | Copyable command block. |
| `SectionHeader.tsx` | n/a | Shared eyebrow/title/lead markup. |
| `LandingEffects.tsx` | n/a | Scroll reveal / motion setup. |
| `data.ts` | n/a | All landing content constants. |
| `links.ts` | n/a | `REPO_URL` and `VERSION` (currently `1.0.1`). |

### Site chrome

| Component | Responsibility |
| --- | --- |
| `SiteNav.tsx` | Sticky nav, anchor links, active-section tracking, scroll progress, light/dark toggle (`xfetch-theme` in `localStorage`, `data-theme` on `<html>`), mobile menu. |
| `Nav.tsx` | Older landing-only anchor nav; not mounted by the root layout. |
| `Footer.tsx` | Brand, links (GitHub, Xscriptor, MIT, contributing), version text. |
| `BgPaths.tsx` | CSS/SVG ambient background (grid, glows, drifting lines). Pure CSS/SVG, reduced-motion aware. |
| `Icons.tsx` | Inline SVG icon set keyed by module/feature name; includes `renderModuleIcon()` and the X logo. |

### Docs

| Component | Responsibility |
| --- | --- |
| `docs/DocsShell.tsx` | Sidebar of grouped nav, mobile drawer, base-path-aware active link, scrolls active item into view. |
| `docs/DocsSearch.tsx` | Client search box and results panel. |
| `docs/DocArticle.tsx` | Article body, TOC, breadcrumbs, language switch, prev/next, copy buttons. |
| `docs/LangFlag.tsx` | Inline SVG flags for `en` (UK), `es`, `de`; other languages render `null`. |

## Data sources

| Source | Used by |
| --- | --- |
| `docs/<lang>/**/*.md` | Docs routes, hub counts, sidebar, search index, sitemap. |
| `docs/<lang>/SUMMARY.md` | Hub metadata and sidebar ordering. |
| `src/components/landing/data.ts` | Features, modules, layout examples, install tabs, plugins, roadmap phases, uninstall text. |
| `src/components/landing/links.ts` | `REPO_URL` and `VERSION`. |
| `public/previews/*` | `/previews` gallery (dimensions hardcoded in `previews/page.tsx`). |
| `package.json` | Dependency versions, scripts. |
| GitHub Actions env | `GITHUB_ACTIONS`, `NEXT_PUBLIC_BASE_PATH`. |

## Key files

| File | Why it matters |
| --- | --- |
| `next.config.ts` | The only place that switches between server build and static GH Pages export. |
| `src/lib/docs.ts` | All docs discovery, grouping, ordering, metadata and search indexing. |
| `src/lib/docs-ui.ts` | Languages, group labels/descriptions, UI strings. |
| `src/app/docs/[lang]/[...slug]/page.tsx` | Static generation contract for every article. |
| `src/components/docs/DocArticle.tsx` | Markdown rendering and link rewriting. |
| `src/components/landing/data.ts` | Single source for landing content (and the stale `configs/config.jsonc` snippet). |
| `.github/workflows/deploy.yml` | The deploy contract (basePath, `.nojekyll`, artifact, permissions). |
| `scripts/ci.sh` / `scripts/ci.ps1` | Local gate: lint + build. |

## Cross-repo links

The site documents and links to:

| Repository | Relationship |
| --- | --- |
| `xfetch-cli/xfetch` | Product; `REPO_URL`, install commands, docs source of truth. |
| `xfetch-cli/configs` | Presets documentation (`docs/en/presets.md`). |
| `xfetch-cli/themes` | Theme registry and `theme-manager` docs. |
| `xfetch-cli/logos` | Logo catalog used by `--gen-config`. |
| `xfetch-cli/plugins` | Official plugin pages under `docs/<lang>/plugins/`. |
| `xfetch-cli/extensions` | Extension pages under `docs/<lang>/extensions/`. |
| `xfetch-cli/effects` | Effects docs (`docs/<lang>/effects.md`). |
| `xfetch-cli/assets` | README banners and site hero banner hotlink. |

## Theming and styling invariants

- Dark theme is the default: `<html data-theme="dark">`. Light mode removes the
  attribute; the choice is stored as `xfetch-theme` in `localStorage`.
- The inline script in `src/app/layout.tsx` runs before hydration to add
  `xf-js` and apply the stored theme; this avoids a flash.
- `globals.css` defines tokens (`--accent`, `--fg`, `--bg`, ...) consumed by
  both the landing and docs CSS.
- The site is responsive down to mobile; the docs sidebar becomes a drawer
  under `lg`.
- Accessibility: skip link (`#content`), aria labels on nav/search/theme
  controls, reduced-motion handling in CSS, semantic headings and tables.

## Invariants to preserve

1. Every doc page must exist in `en`, `es` and `de`; missing files simply do not
   render and break language-switch symmetry.
2. Adding a page requires adding it to all three `SUMMARY.md` files for the
   intended order. Without a SUMMARY entry it still renders, but it is sorted
   alphabetically after the listed root pages.
3. Keep `dynamicParams = false`; otherwise unknown doc slugs could be rendered
   at runtime, which is incompatible with static export.
4. Do not use `next/image` for repository assets without changing the export
   config; images are unoptimized.
5. Any absolute URL to the site must include `/web/` for production or use the
   `NEXT_PUBLIC_BASE_PATH` prefix.
6. Docs links should stay relative (`.md`), not hardcoded `/docs/...`, so
   `DocArticle` can rewrite them per language.
