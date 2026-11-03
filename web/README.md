# web

Landing page and documentation site for the `xfetch` ecosystem, published as a
GitHub Pages project site at `https://xfetch-cli.github.io/web/`.

Last verified: 2026-11-03 against `web@09782e2d95582665e212213657652d6ab43c49ca` (`main`).

## Role

`xfetch-cli/web` is the public face of xfetch and also the maintained home of
the user-facing documentation. It has two jobs:

1. Serve the marketing landing page (`/`) and the screenshot gallery
   (`/previews`).
2. Render the Markdown documentation stored in `docs/<lang>/` for English
   (`en`), Spanish (`es`) and German (`de`) under `/docs`.

It does not contain the Rust binary, the plugins, the presets or the theme
registry. It is a consumer (and documenter) of those repositories.

## Location

| Item | Value |
| --- | --- |
| Local clone | `~/repos/xfetch-cli/web` |
| Remote (fetch/push) | `https://github.com/xfetch-cli/web.git` |
| Default branch | `main` (`origin/HEAD -> origin/main`) |
| HEAD | `09782e2d95582665e212213657652d6ab43c49ca` |
| HEAD date | 2026-10-25 19:36:52 +0100 |
| HEAD subject | `Merge pull request #16 from xfetch-cli/design/web-redesign` |
| Package | `xfetch-web` 0.2.0, `private: true`, MIT |
| Published URL | `https://xfetch-cli.github.io/web/` |

## Stack

| Layer | Choice | Version / notes |
| --- | --- | --- |
| Framework | Next.js App Router | `next ^16.4.3` |
| UI runtime | React | `react` / `react-dom` 19.2.4 |
| Language | TypeScript | `typescript ^5`, `strict: true` |
| Styling | Tailwind CSS v4 | `@tailwindcss/postcss` + `src/app/globals.css` and `src/app/landing.css` |
| Animation | framer-motion | `^12.42.2` |
| Markdown | react-markdown + remark-gfm | `^10.1.0` / `^4.0.1` |
| Shared UI | `@xscriptor/xcomponents` | `^0.2.3` |
| Lint | ESLint 9 flat config | `eslint-config-next` 16.4.3 |
| Hosting | GitHub Pages | static export, `basePath: /web` |

`package.json` overrides `postcss` to `^8.5.10` and `sharp` to `^0.35.5` (security
fixes; the site does not use `next/image`, images are unoptimized).

## Quick start

```bash
cd ~/repos/xfetch-cli/web
npm ci

# Local dev server (http://localhost:3000)
npm run dev

# Lint and build (what CI runs)
npm run lint
npm run build
```

Node 20 is the version used by GitHub Actions (`actions/setup-node` with
`node-version: 20`). The local clone is normally used with the system Node; any
Node version supported by Next.js 16 works for development.

Static export exactly as GitHub Pages builds it:

```bash
GITHUB_ACTIONS=true NEXT_PUBLIC_BASE_PATH=/web npm run build
# result: ./out/ (base path /web is compiled into asset URLs)
```

To preview the exported site locally, serve a parent directory that contains an
`out` copy named `web` (the build is not rooted at `/`):

```bash
mkdir -p /tmp/xfetch-web-preview
rm -rf /tmp/xfetch-web-preview/web
cp -r out /tmp/xfetch-web-preview/web
npx serve /tmp/xfetch-web-preview
# open http://localhost:3000/web/
```

## Documentation index

| File | Contents |
| --- | --- |
| `README.md` | This file: role, location, stack, quick start. |
| `architecture.md` | Directory/route map, rendering/export mode, docs pipeline, components, data sources, cross-repo links. |
| `content.md` | Every documented page, language layout, mapping to the xfetch CLI docs, update workflow and known drift. |
| `operations.md` | Install/dev/build/lint/deploy, GitHub Actions, common tasks, troubleshooting. |

## Upstream project docs

- xfetch core: `https://github.com/xfetch-cli/xfetch`
- Configs/presets: `https://github.com/xfetch-cli/configs`
- Themes registry: `https://github.com/xfetch-cli/themes`
- Logos catalog: `https://github.com/xfetch-cli/logos`
- Plugins, extensions, effects: `xfetch-cli/plugins`, `xfetch-cli/extensions`,
  `xfetch-cli/effects`
