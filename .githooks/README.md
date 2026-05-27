# Git hooks

Repo-managed hooks. Enable once per clone:

```bash
git config core.hooksPath .githooks
```

On Windows, hooks run via Git Bash (bundled with Git for Windows). No extra setup.

## Hooks

| Hook | Purpose |
| --- | --- |
| `pre-commit` | Refuses commits that include changes under `gen/**`. Regeneration is owned by `.github/workflows/generate.yml`. Override with `ALLOW_GEN=1 git commit ...` when bootstrapping. |
