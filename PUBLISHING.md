# Publishing

This directory is intended to be pushed as its own GitHub repository.

## Expected public layout

```text
README.md
LICENSE
CONTRIBUTING.md
CODE_OF_CONDUCT.md
SECURITY.md
.github/workflows/validate.yml
skills/<slug>/
```

## Suggested release flow

1. Initialize a dedicated Git repository in this directory.
2. Add the target remote.
3. Review the contents of `skills/`.
4. Commit the first import as the initial public version.

## Notes

- The actual skill data is under `skills/`.
- This folder is intentionally separated from the CLI and website workspace.
