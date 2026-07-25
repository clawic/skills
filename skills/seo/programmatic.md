# Programmatic — Generating Pages At Scale Without Getting Filtered

Programmatic SEO is one page per row of a dataset: "flights from {a} to {b}", "{job title} salary in {city}", "{software} alternatives". It works when each page answers a query that exists with data nobody else has assembled. It fails as scaled content abuse when the pages exist because the template could produce them.

## The Go / No-Go Test

Answer all four before generating anything. A no on any of them means stop.

1. **Demand per row**: does anyone search this combination? Sample 30 rows and check volume, autocomplete, and whether page 1 exists at all. Demand is usually a power law — thousands of rows and a few dozen with real volume.
2. **Data you actually have**: is there something on the page besides the two variables? Prices, counts, comparisons, availability, real reviews. A template with the variable swapped and nothing else is a doorway page.
3. **Uniqueness per page**: what percentage of the rendered page differs between two neighboring rows? Below roughly a third and the pages are near-duplicates competing with each other.
4. **Maintenance**: who updates the data when it goes stale, and how? Stale programmatic pages decay faster than they were built.

## Building The Page Type

- Design one page manually first, get it ranking, and only then generate. A template validated by a single ranking page is worth a thousand unvalidated ones.
- The variable must appear where it matters: title, H1, URL, opening sentence, and the data itself — not sprinkled through boilerplate.
- Give the page a job beyond the keyword: a table, a calculator, a comparison, a map, a filtered list. That is the content, and the prose is the framing.
- Write the unique sections from data, not from a spinner. Sentence templates with swapped adjectives read exactly like what they are.
- Provide a real next step for the visitor: booking, filtering, contact, download. Pages with no destination convert nothing and confirm they exist only for search.

## URL And Structure

- Predictable, readable pattern: `/salary/{job}/{city}/` beats `/p?job=12&city=88`.
- One canonical order for multi-variable pages: `/flights/madrid-to-lisbon/` and `/flights/lisbon-to-madrid/` are different intents; `/a/b` and `/b/a` for a symmetric comparison are not — pick one and canonical the other.
- Hub pages per dimension (`/salary/{job}/`, `/salary/city/{city}/`) so leaves are reachable in two or three clicks and are not orphaned in a sitemap-only existence.
- Cross-link neighbors: nearby cities, similar roles, adjacent alternatives. This is the crawl path and, on these page types, most of the internal link graph.

## Rollout Discipline

- Publish in waves — hundreds, not tens of thousands — and only if the previous wave got indexed and earned impressions. Indexing rate is the honest quality signal on this page type.
- Order waves by demand: highest-volume rows first, so the template proves itself on winnable queries.
- Kill rows that fail: no impressions after a few months of being indexed means the row had no demand or the page has nothing to offer. Prune it rather than accumulating dead weight that drags sitewide quality. Deliberate exception to the 6-12 month window an editorial page gets before pruning: a template row costs maintenance forever, cannot be improved individually, and a mass of demand-free rows is exactly the scaled-content-abuse pattern.
- Watch the Page indexing report by section. "Crawled — currently not indexed" climbing across a wave is Google's verdict on the template, delivered before any ranking data.

## Where It Crosses The Policy Line

Google's scaled content abuse policy is about purpose and value, not about automation: pages mass-produced primarily to rank, offering little to a visitor, whether written by a model, a spinner, or an intern. The tells that get sites caught:

- Thousands of URLs published on one day with no demand behind most of them.
- Pages whose only difference is the substituted variable.
- Location or "{keyword} in {city}" pages for places the business does not serve — the textbook doorway pattern.
- Datasets scraped from a competitor and reformatted.
- No internal reason the pages exist besides search.

`risk_posture: conservative` (the default) means: prove demand, hold the wave size small, and never generate location pages for markets the business cannot actually serve.

## Programmatic Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Generating the full dataset on day one | The bad rows outnumber the good and define the site's average | Waves, ordered by demand |
| One template, one paragraph, one variable | Near-duplicates; Google indexes a fraction and trusts none | Data-driven unique blocks |
| Skipping the manual prototype | You scale a page type nobody proved can rank | One page ranking first |
| City pages for cities you do not serve | Doorway pages, an explicit policy violation | Only real service areas, with real local content |
| Sitemap-only discovery | Leaves with no internal links get crawled late and dropped early | Hub pages and neighbor links |
| Never pruning | Dead rows accumulate into a sitewide quality problem | Scheduled review; delete or consolidate what earns nothing |
| Measuring by pages published | Publishing is not the outcome | Measure indexed rate, impressions per row, and conversions |
