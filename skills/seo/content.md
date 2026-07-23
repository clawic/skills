# Content SEO

## Search Intent

Match content type to query intent — mismatch = no ranking:
- **Informational**: "how to", "what is" → guides, tutorials
- **Navigational**: brand + product → homepage, product page
- **Transactional**: "buy", "price", "discount" → product/service pages with CTAs
- **Commercial investigation**: "best", "vs", "review" → comparisons, reviews

The check is empirical: search the query, page 1 is the format spec (→ SKILL.md rule 2). Page 1 mixing formats = fractured intent — easier to enter, less stable to hold.

## Information Gain

Matching competitors' outlines produces the 11th version of the same page — Google has no reason to rank it. Every page needs something the current SERP lacks: original data, first-hand testing, a tool or calculator, expert quotes, a sharper answer structure. Go/no-go question before writing: "what can we say that nobody ranking can?" No answer = don't write it.

## E-E-A-T

Not a score — no meter exists; the rater guidelines shape Google's systems indirectly. Demonstrate, never claim:
- **Experience** (added to the acronym in 2022): first-hand evidence — original photos, test data, screenshots, "we measured"
- **Expertise**: author byline with verifiable credentials, linked to a bio page with history
- **Authoritativeness**: citations to and from recognized sources; being referenced elsewhere
- **Trust**: contact page, about page, HTTPS, claims with sources

Decisive for YMYL (health, finance, legal, safety): weak E-E-A-T there means no ranking regardless of content quality.

## AI-Assisted Content

Google's spam policy targets scaled content abuse — mass-producing pages to manipulate rankings — not AI use per se; AI-assisted content demonstrating E-E-A-T is acceptable per Google's own guidance. The real trap: raw model output is by construction the consensus of existing content — zero information gain. Use AI for structure and drafting; the ranking ingredients (data, experience, opinion) must come from outside the model.

## Thin and Dead Content

"Thin" is about value, not word count — a 150-word page that fully answers its query is not thin; a 1,500-word page saying nothing is. Decision rule for a page with no impressions and no links after 6-12 months indexed, in order:

1. **Improve** — if the topic deserves a page and you can add information gain
2. **Consolidate** — 301 into the closest strong page, merge the unique parts
3. **Remove or noindex** — if neither applies

Sitewide quality reassessments judge the whole domain: dead weight drags your winners.

## Freshness

- Query Deserves Freshness: time-sensitive topics need genuinely recent content
- "Best X [year]" queries: update the content, then the date — date-only bumps without changes are detectable and treated as deceptive
- Refreshing a decayed winner (traffic well down from its peak) usually beats writing new: existing links and history compound
- Evergreen content: no dates in the URL, revisit on a schedule

## Structure

- Answer the query in the first ~100 words — featured snippets and AI Overviews both extract answer-first pages
- Question-format H2s matching People Also Ask phrasing → snippet and PAA capture
- Lists and tables for procedural and comparison queries — the formats snippets extract
- Short paragraphs (2-3 sentences); table of contents on long pieces
- FAQ section at the bottom for remaining PAA queries (markup in `schema.md`)
