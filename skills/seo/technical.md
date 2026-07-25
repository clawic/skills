# Technical — Crawl, Render, Index, Serve

Pipeline model: **discover → crawl → render → index → serve**. Every technical problem is a break at one named stage, and each stage has its own control. Fixing the wrong stage is why technical SEO feels slow.

**Contents:** [Which Control Does What](#which-control-does-what) · [robots.txt](#robotstxt) · [Indexing Diagnosis](#indexing-diagnosis) · [Canonicals and Duplicates](#canonicals-and-duplicates) · [Sitemaps](#sitemaps) · [Status Codes That Matter](#status-codes-that-matter) · [Crawl Budget](#crawl-budget) · [Log Files](#log-files) · [Mobile and HTTPS](#mobile-and-https)

## Which Control Does What

| Control | Stops crawling | Stops indexing | Passes signals | Use for |
|---|---|---|---|---|
| `robots.txt Disallow` | Yes | No | No | Saving crawl budget on infinite or worthless URL spaces |
| `<meta name="robots" content="noindex">` | No | Yes | Links followed until the page is dropped | Pages that must exist for users but not in search |
| `X-Robots-Tag` HTTP header | No | Yes | Same as meta | Non-HTML files: PDFs, images, JSON |
| `rel=canonical` | No | No (a hint) | Yes, consolidates | Duplicate or near-duplicate URLs of the same content |
| 301 redirect | No | Removes the old URL | Yes | Permanent moves and consolidation |
| 404 / 410 | No | Yes, eventually | No | Pages that should cease to exist (410 is faster) |
| `nofollow` on a link | No | No | No (a hint since 2020) | Untrusted or paid links |
| Password / auth | Yes | Yes | No | Staging and private areas |

The combination that breaks sites: `Disallow` plus `noindex`. Blocked from crawling, Google never reads the noindex, and the URL can stay indexed as a title-only result. Allow the crawl until the page drops out, then block.

## robots.txt

- Per host, protocol, and port: `https://example.com/robots.txt` does not govern `https://shop.example.com` or the http:// variant.
- Google reads up to 500 KiB and ignores the rest — long generated files silently lose their tail rules.
- `noindex` in robots.txt is unsupported (Google stopped honoring it in September 2019). Rules that "worked for years" may simply have stopped.
- Never block CSS, JS, or the API endpoints a page needs: blocked resources produce a broken render, and Google indexes what it can render.
- Longest matching rule wins, and `Allow` beats `Disallow` at equal specificity — the way to open one path inside a blocked directory.
- A 5xx on robots.txt makes Google pause crawling of the whole host. A 404 is safe (treated as "crawl everything"); an error page returning 200 with HTML is not.

## Indexing Diagnosis

Read the Page indexing status, then act:

| Status | What it means | Action |
|---|---|---|
| Discovered — currently not indexed | Google knows the URL, has not chosen to crawl it | Crawl-demand problem: internal links, sitemap, better content — resubmitting changes nothing |
| Crawled — currently not indexed | Crawled, judged not worth indexing | Thin, duplicate, or no demand; improve or consolidate |
| Duplicate, Google chose different canonical | Your canonical was overridden | Align the signals: internal links, sitemap entry, hreflang, redirects, and content similarity |
| Alternate page with proper canonical tag | Working as intended | Nothing |
| Soft 404 | Thin, empty, or an error page returning 200 | Return a real 404, or add real content |
| Excluded by noindex | A tag or header says so | Find which layer emits it: CMS setting, plugin, CDN, framework metadata |
| Blocked by robots.txt | A Disallow rule matched | Decide: crawlable or truly excluded, never both |
| Page with redirect | Not a problem; the target is what matters | Verify the target returns 200 and is canonical |
| Not found (404) | The URL 404'd when crawled | Deliberate deletion: leave it, or 410 to speed it up. Accident: restore the page or 301 it to the closest equivalent, then fix the links and sitemap entries still pointing there |
| Server error (5xx) | The host failed under crawl, not an indexing setting | Crawl stats and server logs; sustained 5xx shrinks crawl rate for weeks |
| Blocked due to unauthorized request (401) / Blocked due to access forbidden (403) | Auth, geoblock, WAF, or bot protection answered Googlebot | Reproduce with the live test, verify Googlebot by reverse DNS in the logs, then allowlist it — staging protection leaking onto production is the usual cause |
| Anything else | A status this table does not list | Run URL Inspection's live test on one affected URL: the fetch result names which stage broke — discover, crawl, render, or index — and the fix follows that stage's control above |

Verify with URL Inspection's **live test** and read the rendered HTML — not browser view-source, which is the pre-render document.

## Canonicals and Duplicates

- Canonical is a hint. Google weighs it against internal links, sitemap inclusion, redirects, hreflang, and content similarity; conflicting signals lose to the majority.
- Self-referencing canonical on every indexable page: costs nothing, settles parameter and case duplicates.
- Canonical must point at a URL that returns 200 and is not itself canonicalized elsewhere. Chained canonicals get ignored.
- Duplicate generators to check on any site: `www` vs bare, `http` vs `https`, trailing slash, uppercase paths, `index.html`, session and tracking parameters, print views, and sort/filter URLs.
- Syndication: ask for a canonical back to your URL, or at minimum a link. With neither, the stronger domain usually wins the query with your text.
- Scrapers rarely outrank you; when one does, it is a symptom of your own indexing problem on that page.

## Sitemaps

- 50,000 URLs and 50 MB uncompressed per file; use a sitemap index above that.
- Only canonical, indexable, 200-status URLs. Redirects, noindex pages, and 404s in a sitemap teach Google to trust the file less.
- `lastmod` is used when it is consistently accurate; a file that stamps today's date on every URL every night gets ignored.
- Split sitemaps by template or section: submitted-vs-indexed per file turns the report into a diagnosis of which page type Google is rejecting.
- Image and video URLs travel in extension tags; news sites use a separate news sitemap covering the last 48 hours.

## Status Codes That Matter

| Code | Effect on search | Notes |
|---|---|---|
| 200 | Indexable | Also what soft-404 error pages return — the trap |
| 301 / 308 | Permanent move, signals transfer | Preferred for consolidation |
| 302 / 307 | Temporary; the old URL can stay indexed | Long-lived 302s get reinterpreted unpredictably |
| 304 | Not modified | Legitimate; helps crawl efficiency |
| 404 | Dropped over time | Fine for genuinely gone pages |
| 410 | Dropped faster than 404 | For deliberate deletions at scale |
| 429 / 503 | Crawl backs off | The correct answer to overload; sustained 5xx shrinks crawl rate for weeks |
| 500 | Crawl backs off, index decays | Any 5xx pattern in Crawl stats is a P0 |

## Crawl Budget

A real constraint only above roughly 1 million pages, or above ~10,000 pages that change daily; smaller sites are limited by demand, not by budget. Where it matters, the waste is almost always faceted URL combinations, infinite calendars, session IDs, endless pagination, soft 404s, and redirect chains. Crawl stats (GSC → Settings) breaks crawling down by response code, file type, and purpose: a host whose crawl is mostly JavaScript files and 404s is telling you exactly what to fix.

## Log Files

The only source showing what Googlebot actually did. Pull them when the site is large or the indexing story does not add up:

- Which templates get crawled and how often — the honest importance ranking of your site.
- URLs crawled that you did not know existed: parameters, old paths, injected spam.
- Pages that earn traffic but are crawled monthly — an internal linking problem.
- Googlebot receiving 5xx or 429 that users never see — CDN or WAF rate limiting by user agent.
- Verify Googlebot by reverse DNS before drawing conclusions; a large share of self-declared Googlebot traffic is not Google.

## Mobile and HTTPS

- Mobile-first indexing is universal: the mobile rendering is the indexed one. Content, links, and structured data that exist only on desktop do not exist.
- Viewport meta tag required; tap targets and legible font sizes affect usability, not ranking directly.
- Intrusive interstitials covering content on entry from search are a demotion signal; reasonably sized cookie and legal notices are not.
- HTTPS everywhere, no mixed content, HSTS once you are certain. After the switch: 301 all http URLs, then update canonicals, internal links, sitemap, and hreflang.
- Serve the same content to Googlebot and to users. Any user-agent branching is cloaking risk, and geolocation redirects mislead a crawler that fetches mostly from US IPs.
