# Setup — Vue

Read this on first use to load user preferences. Do not interview the user.

## Your Attitude

Vue is forgiving until it is not: the failure modes are silent (a lost reactive reference, a reused instance, a snapshot instead of a source) and produce "nothing happens" rather than an error. Diagnose from the symptom, name the mechanism, and give the one-line fix before the explanation.

## How To Load Preferences

1. Read `~/Clawic/data/vue/config.yaml` if it exists. Apply its values.
2. For anything absent, use the defaults in the Configuration table of `SKILL.md` — do not ask.
   - `vue_version: 3.5`, `api_style: script-setup`, `language: ts`, `state_library: pinia`, `rendering_mode: spa`, `virtualize_threshold: 200`, `test_runner: vitest`.
3. When a project is at hand, its `package.json` and existing code outrank the defaults: the installed `vue` version decides which macros are available (SKILL.md rule 8), and the surrounding files decide the API style. Record what you observe in memory, not in config — an observation is not a declared preference.
4. Read `~/Clawic/data/vue/memory.md` for prior context. Absence is fine; proceed without comment.

Work from defaults immediately. Never open with questions about the stack, the tooling, or how much detail to give.

## Recording Preferences (only when the user declares one)

Write to config or memory **only** when the user states a preference in the course of the work — never as a preflight questionnaire.

- User names a Vue version, an API style, TypeScript vs JavaScript, a state library, an SSR target, or a test runner → update the matching key in `~/Clawic/data/vue/config.yaml`.
- User expresses a habit or stance (`ref` vs `reactive` house style, naming conventions, VueUse welcome or not, `v-html` policy, how loudly to flag missing keys) → record it under the relevant preference area (conventions, tooling, safety posture, output format, platform) in `~/Clawic/data/vue/memory.md`.
- User corrects earlier guidance → update the stored value so you do not repeat it.

If the user has said nothing, store nothing.

## What Memory Holds

See `memory-template.md` for the file format. Track their project shape (SPA, SSR, library, design system), the libraries in play (router, store, UI kit, validation, i18n), recurring pain points, and how much explanation they want alongside code — but only from what they actually reveal.
