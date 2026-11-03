# Formula/xfetch.rb

Single-formula Homebrew tap for the XFetch CLI. This document describes the
formula exactly as committed at `homebrew-tap@cc9ca03`.

## File facts

| Item | Value |
| --- | --- |
| Path | `Formula/xfetch.rb` |
| Size | 778 bytes, 25 lines |
| SHA-256 of file | `80efd0b6c9aef96fbcdadcce159e90053f28ca17c74cf731d2cf3b10814a76cb` |
| Class name | `Xfetch < Formula` |
| Formula name | `xfetch` |
| Homebrew tap | `xfetch-cli/tap` |
| Upstream project | `https://github.com/xfetch-cli/xfetch` |

The path, file name, and class name are coupled: Homebrew expects
`Formula/xfetch.rb` to define `class Xfetch`. Renaming the file changes the
installable formula name.

## Current contents

```ruby
class Xfetch < Formula
  desc "A fast, customizable system information fetch tool for the terminal"
  homepage "https://github.com/xfetch-cli/xfetch"
  license "MIT"
  version "1.0.2"

  on_macos do
    on_arm do
      url "https://github.com/xfetch-cli/xfetch/releases/download/v1.0.2/xfetch-1.0.2-aarch64-apple-darwin.tar.gz"
      sha256 "8379ceaab87463dcccd9862efc6ecfbb2d0478a415e0b703eba74eb5b8a04252"
    end
    on_intel do
      url "https://github.com/xfetch-cli/xfetch/releases/download/v1.0.2/xfetch-1.0.2-x86_64-apple-darwin.tar.gz"
      sha256 "6a7688586559506189ad4bb16b92ff135852ddf21ccdd6a91ba3881f8cf9a560"
    end
  end

  def install
    bin.install "xfetch"
  end

  test do
    assert_match version.to_s, shell_output("#{bin}/xfetch --version")
  end
end
```

## Field reference

| Field | Value | Notes |
| --- | --- | --- |
| `desc` | `A fast, customizable system information fetch tool for the terminal` | Kept short; no trailing period |
| `homepage` | `https://github.com/xfetch-cli/xfetch` | Main repo, not this tap |
| `license` | `MIT` | Matches the core repo |
| `version` | `1.0.2` | No leading `v` |
| `on_macos` / `on_arm` | `aarch64-apple-darwin` tarball | Apple Silicon |
| `on_macos` / `on_intel` | `x86_64-apple-darwin` tarball | Intel Macs |
| `sha256` (arm64) | `8379ceaab87463dcccd9862efc6ecfbb2d0478a415e0b703eba74eb5b8a04252` | 64 hex chars |
| `sha256` (x86_64) | `6a7688586559506189ad4bb16b92ff135852ddf21ccdd6a91ba3881f8cf9a560` | 64 hex chars |
| `install` | `bin.install "xfetch"` | Archive root contains `xfetch` |
| `test` | `assert_match version.to_s, shell_output("#{bin}/xfetch --version")` | Smoke test only |

The formula deliberately has:

