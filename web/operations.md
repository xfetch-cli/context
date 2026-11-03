# web operations

Install, run, build, deploy and troubleshoot the site at `xfetch-cli/web`.

Last verified: 2026-11-03 against `web@09782e2d95582665e212213657652d6ab43c49ca` (`main`).

## Prerequisites

| Tool | Version | Notes |
| --- | --- | --- |
| Node.js | 20 (CI) | Any version supported by Next.js 16 works locally. |
| npm | bundled with Node | The lockfile is `package-lock.json`; use `npm ci`. |
| Python | not needed | The docs are Markdown; no docs build script. |
| Git | any | Deploy is driven by pushes to `main`. |

No Rust toolchain, database or external service is required.

## Install and dev

```bash
cd ~/repos/xfetch-cli/web
npm ci
npm run dev
# http://localhost:3000
```

`npm install` works too but only `npm ci` is reproducible from the lockfile and
is what CI uses.

## Lint, build and local CI

```bash
npm run lint     # eslint (flat config, next core-web-vitals + typescript)
npm run build    # next build
npm start        # serve the production server build
```

Local CI wrapper (the project's documented pre-commit gate):

```bash
bash scripts/ci.sh         # Linux/macOS
./scripts/ci.ps1           # Windows PowerShell
```

Both scripts `cd` to the repository root and run, in order:

1. `npm run lint`
2. `npm run build`

There is no unit-test or end-to-end suite in this repository. The deploy
workflow is the real integration test.

Important: a local `npm run build` without `GITHUB_ACTIONS=true` produces a
server build, not a static `out/` directory. See "Static export" below.

## Static export (GitHub Pages)

The production artifact is a static export with base path `/web`:

```bash
GITHUB_ACTIONS=true NEXT_PUBLIC_BASE_PATH=/web npm run build
touch out/.nojekyll
```

Expected result: `out/` contains `index.html`, `previews/`, `docs/`, and a
`404.html` (Next.js export). `next.config.ts` only enables
`output: "export"`, `basePath: "/web"` and `images.unoptimized` when
`GITHUB_ACTIONS === "true"`.

Preview locally:

```bash
mkdir -p /tmp/xfetch-web-preview
rm -rf /tmp/xfetch-web-preview/web
cp -r out /tmp/xfetch-web-preview/web
npx serve /tmp/xfetch-web-preview
# http://localhost:3000/web/
```

Opening `out/index.html` directly with `file://` does not work: the export uses
absolute `/web/...` asset URLs.

## Deploy pipeline

File: `.github/workflows/deploy.yml`.

Triggers:

- `push` to `main`, ignoring changes to `README.md` and `.gitignore`.
- `workflow_dispatch` (manual run).

Concurrency: group `pages`, `cancel-in-progress: false`.

### build job

| Step | Detail |
| --- | --- |
| `actions/checkout` | Pinned to `11d5960a...` (v4.4.0). |
| `actions/setup-node` | Pinned to `49933ea5...` (v4.4.0), Node 20, `cache: npm`. |
| `npm ci` | Clean lockfile install. |
| `npm run build` | `env: NEXT_PUBLIC_BASE_PATH=/web`; `GITHUB_ACTIONS` is already true. |
| `touch out/.nojekyll` | Prevents GitHub Pages/Jekyll from hiding `_next/` assets. |
| `actions/configure-pages` | Pinned `983d7736...` (v5.0.0), `pages: read`. |
| `actions/upload-pages-artifact` | Pinned `56afc609...` (v3.0.1), path `out`. |

Job permissions: `contents: read`, `pages: read`. Deployment/OIDC permissions
are intentionally granted only to the deploy job.

### deploy job

- `needs: build`
- `environment: github-pages`
- permissions: `pages: write`, `id-token: write`
- `actions/deploy-pages` pinned to `d6db9016...` (v4.0.5)

The published site is `https://xfetch-cli.github.io/web/`. The repository must
have Pages configured to use GitHub Actions as the source.

## Security and dependency automation

| File | Behaviour |
| --- | --- |
| `.github/workflows/security-audit.yml` | On every push, every PR, and Mondays 06:23 UTC: checkout with `persist-credentials: false`, Node 20, `npm ci --ignore-scripts`, `npm audit --audit-level=high`. |
| `.github/dependabot.yml` | Weekly npm updates (minor/patch grouped as `npm-minor-patch`, max 5 PRs) and GitHub Actions updates (grouped as `github-actions`, max 5 PRs). |

All workflow actions are pinned by commit SHA. Update the SHA and the version
comment together.

## Common tasks

### Add a documentation page

1. Create `docs/en/<slug>.md`, `docs/es/<slug>.md`, `docs/de/<slug>.md`.
2. Start each file with `# Title` and a one-line description.
3. Add a numbered entry to all three `SUMMARY.md` files (or accept automatic
   alphabetical placement after listed root pages).
4. For nested pages, use `docs/<lang>/plugins/<name>.md` or
   `docs/<lang>/extensions/<name>.md`; nav picks them up automatically.
5. Run `bash scripts/ci.sh`.
6. Check `/docs/en/<slug>`, `/docs/es/<slug>`, `/docs/de/<slug>` and the
   language switcher.

### Add a language

This is a code change, not just adding a folder:

1. Add the language code to `DOCS_LANGS` and `LANG_META` in
   `src/lib/docs-ui.ts`.
2. Add `GROUP_LABELS`, `GROUP_DESCRIPTIONS` and `DOCS_UI` entries for it.
3. Add a flag in `src/components/docs/LangFlag.tsx`.
4. Create `docs/<lang>/` with all 44 pages plus `SUMMARY.md`.
5. `generateStaticParams` and the sitemap pick it up from `DOCS_LANGS`; no route
   file changes are needed.
6. The `/docs` portal card grid is `sm:grid-cols-3`; consider layout changes if
   the number of languages grows past three.

### Update the displayed xfetch version

1. `src/components/landing/links.ts` → `VERSION`.
2. `docs/en/SUMMARY.md`, `docs/es/SUMMARY.md`, `docs/de/SUMMARY.md` → version
   bullet.
3. `docs/<lang>/changelog.md` if the release notes are part of the update.
4. Run CI and search for the old version string:

   ```bash
   grep -rn "1\.0\.1" src docs
   ```

### Add a preview screenshot

1. Put the file in `public/previews/`.
2. Add an entry to the relevant array (`LINUX_STILLS`, `MACOS_STILLS`,
   `WINDOWS_STILLS`) or the demo section in `src/app/previews/page.tsx`.
3. If it is a WebP still, add its intrinsic dimensions to `IMAGE_DIMENSIONS` so
   the page avoids layout shift.
4. Preview at `/previews` and confirm the base-path prefix works in production.

### Change navigation or theme UI

- Landing anchors: `SiteNav.tsx` (`NAV_ITEMS`, `SECTION_IDS`) and the section
  components' `id` attributes.
- Docs sidebar labels: `GROUP_LABELS` in `src/lib/docs-ui.ts`.
- Docs group assignment: `GROUP_BY_SLUG` in `src/lib/docs.ts`.
- Theme toggle: `SiteNav.tsx` + the boot script in `src/app/layout.tsx`, key
  `xfetch-theme`, attribute `data-theme`.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `npm run build` has no `out/` | `GITHUB_ACTIONS` is not `true`. | Set `GITHUB_ACTIONS=true` for a local export; leave it unset for a server build. |
| Deployed site has broken CSS/JS under `/web/` | `.nojekyll` missing or base path not set. | The workflow touches `out/.nojekyll`; keep `NEXT_PUBLIC_BASE_PATH=/web`. |
| Images 404 only in production | A hardcoded `/previews/...` path ignores the base path. | Use `NEXT_PUBLIC_BASE_PATH` as `previews/page.tsx` does. |
| `/docs/<slug>` 404s | Docs moved to `/docs/<lang>/<slug>`. | The 404 page offers `/docs/en/<slug>`; update the link. |
| A doc page is missing from the sidebar | Slug absent from `SUMMARY.md` and/or grouped under `more`. | Add it to `SUMMARY.md` or `GROUP_BY_SLUG`. |
| Language switch 404s | Page exists in one language only. | Add the missing Markdown files. |
| Relative doc links 404 | Target is not a known slug and has no `README` fallback. | Use the correct `.md` relative path or an absolute URL. |
| `npm ci` fails after a lockfile edit | `package.json` and `package-lock.json` are out of sync. | Regenerate with `npm install` and commit both. |
| Lint fails on unused imports | `eslint-config-next` treats unused vars as errors. | Remove the import or explain a justified disable. |
| Build fails after a Node upgrade | Next.js 16 requires a supported Node range. | Use Node 20 (CI) or a current LTS. |
| Security audit fails | `npm audit --audit-level=high` found a high/critical advisory. | Update the dependency or add an override in `package.json`. |
| `sharp`/`postcss` install warnings | Overrides pin `sharp ^0.35.5` and `postcss ^8.5.10`. | Do not remove the overrides without checking audits. |
| Docs search misses content | Query under 2 characters, or the page is `isIndex` (excluded). | Search longer terms; index pages are intentionally excluded. |

## Verification status

At the time of writing the validators for the sibling repositories were run
successfully, but the web build was not executed in this environment because
`node_modules/` is not installed and installing it would modify the working
tree. Commands above were verified against `package.json`, `next.config.ts`,
`scripts/ci.sh`, `scripts/ci.ps1` and the workflow files.
