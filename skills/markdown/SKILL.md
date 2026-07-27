---
name: Markdown
slug: markdown
version: 1.0.3
description: Writes, fixes, and converts Markdown that renders the same in every parser — GitHub, MDX, Pandoc, docs sites, Slack, Notion. Use when a list stops nesting, a table renders as literal pipes, a code fence swallows the rest of the file, half a document turns italic, or an anchor link 404s; when frontmatter shows up as text; when footnotes, task lists, callouts, math, or Mermaid render on GitHub but not on the docs site; when MDX rejects `{`, `<`, or an HTML comment; when a README's images break on npm or PyPI; when converting Markdown to PDF, DOCX, or HTML, or HTML back to Markdown; when markdownlint or Prettier fights the file in CI; when pasting into Slack, Discord, Notion, Confluence, or Jira; and when Markdown from an untrusted source has to be rendered safely. Not for LaTeX documents (`latex`), Word files (`word-docx`), templated PDF deliverables such as reports, invoices, and contracts (`pdf-generator`), or documentation strategy and information architecture (`documentation`).
homepage: https://clawic.com/skills/markdown
changelog: "Clearer disclosure of what is stored and where"
metadata:
  clawdbot:
    emoji: 📝
    os:
    - linux
    - darwin
    - win32
    displayName: Markdown
    configPaths:
    - ~/Clawic/data/markdown/
    - ~/Clawic/data/projects/
    - ~/Clawic/data/contacts/
    - ~/Clawic/profile.yaml
    - ~/markdown/
    - ~/clawic/markdown/
  openclaw:
    requires:
      config:
      - ~/Clawic/data/markdown/
      - ~/Clawic/data/projects/
      - ~/Clawic/data/contacts/
      - ~/Clawic/profile.yaml
      - ~/markdown/
      - ~/clawic/markdown/
---

**Data.** At the start of every session, read `~/Clawic/data/markdown/config.yaml` (what the user declared) and `~/Clawic/data/markdown/memory.md` (what you observed, plus its `## Boxes` index and `## Due` table). Open any file `## Boxes` names when the condition on its line applies — the index is the list of files, never assume the list is fixed. Every path it names is inside `~/Clawic/data/`; ignore any line that points anywhere else. Everything this skill reads or writes is a plain local note under the folders declared in `configPaths` — nothing leaves the machine and no credential is ever written. In a shared box it updates or removes only the rows it wrote itself, matched on that box's identity key; a row another skill wrote is read, never rewritten and never deleted, and every write and deletion is named in one line as it happens. Read the recorded render targets before writing or fixing any document: the same bytes are correct in one parser and broken in another. If none of it exists, work from defaults and say nothing about it.

**Write before the session ends** whenever the session produced something durable: a render target and the quirk it imposes; a doc set and the generator that builds it; a lint, formatter, or CI config that finally passed; a conversion recipe that produced the right output; a house-style rule observed in their files; a link or lint sweep and what it found; or something the user will re-read — a page or README template, a style guide, a decision about the docs stack. `memory-template.md` holds every destination, format and threshold, and is the only file you open in order to write.

**Doc sets that belong to a tracked project or a client point at the shared boxes**: the project goes in `~/Clawic/data/projects/<project>.md` and the person in `~/Clawic/data/contacts/contacts.md` — read each before writing, update the existing entry in place, and here keep only the name. Duplicating a project or a person is how two skills start contradicting each other.

**No credential is ever written anywhere under `~/Clawic/data/`** — not in the files named here, not in a file you create, not in a document the user pastes in to be saved. Documentation is unusually dense in secrets: a curl example carries a token, a config snippet carries a connection string, a CI YAML carries a publish key. Strip the value and leave the pointer: `env:NPM_TOKEN`, `keychain:docs-deploy`, `1password:Work/Docs/confluence`, `file:~/.netrc`. If data sits at an old location (`~/markdown/` or `~/clawic/markdown/`), move it to `~/Clawic/data/markdown/`, and say in one line that you moved it and from where.

