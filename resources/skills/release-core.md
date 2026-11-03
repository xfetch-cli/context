# Skill: core release

## Preconditions

- Addon catalogs are on `main`.
- The release branch is reviewed and merged.
- `Cargo.toml` has the intended version.
- The tree is clean.

## Prepare

1. Bump `Cargo.toml` and `Cargo.lock`.
2. Update `CHANGELOG.md`.
3. Update installers/docs examples to the release version.
4. Run:

```bash
cd ~/repos/xfetch-cli/xfetch
cargo fmt --all --check
cargo clippy --all-targets -- -D warnings
cargo test --all-targets
cargo audit --deny warnings
```

5. Merge the release PR.

## Tag and verify

```bash
git checkout main
git pull --ff-only
git tag -a vX.Y.Z -m "xfetch vX.Y.Z"
git push origin vX.Y.Z
```

The release workflow requires the tag commit to be on `main` and the tag to
match `Cargo.toml`. Verify:

- six target archives
- `SHA256SUMS`
- `SHA256SUMS.bundle`
- GitHub release published

Do not move or reuse the tag after publication.

## Publish

See `crates-io-publish.md`.

## After the release

1. Bump `homebrew-tap` with asset URLs and SHA-256 values.
2. Update `web` version/docs/changelog.
3. Bump the AUR packages.
4. Update `current-state.md` in the context repository.
