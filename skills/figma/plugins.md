# Plugins — Choosing, Trusting, and Writing

A plugin is third-party code running with read access to the open file. Both halves of that sentence matter: it is the only way to bulk-edit a Figma document, and it is a data-exfiltration surface.

## When a Plugin Is Worth It

- The action repeats 10 or more times, or repeats on a schedule. Below that, doing it by hand is faster than finding, vetting, and learning a plugin.
- The operation is mechanical and verifiable: rename, restyle, find, replace, count, audit. Judgment work does not automate well.
- Read-only audits that do not need to write to the file belong on the REST API instead, which runs outside the editor and can cover many files at once.

## Trust and Permissions

- A plugin can read the entire open file, including pages you are not looking at, and can call external services if its manifest declares network access. The manifest's declared domains are visible before install — read them.
- Vet on: who publishes it, how many installs, when it was last updated, what domains it contacts, and whether it needs write access at all.
- Never run an unvetted plugin on a file containing unreleased product work, customer data in mocks, or credentials pasted into a sticky.
- Organizations can allowlist plugins and publish private org-only plugins. On a regulated file, an allowlist is the control; "we told people to be careful" is not.
- A plugin that asks for network access to do something purely local is the clearest red flag available.

## A Curated Toolkit

Keep the list short — every installed plugin is a search-result competitor and a trust decision.

| Need | Category |
|---|---|
| Icons | An icon-set browser, with a curated import policy: pull the fifty icons the product uses, never the full set |
| Realistic content | A content generator for names, dates, avatars and copy at realistic lengths — replaces `Lorem ipsum` |
| Token pipeline | A token plugin that exports variables to JSON and syncs to a repo, feeding a platform-code transformer |
| Accessibility | A contrast and simulation plugin, plus an annotation kit for roles and focus order |
| Cleanup | An audit plugin for detached instances, unused components, and stray styles |
| Figma-to-code | Scaffolding tools — useful for a starting point, never for production output |
| User testing | A prototype-testing service that records unmoderated sessions |

## Writing Your Own

The plugin runtime is a sandbox around the document plus an optional UI iframe.

- **Manifest**: `editorType` declares which editors it runs in (design file, FigJam, and other surfaces are separate). `networkAccess` must declare allowed domains — an empty allowlist is the honest default for a local-only plugin.
- **Document access**: modern plugins load pages on demand. Nodes outside the current page are not available synchronously; use the async page-loading API before walking the whole document, and use the async node getters rather than the deprecated synchronous ones.
- **No network in the sandbox**: the plugin code itself cannot fetch. The UI iframe can. The pattern is a `postMessage` bridge — sandbox asks the UI to fetch, UI posts the result back.
- **Performance**: set the flag that skips invisible instance children before traversing, batch reads before writes, and avoid awaiting per node in a loop over thousands of nodes. A naive traversal of a large file hangs the editor.
- **Fonts**: text edits require loading the font first, and a plugin that renames text without loading fonts fails on exactly the nodes that matter.
- **Undo**: group mutations so one Cmd+Z reverses the whole operation, or users will not trust the plugin twice.

Recipes worth building in-house rather than installing:

- Bulk rename against a naming convention the team owns.
- Find every detached instance and report by page.
- Audit every text node for hardcoded colors and unbound spacing.
- Swap a deprecated component for its replacement across the file, preserving overrides.

## Plugins vs Widgets

- Plugins run for one person, on demand, and close when done.
- Widgets live on the canvas and are visible to everyone in the file — voting, checklists, trackers, ticket embeds. They persist state in the document.
- Use a widget when the output is shared context; a plugin when the output is a change to the file.

## Symptom → Cause

| Symptom | Cause | Fix |
|---|---|---|
| Plugin hangs the editor on a big file | Synchronous traversal of every node | Skip invisible instance children; batch; use async getters |
| Plugin cannot see other pages | Dynamic page loading | Load pages explicitly before traversing |
| Text edits fail silently | Font not loaded before the edit | Load the font, then write |
| Fetch fails inside the plugin | Sandbox has no network | Move the fetch to the UI iframe and bridge with postMessage |
| One operation takes twenty undos | Mutations not grouped | Group the mutation into a single undo step |
| Plugin asks for unexpected domains | Network access beyond its function | Do not install; find an alternative |
| A vetted plugin broke after an update | Plugins auto-update | Pin critical workflows to a private org plugin you control |
| Team has 40 plugins installed | No curation policy | Allowlist; delete the rest |
