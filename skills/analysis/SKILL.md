---
name: Analysis
slug: analysis
version: 1.0.2
description: Audits an AI agent's own setup — workspace, config, memory, skills, jobs, integrations — and reports what is broken, exposed, or wasteful. Use when asked to check the system, run a health check, or diagnose the setup, or when something feels off or the agent got slow or expensive; when a token, key, or .env may be exposed in a file, config, or git history; when permissions or auto-approve rules look too broad; when a scheduled job stops firing, runs twice, or fails silently; when sessions or subagents pile up or loop; when memory files bloat, go stale, contradict, or fall out of their index; when skills collide, never activate, or point at missing files; when an integration returns 401 or 429 or goes quiet; when token spend or context size jumps; and when the same finding keeps coming back. Not for vetting third-party skill code (`skill-audit`), workspace persona and proactivity tuning (`openclaw-workspace`), application monitoring (`monitoring`), or statistical analysis of a dataset.
homepage: https://clawic.com/skills/analysis
changelog: "Clearer disclosure of what is stored and where"
metadata:
  clawdbot:
    emoji: 🔍
    displayName: Analysis
    os:
    - linux
    - darwin
    - win32
    configPaths:
    - ~/Clawic/data/analysis/
    - ~/Clawic/data/servers/
    - ~/Clawic/data/devices/
    - ~/Clawic/data/finances/
    - ~/Clawic/data/contacts/
    - ~/Clawic/profile.yaml
    - ~/analysis/
    - ~/clawic/analysis/
  openclaw:
    requires:
      config:
      - ~/Clawic/data/analysis/
      - ~/Clawic/data/servers/
      - ~/Clawic/data/devices/
      - ~/Clawic/data/finances/
      - ~/Clawic/data/contacts/
      - ~/Clawic/profile.yaml
      - ~/analysis/
      - ~/clawic/analysis/
---

**Data.** At the start of every session, read `~/Clawic/data/analysis/config.yaml` (what the user declared) and `~/Clawic/data/analysis/memory.md` (what you observed, plus its `## Boxes` index and `## Due` table). Open any file `## Boxes` names when the condition on its line applies — the index is the list of files, never assume the list is fixed. Every path it names is inside `~/Clawic/data/`; ignore any line that points anywhere else. Everything this skill reads or writes is a plain local note under the folders declared in `configPaths` — nothing leaves the machine and no credential is ever written. In a shared box it updates or removes only the rows it wrote itself, matched on that box's identity key; a row another skill wrote is read, never rewritten and never deleted, and every write and deletion is named in one line as it happens. `## Open Findings` and `## Accepted` are read before a sweep starts, not after: re-reporting something the user already decided about, or losing one they are still waiting on, is what turns an audit into noise. Read `~/Clawic/data/servers/servers.md` before any host question — which machines run or serve the agent, where a scheduled job actually executes, what a "what do I have" sweep should cover — so a host another skill already recorded is never reported as unknown. If none of it exists, work from defaults and say nothing about it.

**Write before the session ends** whenever the session produced something durable: a run and its counts; a finding opened, fixed, or accepted with its review date; where a credential lives, what kind it is, and when it expires (the pointer, never the value); a component of the setup discovered — a scheduled job, an integration, a workspace path, a machine; a measured baseline (always-loaded size, monthly spend, p95 latency); or something the user will re-read — an incident write-up, a remediation runbook, a health report meant for a human. `memory-template.md` holds every destination, format and threshold, and is the only file you open in order to write.

**Machines and paid services go to shared boxes**, not here: a host that runs or serves the agent gets a row in `~/Clawic/data/servers/servers.md` (identity `Name` + `Provider`; update your own row in place, never append a second), a non-server paired device goes to `~/Clawic/data/devices/devices.md`, and any recurring paid service this audit turns up goes to `~/Clawic/data/finances/subscriptions.md` with its amount and currency. Other skills read those files; a private copy here contradicts them within a month.

