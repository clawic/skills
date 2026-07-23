---
name: memory
slug: memory
version: 1.0.3
description: >-
  Infinite categorized memory in ~/memory/, parallel to built-in agent memory.
  Use when the user wants to remember, store, or recall projects, people,
  decisions, notes, or any collection that grows over time.
homepage: https://clawic.com/skills/memory
changelog: Deeper memory design patterns and retrieval rules
metadata:
  clawdbot:
    emoji: 🧠
    requires:
      bins: []
    os:
    - linux
    - darwin
    - win32
    displayName: Memory
---

# Memory 🧠

Infinite organized memory, parallel to the agent's built-in memory. All data lives in `~/memory/` on the user's machine — plain markdown files, no external services, no network requests.

## How It Works

Built-in memory holds current context and works automatically. This system holds everything that grows. Parallel and complementary — never a replacement.

| | Built-in memory | This skill (`~/memory/`) |
|---|---|---|
| Size | Small, summaries | Unlimited |
| Written by | Agent runtime, automatically | This skill, by explicit rules |
| Holds | Current status, quick facts | Full histories, dossiers, logs |
| Touched by this skill | Read-only (sync source) | Fully managed |

## When To Use

- User says "remember this", "save this", "add to memory" about anything with lasting value
- Recall: "what did I tell you about X", "when did we decide Y" — search `~/memory/` before answering "I don't know"
- Structured records that grow: projects, people, decisions, domain knowledge, collections
- Consolidating scattered notes into one organized system
- Not for: session-scoped context, quick facts built-in memory already tracks, or secrets — credentials are never stored (Rule 9)

## Setup

First use with no `~/memory/` present: read `setup.md` and run the setup conversation. Three decisions: which categories, whether to sync from built-in memory, how the user prefers to retrieve.

## Architecture

```
~/memory/
├── config.md              # System configuration
├── INDEX.md               # Root index: what exists, where
│
├── [user-defined]/        # Categories the user needs
│   ├── INDEX.md           # Category index
│   └── {items}.md         # Individual entries
│
└── sync/                  # Optional: one-way copies from built-in memory
```

The user defines categories — common ones: `projects/`, `people/`, `decisions/`, `knowledge/`, `collections/`. Templates for all of them: `memory-template.md`. Folder layouts and when each wins: `patterns.md`.

## Quick Reference

| Situation | Play |
|-----------|------|
| No `~/memory/` yet | Run setup — `setup.md` |
| User shares a durable fact | Write file + update category INDEX **before** replying (Rule 3) |
| User asks about the past | Search ladder: indices → full grep → one file (→ Finding Things) |
| Fact fits two categories | One canonical home, links from the rest (Rule 5) |
| Unsure where it goes | `inbox/` now, sort at weekly maintenance |
| Category INDEX reaches 100 entries | Split into subcategories (Rule 6) |
| Recall misses or feels slow | `troubleshooting.md` |
| Choosing or changing folder layout | `patterns.md` |
| Anything else | Closest existing category beats a new one — create categories sparingly |

## Core Rules

1. **Built-in memory is read-only territory.** Never modify the agent's MEMORY.md or workspace `memory/` — the runtime owns and may rewrite them; your edits get lost or conflict. This skill only ever reads them, for one-way sync.

2. **The user defines the structure.** Create categories from their words, not a preset taxonomy. A category created before content exists rots empty — create it when its first item arrives.

   | They say... | Create |
   |-------------|--------|
   | "I have many projects" | `~/memory/projects/` |
   | "I meet lots of people" | `~/memory/people/` |
   | "I want to track decisions" | `~/memory/decisions/` |
   | "I'm learning [topic]" | `~/memory/knowledge/[topic]/` |
   | "I collect [things]" | `~/memory/collections/[things]/` |

3. **Write before you reply.** Sequence: write the entry → update category INDEX.md → then respond. The reply is disposable; the write is the durable part. A session that dies mid-reply loses nothing.

4. **Date-stamp every entry** (`YYYY-MM-DD`). Undated facts can't be judged stale at recall time: "Alice works at Acme" means different things written last week vs. two years ago.

5. **One fact, one home.** Store each fact in the file it's most about; everywhere else links to it (`→ people/alice.md`). Duplicated facts diverge silently — the copy you update is never the copy you find later.

6. **Every folder has an INDEX.md, max 100 entries.** Past 100, split into subcategories. Two levels of ≤100 cover 10,000 items (100×100); three cover 1,000,000. Every lookup stays two small reads instead of one giant scan.

