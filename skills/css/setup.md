# Setup — CSS

Read this on first use to load user preferences. Do not interview the user.

## Your Attitude

CSS fails without errors: the rule parses, nothing changes, and the developer adds three more properties. You name the mechanism first and write the smallest declaration that fixes it. Modern features over hacks, accessibility floor never negotiable, no `!important` smuggled in as a shortcut.

## How To Load Preferences

1. Read `~/Clawic/data/css/config.yaml` if it exists. Apply its values.
2. For anything absent, use the defaults in the Configuration table of `SKILL.md` — do not ask.
   - `authoring_mode: plain-css`, `browser_support: evergreen`, `naming_convention: none`, `rem_base: 16`, `a11y_target: aa`, `explanation_depth: mechanism`, `output_shape: diff`.
3. Read `~/Clawic/data/css/memory.md` for prior context (their stack, recurring bugs, design tokens in play). Absence is fine; proceed without comment.
4. Infer from the codebase before asking: an existing `tailwind.config`, `.scss` files, or a tokens file answers `authoring_mode` and `naming_convention` without a question. What you infer from files is memory, not declared config.

Work from defaults immediately. Never open with questions about frameworks, browser targets, or how much accessibility detail they want.

## Recording Preferences (only when the user declares one)

Write to config or memory **only** when the user states a preference in the course of the work — never as a preflight questionnaire.

- User names an authoring flavor, a support target, a naming convention, a root font-size, or an accessibility level → update the matching key in `~/Clawic/data/css/config.yaml`.
- User asks for less theory ("just give me the fix") or for the whole file instead of the changed lines → that is `explanation_depth` / `output_shape`, declared config, not an observation: write the key immediately.
- User expresses a habit or stance (nesting depth, longhand vs shorthand, comment density, appetite for Chromium-first features, banned techniques, which surfaces they ship to) → record it under the relevant preference area (tooling, conventions, platform, risk posture, constraints, output) in `~/Clawic/data/css/memory.md`.
- User corrects earlier guidance → update the stored value so you don't repeat it.

If the user has said nothing, store nothing.

## What Memory Holds

Track their stack and build chain, the design tokens or scale in use, recurring bug classes (stacking contexts, layout shift, Safari differences), and the components that keep coming back — but only from what they actually reveal. Explanation depth and deliverable shape are declared preferences: they live in `config.yaml`, not here.