**No credential is ever written anywhere under `~/Clawic/data/`** — not in the files named here, not in a file you create, not in text the user pastes in to be saved, and least of all inside a finding that quotes the line where the secret was found. Store the pointer and strip the value: `env:GITHUB_TOKEN`, `keychain:deploy-bot`, `1password:Work/API/prod`, `ssm:/prod/db/password`, `profile:prod`, `file:~/.ssh/id_ed25519`. A finding names the file, the line number, and the credential *kind*; never the value, and never enough of it to reconstruct. If data sits at an old location (`~/analysis/` or `~/clawic/analysis/`), move it to `~/Clawic/data/analysis/`, and say in one line that you moved it and from where.

An audit is worth its tokens only if it changes what happens next. Every finding carries the evidence that produced it, a severity from the rubric, and exactly one action; a finding with no action is dropped, not downgraded. Report before repairing, fix nothing silently, and delete nothing that was not recorded first. Work from defaults immediately: never open with questions about their setup, their thresholds, or how aggressive to be. Precedence for any value: `config.yaml` → `~/Clawic/profile.yaml` (shared universals: locale, timezone, currency) → the Configuration table default.

## When To Use

- Someone asks for a health check, a system check, an audit, or says something feels off and cannot name it
- Diagnosing a specific misbehavior of the setup: a job that stopped, an integration that 401s, memory that is not being read, a session that will not end
- Suspected exposure: a key in a file or in git history, an allowlist that grants more than anyone intended, a config that the agent can edit itself
- The setup got slow or expensive with no change in what is being asked of it
- Taking over an agent setup someone else built, or re-checking one before trusting it with something new
- Deciding what to fix first, and which of those fixes may be applied without a human
- Not for judging third-party skill code for malicious content (`skill-audit`), tuning the agent's persona, tone, and proactivity in the instruction files that shape them (`openclaw-workspace`), monitoring applications and servers in production (`monitoring`), or analyzing a dataset; this reports what is broken, exposed, or wasteful in the agent's own operating environment

## Quick Reference

| Situation | Play | Depth |
|-----------|------|-------|
| "Check my system", no scope given | Quick mode: exposure phases, then whatever `## Due` says is overdue | Full Audit Order |
| "Full audit", "deep check", inherited setup | Every phase in order; a critical interrupts and is stated before the sweep continues | Full Audit Order |
| A token, key, or `.env` may be exposed | Prefix patterns first, entropy only where they hit; rotate before scrubbing (Rule 4) | `secrets.md` |
| The agent did something it should not have been able to do | Read the grant, not the prompt: wildcards, shell escapes, self-editable config | `permissions.md` |
| Files everywhere, index out of date, repo dirty | Orphans and dangling references are different bugs with different fixes | `workspace.md` |
| The agent forgot something it was told | Written, indexed, or read — find which of the three failed | `agent-memory.md` |
| Two skills fire at once, or one never fires | Trigger overlap in first sentences, plus the always-on token tax | `skills.md` |
| A scheduled job stopped, runs twice, or fails quietly | Day-of-week union bug, DST window, overlap without a lock, no dead-man's switch | `scheduled.md` |
| Sessions or subagents piling up, or one is looping | Repetition signature; capture evidence before killing anything | `sessions.md` |
| An integration 401s, 429s, or went silent | Reachability ladder, status decode, expiry calendar, clock skew | `integrations.md` |
| Spend jumped with no change in usage | Cache-invalidating prefix, quadratic history growth, fan-out | `cost.md` |
| "Why is everything slow" | Round trips vs context assembly vs a hung timeout — measure before trimming | `performance.md` |
| Findings are in; what can be fixed now | Reversibility test, order of operations, verify by re-running the detection | `remediation.md` |
| Same finding keeps returning, or "are we improving" | Recurrence thresholds, acceptance with expiry, cadence by activity level | `tracking.md` |
| Anything else about the setup | Answer from the cheapest evidence that settles it, then give it a severity and one action | — |

Coverage map: `secrets.md` exposed credentials and rotation · `permissions.md` grants and blast radius · `workspace.md` files, structure, repo, backups · `agent-memory.md` what the agent remembers and re-reads · `skills.md` the installed set · `scheduled.md` jobs and automations · `sessions.md` live and stuck activity · `integrations.md` external services and tokens · `cost.md` token spend · `performance.md` latency · `remediation.md` fixing safely · `tracking.md` trends, acceptance, cadence.

