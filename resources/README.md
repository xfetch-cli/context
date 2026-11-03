# Resources for agents and skills

Reusable prompts and procedures for working on the xfetch ecosystem with an AI
agent. They are intentionally short; the deep context is in the repository
folders.

## How to use

1. Pick the role that matches the task from `agents/`.
2. Pick the procedure from `skills/`.
3. Load the relevant context from `../<repo>/` and `../ecosystem.md`.
4. Run the local checks before reporting success.
5. Update `../current-state.md` and the relevant documents when state changes.

## Agents

| Agent | Use when |
| --- | --- |
| `agents/ecosystem.md` | Cross-repo architecture, contracts, versioning or release-train work. |
| `agents/core.md` | Working on `xfetch-cli/xfetch` (probes, UI, addons, wasm). |
| `agents/addons.md` | Working on plugins, effects or extensions and their catalogs. |
| `agents/content.md` | Working on web, configs, themes, logos or assets. |
| `agents/release.md` | Preparing releases, tags, crates.io, Homebrew, web and AUR. |

## Skills

| Skill | Use when |
| --- | --- |
| `skills/build-test.md` | Building and testing any Rust or data repo locally. |
| `skills/addon-development.md` | Adding or changing a plugin, effect or extension. |
| `skills/release-core.md` | Preparing and publishing an xfetch release. |
| `skills/crates-io-publish.md` | Publishing SDK crates or `xfetch-cli` to crates.io. |
| `skills/context-maintenance.md` | Updating this context repository safely. |

## Repository map

See `repos.md` for the context document, agent and skill to use per repository.

## Rules

- English, no emojis.
- No credentials, private paths or personal data.
- Use `~/repos/xfetch-cli/<repo>` in commands.
- Do not merge, tag or publish without explicit approval.
- Source code wins over this context when they disagree.
