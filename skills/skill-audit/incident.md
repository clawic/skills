# Incident — When an Installed Skill Turns Out Hostile

Triggers: the agent did something nobody asked; unexplained network, file, or tool activity; a sweep or update audit flags an installed skill; the registry pulls a skill you have.

## Order of Operations (contain before you investigate)

1. **Quarantine.** Move the folder out of every agent's skill directory into `~/Clawic/data/skill-audit/quarantine/<slug>-<date>/`. Move, never delete — it is evidence, and diffing it against the registry copy shows local tampering.
2. **Kill loaded context.** A removed skill still governs any session that already loaded it. End those sessions; the fix applies only to fresh ones.
3. **Sweep for spread.** Persistence grabs (`injection-patterns.md`) mean checking everything the skill could have written: agent memory files, CLAUDE.md/AGENTS.md, hooks and settings, shell rc files, crontabs, other skills' folders. Diff each against a known-good copy or git history.
4. **Audit the quarantined copy.** Full five passes; find the line that explains the observed behavior. If nothing explains it, the skill may be the wrong suspect — widen to a full sweep (`sweep.md`) before closing.

## Blast Radius

Rotate what was REACHABLE, not just what you saw leave:

- Window: install/update date → quarantine date; the audit log gives the left edge.
- Reachable = readable by the agent while the skill was loaded: env vars in agent context, credentials in files under directories the agent worked in, tokens in dotfiles.
- Any network sink in the skill (`exfiltration.md` ledger) → assume reachable secrets left; rotate the keys, tokens, and passwords in scope.
- No sink and no execution → exposure is limited to in-context data; judge from session transcripts.

## Report Upstream

- To the registry: slug, owner, version, file:line evidence, pattern class. Evidence-quality reports get skills pulled; "seems malicious" does not.
- To anyone you recommended it to: the audit log shows which version you vouched for — say which.

## Close Out

- Log the incident in `audit-log.md`: date, slug, version, what it did, what was rotated.
- Novel pattern that nothing in `checks.md` matched → add its grep to `~/Clawic/data/skill-audit/patterns.md`, which runs alongside the battery from then on. An incident that doesn't improve the battery will repeat.
