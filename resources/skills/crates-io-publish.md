# Skill: crates.io publish

## Authentication

```bash
cargo login
```

The token is stored in `~/.cargo/credentials.toml`. Never commit it or paste it
into logs.

## Preconditions

- Version bumped and changelog updated.
- Metadata complete (license, description, repository, homepage,
  documentation, readme, keywords, categories).
- Git tree clean.
- Local gates pass.
- `cargo publish --dry-run` passes.

## Order

1. `xfetch-api-common`
2. `xfetch-plugin-api`
3. `xfetch-extension-api`
4. `xfetch-effect-api`
5. `xfetch-guest-api`

`xfetch-guest-api` is independent, but keep the order consistent. The protocol
crates depend on `xfetch-api-common`.

## Publish

```bash
cd ~/repos/xfetch-cli/api
cargo publish -p xfetch-api-common
cargo publish -p xfetch-plugin-api
cargo publish -p xfetch-extension-api
cargo publish -p xfetch-effect-api
cargo publish -p xfetch-guest-api
```

For the core, publish only after the tag and assets:

```bash
cd ~/repos/xfetch-cli/xfetch
cargo publish --dry-run -p xfetch-cli
cargo publish -p xfetch-cli
```

## Verify

```bash
cargo search xfetch-api-common
cargo info xfetch-plugin-api
cargo install xfetch-cli --version <version>
```

## Rules

- crates.io versions are immutable.
- Fix a mistake with a new patch; use `cargo yank` only to stop new
  resolutions.
- Do not publish `xfetch-cli` 1.0.2 to crates.io; 1.0.3 is the definitive
  candidate.
