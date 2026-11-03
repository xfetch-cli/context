# Repository map for agents

Pick the context document, agent and skill for the repository you are working
on. Paths assume source clones under `~/repos/xfetch-cli/`.

| Repo | Context | Agent | Skill |
| --- | --- | --- | --- |
| `xfetch` | `../xfetch/README.md` and `../xfetch/architecture.md` | `agents/core.md` | `skills/build-test.md`, `skills/release-core.md` |
| `api` | `../api/README.md` and `../api/protocols.md` | `agents/ecosystem.md` | `skills/build-test.md`, `skills/crates-io-publish.md` |
| `plugins` | `../plugins/README.md` and `../plugins/protocol.md` | `agents/addons.md` | `skills/addon-development.md` |
| `effects` | `../effects/README.md` and `../effects/protocol.md` | `agents/addons.md` | `skills/addon-development.md` |
| `extensions` | `../extensions/README.md` and `../extensions/protocol.md` | `agents/addons.md` | `skills/addon-development.md` |
| `web` | `../web/README.md` and `../web/architecture.md` | `agents/content.md` | `skills/build-test.md` |
| `configs` | `../configs/README.md` and `../configs/inventory.md` | `agents/content.md` | `skills/build-test.md` |
| `themes` | `../themes/README.md` and `../themes/registry.md` | `agents/content.md` | `skills/build-test.md` |
| `logos` | `../logos/README.md` and `../logos/catalog.md` | `agents/content.md` | `skills/build-test.md` |
| `assets` | `../assets/README.md` and `../assets/inventory.md` | `agents/content.md` | `skills/build-test.md` |
| `homebrew-tap` | `../homebrew-tap/README.md` and `../homebrew-tap/formula.md` | `agents/release.md` | `skills/release-core.md` |
| `.github` | `../.github/README.md` and `../.github/profile.md` | `agents/content.md` | `skills/build-test.md` |
| `context` | `../README.md` | `agents/ecosystem.md` | `skills/context-maintenance.md` |

## Cross-repo work

Use `agents/ecosystem.md` plus `../ecosystem.md`, `../repositories.md` and
`../conventions.md`. Release work uses `agents/release.md`,
`skills/release-core.md` and `skills/crates-io-publish.md`.
