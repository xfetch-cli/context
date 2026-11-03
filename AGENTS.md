# Agent instructions

This repository is documentation and reusable agent material for the xfetch
ecosystem. It does not contain the source code of the applications.

## Local layout

Source repositories are expected at:

```text
~/repos/xfetch-cli/
├── xfetch/
├── api/
├── plugins/
├── effects/
├── extensions/
├── web/
├── configs/
├── themes/
├── logos/
├── assets/
├── homebrew-tap/
├── .github/
└── context/
```

## Read order

1. `README.md` for the map.
2. `ecosystem.md` for contracts and cross-repo flow.
3. `repositories.md` for paths, remotes and branch state.
4. The relevant repo folder, for example `xfetch/`, `api/` or `plugins/`.
5. `resources/` for an agent role or task skill.
6. `current-state.md` before assuming a branch, PR or release is current.

## Rules

- Read the source repository before editing it. Do not infer behavior from this
  context alone.
- Do not modify source repositories from this context repository unless the
  user explicitly asks.
- Keep the JSON addon protocol at `1` and the WIT at
  `xfetch:runtime@0.1.0` unless the task is a deliberate protocol release.
- Preserve the roles: plugins add information, effects transform printed
  lines, extensions mutate configuration.
- Run the local checks before claiming success:
  - Rust repos: `cargo fmt --all --check`,
    `cargo clippy --all-targets -- -D warnings`, `cargo test`,
    `cargo audit --deny warnings`.
  - Data repos: `bash scripts/ci.sh` where available.
- Do not publish to crates.io, push tags, merge release PRs or change licenses
  without explicit user approval.
- Never add credentials, tokens, private paths or personal data. Use
  `~/repos/xfetch-cli/...` in examples.
- Write in English, with no emojis.
- When architecture, versions or contracts change, update this context
  repository and the `Last verified` markers.

## Resources

- `resources/agents/`: role prompts for ecosystem, core, addons, content and
  release work.
- `resources/skills/`: repeatable procedures for build/test, addon work,
  core release, crates.io publish and context maintenance.
- `resources/repos.md`: which context document and skill to use per repo.
