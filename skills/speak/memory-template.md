# Memory Template — Speak

Create `~/Clawic/data/speak/preferences.md` with this structure:

```markdown
# Speak Preferences

## Status
status: ongoing
last: YYYY-MM-DD

## Voice
<!-- one line per persona: provider: voice-id (model/version if pinnable) -->

## Rate
<!-- baseline + evidence dates: 1.15 (asked "faster" 2026-07-12, 2026-07-19) -->

## Lexicon
<!-- one line per fix: token -> spoken form (engine if engine-specific, date) -->
<!-- Nginx -> "engine x" (all engines, 2026-07-12) -->

## Style
<!-- confirmed two-signal styles: "no chunk check-ins during briefings" -->

## Avoid
<!-- what did NOT work: "no long monologues", "skip parentheticals" -->

## Contexts
<!-- qualifier: rule — "driving: two sentences max", "work calls: formal", "quiet hours: 22-08" -->

## Engine Notes
<!-- SSML tier per engine (ssml.md test results), character limits hit, fallback order -->

---
*Updated: YYYY-MM-DD*
```

## Status Values

| Value | Meaning |
|-------|---------|
| `ongoing` | Still learning their ear |
| `complete` | Voice, rate, and lexicon stable |

## Entry Rules

- Evidence dates on every stored line — a later session must be able to tell preference from mood (SKILL.md rule 7).
- Lexicon and language lines store on one signal; every other section needs two (SKILL.md Preference Memory).
- Declared values live in `config.yaml`, not here; this file is what the agent observed and confirmed. On conflict, the declared value wins.
- Group similar lines; keep each one line. This file is read at every voice session start — its length is a latency tax.