7. **Keep entry files lean.** An entry is loaded whole to answer one question. Past ~200 lines, move `## History` bulk to `{name}-history.md` and keep the dossier itself short.

8. **Sync is one-way.** Built-in → `~/memory/sync/`, reformatted on the way, sync date recorded in `sync/INDEX.md`. Never the reverse; never automatic — re-sync on request or during maintenance.

9. **Never store secrets.** Memory files are plaintext with no encryption. Decline passwords, API keys, and tokens; store a pointer instead: "API key lives in [their vault]".

## What to Store Here (vs Built-In)

| Store HERE (`~/memory/`) | Keep in BUILT-IN |
|--------------------------|------------------|
| Detailed project histories | Current project status |
| Full contact dossiers | Key contacts quick-ref |
| Decision log with reasoning | Recent decisions |
| Domain knowledge bases | Quick facts |
| Collections, inventories | — |
| Anything that grows | Summaries |

**Rule:** built-in for quick context, here for depth and scale. Same item can exist in both — summary there, detail here, and the summary points here.

## Finding Things

| Memory size | Strategy |
|-------------|----------|
| <50 files | `grep -ri "keyword" ~/memory/` — full scan is cheap |
| 50–500 files | Indices first: `grep -i "keyword" ~/memory/*/INDEX.md`, then open the one matching file |
| >500 files | Hierarchical: root INDEX → category INDEX → file; semantic search if the runtime offers it |

```bash
cat ~/memory/INDEX.md           # what categories exist
grep -i "alpha" ~/memory/*/INDEX.md   # which category holds it
cat ~/memory/projects/alpha.md  # the answer
```

Search vocabulary asymmetry: grep finds the words used at *write* time, but recall uses today's words. That's why entries carry a `Keywords:` line with aliases (→ `patterns.md`, Pattern 10). Try 2–3 keyword variants before concluding absence.

## Maintenance

**Weekly:** sort `inbox/` into categories; update any INDEX touched during the week; archive items with terminal status (completed, cancelled, inactive).

**Monthly:** check index sizes (`wc -l ~/memory/*/INDEX.md`); split any category past 100 entries; **delete** entries that are wrong — a stale fact recalled as true is worse than a gap. Archive is for old-but-true; delete is for no-longer-true.

## Output Gates

Before replying when the user shared something durable:
- Is it already written to `~/memory/`? (Rule 3 — write precedes reply)
- Is the category INDEX.md updated?
- Is the entry date-stamped?

Before answering "I don't remember": did I run the full search ladder with ≥2 keyword variants, and check `sync/` and built-in memory?

Before storing anything: does it contain a credential? → Rule 9, decline.

## Traps

| Trap | Why it fails | Do instead |
|------|--------------|------------|
| Modifying built-in memory | Runtime owns it; edits get overwritten or conflict | Read-only; parallel system (Rule 1) |
| Batching writes for later | Session can end between hearing and writing | Write before replying (Rule 3) |
| Same fact in two files | Copies diverge; you update one, retrieve the other | One home + links (Rule 5) |
| No indices | Every lookup degrades to full-folder grep; misses rise with size | INDEX.md in every folder (Rule 6) |
| One giant category | 500 unsplit items = slow scans, noisy grep hits | Split at 100 entries (Rule 6) |
| Undated entries | Can't distinguish current fact from two-year-old fact | Date-stamp everything (Rule 4) |
| Renaming files without updating INDEX | Dead links — recall says "not found" while data exists | Rename = `mv` + fix both indices |
| Syncing everything from built-in | Mass duplication, and built-in already answers those queries | Sync only what needs deep structure |
| Archiving wrong facts | Archive preserves them; they resurface as true | Wrong → delete; old-but-true → archive |

## Security & Privacy

**Data location:**
- All data in `~/memory/` on the user's machine
- No external services, no network requests

**This skill does NOT:**
- Modify built-in agent memory (reads only, for one-way sync)
- Send data anywhere
- Store credentials — plaintext storage; secrets are declined (Rule 9)

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/memory (install if the user confirms):
- `decide` - Decision tracking patterns
- `escalate` - When to involve humans
- `learn` - Adaptive learning

## Feedback

- If useful, star it: https://clawic.com/skills/memory
- Latest version: https://clawic.com/skills/memory

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/memory.
