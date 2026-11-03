# Skill: context maintenance

## When to update

- After a release, tag or crates.io publish.
- After a branch/PR merge that changes the current state.
- After an architecture or protocol change.
- When a documented command or path changes.

## Rules

- Update `current-state.md` first.
- Keep `repositories.md` heads and branches current.
- Keep `Last verified` markers accurate.
- Use `~/repos/xfetch-cli/<repo>` in commands.
- No credentials, tokens, private paths or personal data.
- No emojis; English only.
- Do not copy stale source docs; link to canonical docs.

## Sanitization checklist

```bash
grep -rn '/home/' . --include='*.md'
grep -rniE 'token|secret|password|api[_-]?key|gho_|ghp_|github_pat_' . --include='*.md'
grep -rn 'private notebook' . --include='*.md'
```

Fix any real path or secret. Generic mentions of `token`,
`CARGO_REGISTRY_TOKEN` or `~/.cargo/credentials.toml` are documentation and are
allowed; actual values are not.

## Validation

```bash
python3 - <<'PY'
import re, pathlib
missing=[]
for p in pathlib.Path('.').rglob('*.md'):
    text=p.read_text(encoding='utf-8')
    for m in re.finditer(r'\[[^\]]*\]\(([^)]+)\)', text):
        t=m.group(1).strip()
        if t.startswith(('http://','https://','mailto:','#')): continue
        t=t.split('#',1)[0].strip('<>')
        if t and not (p.parent/t).resolve().exists():
            missing.append((str(p),t))
print(missing or 'links ok')
PY
```

Then commit and push the context repository.
