# Migration Guide - Productivity

## v1.0.4 Productivity Operating System Update

This update keeps the same home folder, `~/Clawic/data/productivity/`, but changes the recommended structure from a light memory-only setup into a fuller operating system with named folders for inbox, goals, projects, tasks, habits, planning, reviews, commitments, focus, routines, and someday items.

### Before

- `~/Clawic/data/productivity/memory.md`
- optional loose notes such as `~/Clawic/data/productivity/<topic>.md`
- older installs may also have copied context guides in a flat layout

### After

- `~/Clawic/data/productivity/memory.md`
- `~/Clawic/data/productivity/inbox/`
- `~/Clawic/data/productivity/dashboard.md`
- `~/Clawic/data/productivity/goals/`
- `~/Clawic/data/productivity/projects/`
- `~/Clawic/data/productivity/tasks/`
- `~/Clawic/data/productivity/habits/`
- `~/Clawic/data/productivity/planning/`
- `~/Clawic/data/productivity/reviews/`
- `~/Clawic/data/productivity/commitments/`
- `~/Clawic/data/productivity/focus/`
- `~/Clawic/data/productivity/routines/`
- `~/Clawic/data/productivity/someday/`
- any old loose notes preserved until the user chooses to merge or archive them

## Safe Migration

1. Check whether `~/Clawic/data/productivity/` already exists.

2. If it exists, keep `memory.md` exactly as it is.

3. Create the new files without deleting the old ones:

```bash
mkdir -p ~/Clawic/data/productivity/{inbox,goals,projects,tasks,habits,planning,reviews,commitments,focus,routines,someday}
touch ~/Clawic/data/productivity/inbox/capture.md
touch ~/Clawic/data/productivity/inbox/triage.md
touch ~/Clawic/data/productivity/dashboard.md
touch ~/Clawic/data/productivity/goals/active.md
touch ~/Clawic/data/productivity/goals/someday.md
touch ~/Clawic/data/productivity/projects/active.md
touch ~/Clawic/data/productivity/projects/waiting.md
touch ~/Clawic/data/productivity/tasks/next-actions.md
touch ~/Clawic/data/productivity/tasks/this-week.md
touch ~/Clawic/data/productivity/tasks/waiting.md
touch ~/Clawic/data/productivity/tasks/done.md
touch ~/Clawic/data/productivity/habits/active.md
touch ~/Clawic/data/productivity/habits/friction.md
touch ~/Clawic/data/productivity/planning/daily.md
touch ~/Clawic/data/productivity/planning/weekly.md
touch ~/Clawic/data/productivity/planning/focus-blocks.md
touch ~/Clawic/data/productivity/reviews/weekly.md
touch ~/Clawic/data/productivity/reviews/monthly.md
touch ~/Clawic/data/productivity/commitments/promises.md
touch ~/Clawic/data/productivity/commitments/delegated.md
touch ~/Clawic/data/productivity/focus/sessions.md
touch ~/Clawic/data/productivity/focus/distractions.md
touch ~/Clawic/data/productivity/routines/morning.md
touch ~/Clawic/data/productivity/routines/shutdown.md
touch ~/Clawic/data/productivity/someday/ideas.md
```

4. If the user has older free-form topic files in `~/Clawic/data/productivity/`, map them gradually:
   - current priorities -> `dashboard.md`
   - goals -> `goals/active.md`
   - projects -> `projects/active.md`
   - actionable work -> `tasks/next-actions.md`
   - habits and routines -> `habits/active.md`
   - focus notes -> `focus/sessions.md` or `focus/distractions.md`
   - weekly reset notes -> `reviews/weekly.md`
   - parked ideas -> `someday/ideas.md`

5. If older copied guide files exist in a flat layout, preserve them as legacy references. Do not delete or rename them automatically.

6. Only clean up legacy files after the user confirms the new structure is working.

## Post-Migration Check

- `memory.md` still contains the user's saved preferences
- active priorities are visible in `dashboard.md`
- next actions live in `tasks/next-actions.md`
- review cadence is captured in `reviews/weekly.md`
- no legacy file was deleted without explicit user approval
