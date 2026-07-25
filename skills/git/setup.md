# Setup — Git

Read this on first use to load user preferences. Do not interview the user.

## Your Attitude

Git is unforgiving in exactly two places: destructive commands and published history. Be precise about those, and fast and unceremonious about everything else. Name what a command will destroy before running it; never improvise a rewrite on a shared branch.

## How To Load Preferences

1. Read `~/Clawic/data/git/config.yaml` if it exists. Apply its values.
2. For anything absent, use the defaults in the Configuration table of `SKILL.md` — do not ask.
   - `integration_style: repo-default`, `commit_style: repo-default`, `branch_naming: type/topic`, `protected_branches: [main, master]`, `force_push_policy: own-branches`, `subject_max: 72`, `message_language: repo-default`, `remote_host: github`, `signing: off`.
3. Resolve the two `repo-default` values from the repository itself, not from the user: `git log --oneline -20` for message style, `git log --merges --oneline -20` for integration style.
4. Read `~/Clawic/data/git/memory.md` for prior context (team conventions, past corrections). Absence is fine; proceed without comment.

Work from defaults immediately. Never open with questions about workflow, tooling, or how careful to be.

## Recording Preferences (only when the user declares one)

Write to config or memory **only** when the user states a preference in the course of the work — never as a preflight questionnaire.

- User names a merge strategy, message convention, branch pattern, protected branch, subject-length limit, commit-message language, host, or signing setup → update the matching key in `~/Clawic/data/git/config.yaml`.
- User expresses a stance or habit (whether you may commit unprompted, confirmation before destructive commands, review gates expected before a push, sync cadence) → record it under the relevant preference area (tooling, conventions, thresholds, restrictions, platform, safety posture, work order, cadence) in `~/Clawic/data/git/memory.md`.
- User corrects earlier guidance ("we merge, we don't rebase") → update the stored value so you don't repeat it.

If the user has said nothing, store nothing. A repository convention observed in `git log` is a fact about the repo, not a declared preference — follow it, but do not write it to `config.yaml` as if the user had asked for it.

## What Memory Holds

See `memory-template.md` for the file format. Track the repos and hosts they work in, the team conventions they have stated, the operations they want confirmation on, and past corrections — but only from what they actually reveal.
