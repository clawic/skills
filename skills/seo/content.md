# Content — Intent, Gain, Refresh, and Pruning

Content SEO is three jobs, and most programs only do the first: deciding what deserves a page, making that page better than what ranks, and maintaining the library so the dead pages do not drag the live ones.

**Contents:** [Search Intent](#search-intent) · [Information Gain](#information-gain) · [The Brief](#the-brief) · [E-E-A-T](#e-e-a-t) · [AI-Assisted Content](#ai-assisted-content) · [Structure](#structure) · [Freshness](#freshness) · [The Refresh Playbook](#the-refresh-playbook) · [Thin, Dead, and Duplicated](#thin-dead-and-duplicated) · [Content Inventory At Scale](#content-inventory-at-scale) · [Content Traps](#content-traps)

## Search Intent

Match the content type to the query intent — mismatch means no ranking at any quality:

- **Informational** ("how to", "what is", "guide"): guides, tutorials, explainers
- **Navigational** (brand plus product): homepage, product page, docs
- **Transactional** ("buy", "price", "discount", "near me"): product, service, or booking pages
- **Commercial investigation** ("best", "vs", "review", "alternatives"): comparisons and shortlists

Modifiers carry the intent more reliably than the head term: "for beginners", "template", "cost", "software", "reddit", and a year all tell you the format before you search. Then verify empirically — search the query, page 1 is the format spec (SKILL.md rule 2). Page 1 mixing formats means fractured intent: easier to enter, less stable to hold.

## Information Gain

Matching competitors' outlines produces the eleventh version of the same page, and Google has no reason to rank it. Every page needs something the current SERP lacks: original data, first-hand testing, a tool or calculator, expert quotes, a sharper answer structure, a genuinely different opinion. Go/no-go before writing: "what can we say that nobody ranking can?" No answer means do not write it — pick a different query or go get the missing input.

## The Brief

A brief that produces a rankable draft contains, and a brief that only lists a keyword and a word count does not:

1. Target query and the two or three variants it clusters with.
2. The intent and the format the SERP demands, stated as a sentence.
3. What the top five cover — the table stakes you cannot skip.
4. The gain: the specific thing this page will have that none of them do, and where it comes from.
5. The searcher's state: what they already know, what they will do next.
6. Internal links in (which existing pages will point here) and out (the money page this justifies).
7. The single question the page must answer in its first 100 words.

## E-E-A-T

Not a score — no meter exists; the rater guidelines shape Google's systems indirectly. Demonstrate, never claim:

- **Experience** (added to the acronym in 2022): first-hand evidence — original photos, test data, screenshots, "we measured"
- **Expertise**: author byline with verifiable credentials, linked to a bio page with history
- **Authoritativeness**: citations to and from recognized sources; being referenced elsewhere
- **Trust**: contact page, about page, HTTPS, claims with sources, transparent ownership

Decisive for YMYL (health, finance, legal, safety): weak trust signals there mean no ranking regardless of writing quality. For YMYL specifically, add reviewer credentials, publication and review dates, sources for every claim of fact, and an explicit correction policy.

## AI-Assisted Content

Google's spam policy targets scaled content abuse — mass-producing pages primarily to manipulate rankings — not AI use per se; AI-assisted content demonstrating experience and expertise is acceptable per Google's own guidance. The real trap is structural: raw model output is the consensus of existing content, which is zero information gain by construction. Use models for structure, drafting, and compression; the ranking ingredients — data, experience, opinion, examples — must come from outside the model. Every factual claim gets checked by a human who can be named on the byline.

## Structure

- Answer the query in the first ~100 words; featured snippets and AI summaries both extract answer-first pages.
- Question-shaped H2s matching People Also Ask phrasing, each followed by a short direct answer.
- Lists and tables for procedural and comparison content — the formats extraction prefers.
- Short paragraphs (2-3 sentences), a table of contents on long pieces, and descriptive subheads a skimmer can navigate by.
- One idea per section, with the conclusion first and the reasoning after. Search readers do not read to the end.
- An FAQ section at the bottom for the remaining People Also Ask queries, as content — the markup no longer produces a rich result for ordinary sites.

## Freshness

- Query Deserves Freshness: time-sensitive topics need genuinely recent content, and the SERP tells you which ones by showing publication dates.
- "Best X {year}" queries: update the content, then the date. Date-only bumps are detectable and treated as deceptive.
- Refreshing a decayed winner usually beats writing new: the URL's links and history compound, a new URL starts at zero.
- Evergreen content: no dates in the URL, a scheduled revisit, and a visible "last reviewed" only where accuracy matters to the reader.

## The Refresh Playbook

For a page whose traffic has fallen while it still ranks in the top 20:

1. Diagnose which happened: position lost (a competitor improved), CTR lost (snippet or SERP layout), or impressions lost (query demand fell or intent shifted).
2. Search the query now. Compare the current page 1 against what your page assumes. Intent shift is the most common cause and the one a rewrite of the same angle will not fix.
3. Update facts, prices, screenshots, and examples first; stale specifics are what readers and raters notice.
4. Add the subtopics the current top five cover and you do not, and remove the sections nobody needs any more.
5. Rework the title and opening for the query as it is phrased today.
6. Re-link: two or three fresh internal links from pages that gained authority since publication.
7. Republish with an accurate modified date, then measure in a 28-day window against the previous one.

## Thin, Dead, and Duplicated

"Thin" is about value, not word count: a 150-word page that fully answers its query is not thin; a 1,500-word page saying nothing is. For a page with no impressions and no links after 6-12 months indexed, in order:

1. **Improve** — the topic deserves a page and you can add information gain
2. **Consolidate** — 301 into the closest strong page and merge the unique parts
3. **Remove or noindex** — if neither applies

Sitewide quality reassessments judge the whole domain: dead weight drags your winners.

## Content Inventory At Scale

Once a site passes a few hundred URLs, decide with a table, not with opinions. Per URL: clicks and impressions (last 12 months), position, backlinks, conversions, last updated, and target intent. Then:

| Segment | Disposition |
|---|---|
| Traffic and conversions | Protect: scheduled refresh, keep links pointing in |
| Impressions, few clicks, position 4-15 | Highest ROI: snippet rewrite plus gap fill |
| Traffic, no conversions, informational | Keep, but add the path to a commercial page |
| No impressions, has links | Repurpose the URL for a topic with demand, or consolidate carefully to preserve the links |
| No impressions, no links, no strategic role | Consolidate or delete |
| Duplicated intent with a stronger page | 301 into the survivor, merge unique content |
| Anything else | Leave it and revisit next inventory; churn without a reason is its own risk |

Run this yearly on a stable site, quarterly on one that publishes heavily.

## Content Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Writing to a word count | Length is a correlation, not a cause; padding is a negative signal to readers | Cover the intent, then stop |
| Publishing without an information-gain answer | You ship the eleventh identical page | Answer the go/no-go question first |
| Refreshing by changing the date | Deceptive and detectable | Change the content, then the date |
| Deleting content in bulk after a quality hit | You remove pages that still earn and lose their links | Consolidate deliberately, page by page, with the data in front of you |
| One page per keyword variant | Cannibalization by design | One page per intent |
| Treating the FAQ as SEO furniture | Nobody reads it and it no longer earns markup | Answer questions readers actually ask, in the body |
| A library with no owner | Facts rot; the site's quality average falls silently | Inventory on a schedule, with dispositions assigned |
