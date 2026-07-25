# Ecommerce — Categories, Facets, Variants, and Stock

Ecommerce SEO is mostly URL governance. A catalog of 5,000 products can generate millions of crawlable URLs through filters, sorts, and variants; the sites that win are the ones where the crawler only meets pages worth indexing.

## Which Page Type Earns What

| Page type | Intent it wins | Optimization priority |
|---|---|---|
| Category / collection | Head and mid-tail commercial ("running shoes", "waterproof hiking boots") | Highest — these carry the volume |
| Filtered category worth indexing | Long-tail commercial with real demand ("women's waterproof hiking boots size 8" only if searched) | Selective: index a curated whitelist, block the rest |
| Product page | Brand plus model, long-tail specifics | Unique content, structured data, reviews |
| Brand pages | "Brand + category" queries | Cheap wins on most catalogs, usually neglected |
| Comparison and buying guides | Commercial investigation ("best X", "X vs Y") | Feeds internal links into the categories |
| Search results pages | Nothing | Noindex |
| Cart, checkout, account | Nothing | Noindex |

Most catalogs under-invest in category pages and over-invest in product descriptions. Category pages are where the volume is.

## Faceted Navigation

The decision is per facet, decided once, and enforced in the template:

| Facet type | Example | Treatment |
|---|---|---|
| High-demand, searched | color, size-as-a-category, gender, material | Indexable static URL, unique H1 and copy, linked from the parent category |
| Combinable but not searched | color + size + price | `rel=canonical` to the clean category, and keep links out of the crawl path |
| Sorting and view options | `?sort=price`, `?view=grid` | Canonical to the clean URL; never linked as crawlable anchors |
| Infinite combinations | multi-select filters | Block crawling of the parameter pattern; a canonical does not save crawl budget, it only fixes indexing |
| Anything else | — | Default to non-indexable until search demand is proven |

Whitelist beats blacklist: pick the handful of facet URLs with proven demand, make them real pages, and treat everything else as a UI state. Verify the result in log files — facets are the number one crawl-budget sink on large catalogs.

## Product Pages

- Manufacturer descriptions are duplicated across every retailer selling the item. Add what only you have: your photos, sizing notes, comparisons, in-house reviews, Q&A, real availability.
- Product schema with price, currency, availability, and genuine reviews is the highest-value markup on any catalog; generate it from the same data source the page renders from, so it can never drift.
- Reviews are content and conversion at once; they also generate the long-tail phrasing customers actually search.
- Thin product pages at scale are an index-bloat problem: for a catalog of near-identical SKUs, consider one indexable page per model with variants selectable on-page.

## Variants

- Default: one indexable URL per **model**, with size and color selectable on the page. Per-variant URLs multiply near-duplicates and split signals.
- Exception: variants with genuine independent demand ("black iPhone 15 case") deserve their own indexable URL — proven by search volume, not by the catalog's structure.
- Variant parameters (`?variant=123`) must canonical to the model URL.
- Model the relationship with `ProductGroup`/`hasVariant` rather than shipping duplicate Product blocks.

## Out Of Stock And Discontinued

| Situation | Do |
|---|---|
| Temporarily out of stock | Keep the URL live and indexable, show stock status, offer alternatives and a notify option. Removing it discards accumulated ranking |
| Permanently discontinued, replacement exists | 301 to the replacement product |
| Permanently discontinued, no replacement, has links or traffic | Keep the page, mark it discontinued, link to the category and closest alternatives |
| Permanently discontinued, no value | 410 |
| Seasonal product returning next year | Keep the URL live year-round; rebuilding it each season starts from zero |

Never let a product 404 silently — catalogs leak traffic through deletions nobody logs.

## Category Page Content

- The copy exists for the query, not for word count: a short intro above the grid answering what the category covers, and deeper buying guidance below the grid where it does not push products off the screen.
- Unique H1 per category, matching the phrasing searchers use rather than the internal taxonomy name.
- Link from the category to its top products and to its sibling categories; this is how deep products get crawled.
- Pagination: indexable, self-canonical pages with real links. Load-more buttons without crawlable page URLs make everything past page 1 invisible.

## Feeds And Shopping Surfaces

- Merchant feed data and page markup must agree on price, availability, and identifiers; mismatch causes suppression in shopping surfaces even when the organic page is fine.
- GTIN/MPN/brand identifiers improve matching in shopping and merchant listings.
- Free listings and organic results are separate systems with separate diagnostics; a feed problem never shows up in Search Console.

## Ecommerce Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Deleting out-of-stock product URLs | Throws away ranking and links that return with the stock | Keep and mark status |
| Canonicalizing filters and calling crawl fixed | Canonicals fix indexing, not crawling; the crawler still spends the budget | Block the crawl path for combinatorial facets |
| Copying manufacturer descriptions | Identical to every competitor; nothing to rank | Add first-hand content |
| Blocking category pagination | Deep products lose their only discovery route | Indexable, self-canonical pagination |
| Publishing a new page per size and color | Near-duplicate explosion, split signals | One model URL unless demand proves otherwise |
| Judging the catalog by product-page traffic | Category pages carry the commercial volume | Report by page type |
| Ignoring internal site-search data | It is a free list of the exact product language customers use | Mine it for category and facet decisions |