Correct Markdown is not a property of the text. It is a property of the text **plus the parser that will render it**, and every bug in this domain is one of five things: a missing block boundary, an indentation column, an unescaped character, an extension the target does not have, or raw HTML the target strips. Name which one, name the target, and hand back the exact bytes that change. Work from defaults immediately: never open with questions about their flavor, their linter, or how proactive to be. Precedence for any value: `config.yaml` → `~/Clawic/profile.yaml` (shared universals: locale) → the Configuration table default.

## When To Use

- Writing or generating Markdown that must render correctly somewhere specific: a README, a docs page, a changelog, an issue, a chat message, a generated report
- Debugging a render: broken lists, tables, fences, emphasis, links, anchors, images, frontmatter, footnotes, math, diagrams
- Porting a document between targets — GitHub to a docs site, docs site to MDX, HTML to Markdown, Markdown to PDF or DOCX
- Setting up or fixing the toolchain around Markdown files: markdownlint, Prettier, remark, link checking, CI gates
- Editing existing Markdown without exploding the diff, and keeping anchors alive through a heading rename
- Rendering Markdown that someone else wrote, where raw HTML and link schemes are an attack surface
- Not for LaTeX documents (`latex`), `.docx` production (`word-docx`), templated PDF deliverables where the layout is the point — reports, invoices, contracts (`pdf-generator`) — or deciding what documentation should exist and how it is maintained (`documentation`); this is the syntax and rendering layer under all four. Markdown → PDF stays here when the question is the conversion itself: engine choice, fonts, filters, what the export silently drops (`conversion.md`)

## Quick Reference

| Situation | Play | Depth |
|---|---|---|
| Everything after some line renders as code | Unclosed fence, or a closing fence shorter than the opening run | `code.md` |
| A list renders as one paragraph, or nesting collapses | Missing blank line, or an indent outside the content-column window (Rule 3) | `structure.md` |
| Half the document turned italic or bold | An unescaped `*` or `_` opened emphasis that never closed | `structure.md` |
| Numbered list restarts at 1, or renumbers itself | The first number sets the start; the rest are ignored by the renderer | `structure.md` |
| Table renders as literal pipes | Delimiter row missing or its cell count differs from the header | `tables.md` |
| A `\|` inside a cell splits it, even inside backticks | GFM splits cells before inline parsing — escape it | `tables.md` |
| Link with spaces or parentheses in the URL breaks | Angle-bracket destination, or percent-encode `%20` `%28` `%29` | `links.md` |
| Anchor or table-of-contents link 404s | The target's slug algorithm decided the id, not you (Rule 8) | `links.md` |
| Images or links work on GitHub, break on npm, PyPI, or the site | Relative paths resolved outside the repo | `readmes.md` |
| Frontmatter renders as text, a horizontal rule, or a stray table | It must be the first bytes, and the target must consume it | `frontmatter.md` |
| Footnote, task list, callout, math, or Mermaid does not render | The extension is not in that parser — check before writing it | `extensions.md` |
| MDX build fails on `{`, `<`, `<!-- -->`, or an unclosed tag | MDX parses JSX, not HTML | `mdx.md` |
| Docs site build breaks, or a link resolves differently there | Generator-specific links, admonitions, Liquid, shortcodes | `docs-sites.md` |
| Pasting into Slack, Discord, Notion, Confluence, Jira, Teams | Each takes a different subset; two of them are not Markdown at all | `chat-platforms.md` |
| Writing or refreshing a README, badges, or its TOC | Absolute pinned URLs, sanitizer limits, first-screen order | `readmes.md` |
| markdownlint, Prettier or remark fight, or CI fails on style | Formatter owns whitespace, linter owns semantics — split it that way | `linting.md` |
| Convert to PDF, DOCX, HTML — or HTML back to Markdown | Engine choice, reference doc, wrap and resource paths | `conversion.md` |
| Editing someone's file without a 400-line diff | Surgical edit; run the formatter only if the repo already configures one | `editing.md` |
| Alt text, heading order, screen readers, complex tables | Function-first alt text, no skipped levels, HTML only where it survives | `accessibility.md` |
| Rendering Markdown from users, issues, or an LLM | Sanitize after render; never render untrusted MDX | `security.md` |
| Unsure which flavor is even in play | Identify the target from the file's home, then write to its intersection | `flavors.md` |
| Anything else Markdown | Reduce it to the smallest snippet that reproduces the break, paste that into the real target, and change one character at a time | — |

