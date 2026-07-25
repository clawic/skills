# Recovery — Drops, Updates, Penalties, and Hacked Sites

Recovery starts with classification. The shape of the loss names the cause faster than any tool: what dropped, when, and how widely.

## Classify the Shape First

| Shape of the loss | Most likely cause | First check |
|---|---|---|
| One page, one query cluster | Intent shift, a stronger competitor, or cannibalization from your own new page | Search the query; look for a newer URL of yours in GSC |
| One page, all its queries, overnight | Page-level: noindex, canonical flip, 404, redirect, template change | URL Inspection → Google-selected canonical and indexing status |
| Whole site, single day, total | Robots.txt, sitewide noindex, DNS/hosting, HTTPS failure, manual action | Fetch robots.txt live; GSC Manual Actions; server logs for Googlebot 5xx |
| Whole site, gradual over 1-3 weeks | Core update rollout | Drop dates vs Google's published update timeline |
| Whole site, impressions flat, clicks down | SERP layout change: AI Overview, ads, packs above you | Compare CTR at unchanged average position; look at the live SERP |
| Category of pages only | Template or quality reassessment of that page type | Compare that template's pages against the ones that held |
| Anything else | Verify the data before treating it as SEO | Year-over-year comparison; analytics tag and property integrity |

## Core Updates

- Rollouts take roughly one to three weeks; judging anything before the rollout ends is reading noise.
- The helpful content system stopped being a separate system in the March 2024 core update — sitewide quality assessment is now part of core ranking. Recovery is therefore a sitewide job, not a page fix.
- A core update is a re-ranking of what already exists, not a punishment for a specific error. There is no line to fix; there is a comparison you lost.
- Work the comparison: sample 20 queries you lost, read the pages that replaced you, and list concretely what they have (first-hand evidence, freshness, depth, brand familiarity, reviews) that you do not.
- Recovery generally arrives at a later update, not between them. Sites that improved substantially and still waited two or three cycles are common; some never return. Say this before the work, not after.
- Deleting content wholesale is a bet, not a fix. Consolidating genuinely dead pages is defensible; deleting pages that still earn impressions removes evidence you needed.

## Manual Actions

GSC → Security & Manual Actions. Types you will actually see, with the fix that ends them:

| Action | Meaning | Fix |
|---|---|---|
| Unnatural links to your site | Paid, exchanged, or scaled inbound links | Remove what you can, disavow the rest at domain level, document every attempt |
| Unnatural links from your site | Selling links or unmarked paid outbound links | Remove or add `rel="sponsored"`/`nofollow` |
| Thin content with little or no added value | Doorway pages, scraped or spun content, thin affiliate pages | Delete or genuinely rebuild; consolidation is not enough if the intent was scale |
| Pure spam | Automated gibberish, cloaking, scraped content | Full remediation; expect a long road |
| User-generated spam | Spam in comments, forums, profiles | Clean, then moderate: nofollow UGC links, approval queues, rate limits |
| Site reputation abuse | Third-party content published on your domain to trade on your ranking | Remove or noindex the hosted sections; first-party oversight does not exempt it under the policy as enforced since 2024 |
| Cloaking / sneaky redirects | Different content or destination for Googlebot vs users | Remove the branching; verify with URL Inspection's rendered HTML |

Reconsideration request structure — Google reads these by hand, so write for a human:

1. What was wrong, stated plainly, with no excuses and no blaming an agency without evidence.
2. What you did, itemized, with counts (URLs removed, links disavowed, sites contacted).
3. Evidence: a linked spreadsheet of actions, before/after examples.
4. What prevents recurrence: process, ownership, monitoring.

Reviews take days to weeks. A rejected request usually means the cleanup was partial — assume that first. Ranking recovery after revocation is separate and slower: the penalty ends, the lost links stay lost.

## Security Issues and Hacked Content

Signatures, in the order they get missed:

- **Cloaked spam pages**: your site ranks for pharma, gambling, or foreign-language terms you never wrote — visible only to Googlebot. Detect with URL Inspection (live test) and by searching `site:` for unrelated terms.
- **Injected redirects**: users from Google get redirected, direct visits look fine, often mobile-only or first-visit-only via cookie.
- **Content injection**: hidden links or text in existing templates, usually in the footer or in a sprite of `display:none` markup.
- **Fake pages at scale**: thousands of new URLs appear in the Page indexing report in days.

Cleanup order: take a forensic copy → patch the entry vector (outdated plugin, stolen credentials, exposed admin) → remove injected content and rogue files → rotate every credential including database and CMS users → remove any spam URLs (410) and request review in GSC → keep monitoring for reinfection for weeks; reinfection through an unpatched vector is the norm, not the exception.

## Deindexation Without a Penalty

Most "we got penalized" cases are self-inflicted. Check in this order: `noindex` shipped by a template or plugin, robots.txt disallow, canonical pointing elsewhere, redirect loop or chain, 4xx/5xx served to Googlebot only, hreflang pointing to a URL that 404s, or a CDN/WAF blocking Googlebot's IP ranges. Verify with a live fetch as Googlebot smartphone, not with a browser.

## Spam Policies Worth Knowing By Name

- **Scaled content abuse**: mass-produced pages, regardless of how they were produced, whose purpose is ranking rather than users.
- **Expired domain abuse**: buying an expired domain to reuse its authority for unrelated content.
- **Site reputation abuse**: renting out subfolders or sections to third-party content.

All three are enforceable as manual actions and as algorithmic suppression. The common tell is intent visible in the pattern: sudden volume, templated sameness, topical mismatch with the host domain.

## Recovery Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Changing everything at once after a drop | You lose attribution and cannot learn | One hypothesis per change window, logged with dates |
| Disavowing after a core update | Core updates are not link penalties; you cut real equity | Disavow only with a manual action naming links or a clear spam attack |
| Reading day-to-day rank fluctuation as recovery | Daily volatility is normal and larger than most fixes | Compare 28-day windows against the same window last year |
| Waiting for a "recovery update" and shipping nothing | Nothing is recomputed for a site that did not change | Improve the comparison, then wait |
| Republishing dates to look fresh | Date-only changes are detectable and treated as deceptive | Update the content, then the date |
