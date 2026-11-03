# .github

Organization-level `.github` repository for the `xfetch-cli` GitHub
organization.

## Role

`xfetch-cli/.github` is the special repository GitHub uses for organization-wide
defaults. In this organization it currently does exactly one thing: it renders
the public organization profile from `profile/README.md`. There are no
organization-wide workflows, issue templates, security policies, or funding
metadata in this repository at HEAD.

GitHub renders the organization profile only when the repository is named
`.github`, is public, and contains `profile/README.md` on its default branch.
The root `README.md` is a maintainer note, not the profile.

## Location

| Item | Value |
| --- | --- |
| Local clone | `~/repos/xfetch-cli/.github` |
| Remote (fetch/push) | `https://github.com/xfetch-cli/.github.git` |
| Default branch | `main` (`origin/HEAD -> origin/main`) |
| HEAD | `9522504d6fae19fbb05b4bc1a7af75e4c1171f26` |
| HEAD date | 2026-10-19 11:33:00 +0200 |
| HEAD subject | `add organization profile` |
| Tracked files | `README.md`, `profile/README.md` |

## Contents

```text
.github/
|-- .git/
|-- README.md
`-- profile/
    `-- README.md
```

`README.md` (3 lines) says:

```text
# XFetch organization profile

The public organization profile is rendered from [profile/README.md](profile/README.md).
```

`profile/README.md` is the full 76-line public profile. See
[`profile.md`](profile.md) for a line-by-line breakdown.

## History

```text
b82ebc5 2026-10-19  Initial commit
                    README.md with "# .github / Organization profile for XFetch"
9522504 2026-10-19  add organization profile
                    rewrote README.md, added profile/README.md
```

No branches other than `main`; no tags; no CI runs from this repository.

## Related repositories

The profile links to the rest of the organization. The canonical organization
URL is https://github.com/xfetch-cli.

| Repository | Purpose |
| --- | --- |
| `xfetch` | Core CLI |
| `plugins` | Official and community plugins |
| `themes` | Theme registry |
| `configs` | Presets and layouts |
| `effects` | Intro animations |
| `extensions` | Config lifecycle extensions |
| `api` | SDK crates and WIT contracts |
| `logos` | ASCII logo catalog |
| `web` | Website and documentation |
| `homebrew-tap` | macOS Homebrew formula |
| `assets` | Brand assets (banner used by the profile) |

## Document index

- [`profile.md`](profile.md) - what the org profile contains, how it is
  rendered, branding, links, and the update workflow.
- [`operations.md`](operations.md) - org-level conventions, visible automation,
  common tasks, and gotchas.

Last verified: 2026-11-03 against `.github@9522504d6fae19fbb05b4bc1a7af75e4c1171f26`.
