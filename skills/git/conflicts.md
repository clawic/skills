# Conflict Traps

## Reading the Conflict

- Default markers hide the base: you see ours and theirs but not what they diverged FROM — the one fact that tells you which side changed what. `merge.conflictStyle zdiff3` (git >=2.35) adds the `|||||||` base block; set it before your next conflict, not during.
- `git log --merge -p -- file` lists exactly the commits from both sides that touched the conflicted file — read it before editing, not after guessing wrong.
- The three stages are addressable directly: `git show :1:file` (base), `:2:file` (ours), `:3:file` (theirs). Diffing the base against each side answers "what did each branch intend" better than any marker layout.
- `git diff --diff-filter=U` lists only unresolved paths. In `git status --short` the two-letter code names the shape: `UU` both modified, `AA` both added, `DU`/`UD` one side deleted.
- "Both added" is not "both modified": both-added has no base version, so three-way tools show nonsense — diff the two versions directly and compose by hand.
- Rename + heavy edits can defeat rename detection: Git shows a delete/add or spurious conflict instead. Confirm with `git log --follow -- newpath` before resolving as if the file were new.
- Whitespace-only conflicts vanish from a diff configured to ignore whitespace — the conflict is real even when `git diff -w` shows nothing.

## Ours/Theirs Semantics

- Merge: ours = your branch, theirs = incoming. Rebase: INVERTED — ours = the upstream you rebase onto, theirs = your own commit being replayed. Why: rebase checks out upstream and applies your commits as patches. Misreading this makes you throw away your own work with full confidence.
- `git checkout --ours file` takes the whole file from one side — including that file's non-conflicting hunks from the other side, discarded with no notice. Fine for lockfiles, wrong for source code.
- `-X ours` / `-X theirs` on the merge command are a different tool: they auto-resolve only the CONFLICTING hunks in favor of one side and merge everything else normally. Still blunt, but not file-level blunt.
- Generated files (lockfiles, snapshots, compiled assets): never hand-merge. Take either side, rerun the generator (`npm install`, etc.), stage the regenerated output.

## Conflicts That Are Not Text

- **Lockfiles** conflict on almost every merge. Regenerate (above), or install a merge driver so Git stops asking — `.gitattributes` travels between clones, the driver definition does not, so each machine also needs the `git config merge.<name>.driver` line (`config.md`).
- **Binary files** cannot be merged; Git asks you to pick a side: `git checkout --theirs asset.png && git add asset.png`. Mark them `binary` in `.gitattributes` so Git stops attempting a diff.
- **Submodule pointer conflicts** appear as a conflicted gitlink with no content to edit. Enter the submodule, check out the commit you actually want, then `git add <path>` in the parent (`submodules.md`).
- **modify/delete** conflicts need a decision about the file's existence before its content: `git rm <path>` accepts the deletion, `git add <path>` keeps the file.

## Finishing

- Marker check + build before `--continue` (exact command → SKILL.md, Conflict Basics). Markers compile fine in some languages — the build alone does not catch them.
- Merge-tool "success" is not resolution: some tools exit without saving; `git diff --staged` to confirm real content got staged before committing.
- Mergetool `.orig` backups are committable garbage: `git config mergetool.keepBackup false`.
- Abort economics: `git merge --abort` costs nothing; `git rebase --abort` discards every already-resolved commit of this rebase — unless `rerere.enabled` is on, in which case your resolutions replay automatically on the next attempt.
- Audit your own resolution afterwards: `git log --remerge-diff -1 <merge-sha>` (git >=2.36) shows what the merge commit changed relative to a mechanical re-merge. That diff IS the resolution, and it is where silent mistakes hide (`forensics.md`).

## Rerere

- `rerere.enabled true` records each resolution by conflict hash and replays it when the same conflict reappears — the difference between resolving a long rebase once and resolving it on every attempt.
- It applies with no prompt, including a resolution you got wrong the first time. `git rerere diff` shows what it is about to do; `git rerere forget <path>` erases a bad memory while the conflict is still open.
- Recordings are local to the clone and never push: rerere helps whoever runs the rebase, not the reviewer.

## Prevention

- Conflict size grows with divergence: rebase the feature branch onto main daily; a week of drift in an actively edited file can cost more to merge than the feature took to write.
- Same region conflicting across 3+ replayed commits → abort and merge instead (SKILL.md, rule 5).
- Preview before merging: `git merge --no-commit --no-ff branch` shows the damage and lets you `git merge --abort` for free.
- Small, focused commits conflict less and resolve easier — a conflict inside a 40-line commit tells you its intent; inside a 2,000-line commit it tells you nothing.
- Structural collisions (two people renaming or moving the same module) are cheap to prevent and expensive to resolve: rename-heavy work is the one case worth announcing before starting it.
