# assets

Canonical brand-asset repository for the `xfetch-cli` GitHub organization.

## Role

`xfetch-cli/assets` stores the shared visual identity used by the other
repositories. It currently contains a single tracked file: the transparent SVG
banner `logo/banner/xfetch.svg`. Every README in the ecosystem that shows the
XFetch banner hotlinks this file over `raw.githubusercontent.com`; the website
landing page does the same. There is no build step, package, or release
process. The `main` branch is the serving surface.

## Location

| Item | Value |
| --- | --- |
| Local clone | `~/repos/xfetch-cli/assets` |
| Remote (fetch/push) | `https://github.com/xfetch-cli/assets.git` |
| Default branch | `main` (`origin/HEAD -> origin/main`) |
| HEAD | `d1863feb0e532db131011497d4a518806b1c0022` |
| HEAD date | 2026-10-19 11:43:35 +0200 |
| HEAD subject | `Revert "add social preview card (#1)" (#2)` |
| Tracked files | 1 (see `inventory.md`) |

## How other repositories reference these assets

All consumers use the raw URL, not a submodule and not a copied file:

```text
https://raw.githubusercontent.com/xfetch-cli/assets/main/logo/banner/xfetch.svg
```

Confirmed consumers (exact files):

| Repository | File | Width | Alt text |
| --- | --- | --- | --- |
| `xfetch` | `README.md` line 2 | `100%` | `XFetch banner` |
| `api` | `README.md` line 2 | `30%` | `XFetch banner` |
| `configs` | `README.md` line 2 | `30%` | `XFetch banner` |
| `effects` | `README.md` line 2 | `30%` | `XFetch banner` |
| `extensions` | `README.md` line 2 | `30%` | `XFetch banner` |
| `plugins` | `README.md` line 2 | `30%` | `XFetch banner` |
| `themes` | `README.md` line 2 | `30%` | `XFetch banner` |
| `web` | `README.md` line 2 | `30%` | `XFetch banner` |
| `web` | `src/components/landing/Hero.tsx` lines 12-19 | `width={1000} height={175}` | `xfetch banner` |
| `.github` | `profile/README.md` line 2 | `60%` | `XFetch` |

`logos/README.md` does not use this banner; it has its own `<h1>` heading.

## Invariants

- The path `logo/banner/xfetch.svg` on `main` is a public contract. Moving or
  renaming it breaks every README and the website build.
- The file must stay a valid, standalone SVG with viewBox aspect ratio
  `1000:175` (approximately `5.714:1`). Consumers size it with a percentage
  width or fixed 1000x175.
- The default branch must remain `main`, because the URL hardcodes `/main/`.
- No license file, README, or build metadata exists in the repository itself;
  the project is MIT-licensed through the `xfetch` core repository.

## Operations in one screen

```bash
cd ~/repos/xfetch-cli/assets
git switch main
git pull --ff-only

# Replace the banner, then validate it.
xmllint --noout logo/banner/xfetch.svg
sha256sum logo/banner/xfetch.svg

# Confirm what GitHub serves after pushing (CDN may be stale briefly).
curl -fsSL -o /tmp/banner.svg \
  https://raw.githubusercontent.com/xfetch-cli/assets/main/logo/banner/xfetch.svg
sha256sum /tmp/banner.svg
```

Do not commit or push unless the task explicitly asks for it.

## Document index

- [`inventory.md`](inventory.md) - complete asset tree, file metadata, SVG
  composition, consumers, replacement procedure, and gotchas.

Last verified: 2026-11-03 against `assets@d1863feb0e532db131011497d4a518806b1c0022`.
