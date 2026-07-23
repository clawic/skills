# Conflict Traps

## Reading the Conflict

- Default markers hide the base: you see ours and theirs but not what they diverged FROM — the one fact that tells you which side changed what. `merge.conflictStyle zdiff3` (git >=2.35) adds the `|||||||` base block; set it before your next conflict, not during.
- `git log --merge -p -- file` lists exactly the commits from both sides that touched the conflicted file — read it before editing, not after guessing wrong.
- "Both added" is not "both modified": both-added has no base version, so three-way tools show nonsense — diff the two versions directly and compose by hand.
- Rename + heavy edits can defeat rename detection: Git shows a delete/add or spurious conflict instead. Confirm with `git log --follow -- newpath` before resolving as if the file were new.
- Whitespace-only conflicts vanish from a diff configured to ignore whitespace — the conflict is real even when `git diff -w` shows nothing.

## Ours/Theirs Semantics

- Merge: ours = your branch, theirs = incoming. Rebase: INVERTED — ours = the upstream you rebase onto, theirs = your own commit being replayed. Why: rebase checks out upstream and applies your commits as patches. Misreading this makes you throw away your own work with full confidence.
- `git checkout --ours file` takes the whole file from one side — including that file's non-conflicting hunks from the other side, silently discarded. Fine for lockfiles, wrong for source code.
- Generated files (lockfiles, snapshots, compiled assets): never hand-merge. Take either side, rerun the generator (`npm install`, etc.), stage the regenerated output.

## Finishing

- Marker check + build before `--continue` (exact command → SKILL.md, Conflict Basics). Markers compile fine in some languages — the build alone does not catch them.
- Merge-tool "success" is not resolution: some tools exit without saving; `git diff --staged` to confirm real content got staged before committing.
- Mergetool `.orig` backups are committable garbage: `git config mergetool.keepBackup false`.
- Abort economics: `git merge --abort` costs nothing; `git rebase --abort` discards every already-resolved commit of this rebase — unless `rerere.enabled` is on, in which case your resolutions replay automatically on the next attempt.

## Prevention

- Conflict size grows with divergence: rebase the feature branch onto main daily; a week of drift in an actively edited file can cost more to merge than the feature took to write.
- Same region conflicting across 3+ replayed commits → abort and merge instead (SKILL.md, rule 5).
- Preview before merging: `git merge --no-commit --no-ff branch` shows the damage and lets you `git merge --abort` for free.
- Small, focused commits conflict less and resolve easier — a conflict inside a 40-line commit tells you its intent; inside a 2,000-line commit it tells you nothing.