- no `bottle` block (it is not distributed through Homebrew's bottle CDN);
- no `head` block (no source build);
- no `depends_on` (no runtime dependencies);
- no `livecheck` block (no automated version discovery);
- no `on_linux` block (Homebrew on Linux cannot install it);
- no `service` block (no daemon managed by Homebrew).

## Supported platforms

```text
macOS arm64  (Apple Silicon) -> xfetch-1.0.2-aarch64-apple-darwin.tar.gz
macOS x86_64 (Intel)         -> xfetch-1.0.2-x86_64-apple-darwin.tar.gz
Everything else              -> no URL; install fails
```

The upstream release builds six targets, but the formula intentionally selects
only two. The upstream release matrix in
`xfetch/.github/workflows/release.yml` is:

| Runner | Rust target | Archive name |
| --- | --- | --- |
| `ubuntu-22.04` | `x86_64-unknown-linux-gnu` | `xfetch-<ver>-x86_64-unknown-linux-gnu.tar.gz` |
| `ubuntu-24.04-arm` | `aarch64-unknown-linux-gnu` | `xfetch-<ver>-aarch64-unknown-linux-gnu.tar.gz` |
| `ubuntu-latest` | `x86_64-unknown-linux-musl` | `xfetch-<ver>-x86_64-unknown-linux-musl.tar.gz` |
| `macos-15-intel` | `x86_64-apple-darwin` | `xfetch-<ver>-x86_64-apple-darwin.tar.gz` |
| `macos-latest` | `aarch64-apple-darwin` | `xfetch-<ver>-aarch64-apple-darwin.tar.gz` |
| `windows-latest` | `x86_64-pc-windows-msvc` | `xfetch-<ver>-x86_64-pc-windows-msvc.zip` |

The formula uses the two `apple-darwin` rows.

## How release assets are selected

1. The release workflow runs on a tag matching `v*` and gates on: tag commit is
   an ancestor of `origin/main`, tag version equals `Cargo.toml` version,
   `cargo test --locked` passes, and `cargo audit --deny warnings` passes.
2. Each build job stages `xfetch` (or `xfetch.exe`), `LICENSE`, and `README.md`,
   then creates `xfetch-${VERSION}-${TARGET}.tar.gz` (zip on Windows), where
   `VERSION="${GITHUB_REF_NAME#v}"`.
3. A checksums job validates each archive contains the expected binary and
   writes `SHA256SUMS`.
4. The `sign` job signs `SHA256SUMS` keylessly with cosign and attests build
   provenance for the archives.
5. The `release` job verifies the signature and checksums, then publishes the
   archives plus `SHA256SUMS` and `SHA256SUMS.bundle` with
   `softprops/action-gh-release`.

The formula points at
`https://github.com/xfetch-cli/xfetch/releases/download/v<version>/...`, so the
tag must exist and the asset names must match exactly.

## Current state

The formula is aligned with the published `v1.0.2` release. The arm64 and
x86_64 URLs and SHA-256 values come from the release `SHA256SUMS` and were
re-verified by downloading both macOS archives. This is the security update
from v1.0.1.

## Bump procedure

Only run these steps after the upstream GitHub Release for the new version is
published and its assets are final.

1. Download the upstream checksums:

```bash
VERSION=1.0.2
curl -fsSL -o /tmp/SHA256SUMS \
  "https://github.com/xfetch-cli/xfetch/releases/download/v${VERSION}/SHA256SUMS"
cat /tmp/SHA256SUMS
grep 'apple-darwin' /tmp/SHA256SUMS
```

2. Cross-check the two values (the release also publishes them through the
   signed checksum bundle; the plain file is the operator-friendly source):

```bash
grep 'xfetch-'"${VERSION}"'-aarch64-apple-darwin.tar.gz' /tmp/SHA256SUMS
grep 'xfetch-'"${VERSION}"'-x86_64-apple-darwin.tar.gz'  /tmp/SHA256SUMS
```

3. Edit `Formula/xfetch.rb`, changing all five version-bearing values
   consistently:

```text
version "1.0.2"
url .../v1.0.2/xfetch-1.0.2-aarch64-apple-darwin.tar.gz
sha256 <new arm64 sha256>
url .../v1.0.2/xfetch-1.0.2-x86_64-apple-darwin.tar.gz
sha256 <new x86_64 sha256>
```

4. Validate syntax and audit (on a machine with Homebrew):

```bash
cd ~/repos/xfetch-cli/homebrew-tap
ruby -c Formula/xfetch.rb
brew audit --strict --online xfetch-cli/tap/xfetch
```

5. Test the actual download and install:

```bash
brew uninstall xfetch 2>/dev/null || true
brew install --build-from-source ./Formula/xfetch.rb   # exercises the formula locally
# or, after tapping the working tree:
# brew tap xfetch-cli/tap
# brew install xfetch
brew test xfetch
xfetch --version
```

`--build-from-source` is only a local formula-evaluation convenience here; the
formula still downloads the prebuilt tarball. There is no compilation step.

6. Commit on a release branch and open a PR. The recent history uses
   `release/<version>` or `release/v<version>` branches merged into `main`:

```bash
git switch -c release/v1.0.2
git add Formula/xfetch.rb
git commit -m "xfetch 1.0.2"
git push -u origin release/v1.0.2
```

Do not commit or push unless the task explicitly requests it.

## Verification checklist

```text
[ ] version field has no leading "v"
[ ] both URLs use the tag with leading "v" (v1.0.2)
[ ] both URLs use the version without "v" in the file name (xfetch-1.0.2-...)
[ ] both sha256 values match SHA256SUMS exactly
[ ] ruby -c Formula/xfetch.rb passes
[ ] brew audit --strict --online passes
[ ] brew install downloads and installs the expected tarball
[ ] brew test xfetch passes
[ ] xfetch --version prints the new version
```

## Gotchas

- Homebrew caches downloaded archives. If you re-test the same version with a
  changed checksum, run `brew fetch --force xfetch` or `brew reinstall xfetch`.
- A copied checksum for the wrong architecture is the most common failure. The
  arm64 and x86_64 lines differ; read the target name, not the line order.
- `brew audit --strict` applies style rules. The existing `desc` begins with
  "A " and may be reported by some Homebrew versions; do not rewrite it as
  part of a version bump unless the task asks for an audit cleanup.
- Never overwrite a published release asset. Homebrew clients cache it and the
  checksum in the tap would change under them. Cut a new patch release instead.
- The formula has no `livecheck`; `brew bump-formula-pr` and automatic livecheck
  are not configured. Updates are manual.
- The tap is not `homebrew-core`; users must run `brew tap xfetch-cli/tap`
  first, and `brew upgrade` only sees the tap after `brew update` refreshes it.

Last verified: 2026-11-03 against `homebrew-tap@cc9ca03ed62401fac76904973ea0315e25bcb50d`.
