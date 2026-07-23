# Setup — Design

Read this on first use to load user preferences. Do not interview the user.

## Your Attitude

Design quality is measurable: hierarchy, spacing, contrast, and alignment either pass their checks or they don't. Diagnose with the gates, fix with the rules, and explain each change by the rule it satisfies — never by taste alone.

## How To Load Preferences

1. Read `~/Clawic/data/design/config.yaml` if it exists. Apply its values.
2. For anything absent, use the defaults in the Configuration table of `SKILL.md` — do not ask.
   - `type_scale_ratio: 1.25`, `base_unit: 8`, `accessibility_target: AA`, `platform: web`, `brand_file: none`.
3. If `brand_file` is set, read `~/Clawic/data/design/brand.md` before proposing any palette or typeface — brand constraints override the derivation procedures in `palettes.md` and `fonts.md`.
4. Read `~/Clawic/data/design/memory.md` for prior context (projects, register, recurring feedback). Absence is fine; proceed without comment.

Work from defaults immediately. Never open with questions about style, tools, or how strict to be.

## Recording Preferences (only when the user declares one)

Write to config or memory **only** when the user states a preference in the course of the work — never as a preflight questionnaire.

- User names a scale ratio, base unit, accessibility target, platform, or supplies brand colors/fonts → update the matching key in `~/Clawic/data/design/config.yaml`; long brand material goes to `brand.md` with the path in config.
- User expresses a leaning (denser layouts, no animations, prefers punch-list critiques, RTL audience) → record it under the relevant preference area (output medium, aesthetic register, motion posture, language and script, critique style) in `~/Clawic/data/design/memory.md`.
- User corrects earlier guidance ("stop suggesting serifs") → update the stored value so you don't repeat it.

If the user has said nothing, store nothing.

## What Memory Holds

See `memory-template.md` for the file format. Track their projects and artifact types, aesthetic register, tools and delivery format, and feedback they repeat — but only from what they actually reveal.
