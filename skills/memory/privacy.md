# Privacy — Secrets, Sensitive Subjects, and Deletion

This store is plaintext markdown in the user's home directory. Anyone with that account, any backup of it, and any process the user runs can read every word. Every rule below follows from that one fact.

## Never Stored

| Category | Instead |
|---|---|
| Passwords, API keys, tokens, recovery codes | A pointer: "API key in 1Password, item 'Northwind prod', rotated 2026-07" |
| Full card numbers, IBANs, full account numbers | The last four digits plus the issuer, if the user needs to recognize it |
| Government ID numbers, passport numbers | The fact that the document exists and where it is kept |
| Anything under `excluded_topics` | Say once that it is excluded; do not store, do not re-ask |
| Private details about third parties the user did not choose to record | Nothing — see below |

Declining is one sentence, no lecture: "I won't store the key itself — I'll note where it lives so you can find it." Then store the pointer, because the pointer is the useful part.

## Third-Party Data

The user's contacts have not agreed to a dossier. The working boundary:

- **Store**: what the user needs recalled to work with that person — role, preferences they told you, shared history, agreements.
- **Don't store**: health, finances, legal exposure, relationship details, or anything the person themselves would be surprised to find written down, unless the user explicitly asks for it stored.
- **Never store**: content the user quoted from a private channel, when the value is the quote rather than the fact.

Same standard applies to the user's own sensitive categories. The difference is only that they can override it for themselves; they cannot override it for someone else.

## Deletion Requests

"Forget that", "delete what you know about X", "remove that from memory" are all the same operation, and it must be complete or it is worse than nothing — a half-deleted entity resurfaces later and reads as a violation.

```
1. Scope it out loud: one fact, one entity, or one category? Read back what will go.
2. grep -ril "<term>" ~/Clawic/data/memory/     # every file that mentions it, including inbox/ and sync/
3. Delete the lines or the files.
4. Remove every INDEX row that pointed at them.
5. Fix inbound links: a "→ people/x.md" pointing at a deleted file is a dead reference AND a leftover trace of the name.
6. Report exactly what was removed, by path. "Done" is not a receipt.
```

What deletion here does **not** cover, and the user should hear it once: built-in agent memory (the runtime owns it, Rule 1), the cloud provider's version history if the store sits in a synced folder, git history if the store is versioned, and system backups. Deleting a git-tracked memory file leaves it fully readable in the history.

## Storage Realities to State Plainly

- **No encryption.** If the user wants it, the store belongs on an encrypted volume or in an encrypted disk image; this skill does not roll its own crypto and does not pretend the files are protected.
- **Sync copies everything.** A store in a cloud folder is a store on someone else's server, and the provider keeps deleted versions after you delete the file.
- **Shared machines.** A shared account means shared memory. Separate OS accounts are the only real boundary; a subfolder is not one.
- **Screen sharing and pasted output.** Reading an entry aloud in a shared session publishes it. With `recall_citations: true` the path is visible too — worth knowing before opening `people/` on a call.

## Redaction Instead of Deletion

When the fact matters but its detail does not, keep the shape and drop the payload:

```markdown
- 2026-07-25 · stated · Northwind's renewal budget discussed; figure held by the user, not stored
- 2026-06-02 · stated · Bob had a family emergency in June (details not recorded)
```

This preserves the timeline — why a deadline moved, why someone was unreachable — without holding the sensitive part. It is the right default for anything under a third party's privacy.

## Answering "What Do You Have On Me?"

A legitimate and frequent question. Answer with the map, not a dump: which categories exist, how many entries in each, and the most recently updated files. Offer to walk one category at a time, and treat every correction that comes out of the walk as a live supersede-or-delete decision on the spot (SKILL.md Rule 8), not a note for later. Users audit the store most willingly right after it surprises them.

## Back To

SKILL.md — Rule 9 (never store secrets), Security & Privacy (the does-NOT block this file expands), Configuration (`excluded_topics`, `delete_policy`, `recall_citations`), Output Gates (the pre-store and pre-delete checks).
