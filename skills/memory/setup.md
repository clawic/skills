# Setup — Memory

Read this on first use, when `~/Clawic/data/memory/` doesn't exist yet. Guide the user through a short conversation, then create the structure and prove it works with one real entry.

**Boundary first:** this system is SEPARATE from built-in agent memory. Built-in keeps working untouched; this adds parallel, organized, unlimited storage.

## The Conversation

### 1. Explain What This Is

"I can set up an organized memory system for you — separate from my built-in memory. It's for anything you want stored long-term: projects, people, decisions, knowledge, collections. It won't change how I normally remember things; it's additional storage that scales as big as you need."

### 2. Ask What They Need

"What would be most useful to have perfectly organized?

- **Projects** — full history, decisions, context per project
- **People** — detailed profiles of everyone you work with
- **Decisions** — why you chose X over Y, findable later
- **Knowledge** — things you're learning, reference material
- **Collections** — books, recipes, ideas, anything you collect"

Let them answer; don't assume. Create only the categories they actually name — 2-3 is a normal start. Empty categories created "just in case" rot (SKILL.md Rule 2); more emerge later from inbox sorting.

If several categories map to one profession ("clients, deals, competitors"), suggest the domain-focused layout — `patterns.md`, Pattern 2.

### 3. Ask About Sync

"My built-in memory already tracks some things. Want me to copy any of it into this system — preferences, key decisions, key contacts? Or start fresh?"

Sync is one-way and manual (SKILL.md Rule 8). Default to starting fresh; sync only what they'll want deeply structured.

### 4. Create the Structure

Create `~/Clawic/data/memory/` with:
- `config.md` — their answers (template: `memory-template.md`)
- `INDEX.md` — root index (template: `memory-template.md`)
- One folder per named category, each with its own INDEX.md

### 5. First Entry — Prove It Works

Ask: "What's something you'd like me to remember right now?"

Store it immediately: write the entry file, date-stamp it, add it to the category INDEX, then show them where it lives. The demo *is* the training — they see the write happen before the reply.

## When Done

The system is live once:
1. `~/Clawic/data/memory/` exists with their categories
2. Every folder has an INDEX.md
3. One real thing is stored and indexed

It grows from there through normal use — every durable fact the user shares gets written before you reply (SKILL.md Rule 3).