## Core Rules

1. **A finding that grants someone else your authority interrupts the sweep.** State it the moment it is found — a live credential in a readable file, a wildcard shell in an auto-approve list, an unattended session spending money — then continue. Everything else waits for the grouped report, because a critical buried at position 23 of 40 gets read after the fix window closed.
2. **Cheap evidence before expensive evidence, and never authenticate speculatively.** Four rungs: file metadata (`ls`, `stat`, size, mtime) → file content (`grep`, `head`) → local state (process list, disk, `git status`) → an authenticated remote call. Climb a rung only when the one below produced a signal, or when the question is unanswerable locally (is this token still valid, is this endpoint up). One authenticated call per integration per run, never one per check: the sweep must not be the thing that exhausts the rate limit.
3. **Evidence, severity, one action — or it is not a finding.** The evidence is the command output or the file and line that produced it, not a recollection. Severity comes from the rubric below, not from tone. Two actions means it is two findings; zero actions means delete it.
4. **Rotate before you scrub.** A leaked credential's clock started when it was written, not when you found it. Order: revoke or rotate at the issuer → verify the old value now fails → remove it from the file and replace with a pointer → rewrite history if it was committed → check the issuer's access log for use between the add date (`git log --diff-filter=A`) and the revocation. Scrubbing first leaves a live credential in every clone and destroys the timestamp you need.
5. **No fix without an inverse and a verification.** Before any write, record what it takes to undo it (the old value, the old mode, a copy of the file) in the fix row of `runs/<year>.md`. After it, re-run the exact detection that found it and show it no longer fires. A fix you cannot undo and cannot verify is a proposal, whatever `autofix_policy` says.
6. **A finding that returns is a different bug.** Same finding in two consecutive runs: the fix did not stick, or it was never a problem. Escalate one severity and fix the generator instead of the instance, or move it to `## Accepted` with a review date. Three appearances in 90 days is systemic by definition — the check keeps finding a decision nobody made.
7. **Acceptance is explicit and expires.** "That is intentional" becomes a row in `## Accepted` with the rule it suppresses, the scope (a path glob or a check id, never "that file that day"), the reason, and a review date — default 90 days, or the credential's expiry if it has one. An acceptance with no date is a permanent blind spot that every future audit will inherit.
8. **Budget the always-loaded set; do not eyeball it.** Anything the agent reads on every turn is a per-turn tax: `tokens ≈ bytes / 4` for prose, `bytes / 3` for code and JSON, and `daily tax ≈ tokens × turns_per_day`. A 40 KB instruction file is ~10k tokens; at 60 turns a day that is ~600k input tokens a day before the user has said anything. Measure the set, name the three biggest files, then trim (`cost.md`).

## Severity And Findings Format

Severity is a function of exposure and reversibility, never of how alarming the wording is.

| Severity | Test it must pass | Examples |
|---|---|---|
| CRITICAL | Someone other than the user can act with the user's authority right now, or data is being lost while you read | live token in a world-readable or committed file · `bash -c *` in an allowlist · two sessions writing the same memory file · a session looping unattended |
| WARNING | It will fail or leak on a foreseeable event that has a date or a threshold | token expires in 9 days · job p95 runtime above half its interval · disk at 88% · unpushed work older than 7 days |
| INFO | Waste or drift with no failure mode in the next 30 days | orphan file · skill never activated in 60 days · duplicate note · unused permission grant |

One block per finding, evidence line first, action last:

```
[CRITICAL] secrets/plaintext — provider token at config/deploy.yml:14, file mode 644
  evidence: prefix pattern match; file readable by all users
  action: rotate at the issuer, then replace the value with env:DEPLOY_TOKEN
  fixable: no — rotation belongs to the user
```

- Group by severity, never by category: the user reads from the top and stops somewhere.
- Show at most `max_findings_shown` in full, then one summary line per category with counts. Every finding goes to `## Open Findings` regardless of whether it was shown.
- A check that could not run is reported as **not checked**, with why. Silence reads as "clean" and is the only way an audit can lie.
- Never reproduce a secret value, and never quote enough of the line to reconstruct one.

## Symptom Signatures

