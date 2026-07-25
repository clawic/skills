---
name: voice-notes
slug: voice-notes
version: 1.0.2
description: Organize voice message transcripts into a structured, searchable knowledge base with tags, links, and progressive note-taking.
homepage: https://clawic.com/skills/voice-notes
metadata:
  clawdbot:
    emoji: 🎤
    requires:
      bins: []
    os:
    - linux
    - darwin
    - win32
    displayName: Voice Notes
---

## When to Use

User sends voice messages. The agent platform handles transcription (via its configured STT). This skill organizes the resulting transcripts into structured notes, links related content, and maintains a scalable tag-based system.

## Important: Transcription is Platform-Handled

This skill does NOT perform transcription. It expects the agent platform to:
1. Receive audio from the user
2. Transcribe it using the platform's configured STT (local or cloud)
3. Pass the transcript text to this skill for organization

The skill only organizes and stores text transcripts locally in `~/Clawic/data/voice-notes/`. Audio files are never accessed or stored by this skill.

## Architecture

All data stored in `~/Clawic/data/voice-notes/`. See `memory-template.md` for setup.

```
~/Clawic/data/voice-notes/
+-- memory.md           # HOT: tag registry + recent activity
+-- index.md            # Note index with tags and links
+-- transcripts/        # Raw transcriptions (text only)
+-- notes/              # Processed notes
+-- archive/            # Superseded content
```

## Quick Reference

| Topic | File |
|-------|------|
| Memory setup | `memory-template.md` |
| Note processing | `processing.md` |
| Linking system | `linking.md` |
| Tag management | `tags.md` |

## Data Storage

All data stored in `~/Clawic/data/voice-notes/`. Create on first use:
```bash
mkdir -p ~/Clawic/data/voice-notes/{transcripts,notes,archive}
```

## Scope

This skill ONLY:
- Receives transcript text from the agent platform
- Stores transcripts and notes in `~/Clawic/data/voice-notes/`
- Links related notes based on content
- Manages user-defined tags

This skill NEVER:
- Performs audio transcription (platform responsibility)
- Accesses audio files
- Deletes content without explicit user confirmation
- Accesses files outside `~/Clawic/data/voice-notes/`
- Sends data externally
- Requires API keys or credentials

## Self-Modification

This skill NEVER modifies its own SKILL.md.
All data stored in `~/Clawic/data/voice-notes/` files.

## Core Rules

### 1. Never Lose Information
| Event | Action |
|-------|--------|
| New transcript | Save immediately to `transcripts/` |
| Edit note | Preserve original in transcript reference |
| Strategy change | Archive old version, link to new |
| User deletes | Confirm first, then move to `archive/` |

### 2. Tag System Over Folders
- Tags defined in `~/Clawic/data/voice-notes/memory.md` under `## Tag Registry`
- User defines granularity (broad vs specific)
- Reuse existing tags before creating new
- Each note can have multiple tags

### 3. Detect Related Content
Before creating new note:
1. Search existing notes for topic overlap
2. If related -> append or link (not duplicate)
3. If continuation -> extend existing note
4. If contradicts -> link as evolution, preserve both

### 4. Document Scaling
When note exceeds ~100 lines:
1. Identify natural sections
2. Split into linked child notes
3. Parent becomes overview with links
4. Like Notion: notes contain notes

### 5. Progressive Disclosure
| Tier | When Loaded |
|------|-------------|
| `~/Clawic/data/voice-notes/memory.md` | Always (tags, recent) |
| `~/Clawic/data/voice-notes/index.md` | When searching |
| Individual notes | On demand |
| Transcripts | For verification only |

### 6. Reorganize Chaotic Input
User may speak stream-of-consciousness:
- Extract clear meaning
- Structure logically
- Preserve nuance (not over-condense)
- Group related points

### 7. First Session Setup
Ask user on first use:
- "Broad categories or detailed tags?"
- "Any existing topics to seed?"

## Common Traps

- Creating new note when should append -> always search first
- Losing tag consistency -> check registry before creating tags
- Over-condensing -> preserve user's intent and nuance
- Deleting "outdated" content -> archive, never delete
