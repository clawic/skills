# Memory Template — Suno

`memory.md` holds what the agent OBSERVED; declared preferences (method, provider, plan, format, language) live in `config.yaml` (SKILL.md Configuration). An observation never overwrites a declared preference without user confirmation.

Create `~/Clawic/data/suno/memory.md` with this structure:

```markdown
# Suno Memory

## Status
status: ongoing
version: 1.0.0
last: YYYY-MM-DD

## Music Preferences (observed)
<!-- Genres they gravitate to -->
<!-- Typical mood/energy -->
<!-- Vocal textures they pick when shown both clips -->

## Successful Prompts
<!-- Exact style strings that landed — verbatim, never reworded -->
<!-- Format: "prompt" -> result description -->

## Rejected Directions
<!-- Styles/moods the user disliked — saves future rolls -->

## Projects
<!-- Ongoing music projects; details in projects/ -->

## Notes
<!-- Observations about their taste; things that worked or didn't -->

---
*Updated: YYYY-MM-DD*
```

## Status Values

| Value | Meaning |
|-------|---------|
| `ongoing` | Still learning their preferences |
| `complete` | Know their style well |
| `paused` | User said "not now" |

## Projects Folder

For users with multiple songs per effort, create `~/Clawic/data/suno/projects/<name>.md`:

```markdown
# Project: [Name]

## Concept
[What the project is about]

## Locked Style
[The verbatim style string + persona + Exclude Styles — the consistency source]

## Songs
- [Song 1]: prompt used, clip URL, plan tier at generation, status
- [Song 2]: prompt used, clip URL, plan tier at generation, status
```

The per-song plan-tier line doubles as the rights record.

## Songs Folder

Optional: `~/Clawic/data/suno/songs/` for downloaded audio, named `YYYY-MM-DD-<slugged-title>-vN.<ext>`.

## Principles

- Learn preferences through creation, not interrogation
- Save working prompts verbatim — rewording resets the odds (SKILL.md rule 2)
- Note reactions to both clips of a run, not just the chosen one
- Never store API keys in memory files
- Update `last` date on each session
