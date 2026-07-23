# Setup — Suno

Read this when `~/Clawic/data/suno/` doesn't exist or is empty.

## Create the Workspace

```
~/Clawic/data/suno/
├── config.yaml       # Declared preferences — variables in SKILL.md Configuration
├── memory.md         # Observed preferences and successful prompts
├── projects/         # Per-project song tracking (create when a second song shares a project)
└── songs/            # Downloaded audio (optional)
```

## Determine the Method From Context, Not Questions

Resolve `default_method` without interrogating:

1. `AIMUSICAPI_KEY` or `EVOLINK_API_KEY` present in the environment → `api`
2. Otherwise, a browser automation tool is available and the user wants delivered audio → `browser`
3. Otherwise → `prompts` (craft prompt + lyrics for manual pasting)

Record the result in `config.yaml`. Ask at most one question, only if genuinely blocked: the user wants audio files delivered but neither an API key nor a browser tool exists — ask which they can provide, then record the answer.

## First Song

- Pull genre, mood, and purpose from the request itself; draft with the layered style formula (SKILL.md Prompt Essentials) and defaults over questions.
- Vocals unstated → assume vocals unless the use case implies a bed (podcast intro, background loop → instrumental).
- Deliver, then record: what the user reacted to goes to `memory.md` (observed); anything they stated outright ("I always want instrumental") goes to `config.yaml` (declared).

## API Keys

If the user has API access, keys live in environment variables — never in files under `~/Clawic/data/suno/`:

```bash
export AIMUSICAPI_KEY="their-key"     # aimusicapi.ai
export EVOLINK_API_KEY="their-key"    # evolink.ai
```

`config.yaml` and `memory.md` may note which provider and env var name are in use — never the value.

## Done When

Method resolved and recorded, first audio (or prompt package) delivered, and any stated preferences persisted. Future requests start from `config.yaml` + `memory.md`, not from zero.
