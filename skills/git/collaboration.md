# Collaboration Traps

## Push/Pull

- `git pull` defaults to fetch + merge: the source of "Merge branch 'main' of ..." noise commits. `pull.rebase true` once, forever (→ `commands.md`).
- Push rejected (non-fast-forward) means "the remote moved", not "force needed" — `git pull --rebase`, then push. Reaching for `--force` here overwrites teammates' work.
- `git fetch` moves refs only, never your working tree — always safe to run; "fetch broke my code" is never true.
- `git pull --ff-only` is the honest default for a branch you never expect to diverge: it refuses instead of quietly inventing a merge you would have to explain in review.

## Force Push

- `--force-with-lease` only protects against remote states you have not seen — and IDE auto-fetch updates refs without warning, voiding the lease. `--force-if-includes` (git >=2.30) additionally requires the remote commits to be integrated in your local history. Both flags, always (SKILL.md, rule 2).
- Force-pushing during review invalidates line comments and the reviewed state. Push `--fixup` commits while review is open; autosquash after approval, before merge.
- Before any force push, look at what you are about to erase: `git log @{u}..` (yours) and `git log ..@{u}` (theirs). A non-empty second list means someone else's work is in the blast radius.
- Branch protection on main (require PR, block force push) is infrastructure, not etiquette — configure it before the first incident, not after.

## Shared Branch Discipline

- Two people on one branch: rewrites need coordination or abstinence — one person's rebase turns the other's next pull into a conflict storm. If you must: announce it; teammates recover with `git pull --rebase` (their identical local commits are skipped via patch-id matching).
- Stacked PRs merge bottom-up; after each merge, `git rebase --onto main just-merged next-branch` — or `rebase.updateRefs` moves the stack for you (→ `commands.md`).
- Long-lived shared branches drift: syncing with main weekly beats a heroic end-of-quarter merge — conflict cost grows faster than linearly with divergence.
- A branch nobody has touched in a month is not "in progress", it is a merge liability. Either finish it, land it behind a flag, or delete it and keep the SHA in the issue.

## Review Mechanics

- Under a squash-merge policy the PR title becomes the permanent commit subject — write PR titles like commit subjects (SKILL.md, rule 6).
- Post-approval pushes are not re-reviewed by default — the "dismiss stale approvals" repo setting exists precisely for this gap.
- Reviewing a rebased branch: `git range-diff` shows old-vs-new versions of each commit — reviewable in minutes, unlike a full re-diff (→ `commands.md`).
- Review locally when the diff is large: `git fetch origin pull/<n>/head:review-n && git switch review-n` (the GitHub refspec; GitLab exposes `merge-requests/<n>/head`, so `remote_host` picks the form) lets you run the tests instead of reading them. On other hosts, add the contributor's fork as a remote.
- Approving before CI finishes merges bugs with a green checkmark on the PR and a red one on main.
- Merge queues re-test each PR against the queue's projected result, which is why a PR that passed CI can still be rejected at merge time — the base changed. That is the queue working, not a flake.
- `CODEOWNERS` turns file paths into required reviewers. It also blocks merges with no explanation when the owning team is empty or misspelled — check it before blaming the review process.

## Patch-Based Contribution

Some projects (kernel-style, mailing lists, air-gapped review) never see a PR:

```bash
git format-patch -3 --cover-letter -o outgoing/     # 3 commits as .patch files
git send-email outgoing/*.patch                     # or attach them anywhere
git am outgoing/0001-*.patch                        # applies with author and message intact
git apply --3way broken.patch                       # falls back to a merge when context drifted
```

`git am` preserves authorship; `git apply` does not (the applier becomes the author). Use `am` for anything you did not write.

## Sync Hygiene

- Start of day: `git fetch --all --prune` + `git status -sb` — the ahead/behind counts tell you your divergence before it grows teeth.
- CI shows failures against an old SHA right after your rebase: expected, the SHAs changed — re-trigger, don't debug.
- Never start work without pulling first: the cheapest conflict is the one whose base was current.
- Credit collaborators in the history, not just in chat: `Co-authored-by:` trailers make pairing visible to anyone reading `git log` years later (`commits.md`).
