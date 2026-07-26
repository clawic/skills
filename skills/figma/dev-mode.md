# Dev Mode and Handoff

The deliverable is a file an engineer can read without asking questions, not a link and a hope. Dev Mode is the surface; the work is making the tree tell the truth.

## What Dev Mode Gives

- Measurements, spacing, and computed values on selection, plus distance-to-selection on hover.
- Code snippets per node in web, iOS, and Android flavors, with variable names substituted where code syntax is set.
- Ready-for-dev status on frames and sections, so engineers see scope rather than every exploration on the page.
- A compare-changes view showing what moved between versions or branches.
- Annotations pinned to nodes, and dev resource links (Storybook story, Jira ticket, GitHub component).

Dev Mode access is seat-gated — a Dev or Full seat per person. If engineers do not have the seat, the deliverable is a written spec page plus exported assets, not a Figma link.

## Making the Tree Honest

The number-one handoff failure is a frame that looks right and is built wrong: absolute positioning where auto layout belongs. The engineer copies the visible layout and ships something that snaps on resize.

Before marking Ready for dev:

- Drag-sweep every frame across the supported width range, then paste the longest realistic string.
- Zero layers named `Frame N`, `Group N`, `Rectangle N`. Those names reach code verbatim.
- Fills, spacing, and radii bound to variables with code syntax set — the snippet then shows `--color-bg-accent`, not `#2563EB`.
- Sad paths present as frames, not as a note: empty, loading, error, disabled, missing image.
- Sections grouped by feature, so engineers navigate by feature rather than by screenshot order.

## Code Connect

Code Connect maps a Figma component to the real component in the codebase, so Dev Mode's snippet shows the actual import and props instead of generated divs.

- Setup lives in the repo: a config plus one file per component, published with the Code Connect CLI from CI.
- The payoff is proportional to component reuse. Map the twenty components that appear on every screen first; the long tail rarely pays back.
- It also converts prop names, so a Figma `Size = lg` variant renders as `size="lg"` — the naming discipline in Figma becomes API discipline in code.
- Unmapped components still generate generic markup. A partially mapped system is legible: mapped components read as real calls, unmapped ones visibly do not.

## Codegen Is a Draft

- Trust the structure: auto layout maps to flexbox, hierarchy maps to nesting, variables map to token names.
- Do not trust raw px values, absolutely-positioned geometry, or generated class names. They are a snapshot of one viewport width.
- Custom codegen plugins can emit the team's own component syntax; that is worth building only once the component names are stable.

## The Dev Mode MCP Server

Figma exposes a Dev Mode MCP server so an agent or IDE reads the file's nodes, variables, and component metadata directly instead of receiving pasted screenshots.

- Scope every request to a selected node or frame. Pulling a whole page returns a node tree large enough to swamp any context window.
- Ask for variables and component names alongside geometry — that is what makes generated code match the design system instead of hardcoding values.
- It is recent and evolving: verify the current setup and endpoint names before wiring a pipeline around it, and treat access through it with the same care as any token: it can read the whole document, not just the selection.

## Spec What Code Cannot Infer

A pixel-perfect file still leaves these undefined. Annotate them or the engineer guesses:

| Missing from the visual | Annotate |
|---|---|
| Keyboard behavior | Focus order, focus-visible treatment, escape and enter handling |
| Semantics | Roles, accessible names, heading levels, landmark structure |
| Error handling | Which validation fires when, exact error copy, where the message attaches |
| Loading | Skeleton vs spinner, threshold before showing it, what stays interactive |
| Motion | Duration, easing, what animates and what does not, reduced-motion fallback |
| Breakpoints | Which layout applies at which width, and what rearranges rather than resizes |
| Content limits | Max characters per field, truncation vs wrap, pluralization |
| Data states | Empty vs zero vs error vs unauthorized — they look similar and behave differently |

## Handoff Rituals That Reduce Rounds

- Walk the file live once, for fifteen minutes, before async review. Every question asked in that walkthrough is a missing annotation.
- Keep one canonical page. Explorations move to `Archive` before the walkthrough, not after.
- Use the compare-changes view in review instead of describing edits in prose.
- Redlines exported as PDFs are dead; Dev Mode measurements replaced them. Time spent annotating spacing by hand is time not spent speccing behavior.

## Symptom → Cause

| Symptom | Cause | Fix |
|---|---|---|
| "It breaks on resize" | Absolute positioning instead of auto layout | Rebuild with Fill and clamps; drag-sweep before redelivery |
| Snippets show hex values, not tokens | Variables unbound, or code syntax not set | Bind fills to variables; add per-platform code syntax |
| Engineers rebuild existing components | No Code Connect mapping, no doc links | Map the top twenty components; add doc links to descriptions |
| Constant "which state is this" questions | Sad paths missing | Add empty, loading, error, disabled frames |
| Engineers cannot find the current screens | Explorations mixed with delivery | Ready-for-dev on sections; archive explorations |
| MCP or codegen output floods the context | Whole-page fetch | Scope to a selected node |
| Same accessibility question every sprint | Behavior never annotated | Annotate focus order, roles, and error copy in the file |
| Handoff diff is described in a paragraph | Compare-changes not used | Send the compare view instead |
