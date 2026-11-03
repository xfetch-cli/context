# Agent: content and data maintainer

## Mission

Work on `web`, `configs`, `themes`, `logos` and `assets`: public docs, presets,
theme registry, ASCII catalog and brand assets.

## Read first

- `../../web/README.md` and `../../web/architecture.md`
- `../../configs/README.md` and `../../configs/inventory.md`
- `../../themes/README.md` and `../../themes/registry.md`
- `../../logos/README.md` and `../../logos/catalog.md`
- `../../assets/README.md` and `../../assets/inventory.md`

## Responsibilities

- Keep versions and docs aligned with the current core release.
- Keep docs in en/es/de with `SUMMARY.md` order.
- Validate JSONC and data registries before committing.
- Keep asset paths stable; the banner URL is a public contract.
- Do not include usernames or absolute home paths in examples.

## Validation

```bash
cd ~/repos/xfetch-cli/web       && npm ci && npm run lint && npm run build
cd ~/repos/xfetch-cli/configs   && bash scripts/ci.sh
cd ~/repos/xfetch-cli/themes    && bash scripts/ci.sh
cd ~/repos/xfetch-cli/logos     && bash scripts/ci.sh
```

Use `xfetch --config <path>` for config/preset checks and
`XFETCH_LOGOS_URL=<branch>` for catalog checks before merging.

## Constraints

- No secrets, private URLs or personal paths.
- Themes must not contain icons.
- Logo art must be validated for size, line width and control characters.
