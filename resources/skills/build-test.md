# Skill: build and test

## Rust repositories

Apply to `xfetch`, `api`, `plugins`, `effects` and `extensions` when they have
a Rust workspace.

```bash
cd ~/repos/xfetch-cli/<repo>
cargo fmt --all --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

Use `--all-targets` for the core. Use `cargo test --all-targets` when examples
or integration tests matter. The API repository does not commit `Cargo.lock`;
the core and addons do.

## Security audit

```bash
cargo generate-lockfile   # only if the repo intentionally has no lockfile
cargo audit --deny warnings
```

Do not commit a generated lockfile if the repository ignores it.

## Local CI scripts

```bash
cd ~/repos/xfetch-cli/xfetch      && bash scripts/ci.sh
cd ~/repos/xfetch-cli/plugins     && bash scripts/ci.sh
cd ~/repos/xfetch-cli/effects     && bash scripts/ci.sh
cd ~/repos/xfetch-cli/extensions  && bash scripts/ci.sh
cd ~/repos/xfetch-cli/configs     && bash scripts/ci.sh
cd ~/repos/xfetch-cli/themes      && bash scripts/ci.sh
cd ~/repos/xfetch-cli/logos       && bash scripts/ci.sh
```

## Windows cross-check

```bash
cd ~/repos/xfetch-cli/<repo>
bash scripts/ci-windows.sh
```

It requires `rustup target add x86_64-pc-windows-gnu`. For the core,
`zstd-sys` also needs `x86_64-w64-mingw32-gcc` on Linux.

## Web

```bash
cd ~/repos/xfetch-cli/web
npm ci
npm run lint
npm run build
```

## Reporting

State exactly which commands ran, their exit status and any skipped checks.
Never claim a test passed without running it.
