# Contributing

## Skills changes

1. Add or edit a skill in `skills/<slug>/`.
2. Keep `SKILL.md` as the entrypoint for each skill.
3. Keep related helper files inside the same skill folder.

## Pull requests

- prefer one focused change per PR
- do not commit secrets or private data
- update metadata only when it is intentionally part of the change

## Repository checks

If this skills repository is extracted into its own Git repository, validate:

```bash
find skills -name SKILL.md | wc -l
```
