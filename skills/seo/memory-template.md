# Memory Template — SEO

File formats for everything under `~/Clawic/data/seo/`. `config.yaml` is what the user **declared**; `memory.md` is what the agent **observed**. An observation never overwrites a declaration.

**Contents:** [config.yaml](#clawicdataseoconfigyaml) · [memory.md](#clawicdataseomemorymd) · [Status Values](#status-values) · [Audit Report Template](#audit-report-template)

## `~/Clawic/data/seo/config.yaml`

Declared preferences only — what the user stated, never what the agent inferred.

```yaml
site_type: ecommerce        # blog | ecommerce | saas | local | news | directory | auto
target_market: en-GB        # locale used for SERP checks and spelling
tool_access: gsc-only       # gsc-only | paid-suite
risk_posture: conservative  # conservative | standard | aggressive
cms: shopify                # wordpress | shopify | webflow | wix | headless | other | auto
voice_file: voice.md        # long-form style guide in this folder; none if unset

# Preference areas — keys added as the user states preferences
reporting:
  depth: priorities-only
  kpi: revenue
conventions:
  title_format: "{page} | {brand}"
scope:
  off_limits: ["/careers/", "/legal/"]
implementation:
  mode: specs               # specs | patches
measurement:
  exclude_branded: true
cadence:
  gsc_review: monthly
```

## `~/Clawic/data/seo/memory.md`

```markdown
# SEO Memory

## Status
status: ongoing
last: YYYY-MM-DD

## Sites

### [site-name]
- Domain: example.com
- Type / platform / market: ecommerce / Shopify / en-GB
- Search Console access: yes/no, property type
- Last audit: [date] → audits/[file]
- Open priorities: [ranked list]
- Constraints: [who ships, release cadence, what is off-limits]

## Keyword Basket

| Query | Site | URL | Position | Checked | Note |
|---|---|---|---|---|---|
| [query] | [site] | [url] | [n] | [date] | [movement, cause] |

## Timeline

| Date | Event | Type |
|---|---|---|
| YYYY-MM-DD | [shipped / Google update / migration / drop] | [ours / Google] |

## Tried And Rejected
<!-- Recommendations the user declined, and the reason. Do not re-propose without new evidence. -->

## Notes
<!-- Site-specific context, patterns, vocabulary the business uses -->

---
*Updated: YYYY-MM-DD*
```

## Status Values

| Value | Meaning |
|-------|---------|
| `ongoing` | Still learning the site and its constraints |
| `complete` | Site profile, access, and constraints are known |

## Audit Report Template

Save to `~/Clawic/data/seo/audits/[site]-[date].md`:

```markdown
# SEO Audit — [site] — [date]

## Verdict
[One paragraph: what is capping this site, and what fixing it is worth.]

## Top 5 Fixes

| # | URL or template | Exact change | Effort | Traffic at stake | Owner |
|---|---|---|---|---|---|

## Findings By Layer
### Penalties and security
### Indexing
### Intent and content
### Technical quality
### Authority

## Baselines
- Clicks / impressions (last 28 days, and same period last year)
- Indexed pages, by section
- Core Web Vitals (field, p75): LCP [x]s, INP [x]ms, CLS [x]

## Not Verified
[Access gaps, missing data, open questions.]

## Measurement Plan
[The exact GSC or analytics view that will show whether this worked, and when to look.]
```