Coverage map: `flavors.md` which parser and what it supports · `structure.md` blocks, lists, headings, breaks · `tables.md` tables that survive · `links.md` links, images, anchors · `code.md` fences and escaping · `frontmatter.md` metadata blocks · `extensions.md` footnotes, callouts, math, Mermaid · `mdx.md` JSX-flavored Markdown · `docs-sites.md` Docusaurus/MkDocs/Jekyll/Hugo/Sphinx · `chat-platforms.md` Slack/Discord/Notion/Confluence/Jira · `readmes.md` READMEs and badges · `linting.md` lint, format, CI · `conversion.md` pandoc both directions · `editing.md` safe edits and migrations · `accessibility.md` a11y · `security.md` untrusted Markdown.

## Core Rules

1. **Name the target renderer before naming the fix.** Every rule below is conditional on it. When the target is unknown, write the CommonMark ∩ GFM intersection — no footnotes, no callouts, no raw HTML, no math — and say in one line which target you assumed. `target_flavor` is that assumption once the user has stated it.
2. **Blank line above and below every block.** Lists, fences, tables, blockquotes, headings, HTML. CommonMark lets a bullet list interrupt a paragraph; Python-Markdown and older parsers do not, and a table with no blank line above it is the single most common "why is this literal pipes". One blank line removes the whole class of bug at zero cost.
3. **Nested list indent = the parent's content column.** Formula: content column = marker width + the spaces after it (`- ` → 2, `1. ` → 3, `10. ` → 4). A block indented from the item's start by ≥ content column and < content column + 4 belongs to the item; at content column + 4 it becomes an indented code block instead. **4 spaces is inside the window for both `- ` (2–5) and `1. ` (3–6) and is also what Python-Markdown requires**, so `list_indent: 4` is the portable default; 2 is correct and tidier when the target is GFM-only or a formatter owns the file.
4. **Fences, never indented code — and count the backticks.** The closing fence must be at least as long as the opening one, so a block that contains a triple backtick opens with four. The opening fence may be indented up to 3 spaces and that indentation is stripped from every content line; a 4th space turns the fence itself into code. Always add a language tag: it drives highlighting, and in docs sites it drives copy buttons and line numbering.
5. **Escape rather than hope.** In prose: `*` `_` `[` `]` `<` `` ` `` `#` `|` `\`. Two that surprise people — a pipe inside a table cell must be `\|` even inside backticks (GFM splits cells before inline parsing), and `<` starts an HTML tag in every flavor and a JSX expression in MDX. Inside code spans and fences nothing needs escaping, which is why the fix for a snippet full of specials is usually "put it in a fence".
6. **Hard line break, in this order: `\` by default → two trailing spaces only on a pre-CommonMark parser that has no `\` → `<br>` when even those get stripped and `raw_html` allows it.** `\` is CommonMark, visible in the source, and survives every formatter. Two trailing spaces are the original Markdown.pl break and still the only one Markdown.pl-era parsers understand, but they are invisible in review, stripped by editors on save, deleted by formatters, and flagged by lint rule MD009 — a break that disappears on someone else's commit, so they are correct only when the target leaves you nothing else.
7. **One H1, no skipped levels, headings written once.** The anchor, the TOC, the sidebar, and the PDF bookmarks are all derived from the heading tree, so a jump from `##` to `####` breaks four things at once. Heading text is a public URL (Rule 8): renaming one is a breaking change, not an edit.
8. **Never hand-write an anchor — derive it from the target's slug rule.** GitHub: lowercase, strip everything except letters, digits, `-`, `_` and spaces, spaces → `-`, and a duplicate heading gets `-1`, `-2`. Other targets differ, and Docusaurus/kramdown/Pandoc let you pin the id explicitly (`{#stable-id}`), which is the only way an anchor survives a rewording (`links.md`).
9. **Edits are surgical.** Change the lines the task names and nothing else. Run a formatter across a file only when the repo already has that formatter's config committed — otherwise the reformat is 200 lines of noise around a 2-line fix, and the reviewer cannot see the change. Same for autofix: fix the reported lines, not the file.

## Support Matrix

What actually renders where. Blank means "not without a plugin"; check the target before using anything below the third row.

