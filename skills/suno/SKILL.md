---
name: suno
slug: suno
version: 1.0.2
description: >-
  Generates AI music with Suno: style prompts, structured lyrics, song extension, and API
  or browser workflows. Use when the user wants songs, jingles, or background tracks.
homepage: https://clawic.com/skills/suno
changelog: Deeper prompt engineering and workflow guidance
metadata:
  clawdbot:
    emoji: 🎵
    os:
    - linux
    - darwin
    - win32
    displayName: Suno
    configPaths:
    - ~/clawic/suno/
---

This skill stores all persistent data under `~/clawic/suno/` (preferences, project tracking, downloaded audio). If you have data at the old `~/suno/` location, move it to `~/clawic/suno/`.

## When To Use

- User wants a song, jingle, theme, or background track generated with Suno
- User has lyrics and wants them turned into audio
- User wants Suno prompts, style tags, or structured lyrics crafted for manual pasting
- User wants programmatic generation through a hosted Suno API
- Not for editing or mixing existing audio files — use `audio` or `ffmpeg`

## Quick Reference

| Situation | Read |
|-----------|------|
| First run, empty memory dir | `setup.md` |
| Writing or reviewing memory | `memory-template.md` |
| Generating via hosted API, polling, errors | `api.md` |
| Driving suno.com with a browser tool; credits and limits | `browser.md` |
| Crafting the style prompt | `prompts.md` |
| Picking style/genre/mood vocabulary | `styles.md` |
| Writing lyrics, structure tags, delivery control | `lyrics.md` |
| Anything else about generating with Suno | This file covers it |

## Setup

On first use (no `~/clawic/suno/` directory), read `setup.md`.

```
~/clawic/suno/
├── [memory.md]       # Created on first use: preferences, successful prompts
├── [projects/]       # Per-project song tracking
└── [songs/]          # Downloaded audio files
```

Structure in `memory-template.md`.

## Core Rules

### 1. Style Field Describes; Lyrics Field Gets Sung
Genre words typed into the lyrics box become sung words ("upbeat pop song" turns into the opening line). Sound goes in the style field, words in the lyrics field, structure in `[bracketed]` tags on their own lines.

### 2. Treat Each Generation as Sampling
One run returns two clips from the same prompt — two rolls, not a draft and a revision. When a roll lands, save its exact style string to `memory.md` and reuse it verbatim; rewording a working prompt resets the odds.

### 3. Pick the Method by Situation
| Situation | Method |
|-----------|--------|
| Deliver audio files programmatically | Hosted API (`api.md`) |
| No API key, browser tool available | Automate suno.com (`browser.md`) |
| User will paste into suno.com themselves | Craft prompt + lyrics only |
| Unclear (default) | Ask: "API key, browser automation, or just prompts?" |

### 4. Force Endings
Clips without an ending cue stop mid-phrase. Close the lyrics with an `[Outro]` section and `[End]` as the final tag; for instrumentals, `[Fade Out]` then `[End]`.

### 5. Long Songs Are Built, Not One-Shotted
1. Generate the strongest opening clip you can
2. Find where it degrades; extend FROM the last good moment (pick the timestamp), not from the raw end
3. Repeat until the `[End]`, then stitch with "Get Whole Song"

Target 2-4 minutes. Suno >=3.5 generates 4-minute clips and >=4.5 allows 8-minute songs, but one-shot epics drift in melody and mix — the extend loop is how coherent long tracks actually get made.

### 6. Translate Artist Names Into Attributes
Moderation rejects or strips artist and brand names — and the credits are still spent. Convert with three knobs: voice texture + era + production. "Like Springsteen" → "raspy heartland rock male vocals, 80s arena production, driving piano and saxophone".

### 7. API Pattern
Generate → poll every 5 seconds → download immediately (audio URLs expire). Generation runs 30-90 seconds. Working code in `api.md`.

## Prompt Essentials

Layered formula (full guide: `prompts.md`, vocabulary: `styles.md`):
```
[genre] [subgenre] [mood] [tempo] [instruments] [vocals] [era/influence]
```
Example: "indie folk melancholic slow acoustic guitar soft female vocals 90s"

- Front-load: earlier terms steer harder in practice — genre first, garnish last.
- Specificity beats quantity: "shoegaze" outsteers "rock, reverb, dreamy, atmospheric".
- Stay within 8-12 style terms (`styles.md`).
- Instrumental: set the instrumental toggle AND leave the lyrics box empty; stray text gets sung.

## Output Gates

Before submitting any generation, check:
- No artist, band, or brand names in prompt or lyrics?
- Everything that should not be sung is inside `[brackets]` on its own line?
- Ending cue present (`[End]` last) for a standalone song?
- Style terms 8-12, no contradictions (lo-fi + polished, happy + mournful)?
- Instrumental toggle matches the request?
- Commercial deliverable → user confirmed they are on a paid Suno plan?

## Traps

| Trap | Why it fails | Do instead |
|------|--------------|------------|
| Artist/brand names in prompt | Rejected or stripped by moderation; credits spent | Voice + era + production attributes (rule 6) |
| Free-plan track for commercial use | Rights attach at generation time under the plan then active; free-tier songs are non-commercial under Suno's terms | Confirm paid plan before generating deliverables; check current terms |
| Regenerating a clip that was 90% right | Discards the roll that worked | Crop the bad part, extend from the last good bar |
| Lyrics far over ~300 words for a ~3-minute song | Rushed, half-spoken delivery | Word budget in `lyrics.md` |
| Different chorus wording on each repeat | Each variant gets its own melody | Paste identical chorus text every time |
| Negations in the style field ("no drums") | Style terms act as attractors; the noun still pulls | Omit the term, or use Suno's Exclude Styles field |

## Security & Privacy

**Data storage.** This skill creates `~/clawic/suno/` on first use:
- **memory file** — Preferences, successful prompts
- **projects folder** — Per-project tracking
- **songs folder** — Downloaded audio (optional)

All data stays local. API keys live in environment variables, never in files.

**This skill does:**
- Generate music via hosted APIs (requires API key from provider)
- Navigate suno.com with browser automation
- Craft optimized prompts for Suno's model
- Write lyrics with proper structure tags
- Track projects and successful patterns locally

**This skill does NOT:**
- Store API keys in plain text files
- Access files outside `~/clawic/suno/`
- Make requests without user direction

**External endpoints.** When using hosted APIs, requests go to:

| Endpoint | Data Sent | Purpose |
|----------|-----------|---------|
| api.aimusicapi.ai | Prompts, lyrics | Music generation |
| api.evolink.ai | Prompts, lyrics | Music generation |
| suno.com | Browser session | Direct platform access |

API keys authenticate requests; prompts and lyrics are sent for processing.

**Guardrails.** By using this skill with APIs, prompts and lyrics are sent to third-party services for music generation. Only use services you trust with your creative content.

## Related Skills
More Clawic skills, get them at https://clawic.com/skills (install if the user confirms):
- `audio` — Audio processing and editing
- `video` — Combine music with video content
- `ffmpeg` — Audio format conversion

## Feedback

- If useful, star it: https://clawic.com/skills/suno
- Latest version: https://clawic.com/skills/suno

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/suno.
