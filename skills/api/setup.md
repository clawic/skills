# Setup — API

## Lookup Procedure

When a user asks about an API, in this order:

1. **Locate the category** — API Categories table in SKILL.md. Service not listed → say so, link its official docs; don't guess endpoints from memory (they drift).
2. **Read the index only** — `head -20` on the category file gives API name → line number.
3. **Read just that section** — `sed -n 'START,ENDp'`; sections are 50-100 lines. Reading the whole 500-1400 line file wastes context.
4. **Answer with the section's Common Traps included** — the traps are why this reference exists; an answer with only the endpoint repeats what official docs already do.
5. **Version-sensitive claims** (current pricing, current rate limits, model names) — the reference may lag; point to the Official Docs link at the end of the section for anything the user will bill against.

## User Preferences

Optional file at `~/clawic/api/preferences.md`. If you have data at the old `~/api/` location, move it to `~/clawic/api/`. Create only when the user expresses a lasting preference (asked for Python examples twice, always uses the same account):

```markdown
# API Preferences

## Code Examples
- Language: curl (or python, javascript)

## Common APIs
- stripe
- openai

## Default Accounts
- stripe: STRIPE_TEST_API_KEY (never prod without asking)
```

Check this file before answering; a stored language preference beats defaulting to curl.

## What This Skill Does Not Do

- Store or manage API keys
- Make API calls automatically
- Access external services
