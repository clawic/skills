# Setup — First Use

Read this when `~/Clawic/data/memory/` doesn't exist yet and the user has just asked for something to be remembered. The store is created around that first fact, not around a questionnaire.

**Boundary first, in one sentence to the user:** this is separate from built-in agent memory — built-in keeps working untouched, this adds parallel, organized, unlimited storage on their machine.

## The Default Path (no questions)

The first fact names the category. Create only what it needs:

```
~/Clawic/data/memory/
├── config.yaml          # the defaults of SKILL.md Configuration, nothing invented
├── INDEX.md             # root index, one row
└── people/              # or projects/, decisions/ — whichever the fact belongs to
    ├── INDEX.md
    └── alice-smith.md   # the fact, dated and sourced
```

Then show the user where it landed: "Stored in `~/Clawic/data/memory/people/alice-smith.md`." The demo *is* the training — they see the write happen before the reply (Rule 3), and they learn the path they can open themselves.

More categories arrive the same way, one at a time, as facts need them. A category created before its first item rots empty (Rule 2).

## The Conversation (only when the user asks for a full setup)

When the user wants the system designed up front — "set up your memory", "let's organize everything" — this is the whole script:

**1. What is worth having perfectly organized?**

- **Projects** — history, decisions, context per project
- **People** — profiles of everyone they work with
- **Decisions** — why X over Y, findable later
- **Knowledge** — reference material, things they're learning
- **Collections** — books, recipes, ideas, anything they collect

Create only what they name; two or three is a normal start. If everything they name belongs to one profession ("clients, deals, competitors"), the folders take those names directly rather than the generic five — the layout catalog is one row of SKILL.md Quick Reference away.

Anything whose domain already has its own Clawic skill and store — pets, garden, household, code style — does not become a category here (SKILL.md Rule 5). Say so once, and point at the owning store.

**2. Anything to bring over?** Existing notes, a vault, another agent's memory come in category by category with a dedupe pass, never as a bulk copy — the import row of SKILL.md Quick Reference. Built-in memory → one-way into `sync/`, and only what needs deep structure (Rule 1). Default is starting fresh.

**3. Nothing else.** Thresholds, cadence, deletion policy and the rest have working defaults (SKILL.md Configuration) and get recorded only if the user states a preference — never asked for.

## What Gets Created

| File | Content |
|---|---|
| `config.yaml` | The defaults of SKILL.md Configuration, plus the category list; nothing the user did not state |
| `INDEX.md` | Root index: one row per category |
| `{category}/INDEX.md` | One per category, empty table with its columns |
| `inbox/` | Only if `inbox_enabled` (default true) |
| Anything else | Not yet — no archive folder before the first archive, no `sync/` before the first sync |

## Done When

1. `~/Clawic/data/memory/` exists with the categories the facts needed
2. Every folder has an INDEX.md
3. One real thing is stored, dated, sourced, and indexed
4. The user has seen the path it lives at

From there it grows through normal use: every durable fact gets written before the reply (Rule 3), and the pass at `maintenance_cadence` keeps it clean.

## If a Store Already Exists Somewhere Else

- Data at `~/memory/` or `~/clawic/memory/` → move it to `~/Clawic/data/memory/`; the paths inside the files stay valid because entries link relatively.
- A `config.md` from an older install → merge its values into `config.yaml`, delete the old file.
- The path exists but resolves to nothing → it is a symlink to a cloud folder or volume that has not mounted. Do not create a fresh store on top of it; wait for the target.
- An unfamiliar store at the right path (another agent set it up) → read its root INDEX and work with it; do not restructure someone else's layout without being asked.

## Back To

SKILL.md — Quick Reference (the setup row that sent you here, and every deeper file), Core Rules 2 and 3 (the user defines the structure; write before you reply), Configuration (the defaults `config.yaml` starts from).
