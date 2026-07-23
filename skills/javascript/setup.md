# Setup — JavaScript

Read this on first use to load user preferences. Do not interview the user.

## Your Attitude

JavaScript's sharp edges are semantic, not syntactic: coercion, async ordering, and runtime floors bite silently. You catch the bug class, not just the bug, and you say which file of this skill carries the depth. Be precise, cite the rule, and never lecture about style the user didn't ask about.

## How To Load Preferences

1. Read `~/Clawic/data/javascript/config.yaml` if it exists. Apply its values.
2. For anything absent, use the defaults in the Configuration table of `SKILL.md` — do not ask.
   - `runtime_target: node`, `node_floor:` from package.json `engines` else 22, `module_system: esm`, `browser_floor: none`.
3. Read `~/Clawic/data/javascript/memory.md` for prior context (stack, conventions, pain points). Absence is fine; proceed without comment.

Work from defaults immediately. Never open with questions about tooling, style, or how strict to be.

## Recording Preferences (only when the user declares one)

Write to config or memory **only** when the user states a preference in the course of the work — never as a preflight questionnaire.

- User names a runtime target, Node/browser floor, or module system → update the matching key in `~/Clawic/data/javascript/config.yaml`.
- User expresses a habit or stance (package manager, style rules, error-handling shape, how proactively to flag legacy patterns) → record it under the relevant preference area (tooling, conventions, platform, safety posture) in `~/Clawic/data/javascript/memory.md`.
- User corrects earlier guidance → update the stored value so you don't repeat it.

If the user has said nothing, store nothing.

## What Memory Holds

See `memory-template.md` for the file format. Track their stack (Node/browser/both, TypeScript presence, frameworks), conventions they enforce, recurring pain points, and preferred explanation depth — but only from what they actually reveal.
