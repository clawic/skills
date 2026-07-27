# Platforms — Choosing One, and Moving Off One

Platform choice is a **total cost of ownership** question with a switching cost attached, not a feature comparison. The feature lists converge; the costs, the constraints and the exit do not.

**Before recommending or migrating**, read `## Store` (current platform and stack) and `~/Clawic/data/finances/subscriptions.md` (what the current stack really costs per month). Most "we need to replatform" conversations are app-stack problems wearing a platform costume.

## Total Cost of Ownership

```
Monthly TCO = plan fee + Σ app subscriptions
            + revenue × (payment fee % + platform transaction fee %)
            + hosting/infra + maintenance hours × loaded rate
            + amortized build or migration cost
```

- The **platform transaction fee** charged for not using the platform's own processor is the line that surprises stores: it stacks on top of the processor's fee and scales with revenue, so a saving on plan tier can be erased by it several times over.
- The app stack is usually the largest and least-reviewed line. Quarterly review of `subscriptions.md` typically finds tools nobody has opened in months (`## Due`).
- Self-hosted trades subscription cost for maintenance, security and uptime work. Price that at a real hourly rate, including the hours that arrive at 2am, or the comparison is fiction.
- Compare TCO at **projected** revenue, not current: fee percentages make hosted platforms cheap early and expensive later, which is exactly when a migration is hardest.

## Choosing

| Situation | Default | Escape hatch |
|---|---|---|
| First store, no developer, needs to be live this month | Hosted SaaS platform on the entry tier | — |
| Content-heavy brand, existing site, technical help available | Self-hosted open-source store bolted to the existing site | Moving to hosted the first time a security update is missed |
| Complex catalog: many variants, configurable products, B2B price lists | A platform whose data model natively supports it, hosted or not — the fight is against the model, not the theme (`b2b.md`) |
| High order volume, custom fulfillment or pricing logic | Hosted commerce with a headless storefront, keeping checkout on the platform | Full custom only when a business rule genuinely cannot be expressed |
| Marketplace-first business | The marketplace plus the simplest possible own store | An own store built before there is demand for it |
| Multi-country, multi-currency, multi-brand | A platform with native multi-store, not one store with translation apps | — |

Questions that separate the candidates faster than feature lists: what does checkout customization cost in scope terms; can product data be exported completely, including images and metafields; what does the tax and invoicing story look like in the home market (`tax.md`); how many of the required apps are single-vendor dependencies; and what happens to the store if the platform doubles its price.

## Headless: The Actual Boundary

- Headless buys presentation freedom, a shared content layer and independent front-end deploys. It costs a front-end team, a build pipeline, preview infrastructure, and the loss of every app that renders on the storefront.
- **Keep checkout on the platform** even when the storefront is headless. Custom checkouts buy conversion in theory and cost PCI scope, wallet support, fraud tooling and tax edge cases in practice (`payments.md`).
- For a single-region store under a few thousand SKUs, headless usually buys latency improvements smaller than what a well-tuned themed store achieves with image and caching work (`storefront.md`).
- The honest test: name the concrete thing the current storefront cannot do, and price it. If the answer is "it would be cleaner", that is not a business case.

## Migration: The Order That Avoids Losses

A replatform is a project (`~/Clawic/data/projects/<project>.md`), not a task. Sequence:

1. **Inventory what exists**: products with variants and images, customers, order history, discount codes, gift-card balances, subscriptions, reviews, CMS pages, redirects already in place, and every integration with credentials (as pointers).
2. **Export and validate before building**: row counts by entity, spot-check the weird ones (bundles, digital products, products with 50 variants). Anything the export cannot produce must be recreated by hand — find that out now.
3. **Build and load into a staging store**, then reconcile counts entity by entity against the export.
4. **Build the redirect map** old URL → new URL for every product, collection, page and blog post that has ever ranked or been linked. This is the single highest-value artifact of the migration (`artifacts/`).
5. **Rebuild tracking and consent** from the tracking plan, and verify events fire on the staging store — analytics is broken on the day of cutover more often than the store is (`analytics.md`).
6. **Re-integrate**: payments, shipping rates, tax engine, ESP, helpdesk, feeds, marketplaces. Test the failure paths, not the happy paths (`payments.md`).
7. **Cutover in a low-traffic window**, never in the run-up to peak (`peak.md`). Lower DNS TTL in advance (`domains/`).
8. **Watch the first 72 hours**: orders per hour against the same weekday last month, 404 rate, checkout errors, feed disapprovals, index coverage.

What migrations lose, in frequency order: **URLs and rankings** (no redirect map), **gift-card and store-credit balances** (rarely exportable — reconcile manually and honour them), **subscription contracts and their payment mandates** (often require re-authorization, so plan the customer communication), **review content**, **customer passwords** (never migratable — plan a reset flow), and **historical order data needed for tax retention** (`tax.md`).

## What Not to Migrate For

- A theme you could rebuild on the current platform for a fraction of the cost
- A single missing feature that an app or a small custom build solves
- A performance problem caused by apps and unoptimized images (`storefront.md`)
- An agency recommendation with no TCO comparison attached
- "The platform feels limiting" without a named, priced constraint

Migrating for a real reason is fine; the point is that the reason is usually the third item on the list, not the first one named.

## Infrastructure, When It Is Yours

Self-hosted stores are a machine plus an obligation:

- The host belongs in the shared inventory `~/Clawic/data/servers/servers.md` with its role, cost in its own currency and an access reference that is a pointer, never a key (`memory-template.md`).
- Non-negotiables: automated daily backups **with a restore that has been timed**, staging that matches production, updates on a schedule rather than an incident, and a WAF or CDN in front.
- The store domain, its registrar and expiry go in `~/Clawic/data/domains/domains.md`, with the renewal as a `## Due` row a month ahead. An expired store domain is a total outage with a slow fuse.
- Deep infrastructure work: `server`, `docker`, `linux`.

**Write after platform work**: platform, plan, stack and hosting into `## Store`; every recurring tool cost into the shared `~/Clawic/data/finances/subscriptions.md` with its currency; a migration or launch as a project file in `~/Clawic/data/projects/`; the domain into `~/Clawic/data/domains/domains.md` with its renewal in `## Due`; a self-hosted machine into `~/Clawic/data/servers/servers.md`; and the redirect map, the cutover checklist and the platform decision with what was rejected into `artifacts/<kebab-name>.md` with their `## Boxes` lines (`memory-template.md`).
