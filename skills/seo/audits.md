# Audits — Scoping, Running, and Delivering One

An audit is a decision document, not an inventory of everything imperfect. The deliverable that changes a business is five ranked fixes with the traffic at stake attached; a 200-row spreadsheet of crawler warnings gets filed and forgotten.

## Scope Before Crawling

Ask what the audit must answer, then pick the shape:

| Trigger | Shape | Time |
|---|---|---|
| "Traffic dropped" | Diagnostic — run the triage order in SKILL.md, stop at the confirmed cause | Hours |
| "We're about to redesign" | Risk audit — inventory, redirect plan, template review | 1-2 days |
| "We want more traffic" | Opportunity audit — striking distance, gaps, page-type economics | 2-4 days |
| "Health check before a quarter" | Full audit — all five layers, template-level | 3-5 days |
| Anything else | Diagnostic first; a full audit on an unknown site buries the one issue that matters | — |

## Data You Need Before Opinions

1. Search Console: 16 months of query and page data, Page indexing report, Crawl stats, Manual actions, Core Web Vitals.
2. Analytics: organic landing pages with conversions — the only way to weight fixes by money.
3. A crawl of the site, rendered (JavaScript on) and unrendered; the diff is the JavaScript dependency.
4. The XML sitemap(s) and robots.txt as fetched from the live host, not the ones in the repo.
5. A backlink export if `tool_access: paid-suite`; otherwise GSC's Links report, which is free and understates.

Missing data is a finding: no analytics goals configured means nobody can prioritize, and that goes at the top of the report.

## Crawl Configuration That Avoids Wrong Conclusions

- Respect robots.txt on the first pass (you see what Google sees), then re-crawl ignoring it to find what is hidden behind it.
- Render JavaScript on the second pass and diff link counts and word counts against the first.
- Crawl as Googlebot smartphone; mobile is the indexed version.
- Large sites: crawl a stratified sample — 500-2,000 URLs per template — instead of all 400,000. Issues live in templates, not URLs.
- Crawl the staging site too when one exists: staging leaks are found by crawling, never by asking.

## Template Thinking

On any site above a few thousand URLs, count issues per template, not per URL:

`impact of a template fix = pages on that template × sessions per page × expected lift`

A missing H1 on one orphan page is noise. The same defect on the product template covering 12,000 pages is the audit's headline. Report the template name, one example URL, and the page count.

## Prioritization

Score every finding, then sort descending:

`priority = (traffic at stake × confidence the fix moves it) / effort in days`

- Traffic at stake: current organic sessions on affected URLs, or for blocked/unindexed pages, the search volume of their target intent discounted to a realistic position (SKILL.md rule 9).
- Confidence: 0.9 for "the page is blocked from indexing", 0.5 for "intent mismatch", 0.2 for anything whose mechanism you cannot name.
- Effort: real engineering days, asked of whoever will ship it — not your guess.
- Floor: a finding whose traffic at stake is under `min_impressions` per month (SKILL.md Configuration; default 100) goes in the grouped list, never in the Top 5 — five headline fixes that each move nothing is how audits lose credibility.

Worked: a stray noindex on 40 pages worth 8,000 sessions/mo, confidence 0.9, effort 0.5 days → 14,400. A CLS fix worth maybe 300 sessions, confidence 0.2, effort 5 days → 12. The order is not a matter of taste.

## The Five Layers, In Order

1. **Penalties and security** — a manual action makes every other finding irrelevant until cleared.
2. **Indexing** — can Google fetch, render, and index the pages that matter?
3. **Intent and content** — do the indexed pages answer the queries they target?
4. **Technical quality** — speed, mobile, duplicates, structure.
5. **Authority** — internal link distribution first, external links second.

Never invert: prescribing a content rewrite for a page carrying a noindex is the classic wasted engagement.

## Deliverable

```
1. Verdict — one paragraph: what is capping this site, and what it is worth
2. Top 5 fixes — each: URL or template, exact change, owner, effort, traffic at stake
3. Everything else — grouped by layer, ranked, no essays
4. What we could not verify — access gaps, missing data, unknowns
5. Measurement plan — the GSC/analytics view that will show whether it worked, and when
```

Write the exact change, never the category: "add `<link rel=canonical>` self-reference to the product template" beats "fix canonicalization". Every recommendation a developer can ticket without asking a question.

## Audit Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Reporting crawler warnings verbatim | Tools flag non-signals; credibility dies at the first "meta keywords missing" | Filter every finding through "what traffic does this cost?" |
| Auditing without Search Console access | You are guessing at what already ranks and at what Google chose to index | Make access a precondition; a read-only user takes minutes to add |
| Ranking findings by severity label | Tool severity is generic, not site-specific | Score with the priority formula |
| One giant fix list, no owner | Nothing ships | Five items, named owners, named effort |
| Auditing a staging or CDN-cached copy | You review a site nobody sees | Verify with a live fetch and compare against a fresh crawl |
