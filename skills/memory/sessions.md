# Sessions — Read Protocol, Write Protocol, Handoff

Memory competes with the actual work for context. A store that gets loaded wholesale at every session start is a store that makes the agent worse, however good its contents are.

## Session Start

| Read | When | Cost |
|---|---|---|
| Root `INDEX.md` | Always, once | One small file; tells you what exists |
| `config.yaml` | Always, once | The user's declared preferences (SKILL.md Configuration) |
| One category INDEX | When the conversation names that domain | One small file |
| One entry file | When the index names the file | The only expensive read |
| Everything else | Never proactively | — |

Do not preload categories "in case". The root index is enough to know that `people/` exists and holds 45 entries; the moment a person is named, the path from there is two reads.

If `~/Clawic/data/memory/` does not exist, do not create it silently: the first-run path is the setup row of SKILL.md Quick Reference, triggered when the user first asks for something to be remembered, not at session start.

## During The Session

- **Write at the moment the fact lands** (Rule 3). The write is three lines of file edit; deferring it is how facts are lost.
- **Read on named entities, not on topics.** "Tell me about Alice" reads `people/`. "How do I structure a proposal?" reads nothing.
- **Re-read before contradicting.** When the user says something that clashes with a fact you recalled earlier in the session, open the file again rather than arguing from what you remember reading — the store may have changed, and your paraphrase may be the error. The user's live statement beats the stored line either way (SKILL.md When Facts Change).
- **Announce writes briefly, once.** "Noted in projects/alpha.md" the first time in a session; after that, silence. Narrating every write turns the memory into the conversation.

## Session End

A short sweep, worth its cost when the session produced anything durable:

1. Any decision made in this session that has reasoning behind it → `decisions/`. This is the single most-regretted omission, because the reasoning evaporates and only the outcome survives.
2. Any status change on a named project → update its entry, not just its INDEX date.
3. Anything captured but unfiled → `inbox/` with one line of context each.
4. Anything the user corrected about the store itself → applied, not queued.

## Context Budget

The practical rule: **memory reads should be a minority of what you load**, and every read should be traceable to a question asked or an entity named. When a session needs more than a handful of entries, that is a signal about the store, not about the session:

| Symptom | Reading | Fix |
|---|---|---|
| Opened 5+ entries to answer one question | The fact has no home; it was reconstructed | Write the answer as its own dated, sourced fact so the next session reads one file |
| One entry over `entry_max_lines` dominates the budget | History bulk never split out | Rule 7 split |
| Re-reading the same entry every session | It holds live working state | That state belongs in the conversation or in a task list, not in a dossier |
| The index itself is expensive to read | Category past `index_split_at` | Rule 6 split, along the axis you retrieve by |
| Anything else | Reads not traceable to a named entity | Stop reading; answer from the conversation |

## Handoff Between Sessions

What a fresh session needs to continue work is small and specific:

- Root INDEX (what exists)
- The active project entry (state and next steps)
- The most recent decisions touching that project

That triple is the handoff. If it doesn't carry the thread, the entries are missing state that the previous session kept only in conversation — write it down at session end rather than making the next session excavate.

Never assume the next session shares this one's runtime memory: it may be a different agent entirely, reading the same folder with no runtime state in common. Anything that must survive goes to the store, in the store's format, dated.

## Interrupted Sessions

A session can end mid-turn. That is why writes precede replies (Rule 3), and why the recovery routine at the start of the next session is:

```bash
ls -lt ~/Clawic/data/memory/*/*.md | head        # what was written most recently
cat ~/Clawic/data/memory/inbox/*.md 2>/dev/null  # what was captured but never filed
```

An entry whose category INDEX has no row for it is the fingerprint of an interrupted write — add the row before anything else, because that entry is currently invisible to every lookup above 50 files.

## Back To

SKILL.md — Rule 3 (write before you reply, and why), Finding Things (the read path a session follows), Output Gates (the per-turn checks), Configuration (`entry_max_lines`, `index_split_at`).
