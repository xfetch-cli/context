# assets inventory

Complete inventory of `xfetch-cli/assets` as of HEAD
`d1863feb0e532db131011497d4a518806b1c0022` (2026-10-19).

## Tree

```text
assets/
|-- .git/
`-- logo/
    `-- banner/
        `-- xfetch.svg
```

There is exactly one tracked file. There is no `README.md`, `LICENSE`,
`social/`, PNG, or icon file in the working tree today.

## File inventory

| Path | Format | Size | Dimensions | Role |
| --- | --- | --- | --- | --- |
| `logo/banner/xfetch.svg` | SVG 1.1 (XML) | 2,024 bytes, 41 lines | `width="100%" height="100%"`, `viewBox="0 80 1000 175"` | Primary horizontal banner used in READMEs, the org profile, and the website hero |

Integrity and metadata:

```text
sha256  c4f84db3b376a02ef4cc6a41992b64015cba14fc449873bf79b0417ce6b0c9bf
lines   41
bytes   2024
aspect  1000 / 175 = 5.714285...
```

The file validates cleanly:

```bash
xmllint --noout logo/banner/xfetch.svg   # exits 0
```

## SVG composition

`logo/banner/xfetch.svg` is a hand-written SVG with a transparent background.
It has no `<defs>`, no gradients, no embedded raster image, and no embedded
font. It relies on the generic `sans-serif` family. Content, in document order:

1. A large capital `X` at `x="80" y="230"`, `font-size="160"`, `font-weight="bold"`,
   `fill="black"`, `stroke="#FFC107"` (amber), `stroke-width="4"`.
2. A white rectangle at `x="220" y="115"` with a black outline, followed by the
   word `fetch` at `x="283" y="132"`, `font-size="32"`, bold, `fill="white"`,
   `stroke="black"`, `stroke-width="1.5"`.
3. Five staggered rows of colored rectangles that suggest terminal fetch output.

The rectangle palette is:

```text
yellow-green  #58A55C   yellow     #FFD500   light blue  #81D4FA
pink          #F48FB1   black      #000000   mint        #B9F6CA
pale green    #C8E6C9   red        #EF5350   green       #4CAF50
amber         #FFCA28   orange     #FFA726   light green #66BB6A
lavender      #E1BEE7   white      #FFFFFF   light orange#FFE0B2
light yellow  #FDD835   cyan       #80DEEA
```

The `viewBox` origin is `y=80`, not `y=0`, because the banner was cropped from
a taller 1000x350 canvas. The effective drawing area is `x=0..1000`,
`y=80..255`.

## History and naming conventions

```text
2f85812 2026-07-21  upload banner xfetch
                    added logo/banner/xfetch.svg, viewBox="0 0 1000 350"
22ea4b9 2026-07-21  update logo
                    cropped viewBox to "0 80 1000 175", added trailing newline