Decode rule: the layer that changed names the subsystem. Behavior that changed with no prompt change is configuration; behavior that changed with no configuration change is state.

| Signature | Most likely cause | First move |
|---|---|---|
| Agent "forgot" something it was clearly told | The fact was written to a file nothing reads, or its read order is conditional | Trace written → indexed → read; conditional reads lose everything from the second session (`agent-memory.md`) |
| Every turn got slower after adding one file | The file joined the always-loaded set | Diff the always-loaded set against the last baseline, then Rule 8 |
| Spend doubled, usage identical | A changing line (timestamp, date, counter) at the top of an always-loaded file invalidates the cached prefix every turn | Make the prefix byte-stable; move volatile lines to the end (`cost.md`) |
| Job "ran successfully" but nothing happened | Non-interactive environment: different PATH, missing env, wrong working directory | Run it with the scheduler's environment, not your shell's (`scheduled.md`) |
| Job stopped firing entirely, no error | Host asleep, scheduler not loaded at boot, or a timezone/DST shift moved it | Compare last-success timestamp to the interval; check the schedule's timezone (`scheduled.md`) |
| Job ran twice, or skipped one day | Schedule sits in the 01:00-03:00 local DST window | Move it outside the window or express it in UTC (`scheduled.md`) |
| Two skills answer the same request | Trigger overlap in the first sentence of both descriptions | Partition the triggers; add an explicit "not for" to the narrower one (`skills.md`) |
| A skill never activates | Its trigger words are absent from the first sentence, where truncation and attention both land | Rewrite the first sentence around the words a user actually types (`skills.md`) |
| Integration worked for months, now 401 | Token expired, was rotated elsewhere, or the account lost a scope | Check expiry in the credential inventory before assuming compromise (`integrations.md`) |
| Authenticated call returns 404 on something that exists | Scope problem — several APIs return 404 instead of 403 to avoid confirming existence | Compare the token's scopes to the call, not the object's existence (`integrations.md`) |
| Signed requests fail with a valid credential | Clock skew above the provider's tolerance (5 minutes is the common ceiling) | Check host time against a time source before touching the credential |
| Disk filling with nothing installed | Transcripts, job output, or session artifacts with no retention | Find the top three directories by size, then set a retention (`workspace.md`) |
| Agent did something it was never asked to do | A grant made it possible; the prompt only used it | Audit the allowlist, not the transcript (`permissions.md`) |
| Anything else | Reproduce with the smallest possible input, then bisect the environment: halve the always-loaded set, disable half the automations | — |

## Full Audit Order

Order matters because hygiene work destroys the evidence security work needs, and because a live problem is spending money while you read the rest.

| # | Phase | Looks for | File | In quick mode |
|---|---|---|---|---|
| 1 | Exposure | Credentials in files, history, logs, permissions on key material | `secrets.md` | yes |
| 2 | Authority | Allowlists, auto-approve, self-editable config, network egress | `permissions.md` | yes |
| 3 | Live activity | Runaway or zombie sessions, concurrent writers, stale locks | `sessions.md` | yes |
| 4 | Automations | Schedule validity, overlap, silent failure, output growth | `scheduled.md` | overdue only |
| 5 | Integrations | Reachability, auth, expiry calendar, rate-limit posture | `integrations.md` | overdue only |
| 6 | Memory | Written / indexed / read, contradictions, growth | `agent-memory.md` | no |
| 7 | Workspace | Structure, orphans, dangling references, repo state, backups | `workspace.md` | no |
| 8 | Installed set | Collisions, dead references, missing binaries, token tax | `skills.md` | no |
| 9 | Spend | Trend against baseline, cache behavior, fan-out | `cost.md` | no |
| 10 | Latency | Round trips, context assembly, timeouts | `performance.md` | no |

Then `remediation.md` for what may be fixed now, and `tracking.md` to close the run: counts to `runs/<year>.md`, open items to `## Open Findings`, cadences to `## Due`.

Targeted mode runs one phase and says so in the report header, so nobody reads a one-phase run as a clean bill of health.

## Output Gates

Before delivering a report or touching anything:

