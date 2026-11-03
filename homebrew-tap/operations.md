# homebrew-tap operations

Operational guide for maintaining `xfetch-cli/homebrew-tap`, the repository
behind the `xfetch-cli/tap` Homebrew tap.

## Tap workflow

Homebrew resolves `xfetch-cli/tap` to the repository
`https://github.com/xfetch-cli/homebrew-tap`. The important consequence is that
the tap name (`tap`) and repository name (`homebrew-tap`) differ; do not rename
the repository to `tap`.

```text
brew tap xfetch-cli/tap
        |
        v
git clone https://github.com/xfetch-cli/homebrew-tap
        |
        v
$(brew --repository)/Library/Taps/xfetch-cli/homebrew-tap/
        |
        +-- Formula/xfetch.rb   -> formula "xfetch"
```

Useful inspection commands:

```bash
brew tap-info xfetch-cli/tap
brew cat xfetch                 # prints Formula/xfetch.rb
brew info xfetch                # version, URLs, install status
brew edit xfetch                # edit the tapped copy (not this clone)
brew outdated                   # shows pending upgrades
```

`brew edit xfetch` edits the tap's clone under `$(brew --repository)`, not the
working clone at `~/repos/xfetch-cli/homebrew-tap`.
Make changes in the working clone and PR them.

## Branch and review workflow

Observed history uses a release branch plus a pull request:

```text
3e444f7 init homebrew formula for xfetch
45ca446 Merge pull request #1 from xfetch-cli/release/0.9.0
de472d4 Update to 0.9.0
d4c1572 Merge pull request #2 from xfetch-cli/release/1.0.0
368d67d Update to 1.0.0
cc9ca03 Merge pull request #3 from xfetch-cli/release/v1.0.1
f9b920d xfetch 1.0.1
```

Convention: branch `release/<version>` (older) or `release/v<version>` (newer),
commit message `xfetch <version>` or `Update to <version>`, PR into `main`.
Merge commits are the norm. The repository has no tags and no CI; review and
verification are manual.

## Release coupling to xfetch tags

The tap is downstream of `xfetch-cli/xfetch` releases. The chain is:

```text
xfetch commit on main
  -> tag vX.Y.Z
  -> .github/workflows/release.yml gate:
       - tag commit is ancestor of origin/main
       - tag version == Cargo.toml [package].version
       - cargo test --locked
       - cargo audit --deny warnings
  -> build six targets
  -> validate archives + SHA256SUMS
  -> cosign-sign SHA256SUMS, attest provenance
  -> publish GitHub Release with archives + SHA256SUMS + SHA256SUMS.bundle
  -> manual tap bump in homebrew-tap
```

There is no automation that bumps this tap. Neither repository contains a
workflow that edits `Formula/xfetch.rb`. The tap bump is a human step after the
release is published.

The tap must use:

- release tag `vX.Y.Z` (with `v`) in the URL;
- version `X.Y.Z` (without `v`) in `version` and in the archive file names;
- the `apple-darwin` archives only, one per architecture.

## Common tasks

### Install and inspect the current formula

```bash
brew tap xfetch-cli/tap
brew install xfetch
brew info xfetch
xfetch --version
```

### Update to a new upstream release

Follow `formula.md` section "Bump procedure". Short version:

```bash
VERSION=1.0.2
curl -fsSL -o /tmp/SHA256SUMS \
  "https://github.com/xfetch-cli/xfetch/releases/download/v${VERSION}/SHA256SUMS"
grep 'apple-darwin' /tmp/SHA256SUMS
# edit Formula/xfetch.rb: version, two URLs, two sha256 values
ruby -c Formula/xfetch.rb
```

Then audit/test on a Mac, commit on `release/v${VERSION}`, push, open a PR.

### Verify a checksum locally

```bash
VERSION=1.0.2
TARGET=aarch64-apple-darwin
curl -fsSL -o /tmp/xfetch.tar.gz \
  "https://github.com/xfetch-cli/xfetch/releases/download/v${VERSION}/xfetch-${VERSION}-${TARGET}.tar.gz"
shasum -a 256 /tmp/xfetch.tar.gz     # macOS
sha256sum /tmp/xfetch.tar.gz         # Linux
tar -tzf /tmp/xfetch.tar.gz          # should contain ./xfetch, ./LICENSE, ./README.md
```

Compare the first column with the `sha256` in the formula.

### Roll back a bad bump

Revert the bump commit or open a PR that restores the previous version, URLs,
and checksums. Do not leave a formula pointing at a deleted or overwritten
release asset. Re-test after rollback:

```bash
brew reinstall xfetch
xfetch --version
```

### Remove the tap locally

```bash
brew uninstall xfetch
brew untap xfetch-cli/tap
```

## Gotchas

- **Tap name vs repository name.** `brew tap xfetch-cli/tap` clones
  `homebrew-tap`, not `tap`. A repository rename would require a new tap name
  and would break every documented install command.
- **Version formatting.** `version "1.0.1"` has no `v`; the URL path is
  `/download/v1.0.1/` with a `v`; the archive name embeds `1.0.1` without `v`.
  Mixing these up produces 404s or checksum failures.
- **Asset immutability.** GitHub allows replacing a release asset, but Homebrew
  clients and caches assume immutability. A replaced asset with a different
  SHA-256 breaks installs for anyone who cached the old one.
- **Architecture mix-ups.** The arm64 and x86_64 checksums are adjacent in
  `SHA256SUMS`; always match the target substring.
- **macOS only.** There is no `on_linux` branch. Linuxbrew (`brew` on Linux)
  cannot install this formula; Linux users should use `install.sh`,
  `install-prebuilt.sh`, `cargo install xfetch-cli`, or AUR `xfetch-core-bin`.
- **No bottles.** `brew install` downloads the upstream release tarball. It is
  not a Homebrew-built bottle, so `brew bottle` and bottle upload workflows do
  not apply.
- **No CI.** A syntax error or wrong checksum is not caught until a user
  installs. Run `ruby -c` and `brew audit --strict --online` before merging.
- **Security drift.** The formula at HEAD serves v1.0.1. The v1.0.2 changelog
  says v1.0.1 binaries remain affected by wasmtime/zstd advisories. Treat the
  v1.0.2 bump as urgent once the release exists.
- **Prebuilt binaries and Gatekeeper.** Homebrew-installed binaries are not
  quarantined, so no signing is required. If a user downloads a release tarball
  manually and moves it into place, Gatekeeper behavior is different; direct
  them to Homebrew or the install scripts.
- **`brew upgrade` visibility.** The tap must be refreshed (`brew update`) for
  a new formula version to appear. `HOMEBREW_NO_AUTO_UPDATE=1` delays updates.
- **`brew test` is shallow.** The test only asserts that `xfetch --version`
  contains the formula version. It does not exercise rendering or plugins.

## Quick reference

```bash
# User-facing commands
brew tap xfetch-cli/tap
brew install xfetch
brew upgrade xfetch
brew uninstall xfetch
brew untap xfetch-cli/tap

# Maintainer commands
brew audit --strict --online xfetch-cli/tap/xfetch
brew fetch --force xfetch
brew reinstall xfetch
brew test xfetch
ruby -c Formula/xfetch.rb
```

Last verified: 2026-11-03 against `homebrew-tap@cc9ca03ed62401fac76904973ea0315e25bcb50d`.
