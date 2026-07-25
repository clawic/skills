# Platforms — Where The Fix Actually Lives

Every recommendation in this skill has to be implemented somewhere. The platform decides whether that is a checkbox, a template edit, or impossible — and each platform ships a default that quietly costs traffic. Check the `cms` variable, then read the section.

## WordPress

- **The staging disaster**: Settings → Reading → "Discourage search engines from indexing this site" emits a sitewide `noindex`. It survives the copy from staging to production more often than anyone admits — check it first on any WordPress site that lost everything overnight.
- Taxonomy bloat is the default: category, tag, author, and date archives all become indexable URLs, most of them near-empty. Decide per taxonomy: index the ones with real content and purpose, noindex the rest.
- Attachment pages generate one thin URL per uploaded image; redirect them to the parent post (modern SEO plugins do it in one setting).
- One SEO plugin, never two. Duplicate title and canonical tags come from a site running two of them.
- Plugin-generated sitemaps override the WordPress core sitemap; the file submitted in Search Console must be the one the plugin actually maintains.
- Page builders often ship multiple H1s and heavy DOM; check the rendered heading structure rather than the editor's outline.
- Performance is a hosting and plugin problem before it is a code problem: object and page caching, a CDN, and removing the plugins that inject scripts on every page beat micro-optimization.

## Shopify

- URL structure is fixed: `/products/`, `/collections/`, `/pages/`, `/blogs/`. You cannot flatten it; do not promise a client otherwise.
- The same product is reachable at `/products/x` and `/collections/y/products/x`. Themes canonical the collection path back to `/products/x` — verify it, because custom themes break it.
- Variants use `?variant=` parameters; they should canonical to the product URL.
- `robots.txt.liquid` has been editable since 2021 — crawl control is possible, but do it there rather than fighting the theme.
- Collection filtering and sorting create large parameter spaces; treat them as facets and keep them out of the crawl path.
- Apps inject scripts liberally; audit what is actually loading before diagnosing Core Web Vitals.
- Blog structure is limited (one level of blogs and articles) — plan content hubs around collections and pages instead.

## Webflow

- Redirects, canonical fields, and per-page metadata are all in the UI; there is no plugin layer to fight.
- CMS collection pages inherit one template — set metadata from collection fields so every item is unique, and check that the default is not "Site name" repeated a thousand times.
- Sitemap is auto-generated; the manual override exists but drifts from reality, so leave it automatic unless you have a reason.
- Hosting is fixed, so server-level headers and edge logic are not available: solve at the page level.

## Wix and Squarespace

- Both now server-render their pages; the old "Wix cannot rank" claim is out of date. The real constraints are control, not rendering.
- Redirect management exists but is manual — migrations onto or off these platforms need the redirect map built by hand.
- Limited control of headers, robots directives, and structured data beyond built-ins; complex markup usually needs a custom code block.
- URL structures include platform-imposed segments in places; plan the information architecture around them rather than against them.

## Next.js and Modern Frameworks

- Choose rendering per route, not per app: static or server-rendered for anything searchable, client-only for authenticated screens.
- Metadata belongs in the framework's server-side metadata API so titles, descriptions, and canonicals arrive in the HTML response.
- Incremental regeneration serves stale HTML by design — confirm the revalidation window matches how fast the content must be correct in the index.
- A missing route must return a real 404 status from the server, not a client-rendered "not found" screen with a 200.
- Middleware and edge redirects must emit 301/308 for permanent moves; framework-level client redirects are invisible to a crawler that never executes them.
- Check the rendered HTML after every dependency upgrade; framework defaults for metadata and rendering change between major versions.

## Headless And Custom Stacks

- Nothing is free: sitemaps, canonicals, redirects, robots.txt, and structured data all have to be built and owned. Enumerate them at project start or they get discovered during the traffic loss.
- The redirect table needs a home — a database, an edge config, a routing file — before the first URL changes.
- Preview and staging environments must be password-protected, not noindexed: a noindex is one deploy away from production.
- Put the SEO checks in CI: canonical present, title unique, robots meta correct, sitemap builds, no 5xx on key routes. Cheaper than any audit.

## Cross-Platform Diagnosis

| Symptom | Likely platform cause |
|---|---|
| Sitewide noindex overnight | WordPress reading setting; staging config promoted; framework env flag |
| Duplicate titles across hundreds of URLs | Template default not bound to per-item fields |
| Two canonical tags | Two SEO plugins, or theme plus plugin both emitting |
| Traffic loss after a theme or app install | Injected scripts, changed URLs, or a new noindex on a template |
| Sitemap missing new content | Plugin-generated file not the one submitted, or a cache never invalidated |
| Redirects work locally but not in production | Edge/CDN rules ahead of the app layer |
| Anything else | Fetch as Googlebot, read the raw response headers and the rendered HTML, and locate which layer emitted the tag |