94ce5e5 2026-10-19  add social preview card (#1)
                    added social/README.md, social/term.png,
                    social/xfetch-social-preview.png, social/xfetch-social-preview.svg
d1863fe 2026-10-19  Revert "add social preview card (#1)" (#2)
                    removed the whole social/ directory
```

The reverted `social/` assets are recoverable from history but are not part of
the current tree:

```bash
git show 94ce5e556f13ebec1f246735b3eb251964d7d2fd:social/README.md
git show 94ce5e556f13ebec1f246735b3eb251964d7d2fd:social/xfetch-social-preview.svg
git cat-file -s $(git rev-parse 94ce5e556f13ebec1f246735b3eb251964d7d2fd:social/term.png)                 # 131562
git cat-file -s $(git rev-parse 94ce5e556f13ebec1f246735b3eb251964d7d2fd:social/xfetch-social-preview.png)  # 175212
```

The reverted social README documented a 1280x640 social preview made from
`xfetch-social-preview.svg` plus a terminal capture `term.png`, regenerated with:

```bash
rsvg-convert -w 1280 -h 640 -o xfetch-social-preview.png xfetch-social-preview.svg
```

Naming convention in use: `<kind>/<variant>/<name>.<ext>`, lowercase, with a
single product name (`xfetch`). Banners live under `logo/banner/`. A social
preview, if restored, belongs under `social/`.

## Which repository uses which asset

All uses target `logo/banner/xfetch.svg` through this raw URL:

```text
https://raw.githubusercontent.com/xfetch-cli/assets/main/logo/banner/xfetch.svg
```

| Consumer | Exact location | Sizing |
| --- | --- | --- |
| `xfetch` | `README.md` line 2 | `width="100%"` |
| `api` | `README.md` line 2 | `width="30%"` |
| `configs` | `README.md` line 2 | `width="30%"` |
| `effects` | `README.md` line 2 | `width="30%"` |
| `extensions` | `README.md` line 2 | `width="30%"` |
| `plugins` | `README.md` line 2 | `width="30%"` |
| `themes` | `README.md` line 2 | `width="30%"` |
| `web` | `README.md` line 2 | `width="30%"` |
| `web` | `src/components/landing/Hero.tsx` lines 12-19 | `width={1000} height={175}`, CSS class `xf-hero-banner` |
| `.github` | `profile/README.md` line 2 | `width="60%"`, alt `XFetch` |

Notes:

- `web/src/components/landing/Hero.tsx` intentionally bypasses `next/image`;
  the comment explains that `next/image` would require `remotePatterns` plus
  `dangerouslyAllowSVG`. Do not convert it without addressing that.
- `web/docs/es/contributing.md` lists `assets/` in the ecosystem tree and
  describes it as logo/banner files, but does not embed the image.
- No consumer vendors a local copy. A repository-wide search finds only the
  single file in this repo matching the SVG header.

## How to add or replace an asset

### Replace the existing banner

1. Keep the exact path `logo/banner/xfetch.svg`; consumers cannot be updated in
   the same change across all repositories.
2. Preserve the viewBox aspect ratio around `1000:175` and keep the
   background transparent unless every consumer is audited for background
   assumptions.
3. Validate XML and check that the file is non-empty:

```bash
cd ~/repos/xfetch-cli/assets
xmllint --noout logo/banner/xfetch.svg
sha256sum logo/banner/xfetch.svg
```

4. Commit on a branch and open a PR (or commit directly only if the task says
   so). After merge, confirm the raw URL serves the new bytes.

### Add a new asset

1. Put it under `logo/<variant>/` (for example `logo/icon/xfetch.svg`) or
   `social/` for social preview artwork.
2. Use lowercase kebab-case file names.
3. Prefer SVG for vector marks and PNG only where the consumer cannot render
   SVG (for example GitHub social preview uploads).
4. Add the asset to this inventory and to `README.md` if it becomes a public
   contract.

### Restore the social preview

Only do this if the task asks for it. GitHub does not accept SVG as an uploaded
social preview; a PNG/JPG/GIF is required, and the setting is not exposed by
the REST or GraphQL API. The historical procedure was:

1. Recover or recreate `social/xfetch-social-preview.svg` and `social/term.png`.
2. Render a 1280x640 PNG under 1 MB:

```bash
rsvg-convert -w 1280 -h 640 -o xfetch-social-preview.png xfetch-social-preview.svg
ls -l xfetch-social-preview.png
```

3. Upload it manually at
   `https://github.com/xfetch-cli/xfetch/settings` under
   `Social preview -> Edit -> Upload an image`.

## Gotchas

- **The raw URL hardcodes `main`.** Renaming the default branch or the file is
  a breaking change for every README and for the deployed website.
- **raw.githubusercontent.com is cached.** A push can take a few minutes to
  appear. Do not add cache-busting query strings to consumer URLs unless you
  update every consumer; instead retry with a fresh `curl` before concluding
  the push failed.
- **`width="100%" height="100%"` is not a display size.** The SVG fills its
  container. README `<img width="30%">` and the Hero component's explicit
  1000x175 are what constrain it. Opening the raw SVG in a browser makes it
  fill the viewport.
- **Transparent background.** The black `X` and the black stroke around
  `fetch` are designed to read against light surfaces; on very dark surfaces
  the colored bars remain visible but the black fills can disappear. Test both
  GitHub light and dark themes before changing the palette.
- **No embedded font.** Rendering depends on the host's generic `sans-serif`
  and on font fallback; the exact glyph metrics vary by OS and browser.
- **GitHub social preview does not accept SVG.** The profile's claim that this
  repo contains "social preview artwork" is currently stale: the `social/`
  directory was added in `94ce5e5` and fully reverted in the current HEAD
  `d1863fe`.
- **Do not copy the banner into consumers.** The single-source-of-truth design
  is intentional; a local copy silently drifts.
- **No size budget besides practical sanity.** The current file is 2 KB. Avoid
  embedding rasters or huge font subsets; README rendering and the website
  hero both fetch it on page load.
- **Line endings and BOM.** Keep UTF-8 without BOM; `xmllint` rejects some
  malformed variants. The current file ends with a newline.

Last verified: 2026-11-03 against `assets@d1863feb0e532db131011497d4a518806b1c0022`.
