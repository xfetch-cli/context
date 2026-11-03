# profile/README.md

The public organization profile for `https://github.com/xfetch-cli`. This file
is rendered by GitHub at the organization landing page when the `.github`
repository is public and this file exists on the default branch.

## File facts

| Item | Value |
| --- | --- |
| Path | `profile/README.md` |
| Size | 76 lines |
| Format | GitHub-flavored Markdown with inline HTML |
| Language | English only |
| Banner | `https://raw.githubusercontent.com/xfetch-cli/assets/main/logo/banner/xfetch.svg` |
| Rendered at | `https://github.com/xfetch-cli` |

## Section-by-section contents

### 1. Banner and title (lines 1-9)

```html
<p align="center">
  <img src="https://raw.githubusercontent.com/xfetch-cli/assets/main/logo/banner/xfetch.svg" width="60%" alt="XFetch" />
</p>

<h1 align="center">XFetch</h1>

<p align="center">
  A fast, cross-platform system information fetch for the terminal, written in Rust.
</p>
```

- Centered banner, width `60%`, alt text `XFetch` (note: the READMEs use
  `XFetch banner` and widths of 30% or 100%; the profile is the only 60% user).
- Centered `<h1>`.
- Centered one-sentence tagline.

### 2. Feature paragraph (lines 11-13)

```text
XFetch keeps the default output clean and readable, then opens up when you want
more: themes, layouts, plugins, intro effects, config extensions, daemon mode,
and WebAssembly support for sandboxed plugins.
```

This is the product positioning: clean defaults plus optional themes, layouts,
plugins, effects, extensions, daemon mode, and WASM sandboxing.

### 3. Install (lines 15-35)

Three groups:

| Group | Command |
| --- | --- |
| Linux and macOS | `curl -fsSL https://raw.githubusercontent.com/xfetch-cli/xfetch/main/install.sh \| bash` |
| Windows (PowerShell) | `irm https://raw.githubusercontent.com/xfetch-cli/xfetch/main/install-prebuilt.ps1 \| iex` |
| crates.io | `cargo install xfetch-cli` |
| macOS Homebrew | `brew tap xfetch-cli/tap && brew install xfetch` |
| Arch Linux (AUR) | `yay -S xfetch-core-bin` |

The Windows line uses `install-prebuilt.ps1`; the website landing data uses
`install.ps1`. Both scripts exist in `xfetch-cli/xfetch`. Keep the intended
distinction in mind: `install-prebuilt.ps1` downloads a release binary, while
`install.ps1` is the other installer path. Do not change this line without
checking `xfetch`'s README.

### 4. Core repositories table (lines 37-44)

| Repository | Description as written |
| --- | --- |
| `xfetch` | Core binary: modules, layouts, daemon mode, themes, and the WASM runtime. |
| `plugins` | Official and community plugins: animated logos, system stats, weather, media, and package managers. |
| `themes` | Official theme registry with 80+ color palettes. |
| `configs` | Ready-to-use presets, layouts, and plugin configurations. |

### 5. Ecosystem table (lines 46-56)

| Repository | Description as written |
| --- | --- |
| `effects` | Installable intro animations over the information lines. |
| `extensions` | Config-level lifecycle extensions that can rewrite the resolved config before rendering. |
| `api` | SDK crates and WIT contracts for plugin, effect, and extension authors. |
| `logos` | ASCII logo catalog for Linux distributions, macOS, and Windows. |
| `web` | Website and documentation. |
| `homebrew-tap` | Homebrew formula for macOS. |
| `assets` | Brand assets, banners, and social preview artwork. |

The `assets` description currently overstates the repository: the social
preview artwork was added in `assets@94ce5e5` and reverted in
`assets@d1863fe`. The current asset tree has only the banner. Update the
description or restore the artwork; do not assume the `social/` directory
exists.

### 6. Documentation (lines 58-63)

```text
Website:       https://xfetch-cli.github.io/web/
Installation:  https://github.com/xfetch-cli/xfetch/blob/main/docs/INSTALLATION.md
Configuration: https://github.com/xfetch-cli/xfetch/blob/main/docs/CONFIGURATION.md
Plugins:       https://github.com/xfetch-cli/xfetch/blob/main/docs/PLUGINS.md
```

All four are absolute links to external repositories or GitHub Pages. They are
not relative to this repository.

### 7. Contributing and license (lines 65-76)

- Issues and PRs belong in `xfetch-cli/xfetch`.
- Plugin and theme contributions belong in their respective repositories.
- A star on the core repository is requested.
- License is MIT, linked to `xfetch-cli/xfetch/blob/main/LICENSE`.

## How the profile is rendered

GitHub's organization profile is not configurable through settings. The rules:

1. The repository must be named exactly `.github`.
2. The repository must be public.
3. The file must be `profile/README.md` on the default branch (`main`).
4. GitHub renders it at `https://github.com/xfetch-cli` and replaces the default
   repository list header with this content.

The root `README.md` in this repository is not used for the profile. Editing it
has no effect on the organization page.

## Branding rules implied by the file

- Banner is hotlinked from the `assets` repository on `main`; this repository
  stores no image.
- Profile uses `width="60%"`, centered; other READMEs use 30% or 100%.
- Product name is `XFetch` in prose, `xfetch` in commands and package names,
  `xfetch-cli` in GitHub URLs and organization names.
- No badges, no screenshots, no GIFs on the profile; the demo GIF is only in
  `xfetch/README.md`.
- Tone is short and factual, aimed at a first-time visitor.

## Update workflow

1. Edit `profile/README.md` in a branch.
2. Preview the Markdown locally or in the PR diff. Inline HTML and tables render
   the same way as on GitHub.
3. Confirm the banner URL still returns 200 and the correct bytes:

```bash
curl -fsSI https://raw.githubusercontent.com/xfetch-cli/assets/main/logo/banner/xfetch.svg
```

4. Confirm every link in the tables resolves. Repositories can be renamed or
   archived; the profile has no automated link checker.
5. Merge to `main`. The organization page updates from the default branch.
6. Hard-refresh `https://github.com/xfetch-cli` if the old content is cached.

Do not commit or push unless the task explicitly requests it.

## Gotchas

- **Wrong file path.** Only `profile/README.md` renders. A root `README.md` or
  `profile/readme.md` (case matters on some systems) will not.
- **Repository visibility.** If `.github` is made private, the profile
  disappears from the public organization page.
- **Banner dependency.** The profile breaks if `xfetch-cli/assets` becomes
  private, is renamed, changes its default branch, or deletes
  `logo/banner/xfetch.svg`.
- **Relative links resolve against this repo.** Links such as
  `docs/INSTALLATION.md` would point into `.github`, which has no `docs/`.
  Always use absolute URLs, as the current file does.
- **Manual synchronization.** The repository tables are hand-maintained. When a
  repository is added, renamed, or archived, this file must be updated in the
  same change window.
- **Stale claim.** The `assets` row mentions "social preview artwork" that is
  not currently present (reverted in `d1863fe`).
- **Install command drift.** The Windows command here is
  `install-prebuilt.ps1`, while the website landing data uses `install.ps1`.
  Treat this as a known divergence, not an error to silently "fix" in one place.
- **No i18n.** The profile is English-only even though the website has
  `en`, `es`, and `de` docs.

Last verified: 2026-11-03 against `.github@9522504d6fae19fbb05b4bc1a7af75e4c1171f26`.
