# Keywords — Research, Sizing, and Competitive Gaps

Keyword research answers three questions in order: what do people search, which of those can we win, and which are worth winning. Volume answers none of them on its own.

## Research Process

1. **Seeds** — product and service terms plus the customer's own language from support tickets, reviews, sales calls, and internal site search. Customer words beat internal jargon every time.
2. **Expand** — Google autocomplete (including "a" through "z" suffix sweeps), People Also Ask trees, related searches, competitor sitemaps and navigation, Reddit and forum phrasings, and your own Search Console queries.
3. **Qualify** — intent first, then the SERP reality check, then volume. In that order, always.
4. **Prioritize** — business value × winnability; low-authority sites start long-tail and earn the head terms later.
5. **Map** — one intent per page (SKILL.md rule 4); variants and questions cluster onto the page that answers them.

## SERP Reality Check

Do this before trusting any difficulty score. Tool "keyword difficulty" mostly measures the backlinks of ranking pages — not content quality, not intent, not what the searcher wants. Search the keyword and read page 1:

- All results are big-brand domains and homepages → skip, whatever the score says
- Forums, Reddit threads, thin pages, or stale dates on page 1 → beatable regardless of volume
- Featured snippet, AI Overview, People Also Ask, and ads stacked on top → organic #1 earns far less than its rank suggests; check where organic results actually start on the screen
- Page 1 mixes formats (guides and product pages) → fractured intent: Google is unsure, entry is easier, holding is harder
- Every result is a page type you cannot produce (a marketplace, a calculator, a government database) → the query is not for you

A usable winnability proxy without paid tools: `winnable ≈ (number of page-1 results that are weaker than the page you can build) ÷ 10`. Three or more weak results means go; zero means the query is a brand fight.

## Sizing Demand Without Paid Tools

- Google Search Console already reports the queries you earn impressions for — the only exact, first-party demand data available. Start there, not in a keyword tool.
- Google Trends gives relative interest, seasonality, and breakout terms; it never gives absolute volume.
- Keyword Planner buckets ranges and merges close variants — fine for relative sizing, wrong as absolutes, and biased toward advertiser terms.
- Zero-volume keywords convert: tools miss most long-tail. If autocomplete suggests it, people search it.
- Autocomplete ordering is a rough popularity proxy within a prefix; the top suggestions are the common phrasings.
- Seasonal terms: check Trends before committing. An annual average hides a cliff, and a page shipped one month after the peak wastes a year.

## CTR By Position

Click studies put position 1 around 25-30% CTR, falling steeply to low single digits by position 10. SERP features compress the whole curve. Traffic estimate: `expected clicks ≈ volume × position CTR × (1 − feature discount)` (SKILL.md rule 9). A #3 ranking under an AI Overview can earn less than a clean #5. Derive your own curve from Search Console before using any published one — vertical differences are large.

## Striking Distance

The highest-ROI keyword work. Search Console → Performance → filter positions 4-15 with impressions at or above `min_impressions` (SKILL.md Configuration; default 100/month, raise it on large sites). These pages already have relevance; they lack CTR or authority:

1. Rewrite title and meta against the snippets currently above you.
2. Fill content gaps — subtopics the ranking pages cover and you do not.
3. Add 2-3 internal links from your strongest related pages.

Check this list before proposing any new article — improving these beats new content on effort per click, usually by an order of magnitude.

## Cannibalization Check

Search Console → Performance → filter by query → Pages tab. Two URLs trading impressions across weeks for one query is cannibalization. Fix: 301 the weaker into the stronger, merge unique sections, and point internal links at the survivor. Two URLs holding steady in different positions for the same query with different intents is not cannibalization — verify by reading the queries each earns before merging anything.

## Keyword Types

**By intent:** informational ("how to", "what is") · navigational (brand names) · transactional ("buy", "price") · commercial ("best", "vs", "review"). The modifier usually names the intent before you search.

**By length:** head terms (1-2 words, high volume, brand-dominated SERPs) versus long-tail (3+ words, lower volume, easier entry, higher conversion). Low-authority sites earn coverage on long-tail before head terms become winnable — not because long-tail is a trick, but because the head term's SERP is a brand contest.

**By value:** a query with 200 searches that names your product category and a budget beats one with 20,000 that names a homework topic. Score business value explicitly before sorting by volume.

## The Keyword Map

The deliverable, one row per target URL — not one row per keyword:

`URL · primary intent · primary query · clustered variants · page type · current position · owner · status`

Rules: every keyword belongs to exactly one URL; queries whose SERPs are near-identical get clustered onto one row (Google already treats them as one intent); and a new row requires proof that no existing URL targets that intent. The map is the artifact that prevents cannibalization months before it happens.

## Competitive Analysis Without Paid Tools

1. `site:competitor.com <topic>` — their coverage depth per topic, and their URL patterns.
2. Their sitemap.xml — full content inventory, publishing cadence, and which sections they invest in.
3. SERP overlap — search your 10 target queries and note who repeats. Those are your true search competitors, often not the companies the business names as rivals.
4. Their navigation and internal linking — the pages they link from everywhere are the pages they are trying to rank.
5. Gap = topics, questions, and formats they rank with and you lack. Prioritize gaps adjacent to your commercial pages; a gap far from revenue is a hobby.
6. Check what they lost: pages that used to rank and no longer do are cheap entry points.

## Tracking

- Weekly for money terms, monthly for long-tail; more frequent tracking measures noise.
- Track a small basket of queries you actually care about rather than a thousand-keyword report nobody reads.
- Log rank moves against Google's announced update dates — that is what separates "an update hit us" from "we broke something" (SKILL.md, Ranking Drop Triage).
- Record positions per query, never the site-wide average position.
- Store the basket and its history in `~/Clawic/data/seo/memory.md` so comparisons survive across sessions.

## Keyword Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Sorting the list by volume | The top rows are usually unwinnable or worthless | Sort by value × winnability |
| Trusting a difficulty score | It mostly measures backlinks of current results | Read page 1 yourself |
| One page per variant | Google clusters variants; you built competitors for yourself | Cluster onto one intent |
| Ignoring zero-volume long-tail | Tools miss the tail where conversion lives | Harvest autocomplete and GSC |
| Researching once at project start | Demand, SERPs, and competitors move | Re-check the target queries at every refresh |
| Targeting queries the business cannot serve | Traffic that cannot convert costs the same to earn | Score business value first |