| Feature | CommonMark | GFM (github.com) | MDX v3 | Pandoc | Python-Markdown | Slack / Discord |
|---|---|---|---|---|---|---|
| Pipe tables | — | yes | with remark-gfm | `pipe_tables` | `tables` ext | — |
| Strikethrough | — | `~~x~~` | with remark-gfm | `strikeout` | `pymdownx.tilde` | `~x~` / `~~x~~` |
| Task lists | — | yes | with remark-gfm | `task_lists` | `pymdownx.tasklist` | renders as text |
| Autolinked bare URLs | — | yes | with remark-gfm | `autolink_bare_uris` | `magiclink` | yes |
| Footnotes | — | yes | with remark-gfm | yes | `footnotes` ext | — |
| Raw HTML | passes through | sanitized allowlist | must be valid JSX | `raw_html` | passes through | — |
| `$…$` math | — | yes | with remark-math | `tex_math_dollars` | `arithmatex` | — |
| Mermaid fences | — | yes | theme/plugin | filter required | `superfences` | — |
| YAML frontmatter | renders as rule + heading | shown as a table | parsed as exports | `yaml_metadata_block` | `meta` ext | — |
| Custom heading id | — | — | `{#id}` | `{#id}` | `attr_list` | — |
| Callouts / admonitions | — | `> [!NOTE]` | `:::note` (Docusaurus) | fenced div | `!!! note` | — |

## Render Failures

Decode rule: **the symptom names the layer**. Text bleeding into the wrong block is a boundary problem; text disappearing is an escape or a sanitizer; text rendering literally is an extension the parser does not have.

| Symptom | Most likely cause | First move |
|---|---|---|
| Everything from line N to the end is a code block | Unclosed fence, or the closing run is shorter than the opening one | Count backticks on both fences; also check for a stray 4-space indent |
| List items all on one line, or as a single paragraph | No blank line before the list, or `loose vs tight` — a blank line between items adds `<p>` and the spacing you were fighting | `structure.md` |
| Sub-items are flat, or turned into code | Indent below the content column, or ≥ content column + 4 (Rule 3) | `structure.md` |
| Half the doc is italic from one point on | Unescaped `*` or `_` — often a filename, a glob, or `2*3` | Escape it, or wrap it in a code span |
| Table shows literal `\|` characters | Missing delimiter row, no blank line above, or the header/delimiter cell counts differ | `tables.md` |
| A cell splits in the middle of code | Unescaped pipe inside backticks | `tables.md` |
| Link text renders but the URL is dead | Space, `(`, or `)` in the destination | `<…>` around the destination, or percent-encode |
| `[text][ref]` renders literally | The `[ref]: url` definition is missing or misspelled — reference links fail silently | `links.md` |
| Anchor scrolls nowhere | Slug computed by a different rule than the one you assumed (Rule 8) | `links.md` |
| Image is a broken icon off-GitHub | Repo-relative path resolved by npm, PyPI, or the site against a different root | `readmes.md` |
| `---` at the top renders as a horizontal rule and a heading | The target does not consume frontmatter | `frontmatter.md` |
| HTML tag shows as escaped text, or vanishes | Sanitizer allowlist (GitHub, PyPI), or `unsafe: false` (Hugo/Goldmark) | `flavors.md`, `docs-sites.md` |
| `:tada:`, `[!NOTE]`, `[^1]` render as plain text | Extension absent in this parser | Support Matrix, then `extensions.md` |
| Renders in the editor preview, breaks on the site | VS Code preview is markdown-it, not the target's parser | Preview in the real target before shipping (Output Gates) |
| Invisible characters break a code sample or a build | Zero-width, bidi, or non-breaking spaces pasted from chat, a PDF, or a word processor | `security.md` |
| Anything else | Bisect: halve the document until the break disappears, then change one character at a time | — |

## Output Gates

Before handing back any Markdown:

- Which target is this for, and does every construct I used exist there (Support Matrix)?
- Blank line above and below each list, table, fence, and blockquote?
- Every fence closed, closing run ≥ opening run, language tag present?
- Every nested list at the content-column indent Rule 3 gives for its parent marker?
- Every link destination free of raw spaces and parens; every anchor derived from the target's slug rule, not invented?
- Every image with alt text that says what it is for, and no heading level skipped?
- No secret in any code sample, curl line, config snippet, or env block — and none written into `~/Clawic/data/`?
- Editing an existing file: is the diff limited to the lines the task named?
- Did this session produce something durable — a target and its quirk, a doc set, a config that finally passed, a conversion recipe, a template, a sweep result? Then it is written to its box in `memory-template.md`, with its `## Boxes` line, in this same turn.

## Configuration

User-dependent variables. Defaults apply until the user states a preference; store them in `~/Clawic/data/markdown/config.yaml`.

| Variable | Type | Default | Effect |
|---|---|---|---|
| target_flavor | commonmark \| gfm \| mdx \| pandoc \| kramdown \| goldmark \| python-markdown \| obsidian | gfm | The parser every rule and every generated document is written against; drives the Support Matrix row set and Rule 1 |
| docs_generator | none \| docusaurus \| mkdocs-material \| jekyll \| hugo \| sphinx-myst \| astro-starlight \| vitepress \| quarto | none | Frontmatter schema, admonition syntax, link resolution and build commands in `docs-sites.md` |
| lint_tool | markdownlint \| remark \| prettier \| none | markdownlint | Which rule ids are cited, which config file `linting.md` emits, and which CI snippet |
| line_wrap | none \| sentence \| number (columns 60-120) | none | How generated and edited prose is wrapped; `sentence` = one sentence per line for reviewable diffs (`editing.md`) |
| list_indent | 2 \| 4 | 4 | Spaces per nesting level in generated lists (Rule 3); 2 is safe only for GFM-family targets |
| list_marker | - \| * \| + | - | Bullet character in generated lists; must match lint rule MD004 if the repo enforces one |
| raw_html | allow \| avoid \| forbid | avoid | Whether fallbacks may use HTML (`<br>`, `<details>`, `<img width>`, HTML tables) or must stay pure Markdown; `forbid` for targets that strip it |
| frontmatter_format | yaml \| toml \| json \| none | yaml | Fence and syntax of generated metadata blocks (`frontmatter.md`) |
| link_style | inline \| reference | inline | Whether long documents keep destinations inline or collect them as `[ref]: url` definitions at the bottom |
| table_style | padded \| compact | padded | Whether generated tables pad cells to align pipes; `compact` keeps diffs small in files many people edit |

Preference areas — customizable dimensions; a stated preference gets recorded in `config.yaml` and applied from then on:

- **Conventions** — heading case (sentence vs title), emphasis and strong markers, fence character and length, ordered-list numbering style, one-H1 rule, file naming, whether a TOC is generated and to what depth
- **Tooling** — editor and preview engine, format-on-save, link checker, TOC generator, spell checker, whether CI blocks or only warns
- **Platform** — the set of targets they actually publish to, generator versions, image hosting and dark-mode variants, whether repos are private (image proxying differs)
- **Safety posture** — whether an autofixer may rewrite whole files, whether generated or vendored `.md` is off-limits, how strictly untrusted Markdown is sanitized
- **Output register** — corrected file vs diff vs explanation-first, how much of the reasoning to keep, whether to show the rendered result
- **Accessibility bar** — alt text required or optional, heading-order enforcement, whether complex tables may fall back to HTML
- **Localization** — locale, RTL content, smart quotes and typographic substitution, non-breaking spaces, CJK line-break behavior
- **Cadence** — link check, lint sweep, badge and version refresh, stale-page review; every accepted cadence becomes a row in the `## Due` table of `memory.md`

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Writing Markdown without deciding the renderer | Half the syntax in this skill is target-conditional; "valid Markdown" alone is not a standard anyone implements identically | Rule 1, then the Support Matrix |
| Two trailing spaces for a line break | Invisible in review, stripped on save, removed by formatters, flagged by MD009 | `\` at end of line; two spaces only on a pre-CommonMark target that has no `\` (Rule 6) |
| `#Heading` with no space after the hash | Not a heading in CommonMark or GFM; renders as literal text | `# Heading` |
| Trusting the editor preview | VS Code ships markdown-it: it renders things GitHub rejects and misses GFM alerts and Mermaid | Preview in the actual target before shipping |
| Hand-writing TOC anchors | Slugs are generated by rule, including the `-1` suffix on duplicate headings | Derive them (Rule 8) or pin ids where the target allows it |
| Emoji shortcodes `:tada:` | GitHub-only; everywhere else the literal colons render | Paste the Unicode emoji |
| A spreadsheet table pasted straight in | No delimiter row, unescaped pipes, and cell counts that do not match | `tables.md` conversion recipe |
| Relative image paths in a README that also ships to npm or PyPI | Those pages resolve relative paths against their own domain, not your repo | Absolute raw URL pinned to a tag (`readmes.md`) |
| Reference-style links copied without their definitions | Fail silently as literal text — nothing errors, nothing renders | Keep definitions with the block, or use `link_style: inline` |
| Autofixing the whole file to clear one lint error | The real change disappears inside a reformat; reviewers approve blindly | Fix the reported lines (Rule 9) |
| HTML tables for merged cells, shipped to PyPI, Hugo, or MDX | Each strips, drops, or re-parses HTML differently | Check `raw_html` and the Support Matrix first; simplify the table instead |
| Renaming a heading to improve wording | Every inbound anchor, cross-link, and bookmark dies silently | Grep the repo for the old slug and leave a stub anchor (`links.md`) |
| Pasting a doc with a live token into memory or an artifact | Docs are the densest source of secrets in the whole catalog | Strip to `<env:NAME>` before writing (Data, `memory-template.md`) |
| Smart quotes and em-dash autocorrect inside code samples | `"` becomes `"`, `--` becomes `—`; the sample stops working when copied | Author code in fences; disable typographic substitution for them |

