# JavaScript — Rendering, SPAs, and Content Google Never Sees

Google indexes in two passes: it crawls the HTML response, queues the URL for rendering in a headless browser, and indexes the rendered result. Rendering does happen — but it is queued, budgeted, and can fail silently. Everything below is about what survives that gap.

## What Actually Breaks

| Symptom | Cause | Fix |
|---|---|---|
| Page indexed but ranks for nothing | Main content arrives after render, or not at all | Server-render the primary content |
| Google indexed an old version of the copy | Render queue lag plus client-side content changes | Ship content changes in the HTML response |
| Internal pages never discovered | Navigation uses `onclick`/`div` handlers, not `<a href>` | Real anchors with real hrefs to real URLs |
| "Crawled — currently not indexed" on a whole SPA route type | Rendered content is empty, or all routes render the same shell | Per-route server responses with unique content |
| Missing pages return 200 with "Not found" text | Client-side 404; Google records a soft 404 | Server returns a real 404, or the route noindexes itself |
| Title and description identical across every URL | Metadata injected after render, or never per-route | Emit metadata in the server response |
| Content behind tabs/accordions indexed, but content behind "load more" is not | Interaction-gated content is never triggered by the renderer | Render all indexable content in the DOM on load |
| Infinite scroll pages are invisible | No crawlable URL exists for anything past the first screen | Paginated URLs with links, alongside the scroll UX |
| Anything else | Compare crawled HTML against rendered HTML | The diff names the defect |

## Rendering Strategies

| Strategy | What Google gets on first fetch | Use when |
|---|---|---|
| Static generation (SSG) | Complete HTML | Content that changes on a deploy cadence — the safest choice for SEO pages |
| Incremental regeneration (ISR) | Complete HTML, possibly stale | Large catalogs where rebuilding everything is impractical |
| Server-side rendering (SSR) | Complete HTML per request | Personalized or fast-changing pages; watch TTFB |
| Streaming SSR with partial hydration | Complete HTML for the shell and main content | Large apps that need interactivity without shipping everything |
| Client-side rendering (CSR) | An empty shell | App screens behind login — never for pages that must rank |
| Dynamic rendering (serve HTML to bots) | Complete HTML, for bots only | Legacy last resort: Google calls it a workaround, not a recommendation, and it doubles the surface you must keep in sync |

Default: server-render or statically generate anything with a URL a searcher could land on; keep CSR for the authenticated application.

## Testing What Google Sees

1. URL Inspection → **Live test** → View crawled/rendered HTML. This is the authority; nothing else is.
2. `view-source:` shows the server response only — the diff between it and the DOM is your JavaScript dependency.
3. Crawl the site twice, JavaScript off and on, and diff word counts, link counts, titles, and canonicals per URL. Any template with a big delta is a risk.
4. Disable JavaScript in a browser and navigate: if you cannot reach a page, neither can a link-following crawler.
5. Check the rendered HTML for the canonical, robots meta, and structured data — not just the body copy.

## Rules That Prevent The Whole Class

- **Robots meta must be correct in the server response.** If the initial HTML says `noindex` and JavaScript removes it later, Google may drop the URL before rendering it. Never use client-side JavaScript to flip indexability.
- **Canonical in the server response.** A rendered canonical is read, but it competes with everything the pre-render document implied; conflicting values are resolved against you.
- **Links are `<a href="/real/url">`.** Router-managed navigation still needs a real href — that is what discovery follows and what passes link signals.
- **One URL per state that matters.** Filters, tabs, and steps that produce distinct content need distinct, linkable URLs via the History API, not a hash or in-memory state.
- **Status codes come from the server.** SPAs cannot emit HTTP status; missing routes need a server-side 404, or at minimum a noindex plus a real message.
- **Bundle size is an SEO metric on JavaScript sites** — it is the direct cause of INP failures and of render timeouts on slow crawl fetches.
- **Never block your own JS/CSS in robots.txt.** A blocked bundle means Google renders a broken page and indexes what is left.

## Hydration And Framework Pitfalls

- A hydration mismatch can blank server-rendered content in the browser: the HTML was fine, the rendered DOM is empty, and Google indexes the empty version. Watch console errors in the live test.
- Client-side redirects (`useEffect` → `router.push`) are seen only if rendering completes; server redirects with real status codes are unambiguous.
- Personalization on a public URL means Google indexes the default variant. Keep personalized blocks out of the primary content.
- Third-party SEO plugins that inject metadata after mount are a common source of duplicate titles across an entire app.
- A/B testing frameworks that flash content or rewrite headings mid-load create both CLS and indexing ambiguity; run experiments server-side where possible.

## When The Site Is Already CSR And Cannot Be Rebuilt

Prioritize, in order: (1) server-render or prerender the templates that carry commercial intent, (2) make navigation crawlable with real anchors and a sitemap, (3) fix metadata and canonicals at the server layer, even if the body still renders client-side, (4) reduce the bundle so rendering succeeds within a reasonable budget. Partial migration by template is normal and beats a rewrite that never ships.
