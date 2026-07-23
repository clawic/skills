---
name: seo
slug: seo
version: 1.0.4
description: Runs SEO site audits, keyword research, content optimization, technical fixes, and link strategy. Use for auditing a site, diagnosing ranking drops, writing content to rank, local SEO, or schema markup.
homepage: https://clawic.com/skills/seo
changelog: Deeper ranking factors and audit playbooks
metadata:
  clawdbot:
    emoji: 🔍
    requires:
      bins: []
    os:
    - linux
    - darwin
    - win32
    displayName: SEO (Site Audit + Content Writer + Competitor Analysis)
    configPaths:
    - ~/clawic/seo/
---

All persistent data (site profiles, audit history, keyword tracking) lives in `~/clawic/seo/`. If you have data at the old `~/seo/` location, move it to `~/clawic/seo/`.

## When To Use

- Auditing a site: indexing, technical health, on-page, content, links
- Diagnosing a ranking or organic traffic drop
- Writing or optimizing content to rank for a target query
- Keyword research and competitor gap analysis
- Local SEO: Google Business Profile, reviews, citations
- Structured data for rich results
- Not for paid search (PPC) or content strategy without a search target — route to `content-marketing`

## Setup

On first use, read `setup.md` for workspace integration.

## Architecture

```
~/clawic/seo/
├── memory.md        # Site profiles, audit history, keyword tracking
├── audits/          # Site audit reports
└── content/         # SEO content drafts
```

See `memory-template.md` for structure.

## Quick Reference

| Situation | Read |
|-----------|------|
| Writing or fixing titles, metas, headers, URLs, images | `on-page.md` |
| Slow pages, crawl or index problems, CWV failing, mobile | `technical.md` |
| Deciding what to write, intent, E-E-A-T, thin content, AI content | `content.md` |
| Business with a physical location or service area | `local.md` |
| Rich results: JSON-LD, Article, FAQ, Product, LocalBusiness | `schema.md` |
| Internal links, backlinks, anchors, penalties, disavow | `links.md` |
| Picking keywords, sizing opportunity, competitor gaps, cannibalization | `keywords.md` |
| Anything else SEO | Run the audit checklist below, then route |

## Core Rules

### 1. Audit Before Prescribing
Fixed order: manual actions → indexing → intent match → technical → content → links. Diagnosing content quality on a page Google never indexed wastes the engagement.

### 2. The SERP Is the Spec
Before writing or prescribing format, search the exact query. Page 1 defines format, depth, and freshness. If page 1 is all product grids, a 3,000-word guide will not rank there at any quality level.

### 3. Improve Before You Create
Pages at position 4-15 in Search Console with real impressions are the highest-ROI work: better snippet, filled content gaps, internal links. Write a new page only when no existing URL targets the intent. Example: a page at position 8 with 1,000 monthly impressions needs a title rewrite and two internal links — not a competing new article, which triggers rule 4.

### 4. One Intent Per Page
Two URLs alternating in Search Console for the same query = cannibalization: Google splits signals and both underperform. Fix: 301 the weaker into the stronger, merge unique content. Map keywords by intent, not by string — Google clusters variants onto one page.

### 5. Technical Floor
Core Web Vitals "good" thresholds (Google, field data at p75): LCP < 2.5s, INP < 200ms, CLS < 0.1. HTTPS, mobile-first, self-referencing canonicals, clean sitemap. Technical debt caps everything content can earn.

### 6. E-E-A-T Is Signals, Not a Score
No E-E-A-T meter exists — demonstrate it: author bios with verifiable credentials, first-hand evidence (photos, data, tests), citations, contact and about pages. Decisive for YMYL topics (health, finance, legal, safety).

### 7. Links: Earn, Never Buy
Bought links risk manual actions that take months to recover from after cleanup. Safe inventory in `links.md`. Internal links are the lever you fully control — use them before any outreach.

### 8. Iterate From Search Console, Not Assumptions
Each symptom has a different fix: CTR far below the position curve (curve in `keywords.md`) → rewrite the snippet. Impressions but no clicks → intent or SERP-feature problem. No impressions → indexing or relevance problem. GSC tells you which; guessing does not.

## Ranking Drop Triage

In order — stop at the first confirmed cause:

1. GSC → Manual Actions + Security Issues. If flagged, that IS the diagnosis.
2. URL Inspection on hit pages: still indexed? Google-selected canonical changed?
3. Recent deploys: robots.txt edits, stray noindex, redirect changes shipped near the drop date.
4. Drop date vs Google's announced algorithm updates. Aligned = sitewide quality reassessment, not a page bug — fix the weakest content across the site.
5. Search the lost queries: did the SERP change format or gain features (AI Overview, more ads) that push you down visually at the same rank?
6. Backlinks: lost links on your side, or a competitor gained them.
7. None conclusive → gap-compare the pages that replaced you (`keywords.md`).

## SEO Audit Checklist

**Indexing:**
- [ ] Site indexed (site:domain.com)
- [ ] No important pages blocked in robots.txt
- [ ] XML sitemap submitted to Search Console, only canonical 200-status URLs in it
- [ ] No stray noindex on pages that should rank
- [ ] No page both robots.txt-blocked AND noindexed — blocked crawl means Google never sees the noindex, so the page can stay indexed

**Technical:**
- [ ] Core Web Vitals passing (thresholds in rule 5)
- [ ] Mobile-friendly, HTTPS with no mixed content
- [ ] No crawl errors in Search Console
- [ ] Redirect chains ≤3 hops (`links.md`)

**On-Page:**
- [ ] Unique title tags (50-60 chars), meta descriptions (150-160 chars)
- [ ] One H1 per page with keyword; proper heading hierarchy
- [ ] Images with alt text; internal links to and from the page

**Content:**
- [ ] Search intent matched (rule 2)
- [ ] No cannibalization (rule 4)
- [ ] No thin or duplicate content; dead pages improved, consolidated, or removed (`content.md`)

**Off-Page:**
- [ ] Google Business Profile complete (local businesses)
- [ ] Backlink profile checked for toxic patterns (`links.md`)

## Content Writing Process

1. **Keyword research** — target keyword, volume, SERP reality check (`keywords.md`)
2. **Intent analysis** — search the query; page 1 is the format spec
3. **Gap + gain** — cover what ranking pages cover, then add what none of them have
4. **Write** — answer the query in the first ~100 words, structure per `content.md`
5. **Optimize** — title, meta, headers, internal links from strong pages, schema
6. **Publish** — request indexing in Search Console, log in `~/clawic/seo/memory.md`, review GSC after a few weeks

## Output Gates

Before delivering recommendations or content:

- Did I search the actual query, or am I prescribing format from memory?
- Did I check for an existing page targeting this intent before recommending a new one?
- Is every number I cite from this skill's files, not improvised?
- Does each recommendation name the exact page and change ("rewrite /pricing title to include X"), not "improve content"?
- Am I promising a date? Ranking changes take weeks (recrawl + reprocessing) — state the mechanism, never a deadline.

## Traps

| Trap | Why it fails | Do instead |
|------|-------------|------------|
| Writing before checking the SERP | Format mismatch = no ranking at any quality | Rule 2: the SERP is the spec |
| New article for a query an existing page ranks 4-15 for | Cannibalization splits signals | Improve the existing page (rule 3) |
| Chasing keyword density | No density threshold exists; stuffing detection is pattern-based | Cover the topic, use variants naturally |
| noindex on a robots.txt-blocked page | Google never crawls it, never sees the noindex | Allow crawl until deindexed, then block |
| Changing URLs without 301s | Links and authority now point at 404s | Map every old URL to its closest new match |
| Buying links or PBNs | Payment and network footprints → manual action | Earn links via assets and digital PR (`links.md`) |
| Reporting rank without checking the rendered SERP | #1 under an AI Overview and 4 ads earns a fraction of historical #1 clicks | Check pixel position, not just rank |

## Where Experts Disagree

- **Word count**: studies show correlation between length and ranking; nobody serious claims causation. Cover the intent fully, then stop — padding is a negative.
- **Disavow**: Google says it is unnecessary without a manual action; some practitioners still disavow after obvious spam attacks. Default: don't, unless a manual action names links.
- **Exact-match anchors**: they move rankings in tests AND they are the first pattern link-spam systems check. The disagreement is the safe ratio, not the risk — keep exact match a small minority of external anchors.

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/seo (install if the user confirms):

- `content-marketing` — Content strategy
- `analytics` — Traffic analysis
- `market-research` — Competitive analysis
- `html` — HTML optimization
- `web` — Web development

## Feedback

- If useful, star it: https://clawic.com/skills/seo
- Latest version: https://clawic.com/skills/seo

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/seo.
