# Keyword Research & Competitive Analysis

## Research Process

1. **Seeds** — product/service terms plus customer language from support tickets, reviews, sales calls; customers' words beat internal jargon
2. **Expand** — Google autocomplete, People Also Ask, related searches, competitor sitemaps
3. **Qualify** — intent first, then the SERP reality check below, then volume
4. **Prioritize** — business value × winnability; low-authority sites start long-tail
5. **Map** — one intent per page (→ SKILL.md rule 4); variants and questions cluster onto the page that answers them

## SERP Reality Check

Do this before trusting any difficulty score. Tool "keyword difficulty" mostly measures backlinks of ranking pages — not content quality, not intent. Search the keyword and read page 1:

- All results are big-brand domains and homepages → skip, whatever the KD score says
- Forums, Reddit threads, thin pages, or stale dates on page 1 → beatable regardless of volume
- Featured snippet + AI Overview + People Also Ask + ads stacked on top → organic #1 earns far less than its rank suggests; check where organic results actually start on screen
- Page 1 mixes formats (guides and product pages) → fractured intent: Google is unsure, entry is easier, ranking is less stable

## CTR By Position

Click studies put position 1 around 25-30% CTR, falling steeply to low single digits by position 10. SERP features compress the whole curve. Traffic estimate: expected clicks ≈ volume × position CTR, then discount for features sitting above you. A #3 ranking under an AI Overview can earn less than a clean #5.

## Striking Distance

The highest-ROI keyword work. GSC → Performance → filter positions 4-15 with meaningful impressions. These pages already have relevance; they lack CTR or authority:

1. Rewrite title/meta against the snippets currently above you
2. Fill content gaps — subtopics the ranking pages cover that you don't
3. Add 2-3 internal links from your strongest related pages

Check this list before proposing any new article — improving these beats new content on effort-per-click.

## Cannibalization Check

GSC → Performance → filter by query → Pages tab. Two URLs trading impressions for one query = cannibalization. Fix: 301 the weaker into the stronger, merge unique sections, point internal links at the survivor.

## Volume Data Distortions

- Google Keyword Planner buckets ranges and merges close variants — fine for relative sizing, wrong as absolutes
- Zero-volume keywords convert: tools miss most long-tail; if autocomplete suggests it, people search it
- Seasonal terms: check Google Trends before committing — an annual average hides the cliff

## Keyword Types

**By intent:** informational ("how to", "what is") · navigational (brand names) · transactional ("buy", "price") · commercial ("best", "vs", "review") — full format mapping in `content.md`.

**By length:** head terms (1-2 words, high volume, brand-dominated SERPs) vs long-tail (3+ words, lower volume, easier entry, higher conversion). Low-authority sites earn topical authority on long-tail before head terms become winnable.

## Competitive Analysis (no paid tools)

1. `site:competitor.com <topic>` — their coverage depth per topic
2. Their sitemap.xml — full content inventory and publishing cadence
3. SERP overlap — search your 10 target queries and note who repeats: those are your true search competitors, often not who the business names
4. Gap = topics, questions, and formats they rank with that you lack; prioritize gaps adjacent to your commercial pages

## Tracking

- Weekly for money terms, monthly for long-tail
- Log rank moves against Google's announced update dates — separates "an update hit us" from "we broke something" (→ SKILL.md, Ranking Drop Triage)

## Free Data Sources

- Google Search Console — actual queries already driving impressions
- Google Trends — relative interest and seasonality
- Autocomplete, People Also Ask, related searches — real user phrasings

This skill does keyword research without external tools by reading SERPs and public Google data.
