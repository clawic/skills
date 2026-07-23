# Preview Changes

How to analyze an update before applying it.

## Fetching New Version

```bash
# Download without installing into any agent's skills folder
npx clawic install <slug> --dir /tmp/preview-<slug>
```

Compare against current (project install shown; adjust the path for the agent in use):
```bash
diff -r .claude/skills/<slug> /tmp/preview-<slug>
```

Read the diff, not just its shape: a one-line change to an instruction can matter more than a 200-line new file of examples.

## Change Categories

Classify every hunk into exactly one:

### Additions — safe by default, with one exception
New auxiliary files, templates, examples: minimal impact. **Exception:** an added imperative rule ("always X", "never Y", a new step in a flow) changes behavior on every future run — classify it as a modification and review it, even though the diff shows only green lines.

### Modifications — review carefully
Changed instructions, updated references, different approaches. Behavior may shift where the user built habits around the old version. Ask: does the user currently rely on the old wording's behavior?

### Deletions — breaking, after one check
Removed files, sections, or features break workflows that depend on them. **Check first:** search the new version for the deleted text. Diff renders a move as delete + add; content that merely relocated is a structural change, not a removal. Only flag "removed" when the content is truly gone.

### Structural changes — breaking
Renamed files, content moved between files, reorganized folders. References, muscle memory, and automations pointing at old paths break even though every word survived.

## Generating the Impact Report

For each change answer, in order:
1. What exactly changed?
2. Does the user currently use this? (Check their data folder and recent usage, don't guess)
3. Will their workflow break?
4. Is migration needed?

A change where answers 2-4 are all "no" goes in one summary line, not its own section — preview length should track user impact, not diff size.

## Presenting to User

Breaking first, then modifications, then additions:

```
## Update Preview: skill-name v1.2.3 → v2.0.0

### ⚠️ Breaking Changes
- `config.md` renamed to `settings.md`
- Section "Quick Deploy" removed

### Modifications
- New approach for authentication
- Updated examples

### Additions
- `templates/` folder with 5 templates
- Troubleshooting section

### Migration Required
- Your preferences in old config.md need to move to settings.md
```

## When to Recommend Update

| Situation | Call |
|---|---|
| Fixes a bug the user has hit, or security patch | Recommend now |
| Minor improvements, no breaking changes | Recommend, low urgency |
| Major version or structural changes | Present preview neutrally; let user decide |
| User has active work depending on current behavior | Defer explicitly — name the work as the reason |
| Else | Present the diff and ask; never apply on your own judgment |