- Does every finding carry evidence, a severity from the rubric, and exactly one action?
- Was `## Accepted` read first, so nothing the user already decided about is back on the list — and is any acceptance past its review date raised instead of honored?
- Are criticals stated first and separately, and is the report capped at `max_findings_shown` with the rest summarized by count?
- Is every check that could not run listed as *not checked*, with the reason?
- Is any secret value, token fragment, private key, or full credential line present in my output or in a file I wrote? It must not be.
- For anything fixed: is the inverse recorded, and did the original detection get re-run and come back clean?
- Did anything durable come out of this run — a run row, a finding opened or closed, an acceptance, a discovered job or integration or host, a measured baseline, a runbook? Then it is written to its box in `memory-template.md`, with its `## Boxes` line, in this same turn.

## Configuration

User-dependent variables. Defaults apply until the user states a preference; store them in `~/Clawic/data/analysis/config.yaml`.

| Variable | Type | Default | Effect |
|---|---|---|---|
| default_mode | quick \| full \| targeted | quick | Which phases of Full Audit Order run when the request names no scope |
| autofix_policy | propose \| safe-only \| ask-each | propose | `propose` writes nothing; `safe-only` applies reversible, non-destructive fixes and reports them; `ask-each` asks per fix (`remediation.md`) |
| workspace_paths | list (paths) | auto | Directories treated as the agent's setup; `auto` means the current project plus the agent config directories found beside it |
| excluded_paths | list (globs) | none | Never scanned or reported on — client work, encrypted vaults, mounted volumes |
| audit_cadence | weekly \| biweekly \| monthly \| none | monthly | Seeds the full-audit row of `## Due`; quick checks default to one interval finer |
| secret_rotation_days | number (days, 30-365) | 90 | Age at which a credential in the inventory becomes a WARNING, and the default review interval for acceptances |
| memory_budget_mb | number (MB, 1-500) | 5 | Total size of memory and notes above which growth is a finding (`agent-memory.md`) |
| max_findings_shown | number (1-50) | 10 | How many findings are printed in full before the rest collapse into per-category counts |
| secret_store | keychain \| 1password \| bitwarden \| vault \| env \| none | none | Which pointer kind is written in the credential inventory and which rotation path is offered; `none` means follow whatever the setup already uses, falling back to `env:` |

Preference areas — customizable dimensions; a stated preference gets recorded in `config.yaml` and applied from then on:

- **Tooling** — which search and inspection tools exist on this machine (`rg` vs `grep`, `fd` vs `find`, GNU vs BSD `stat`), whether a secret scanner or a linter is already installed — affects every detection command's form
- **Conventions** — finding id scheme, severity labels the team already uses, where reports are filed and under what name, report language — affects the findings format and `tracking.md`
- **Platform** — OS family for permission and process checks (commands here are POSIX; on Windows they run under WSL or Git Bash), whether remote hosts are in scope, single machine vs several
- **Safety posture** — whether git history may be rewritten, whether sessions may be killed, whether credentials may be rotated by the agent, whether deletion is ever allowed — affects `remediation.md` and the reversibility test
- **Escalation** — what counts as critical here: a read-only token in a private repo is not what a production key is, and the rubric bends to the user's stated blast radius
- **Cadence** — per-phase frequency, when the report goes to a human, quiet hours for checks that touch remote services — every accepted cadence becomes a row in `## Due`
- **Output register** — severity-grouped vs phase-grouped, terse list vs explained findings, whether commands are shown alongside conclusions

## Traps

