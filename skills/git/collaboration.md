# Collaboration Traps

## Push/Pull

- `git pull` defaults to fetch + merge: the source of "Merge branch 'main' of ..." noise commits. `pull.rebase true` once, forever (→ `commands.md`).
- Push rejected (non-fast-forward) means "the remote moved", not "force needed" — `git pull --rebase`, then push. Reaching for `--force` here overwrites teammates' work.
- `git fetch` moves refs only, never your working tree — always safe to run; "fetch broke my code" is never true.

## Force Push

- `--force-with-lease` only protects against remote states you have not seen — and IDE auto-fetch updates refs silently, voiding the lease. `--force-if-includes` (git >=2.30) additionally requires the remote commits to be integrated in your local history. Both flags, always (SKILL.md, rule 2).
- Force-pushing during review invalidates line comments and the reviewed state. Push `--fixup` commits while review is open; autosquash after approval, before merge.
- Branch protection on main (require PR, block force push) is infrastructure, not etiquette — configure it before the first incident, not after.

## Shared Branch Discipline

- Two people on one branch: rewrites need coordination or abstinence — one person's rebase turns the other's next pull into a conflict storm. If you must: announce it; teammates recover with `git pull --rebase` (their identical local commits are skipped via patch-id matching).
- Stacked PRs merge bottom-up; after each merge, `git rebase --onto main just-merged next-branch` — or `rebase.updateRefs` moves the stack for you (→ `commands.md`).
- Long-lived shared branches drift: syncing with main weekly beats a heroic end-of-quarter merge — conflict cost grows faster than linearly with divergence.

## Review Mechanics

- Under a squash-merge policy the PR title becomes the permanent commit subject — write PR titles like commit subjects (SKILL.md, rule 6).
- Post-approval pushes are not re-reviewed by default — the "dismiss stale approvals" repo setting exists precisely for this gap.
- Reviewing a rebased branch: `git range-diff` shows old-vs-new versions of each commit — reviewable in minutes, unlike a full re-diff (→ `commands.md`).
- Approving before CI finishes merges bugs with a green checkmark on the PR and a red one on main.

## Sync Hygiene

- Start of day: `git fetch --all --prune` + `git status -sb` — the ahead/behind counts tell you your divergence before it grows teeth.
- CI shows failures against an old SHA right after your rebase: expected, the SHAs changed — re-trigger, don't debug.
- Never start work without pulling first: the cheapest conflict is the one whose base was current.
