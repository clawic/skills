# Migration Guide - Proactivity

## v1.0.1 Architecture Update

This update keeps the same home folder, `~/Clawic/data/proactivity/`, and preserves existing files.
The new version adds active-state files for recovery and follow-through.

### Before

- `~/Clawic/data/proactivity/memory.md`
- `~/Clawic/data/proactivity/domains/`
- `~/Clawic/data/proactivity/patterns.md`
- `~/Clawic/data/proactivity/log.md`

### After

- `~/Clawic/data/proactivity/memory.md`
- `~/Clawic/data/proactivity/session-state.md`
- `~/Clawic/data/proactivity/heartbeat.md`
- `~/Clawic/data/proactivity/patterns.md`
- `~/Clawic/data/proactivity/log.md`
- `~/Clawic/data/proactivity/domains/`
- `~/Clawic/data/proactivity/memory/working-buffer.md`

## Safe Migration

1. Create the new files without deleting the old ones:
```bash
mkdir -p ~/Clawic/data/proactivity/memory
touch ~/Clawic/data/proactivity/session-state.md
touch ~/Clawic/data/proactivity/heartbeat.md
touch ~/Clawic/data/proactivity/memory/working-buffer.md
```

2. Keep `memory.md`, `patterns.md`, and `log.md` exactly as they are.

3. If old proactive rules live in free-form notes, copy them into the new sections in `memory.md`.

4. Start writing only live task state to session state and working buffer.

5. Do not delete or rename any legacy file unless the user explicitly asks for cleanup later.