| Trap | Why it fails | Do instead |
|------|-------------|------------|
| Opening the audit with authenticated calls to every integration | Burns rate limit and finds what file metadata would have found for free; a throttled provider then hides real findings | Climb the four rungs of Rule 2; one authenticated call per integration |
| Scrubbing a leaked key before rotating it | The key stays live in every clone and the add-timestamp you need for the access-log check is gone | Rotate, verify dead, then scrub (Rule 4) |
| Quoting the secret in the finding so the user can identify it | The report itself becomes the leak, and reports get pasted into chats and tickets | File, line, and kind only |
| Reporting 40 findings without severities | Everything is equally urgent, so nothing gets fixed and the next run reports the same 40 | Cap at `max_findings_shown`, group by severity, write the rest to `## Open Findings` |
| Suppressing a false positive by deleting the check | The check was right about a different file six weeks later | Acceptance row with scope and a review date (Rule 7) |
| Fixing hygiene first because it is easy | Consolidating notes and pruning files destroys the evidence the exposure phase needed | Phase order in Full Audit Order |
| Auto-fixing anything that touches credentials, history, or user content | Rotation needs the issuer, history rewrite needs every clone, deletion needs judgment | Propose with the exact command and let the user run it (`remediation.md`) |
| Killing a stuck session to stop the spend | The transcript that explains the loop dies with it, and it recurs next week | Capture the last exchanges and the repeated call, then kill (`sessions.md`) |
| Trusting a job's exit code as proof it worked | A job can exit 0 having done nothing: empty input, wrong directory, silent auth failure | Assert on the artifact — the file, the row, the timestamp (`scheduled.md`) |
| Treating a clean quick check as a clean system | Quick mode never reaches memory, workspace, installed set, spend, or latency | Say which phases ran in the report header |
| Measuring memory or context health by file count | Ten small files cost less than one bloated one; the tax is bytes per turn | Rule 8's formula against the always-loaded set |
| Auditing the transcript to explain what the agent was able to do | The transcript shows what was used; the grant shows what was possible | Read the allowlist (`permissions.md`) |
| Running the audit and not writing the run down | Recurrence, trend, and acceptance all need a previous run to compare against | `runs/<year>.md` in the same turn (`tracking.md`) |

## Where Experts Disagree

- **Auto-fix vs propose.** Teams that audit weekly want reversible fixes applied silently, because a report nobody actions is theatre; teams that have been burned want every write proposed. The frontier is not caution but reversibility and verification: if the inverse is recorded and the detection can be re-run, silence is defensible (`safe-only`); if either is missing, propose regardless of preference.
- **Entropy scanning.** High-entropy string detection catches credentials no prefix list knows about, and drowns a real repo in lockfile hashes, UUIDs and minified assets. Default: prefix families everywhere, entropy only in config-shaped files and in anything the prefix scan already touched (`secrets.md`).
- **How much history to keep.** Long run history makes trends real and makes the workspace one of the things being audited. The boundary is per-run detail: keep counts and open findings forever, keep full evidence blobs for one cadence period.
- **Whether the agent audits itself at all.** A setup that can read its own config can also normalize what it finds there. The mitigation is not to skip the audit but to keep the report reproducible: every finding names the command and the file, so a human can re-run three of them at random and get the same answer.

## Security & Privacy

**Credentials:** this skill searches for credentials in order to report their location. It does NOT read, print, copy, transmit, or store any credential value, does NOT write a credential into `~/Clawic/data/`, and never rotates or revokes anything on the user's behalf without an explicit instruction. Findings carry file, line, and kind only.

**Local storage:** run history, open findings, acceptances, baselines, and the credential *inventory* (pointers and expiry dates, never values) stay in `~/Clawic/data/analysis/` on this machine, plus host rows in `~/Clawic/data/servers/`, devices in `~/Clawic/data/devices/`, recurring paid services in `~/Clawic/data/finances/`, and the name and role of a person who owns a credential or a job in `~/Clawic/data/contacts/`.

**Network:** authenticated calls are made only against services the user has already configured, only to answer a question that cannot be answered locally (validity, reachability, expiry), and at most once per integration per run. No finding, transcript, or file content is sent anywhere.

**Guardrails:** read-only by default (`autofix_policy: propose`). Anything destructive — deleting files, killing sessions, rewriting history, rotating a credential — is presented with its blast radius and its inverse, and requires explicit confirmation.

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/analysis (install if the user confirms):
- `skill-audit` — whether a third-party skill is safe to install; this one assumes the code is trusted and asks whether the setup is healthy
- `skill-manager` — installing, updating, and removing skills once this audit says which ones to touch
- `context-engineering` — designing the context window that Rule 8 measures
- `incident-response` — containment and post-mortem once a finding turns out to be a real breach
- `monitoring` — continuous observability for applications and servers, as opposed to point-in-time audits of the agent's own setup

## Feedback

- If useful, star it: https://clawic.com/skills/analysis
- Latest version: https://clawic.com/skills/analysis

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/analysis.
