# homebrew-tap

Homebrew tap for the XFetch CLI on macOS.

## Role

`xfetch-cli/homebrew-tap` is the Homebrew tap repository behind the
`xfetch-cli/tap` tap name. It contains one formula,
`Formula/xfetch.rb`, which installs a prebuilt macOS binary from the
`xfetch-cli/xfetch` GitHub Releases. Nothing is compiled from source and no
Rust toolchain is required.

Homebrew maps a tap named `xfetch-cli/tap` to the repository
`https://github.com/xfetch-cli/homebrew-tap`; the repository name does not have
to match the tap name.

## Location

| Item | Value |
| --- | --- |
| Local clone | `~/repos/xfetch-cli/homebrew-tap` |
| Remote (fetch/push) | `https://github.com/xfetch-cli/homebrew-tap.git` |
| Default branch | `main` (`origin/HEAD -> origin/main`) |
| HEAD | `cc9ca03ed62401fac76904973ea0315e25bcb50d` |
| HEAD date | 2026-10-25 20:12:27 +0100 |
| HEAD subject | `Merge pull request #3 from xfetch-cli/release/v1.0.1` |
| Tracked files | `README.md`, `Formula/xfetch.rb` |

Remote branches visible in the local clone: `origin/main`,
`origin/release/1.0.0`, `origin/release/v1.0.1`. There are no tags in this
repository.

## Contents

```text
homebrew-tap/
|-- .git/
|-- README.md
`-- Formula/
    `-- xfetch.rb
```

There are no CI workflows, bottles, helper scripts, or license files here.

## Install

```bash
brew tap xfetch-cli/tap
brew install xfetch
```

The first command clones `github.com/xfetch-cli/homebrew-tap` into Homebrew's
tap directory. The second downloads the matching prebuilt tarball from the
`xfetch` v1.0.1 release and installs the `xfetch` binary into Homebrew's `bin`.

## Update

```bash
brew upgrade xfetch
```

## Uninstall

```bash
brew uninstall xfetch
brew untap xfetch-cli/tap
```

## Supported platforms

| Platform | Supported | Mechanism |
| --- | --- | --- |
| macOS arm64 (Apple Silicon) | Yes | `aarch64-apple-darwin` tarball |
| macOS x86_64 (Intel) | Yes | `x86_64-apple-darwin` tarball |
| Linux | No | No `on_linux` URL; use `install.sh`, the prebuilt installer, crates.io, or AUR |
| Windows | No | Use `install.ps1` or `install-prebuilt.ps1` |

Installing prebuilt binaries through Homebrew avoids macOS quarantine, so no
code signing or Gatekeeper bypass is needed. The formula does not define
`bottle` blocks; "prebuilt" here means the upstream release tarball, not a
Homebrew-built bottle.

## Related documentation in other repositories

- Main repository: https://github.com/xfetch-cli/xfetch
- Installation guide (Homebrew section):
  https://github.com/xfetch-cli/xfetch/blob/main/docs/INSTALLATION.md
- Release workflow that produces the macOS tarballs:
  `xfetch/.github/workflows/release.yml`
- Website install instructions:
  https://xfetch-cli.github.io/web/
- Organization profile: `https://github.com/xfetch-cli`

## Document index

- [`formula.md`](formula.md) - formula fields, URLs, checksums, platform
  selection, bump procedure, and verification.
- [`operations.md`](operations.md) - tap workflow, release coupling, common
  tasks, and gotchas.

Last verified: 2026-11-03 against `homebrew-tap@cc9ca03ed62401fac76904973ea0315e25bcb50d`.
