# .github operations

Operational notes for `xfetch-cli/.github`, the organization-level `.github`
repository.

## What lives here (and what does not)

At HEAD the repository contains only:

```text
.github/
|-- README.md           # maintainer note, 3 lines
`-- profile/README.md   # public organization profile, 76 lines
```

It does **not** contain:

- `.github/workflows/` (no organization workflows run from here)
- `.github/ISSUE_TEMPLATE/`
- `.github/PULL_REQUEST_TEMPLATE.md`
- `.github/workflow-templates/` (organization starter workflows)
- `SECURITY.md`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SUPPORT.md`, `FUNDING.yml`

Those files, where they exist, live in the individual repositories. For
example, `xfetch-cli/xfetch` has its own `SECURITY.md`, `CONTRIBUTING.md`,
`CODE_OF_CONDUCT.md`, `SUPPORT.md`, `.github/dependabot.yml`,
`.github/ISSUE_TEMPLATE/`, and `.github/workflows/`.

## Org-level conventions visible in this repository

There is no written org-wide policy file. The conventions are inferred from the
profile and the sibling repositories:

1. **Repository naming.** Lowercase, hyphenated: `homebrew-tap`, `plugins`,
   `themes`, `configs`, `effects`, `extensions`, `logos`, `assets`, `web`,
   `api`, and the core `xfetch`.
2. **Default branch.** Every cloned repository here uses `main`.
3. **Single source of brand imagery.** All READMEs and the profile hotlink the
   banner from `xfetch-cli/assets` on `main`; no repository vendors a copy.
4. **Docs are centralized.** User documentation lives in `web/docs/<locale>/`
   and in `xfetch/docs/`; the profile links there rather than duplicating
   content.
5. **Issue intake.** Feature requests and bug reports go to `xfetch-cli/xfetch`;
   plugins and themes go to their own repositories.
6. **Per-repository automation.** Each repository owns its CI. There is no
   central CI repository.

## Automation visible here

None. There are no workflows, hooks, bots, or schedulers in this repository.
`git log` contains only two commits (`b82ebc5` initial README, `9522504` add
organization profile).

Automation that exists elsewhere in the organization, for context:

| Repository | Automation |
| --- | --- |
| `xfetch` | `release.yml` on `v*` tags: gate (tag on main, Cargo version match, tests, `cargo audit`), build six targets, checksums, cosign signature + provenance attestation, publish release |
| `xfetch` | `rust-tests.yml` on push to `main` and PRs |
| `xfetch` | `security-audit.yml` |
| `xfetch` | `roadmap-sync.yml` plus `.github/scripts/sync_roadmap.py` |
| `xfetch` | `.github/dependabot.yml` |

Other repositories may have their own workflows; they are not defined here.

Important GitHub behavior: adding `.github/workflows/*.yml` to this repository
does not create organization-wide workflows. Only a repository's own workflows
run for that repository. Organization-level starter workflows are a separate
feature and belong in `.github/workflow-templates/`; reusable workflows must
live in a normal repository and be called with
`uses: <owner>/<repo>/.github/workflows/<file>.yml@<ref>`.

## Common tasks

### Update the organization profile

```bash
cd ~/repos/xfetch-cli/.github
git switch main
git pull --ff-only
# edit profile/README.md
git diff -- profile/README.md
# commit/PR only if requested
```

Then verify the rendered page at `https://github.com/xfetch-cli` after merge.

### Add a new repository to the profile

1. Decide whether it belongs under "Core repositories" or "Ecosystem".
2. Add a row with an absolute `https://github.com/xfetch-cli/<repo>` link and a
   one-line description.
3. Update the same table in this book's [`profile.md`](profile.md).
4. Verify the link resolves:

```bash
curl -fsSI https://github.com/xfetch-cli/<repo>
```

### Add organization-wide default community health files

GitHub falls back to files in this repository when a repository lacks its own.
Supported defaults include `SECURITY.md`, `CONTRIBUTING.md`,
`CODE_OF_CONDUCT.md`, `SUPPORT.md`, `FUNDING.yml`, `ISSUE_TEMPLATE/`, and
`PULL_REQUEST_TEMPLATE.md` placed in the repository root or `.github/`
directory, depending on the file type.

Procedure:

1. Add the file with the exact name GitHub expects.
2. Keep it generic enough for every repository.
3. Remember per-repository files override these defaults.
4. Test in a repository that lacks its own copy.

### Add organization starter workflows

Starter workflows shown in the "Actions -> New workflow" UI for organization
members live in:

```text
.github/workflow-templates/<name>.yml
.github/workflow-templates/<name>.properties.json
```

This is distinct from `.github/workflows/`, which would only run for this
repository.

### Verify the banner and links

```bash
curl -fsSI https://raw.githubusercontent.com/xfetch-cli/assets/main/logo/banner/xfetch.svg
curl -fsSI https://xfetch-cli.github.io/web/
curl -fsSI https://github.com/xfetch-cli/xfetch/blob/main/docs/INSTALLATION.md
```

## Gotchas

- **Profile path is exact.** `profile/README.md` on the default branch, in a
  public repository named `.github`. Anything else is ignored.
- **The root README is not the profile.** Editing `README.md` here does not
  change `https://github.com/xfetch-cli`.
- **No org-wide workflow magic.** Do not add a workflow here expecting it to run
  in every repository; it will not.
- **Community health defaults are fallbacks.** A repository that has its own
  `SECURITY.md` or issue templates ignores the org-level version.
- **Relative links are relative to this repository.** Use absolute URLs in
  `profile/README.md`; a bare `docs/CONFIGURATION.md` would resolve to
  `xfetch-cli/.github/docs/CONFIGURATION.md` and 404.
- **Banner coupling.** The profile depends on the `assets` repository's public
  `main` branch. A rename, default-branch change, or deletion breaks the image
  on the org page.
- **Manual drift.** The repository tables and install commands are static text.
  Nothing checks them. Repository renames, new repos, changed install scripts,
  and moved docs all require an edit here.
- **Stale content already present.** The profile says `assets` contains "social
  preview artwork" but the current `assets` HEAD has only the banner. The
  Windows install command also differs from the website's (`install-prebuilt.ps1`
  vs `install.ps1`).
- **Caching.** GitHub may serve a cached profile after a merge; a hard refresh
  or a short wait resolves it. The raw banner URL is also CDN-cached.
- **No CI to catch mistakes.** There is no markdown lint, link checker, or
  preview workflow. Validate links manually before merging.

## Quick reference

```bash
# Inspect
cat README.md
cat profile/README.md
git log --oneline --all -5
git remote -v

# Validate external dependencies
curl -fsSI https://raw.githubusercontent.com/xfetch-cli/assets/main/logo/banner/xfetch.svg

# Rendered profile
# https://github.com/xfetch-cli
```

Last verified: 2026-11-03 against `.github@9522504d6fae19fbb05b4bc1a7af75e4c1171f26`.
