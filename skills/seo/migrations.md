# Migrations — Domains, Replatforms, Redesigns, URL Changes

Every migration is the same bet: Google must re-crawl every old URL, follow a redirect, and transfer signals to the new one. Traffic loss is almost never caused by "the new site being worse" — it is caused by URLs that lost their thread.

## Isolate the Variables

| Change | Risk | Rule |
|---|---|---|
| Domain only, same URLs | Moderate | Safest migration there is: one-to-one redirects, Change of Address tool |
| URL structure only | High | Requires a complete map; the most common source of permanent loss |
| Template/design only, same URLs | Low-moderate | Risk is in removed content, links, and JavaScript rendering |
| CMS replatform | High | Combines URL, template, and technical stack changes at once |
| HTTP → HTTPS | Low | Full-site 301, update canonicals, internal links, and sitemaps |
| Two or more of the above together | Compounding | Sequence them weeks apart; a combined migration is undiagnosable when it goes wrong |

Never combine a domain change with a URL restructure and a redesign in one release unless the deadline is immovable — and then, expect to debug blind.

## Pre-Launch Inventory (do this before any code freeze)

1. Crawl the entire old site; store every 200-status URL.
2. Export 16 months of GSC data: every URL with clicks or impressions. Anything with impressions needs a destination.
3. Export the backlink profile; URLs with external links are the highest-value redirect targets.
4. Save the current sitemap, robots.txt, and hreflang set.
5. Record baselines: clicks, impressions, indexed page count, average position by template, top 200 landing pages, Core Web Vitals.

The union of crawl + GSC + backlinks is the redirect map's source. A crawl alone misses orphaned pages that still rank and still earn links.

## The Redirect Map

- One row per old URL: `old → new (301)`. No wildcards until the exceptions are enumerated; wildcards are how 4,000 blog posts land on a category page.
- Map to the closest equivalent page. Bulk-redirecting unmatched URLs to the homepage produces soft 404s and transfers nothing.
- No genuine equivalent and no traffic or links → let it 410. Deliberate removal beats a misleading redirect.
- Chains ≤3 hops, and collapse old chains while you are in there: if the old URL already redirects once, point the new rule at the final destination.
- Preserve query strings only where they mean something; strip tracking parameters at the redirect.
- Test the map before launch against a staging host: every source URL must return exactly one 301 and land on a 200.

## Launch Sequence

1. Remove staging blocks: the `noindex` or `Disallow: /` that protected staging is the single most common launch disaster. Password-protect staging instead of noindexing it, so there is nothing to forget.
2. Deploy redirects at the same moment as the new site, not after.
3. Publish the new sitemap; keep a temporary sitemap of the OLD URLs submitted for a few weeks so Google recrawls them and finds the redirects faster.
4. Verify the new property in Search Console before launch; for a domain change, use the Change of Address tool (requires both properties verified and the redirects live).
5. Update internal links to point at final URLs — do not rely on redirects for your own navigation.
6. Update canonicals, hreflang, and structured data URLs to the new addresses.
7. Update the backlinks you control: social profiles, directories, partner sites, email footers.
8. Keep the old domain and its redirects for at least a year; Google's guidance is to leave a domain-change redirect in place for a minimum of one year.

## Post-Launch Monitoring

| Window | Watch | Alarm |
|---|---|---|
| Day 0-2 | Live fetches of top 50 URLs, robots.txt, sitemap fetch status | Any 404, any redirect chain, any noindex |
| Week 1 | GSC Crawl stats (spike expected), Page indexing report, server 5xx rate | 5xx under crawl load; "Not found" climbing on mapped URLs |
| Week 2-4 | Indexed count of new URLs, clicks by template | New URLs not being indexed at all |
| Week 4-8 | Clicks vs baseline by template and by landing page | Still below ~80% of baseline with clean redirects → content or template problem, not redirects |
| Ongoing | Old-URL 404 log | Any unmapped URL that receives requests gets a rule added |

A dip in the first weeks is normal while Google recrawls. Turbulence lasting past 8 weeks on a mid-size site is a defect, not patience.

## Failure Modes

| Symptom | Cause |
|---|---|
| Traffic drops the day after launch, redirects verified | Sitewide noindex or robots.txt from staging still live |
| Indexed count collapses over two weeks | Canonicals still pointing at the old domain, or hreflang not updated |
| Some templates recover, one never does | That template's content shrank in the redesign, or its links are now JavaScript-only |
| Rankings return but for different URLs | Redirect targets do not match intent; remap to closer equivalents |
| Old URLs still ranking months later | Redirects are 302, or served only to browsers and not to Googlebot |
| Crawl rate collapses after migration | 5xx or slow responses under crawl load taught Googlebot to back off |
| Images and video lose their traffic | Image URLs were not redirected; media is a separate index |
| Anything else | Unknown until reproduced: fetch the old URL as Googlebot, follow the chain to a 200, and diff the rendered HTML against the pre-launch crawl — the first step that differs from the plan is the cause |

## Migration Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| 302 for a permanent move | Signals transfer is slower and less certain than with 301 | 301 for permanent, 302 only for genuine temporary states |
| Redirecting everything to the homepage | Google treats irrelevant redirects as soft 404s | Closest match, or 410 |
| Launching redirects "next sprint" | Every crawl in between records a 404 | Same release, or no release |
| Trusting a plugin's automatic redirects | Slug-matching rules silently miss renamed and merged pages | Explicit map, tested URL by URL |
| Blocking the old site in robots.txt after the move | Googlebot cannot see the redirects it must follow | Leave the old host crawlable |
| Judging success on day 3 | Recrawl of the whole URL set takes weeks | Compare at week 4 and week 8 against baseline |