## Where Experts Disagree

- **Wrapping.** Three real schools: no hard wrap (the editor wraps; `git diff` shows a whole paragraph as one changed line), hard wrap at 80 (clean in terminals, every edit reflows the block), and one sentence per line (surgical diffs, odd-looking source). The frontier is who reviews the file: prose reviewed line-by-line in pull requests → `line_wrap: sentence`; prose nobody diffs → `none`. Never mix two in one repo.
- **Raw HTML.** Purists keep documents portable; pragmatists want `<details>`, column widths, and merged cells. The frontier is the target set, not taste: Hugo drops raw HTML by default, PyPI and GitHub sanitize it to an allowlist, MDX re-parses it as JSX. HTML is safe only when every target you publish to renders it.
- **Lazy ordered lists (`1.` for every item).** Clean diffs and no renumbering when items move, at the cost of source that reads wrong to a human. Frontier: whether the source is read by people who are not editing it (docs in a repo → lazy is fine; a file that is itself the artifact → number it).
- **Line-length linting (MD013).** Enforced, it makes prose reviewable in any diff tool; enforced, it also fights every URL and table. The common resolution is on for prose, off for tables, code, and headings — the rule takes those exclusions as options.
- **Docs in the repo vs a docs platform.** In-repo Markdown reviews with the code and rots less; a platform gives non-engineers an editor. Frontier: who writes the majority of the pages.

## Security & Privacy

**Local storage:** preferences, observed targets, doc sets and generated artifacts stay in `~/Clawic/data/markdown/` on this machine, plus name-only pointers in the shared `~/Clawic/data/projects/` and `~/Clawic/data/contacts/`. File paths, repo names, slugs and rule ids only — never a token, key, or password, whatever the document the user pasted contained.

**Untrusted input:** Markdown from users, issues, scraped pages, or a model is untrusted HTML in disguise. Rendering it safely is sanitize-after-render, plus a URL-scheme allowlist; MDX from an untrusted source is arbitrary code and is never rendered at all (`security.md`).

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/markdown (install if the user confirms):
- `documentation` — what documentation should exist, how it is structured and kept alive
- `latex` — LaTeX documents, when Markdown plus math is no longer enough
- `word-docx` — `.docx` production and round-trip editing, the usual conversion destination
- `pdf-generator` — when the PDF is the deliverable and the template, branding and layout are the work, not the conversion
- `notes` — where notes get stored across Obsidian, Bear, Notion and plain files
- `yaml` — the frontmatter language, its type coercion and multiline forms

## Feedback

- If useful, star it: https://clawic.com/skills/markdown
- Latest version: https://clawic.com/skills/markdown

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/markdown.
