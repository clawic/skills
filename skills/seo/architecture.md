# Architecture — Structure, Clusters, and Index Discipline

Site architecture decides two things Google reacts to: what it can reach cheaply, and what it believes the site is about. Both are decided by where pages sit and how they link, not by how the navigation looks.

## The Three Structural Rules

1. **Depth**: every page that should rank is within three clicks of the homepage. Depth correlates with crawl frequency and with how much internal signal a page receives.
2. **Grouping**: URLs live in folders that match topics (`/guides/email-marketing/`, `/tools/`, `/products/`). The folder is a reporting dimension in Search Console and a signal of scope.
3. **Consolidation**: one URL per intent. Every extra near-duplicate divides links, impressions, and crawl attention.

## Flat vs Deep

- Flat (everything one level down) breaks above a few hundred URLs: it destroys topical grouping and makes every page equally unimportant.
- Deep folder trees look tidy and starve leaf pages. Depth is measured in **clicks**, not slashes: a five-slash URL linked from the homepage is one click deep.
- Fix depth with hub pages and contextual links, not by shortening URLs.

## Topic Clusters

The working version of "topical authority", stripped of mysticism:

- **Hub page**: targets the head term, explains the whole topic, links to every spoke.
- **Spokes**: one page per distinct sub-intent, each linking back to the hub and to two or three sibling spokes where genuinely relevant.
- The value comes from real coverage plus reachable links, not from the diagram. Twelve thin spokes are worse than three complete ones.
- Build the cluster around what searchers actually ask, and merge sub-intents Google already answers on one page — its SERP is the arbiter of how finely to split.
- Commercial clusters need a path to money: every hub should link to the product, service, or comparison page it justifies.

## Internal Link Distribution

- Links in main navigation and footers appear on every page and therefore say little about relative importance; contextual body links are the ones that differentiate.
- Give the pages that make money more internal links than the ones that do not. This is the only "authority sculpting" that works, and it costs nothing.
- Every new page gets links from two or three strong, topically related pages on the day it publishes. Orphans get crawled late and rank late.
- Audit periodically: pages with traffic but few internal links are the fastest wins; pages with many internal links and no traffic are wasted signal.
- Breadcrumbs give Google an explicit hierarchy and give the SERP a cleaner display line.

## Index Discipline

Every indexable URL should have a reason to exist. Common bloat, and the disposition each deserves:

| URL type | Default disposition |
|---|---|
| Tag and author archives with one or two items | Noindex, or delete the taxonomy |
| Paginated archive pages beyond page 1 | Indexable but self-canonical; never canonical to page 1 (Google dropped support for `rel=next/prev` as an indexing signal, and canonicalizing hides deep items) |
| Search results pages of your own site | Noindex; they are the textbook low-value URL |
| Filter and sort combinations | Canonical to the clean URL; block combinatorial explosions from crawling |
| Print, AMP-legacy, and duplicate mobile URLs | Consolidate to one canonical URL |
| Thin location or template pages with no unique content | Consolidate or write real content |
| Utility pages (cart, thank-you, account) | Noindex |

The test for any bloat question: "if this URL got a visitor from Google, would it be a good landing page?" No means it should not be indexable.

## Subdomain vs Subfolder

- Google states it can handle both. Migration case studies keep showing subfolder gains, and the honest reading is that consolidation plus a relaunch's links explains much of the movement.
- Default for new builds: subfolder (`example.com/blog/`), because reporting, internal linking, and shared authority are simpler.
- Legitimate reasons for a subdomain: a genuinely separate product with its own audience, a hosted platform that cannot serve from a path, or regulatory separation.
- Do not migrate a working subdomain for this reason alone; the migration risk exceeds the expected gain.

## New Site From Zero

- Domain: short, brandable, and permanent. An exact-match keyword domain buys almost nothing now and constrains the brand forever; an expired domain with history is a policy risk (expired domain abuse) unless you continue its actual subject.
- Decide the URL taxonomy before publishing anything. Renaming folders later costs a migration.
- Ship the money pages and the top-of-cluster hubs first, not thirty blog posts. A new site's first job is to be findable for its own name and its clearest commercial intents.
- Expect months before competitive queries move: a new domain has no links, no history, and no coverage. Plan the first two quarters on long-tail and brand.
- Set up Search Console and analytics before launch, so the baseline exists.
- Do not buy links to accelerate. A new domain with a sudden link profile is the easiest pattern in the world to spot.

## Navigation Design

- The main nav is a statement of what the site is about; keep it to the sections that should rank.
- Mega-menus linking to hundreds of URLs flatten importance and add noise to every page. Link the section hubs, let hubs link the rest.
- Faceted navigation must be crawl-controlled by design, not patched later.
- Related-content modules generated by real relevance beat "recent posts" widgets, which link the newest pages regardless of relevance.
- HTML sitemaps are a fallback for discovery, not a substitute for a navigable structure.

## Architecture Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Building the cluster diagram before checking the SERPs | Google may answer four of your "sub-intents" on one page | Let the SERP decide the split |
| Publishing spokes with no hub | Nothing consolidates the topic; the pages compete | Ship the hub first, or at the same time |
| Reorganizing URLs for tidiness | Every move risks ranking for a cosmetic gain | Restructure only when depth or duplication actually costs traffic |
| Noindexing pagination | Deep items lose their discovery path | Self-canonical paginated pages, indexable |
| One page per keyword variant | Cannibalization by design | One page per intent |
| Treating internal links as decoration | It is the only authority lever you fully control | Plan link placement with the page, not after |
