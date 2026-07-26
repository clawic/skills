# Setup — Rust

Read this on first use to load user preferences. Do not interview the user.

## Your Attitude

Rust rewards precision and punishes guessing. Give the fix and the reason it is the right rung, not a list of things to try. When the compiler is right — and it usually is — say what it is protecting against rather than routing around it. How much of that reasoning ships with the fix is `explanation_depth`; the fix itself never changes.

## How To Load Preferences

1. Read `~/Clawic/data/rust/config.yaml` if it exists. Apply its values.
2. For anything absent, use the defaults in the Configuration table of `SKILL.md` — do not ask.
   - `edition: 2024`, `msrv: none`, `async_runtime: tokio`, `error_style: auto`, `unsafe_policy: reviewed`, `lint_level: default`, `explanation_depth: standard`.
3. Read `~/Clawic/data/rust/memory.md` for prior context (their crates, stack, recurring pain points). Absence is fine; proceed without comment.

Work from defaults immediately. Never open with questions about their toolchain, their runtime, or how much detail they want.

## Recording Preferences (only when the user declares one)

Write to config or memory **only** when the user states a preference in the course of the work — never as a preflight questionnaire.

- User names an edition, MSRV floor, async runtime, error crate, unsafe policy, lint strictness, or how much explanation they want → update the matching key in `~/Clawic/data/rust/config.yaml`.
- User expresses a habit or stance (nextest over cargo test, module layout, a private registry or vendored sources, when clippy runs, appetite for new dependencies, target platform, how proactively to flag panics) → record it under the relevant preference area (tooling, conventions, platform, dependencies, registries and sources, work order and gates, output format, safety posture, cadence) in `~/Clawic/data/rust/memory.md`.
- User corrects earlier guidance → update the stored value so you do not repeat it.

If the user has said nothing, store nothing.

## What Memory Holds

Memory holds what you observed, never what the user declared — declared values live in `config.yaml`. Track the crates they work on, whether they write libraries or binaries or firmware, the parts of the language they keep hitting, and which explanations they read versus skip — but only from what they actually reveal.
