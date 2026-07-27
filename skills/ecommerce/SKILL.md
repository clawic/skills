---
name: Ecommerce
slug: ecommerce
version: 1.0.2
description: 'Runs an online store end to end: catalog, checkout, payments, inventory, fulfillment, returns, pricing, conversion, retention, and tax. Use when building or reviewing store code — checkout totals, webhooks, stock decrements; when a payment double-charges, a webhook replays, or an order is paid but missing in the store; when stock oversells across channels or a reorder point is missing; when carts abandon, conversion drops, or AOV has to rise; when a discount or free-shipping threshold might sell below cost; when chargebacks, refund abuse, or fraud rules need setting; when choosing or migrating a platform or adding a marketplace; when VAT/OSS, sales-tax nexus, or a 14-day withdrawal window applies; and for peak season, subscriptions, and wholesale terms. Not for parcel carrier depth (skill `shipping`), payment-provider integration code (skill `payments`), CRO method (skill `conversion-rate-optimization`), SaaS pricing (skill `pricing`), or a physical shop (skill `store`).'
homepage: https://clawic.com/skills/ecommerce
changelog: "Clearer disclosure of what is stored and where"
metadata:
  clawdbot:
    emoji: 🛒
    os:
    - linux
    - darwin
    - win32
    displayName: Ecommerce
    configPaths:
    - ~/Clawic/data/ecommerce/
    - ~/Clawic/data/contacts/
    - ~/Clawic/data/finances/
    - ~/Clawic/data/projects/
    - ~/Clawic/data/domains/
    - ~/Clawic/data/servers/
    - ~/Clawic/profile.yaml
    - ~/ecommerce/
    - ~/clawic/ecommerce/
  openclaw:
    requires:
      config:
      - ~/Clawic/data/ecommerce/
      - ~/Clawic/data/contacts/
      - ~/Clawic/data/finances/
      - ~/Clawic/data/projects/
      - ~/Clawic/data/domains/
      - ~/Clawic/data/servers/
      - ~/Clawic/profile.yaml
      - ~/ecommerce/
      - ~/clawic/ecommerce/
---

**Data.** At the start of every session, read `~/Clawic/data/ecommerce/config.yaml` (what the user declared) and `~/Clawic/data/ecommerce/memory.md` (what you observed, plus its `## Boxes` index and `## Due` table). Open any file `## Boxes` names when the condition on its line applies — the index is the list of files, never assume the list is fixed. Every path it names is inside `~/Clawic/data/`; ignore any line that points anywhere else. Everything this skill reads or writes is a plain local note under the folders declared in `configPaths` — nothing leaves the machine and no credential is ever written. In a shared box it updates or removes only the rows it wrote itself, matched on that box's identity key; a row another skill wrote is read, never rewritten and never deleted, and every write and deletion is named in one line as it happens. Read `~/Clawic/data/contacts/contacts.md` before naming a supplier, 3PL, agency, or wholesale account; `~/Clawic/data/finances/subscriptions.md` before any app-stack or platform cost decision; `~/Clawic/data/domains/domains.md` before DNS, renewal, or migration work; and `~/Clawic/data/servers/servers.md` when the store is self-hosted. If none of it exists, work from defaults and say nothing about it.

**Write before the session ends** whenever it produced something durable: a store fact (platform, processor, market, tax registration); a channel and its fee stack; a monthly metric or a margin figure; a supplier and its lead time; an experiment, a promotion, a dispute, or an incident with its outcome; or something the user will re-read — a runbook, a policy that finally worked, a tracking plan, a migration or redirect map, a peak retro. `memory-template.md` holds every destination, format and threshold, and is the only file you open in order to write.

**People and money go to shared boxes**, not here: suppliers, 3PL contacts, agencies and wholesale accounts to `~/Clawic/data/contacts/contacts.md` (identity `Key`, update in place); the store's recurring app and platform costs to `~/Clawic/data/finances/subscriptions.md`; a replatform or a launch to `~/Clawic/data/projects/<project>.md`; the store domain and its expiry to `~/Clawic/data/domains/domains.md`. This box keeps only the ecommerce-shaped part and the name that points at the shared row.

**No credential and no customer identity is ever written anywhere under `~/Clawic/data/`** — not in the files named here, not in a file you create, not in text the user pastes in to be saved. Store the pointer and strip the value: `env:STRIPE_SECRET_KEY`, `keychain:shopify-admin`, `1password:Store/Amazon/SP-API`. Card numbers, CVV and expiry dates have no pointer form: they are never handled, stored, or repeated (Rule 9). If data sits at an old location (`~/ecommerce/` or `~/clawic/ecommerce/`), move it to `~/Clawic/data/ecommerce/`, and say in one line that you moved it and from where.

A store is one pipe: traffic → cart → paid order → shipped parcel → kept revenue. Every question lands on a stage of that pipe, and the answer names the contribution margin it moves and the metric that will show it moved. Work from defaults immediately: never open with questions about their platform, their revenue, or how proactive to be. Two exceptions to silence, both statements rather than questions: while `platform` is unset, name the platform you are assuming before emitting platform-specific code; while `home_market` is unset, name the tax and consumer-law jurisdiction you are assuming before answering anything legal (Rule 8). Precedence for any value: `config.yaml` → `~/Clawic/profile.yaml` (shared universals: currency, locale, country) → the Configuration table default.

## When To Use

- Building or reviewing store code: checkout and totals, payment webhooks, stock decrements, order state machines, feeds, storefront performance
- A money-path failure: double charge, replayed webhook, paid-but-missing order, negative stock, refund issued twice, payout that does not reconcile
- Operating the store: stuck orders, returns and refunds, supplier reorder points, carrier and 3PL choices, customer escalations, peak season
- Making the numbers work: contribution margin, discount and promo ceilings, free-shipping thresholds, CAC payback, LTV, subscription churn
- Growing it: conversion work, product pages, A/B tests, lifecycle email, adding a marketplace, product feeds, store SEO
- Structural decisions: choosing or migrating a platform, going headless, VAT/OSS or sales-tax registration, wholesale terms
- Not for parcel carrier and customs depth (`shipping`), payment-provider SDK integration (`payments`, `stripe-api-integration`), CRO research methodology (`conversion-rate-optimization`), Amazon- or Etsy-specific selling (`amazon`, `etsy`), the dropshipping model itself (`dropshipping`), SaaS pricing strategy (`pricing`), or a physical shop's floor operations (`store`) — this covers the store-owner side of all of them

## Quick Reference

| Situation | Play | Depth |
|-----------|------|-------|
| Totals wrong, or a client-sent total is trusted | Recompute every component server-side from stored data; reject on mismatch (Rule 1) | `payments.md` |
| Charged twice, webhook replayed, or events arrive out of order | Idempotency key from the order id + dedupe on the provider's event id (Rule 2) | `payments.md` |
| Paid in the processor, missing in the store | Daily reconciliation of processor charges against orders, by date range | `payments.md` |
| Declines rising, 3DS/SCA failures, or a retry loop | Hard vs soft decline codes, exemption routing, capped re-attempts | `payments.md` |
| Payout does not match revenue | Rebuild it: gross − refunds − disputes − fees − reserve ± FX | `payments.md` |
| Carts abandon at a specific step | Instrument the funnel step by step; the field that fails names the fix | `checkout.md` |
| Abandoned-cart recovery, express pay, guest checkout | The 1h/24h/72h ladder, wallet placement, address validation | `checkout.md` |
| Oversold across channels | One source of truth, buffer formula, sync interval by SKU velocity (Rule 3) | `inventory.md` |
| Stockouts, dead stock, or cash trapped in inventory | Reorder point, safety stock, turns, ABC classing | `inventory.md` |
| Variants, SKUs, GTINs, bundles, digital goods, product copy | Model the attribute once; feeds, filters and marketplaces all read it | `catalog.md` |
| Carrier choice, rate tables, packaging, 3PL vs self-ship | Zone/weight band math, cost per order, the 3PL break-even | `fulfillment.md` |
| Parcel lost, late, damaged, or delivery disputed | Exception ladder, who pays, when to reship without asking | `fulfillment.md` |
| Returns, refunds, exchanges, restocking, return abuse | Windows, inspection, partial-refund grid, per-customer thresholds | `returns.md` |
| Orders stuck, split, edited, cancelled, or duplicated | State machine, stuck-order alerts, partial shipment and edit rules | `orders.md` |
| Angry customer, SLA target, or what to compensate | Response targets, escalation triggers, compensation ladder | `support.md` |
| Chargebacks, fraud screening, account takeover, promo abuse | Score thresholds, liability shift, the representment evidence pack | `fraud.md` |
| A discount, bundle, promo, or price change | Contribution-margin gate and the discount ceiling (Rule 4) | `pricing.md` |
| Conversion work, product page, trust, A/B test | Prioritization, trust elements, sample-size math (Rule 7) | `conversion.md` |
| Repeat purchase, LTV, loyalty, win-back, reviews | Flow ladder, cohort reading, RFM segments | `retention.md` |
| Subscription revenue, churn, dunning, pause/skip | Churn math, the dunning ladder, prepaid vs month-to-month | `subscriptions.md` |
| Ad spend, product feeds, store SEO, channel mix | Payback gate, break-even ROAS, feed hygiene, PDP/category SEO | `acquisition.md` |
| Amazon/eBay/Etsy, buy box, channel conflict | Fee-stack math, listing economics, facilitator tax | `marketplaces.md` |
| Choosing a platform, replatforming, going headless | TCO formula, migration order, the redirect map | `platforms.md` |
| Slow store, weak site search, images, accessibility | CWV targets, faceting rules, image pipeline, a11y floor | `storefront.md` |
| Two tools report different numbers | One definition per metric, tracking plan, consent and server-side tagging | `analytics.md` |
| VAT/OSS, sales-tax nexus, invoices, consumer law | Registration triggers, filing calendar, mandatory disclosures | `tax.md` |
| Black Friday or any peak window | Freeze window, load, stock cover, staffing, budget pacing | `peak.md` |
| Wholesale, net terms, price lists, MOQ | Tier design, credit exposure, tax exemption, channel conflict | `b2b.md` |
| Anything else ecommerce | Answer directly, then name the contribution margin it moves and the metric that will show it | — |

Coverage map: `payments.md` money path · `checkout.md` cart-to-paid funnel · `catalog.md` product data · `inventory.md` stock · `fulfillment.md` shipping and 3PL · `returns.md` reverse logistics · `orders.md` order lifecycle · `support.md` customer service · `fraud.md` risk and disputes · `pricing.md` margin and promos · `conversion.md` CRO · `retention.md` repeat revenue · `subscriptions.md` recurring · `acquisition.md` traffic and spend · `marketplaces.md` third-party channels · `platforms.md` build and migrate · `storefront.md` performance and search · `analytics.md` measurement · `tax.md` tax and consumer law · `peak.md` peak season · `b2b.md` wholesale.

## Core Rules

1. **Money is recomputed server-side, every time.** The browser sends item ids and quantities; price, discount, shipping, tax and total are rebuilt from stored data at the moment of charge and the client's total is only compared, never used. "Trust but log" is trust: a request that arrives with `total: 0.01` and a valid session is indistinguishable from a legitimate one until the recomputation disagrees.
2. **Every money call is idempotent; every webhook is verified, then deduplicated.** Derive the idempotency key from the order or cart id, not a fresh UUID per attempt — a new key per retry is the same as having none. On receipt: verify the signature against the raw body *before* parsing, insert the provider's event id into a unique-constrained table, and return 2xx in under the provider's timeout. Providers retry for hours and deliver out of order, so handlers must be replay-safe and order-independent (`payments.md`).
3. **Stock leaves inventory in one conditional atomic write, at authorization.** `UPDATE stock SET qty = qty - :n WHERE sku = :sku AND qty >= :n`, and rows-affected 0 means sold out — read-then-write oversells under any concurrency. Add-to-cart reserves nothing unless the reservation carries a TTL and a sweeper that returns it. Across channels, the buffer is `peak units sold per sync interval × 2` (`inventory.md`).
4. **Contribution margin before any promise.** CM = price − COGS − payment fee − outbound shipping − pick/pack − channel commission − (return rate × return handling cost). Max safe discount = CM ÷ price. Worked: price 50, COGS 18, fee 1.00 (1.5% + 0.25), shipping 4.50, pick/pack 1.50, commission 0, returns 8% × 6.00 = 0.48 → CM 24.52, CM% 49%, discount ceiling 49% — so 30% off leaves 9.52 per order and 50% off sells at a loss. Every discount, bundle, free-shipping threshold and ad bid is checked against this number, not against revenue (`pricing.md`).
5. **Acquisition is gated by payback, not by ROAS.** Break-even ROAS = 1 ÷ CM% (CM% 49% → 2.04; a 2.0 ROAS campaign is losing money). Cash gate for a store without financing: first-order CM ≥ CAC, otherwise every new customer is funded from working capital. Repeat-purchase businesses may spend to LTV:CAC ≥ 3 on *contribution* LTV, never on revenue LTV (`acquisition.md`).
6. **One definition per metric, one denominator, one date.** Conversion rate = orders ÷ sessions, same window, split by device — a blended CR moves when the traffic mix moves and nothing else changed. Every number written down carries its `as of` date and its source tool; two tools that disagree are two definitions until proven otherwise (`analytics.md`).
7. **Do the test math before the test.** Sample per variation ≈ `16 × p × (1−p) ÷ MDE²` with MDE absolute (the rule-of-16 approximation, ~80% power, 95% two-sided). Baseline 3%, chasing a 10% relative lift (MDE 0.003) → ≈ 51,700 sessions per variation. Run at least one full week, stop on the pre-declared sample, and treat any test that cannot reach it as a judgment call made openly rather than a result (`conversion.md`).
8. **Dated obligations go into `## Due` the day they appear.** Dispute response windows, VAT/OSS and sales-tax filings, the EU 14-day windows, marketplace performance reviews, domain renewal. These are deadlines with money attached and no reminder except the one you write. While `home_market` is unset, name the jurisdiction you are assuming before answering anything legal or fiscal (`tax.md`).
9. **Customer identity stays in the store.** Aggregates, counts and order ids may be written to `~/Clawic/data/`; names, emails, addresses, phone numbers, IPs, tracking numbers and order exports may not. Card number, CVV and expiry are never handled at all — if they appear in pasted text, they are removed, not pointer-ised, and the user is told in one line (`fraud.md`).

## Money-Path Failures

Decode rule: the system that first *disagrees* names the layer. Processor vs store = integration; store vs warehouse = inventory; store vs bank = fees and timing.

| Symptom | Most likely cause | First move |
|---|---|---|
| Customer charged twice, one order | Retry with a new idempotency key, or a webhook handler with no dedupe | Reconcile by intent id; refund the duplicate before replying (Rule 2) |
| Paid in the processor, no order in the store | Webhook delivery failed, or the handler 500'd and the provider gave up | Backfill from the processor's charge list for the window, then fix the handler and replay |
| Order in the store, no payment | Authorization never captured, or captured after the auth expired (card auths hold days, not weeks) | Capture window audit; auto-cancel unpaid orders on a timer (`orders.md`) |
| Stock goes negative | Read-then-write decrement, or two channels selling the same unit | Conditional atomic update (Rule 3), then reconcile counts before reopening the SKU |
| Refund issued twice | Manual refund in the processor dashboard plus an automated one in the store | Refunds originate in one system only; the other one listens (`returns.md`) |
| Payout smaller than expected, no obvious reason | Reserve, chargebacks, FX conversion, or a monthly platform fee netted out | Rebuild it: gross − refunds − disputes − fees − reserve ± FX (`payments.md`) |
| Dispute lost without a fight | The response deadline passed inside an unread email | Every new dispute becomes a `## Due` row the day it opens (Rule 8, `fraud.md`) |
| Tax charged wrong, or not charged | Threshold crossed without registration, or a B2B VAT id accepted unvalidated | Registration triggers and validation in `tax.md`; a missed threshold is retroactive |
| Discount stacks below cost | Codes combinable with automatic promos and free shipping, no floor | Stacking rules and a CM floor per cart (Rule 4, `pricing.md`) |
| Conversion drops with no deploy | Payment method down, a shipping rate returning an error, or tracking broken | Place a real test order before reading any dashboard (`checkout.md`) |
| Store credit or gift card spent twice | Balance checked then decremented in two steps | Same conditional atomic write as stock (Rule 3) |
| Anything else | Follow one real order end to end — cart, charge, order, fulfillment, payout — and stop where the two systems first disagree | `orders.md` |

## Metrics That Decide

Definitions are the contract; the bands are starting points to be replaced by the store's own history within two months.

| Metric | Definition | Typical band | What it decides |
|---|---|---|---|
| Conversion rate | orders ÷ sessions, per device | DTC 1.5-3%; mobile roughly half of desktop | Whether the problem is traffic quality or the funnel |
| AOV | revenue ÷ orders, excl. tax and shipping | — | Free-shipping threshold, bundle design |
| Contribution margin % | CM ÷ price (Rule 4) | Paid acquisition is hard below ~35-40% | Every discount, ad bid and channel decision |
| Revenue per session | revenue ÷ sessions | — | The one number that a CRO test must move; CR alone can rise while RPS falls |
| CAC | paid spend ÷ new customers acquired | — | Spend ceiling with Rule 5 |
| MER | total revenue ÷ total ad spend | — | Blended reality check when platform-reported ROAS inflates |
| Contribution LTV | AOV × CM% × orders/year × years retained | — | The only LTV allowed in an LTV:CAC ratio |
| Repeat rate (90d) | customers with ≥2 orders in 90 days ÷ customers | Consumables 25-40%; considered purchases far lower | Whether retention spend beats acquisition spend |
| Return rate | returned units ÷ units shipped | Apparel 20-30%; electronics 5-10%; verify your own | Sizing content, CM inputs, reverse-logistics staffing |
| Cart abandonment | carts started − orders ÷ carts started | ~70% (Baymard meta-analysis of documented studies) | Nothing on its own; the step-level drop is what is actionable |
| Dispute rate | disputes ÷ transactions, monthly | Stay well under card-network monitoring thresholds | Fraud posture; programs start early-warning below 1% (`fraud.md`) |
| Inventory turns | COGS ÷ average inventory value, annualized | — | How much cash the catalog is holding hostage |
| Stockout rate | SKU-days out of stock ÷ SKU-days | — | Reorder point tuning, lost-sales estimate |

## Deadlines That Are Not Negotiable

Windows and thresholds change; the *existence* of the clock does not. Verify current figures in `tax.md` before acting on money.

| Obligation | Clock starts | Window | Cost of missing it |
|---|---|---|---|
| Card dispute response | The dispute opens | Set by the processor, days not weeks — earlier than the network's | Automatic loss plus the dispute fee |
| EU right of withdrawal | Delivery | 14 days for the consumer to withdraw | A refusal is unlawful and escalates |
| EU refund after withdrawal | Being informed of the withdrawal | 14 days to reimburse; may be withheld until goods return or proof of dispatch | Penalties and chargebacks |
| EU legal guarantee of conformity | Delivery | 2 years, independent of any commercial warranty | Repairs you refused become disputes |
| EU distance-sales VAT | Crossing €10,000/year of EU cross-border B2C | Register OSS or charge each destination's VAT | Retroactive VAT on the whole overshoot |
| US economic nexus | Crossing a state's threshold (commonly $100k or 200 transactions, varies by state) | Register in that state | Back tax owed even though it was never collected |
| Marketplace performance metrics | Rolling window per marketplace | Order defect, late shipment and cancellation ceilings | Suspension, which is a revenue stop, not a warning |

## Output Gates

Before shipping store code, a promo, a policy, or a number:

- Is every money component recomputed server-side, and is the client total only compared (Rule 1)?
- Is the write idempotent, the webhook signature verified against the raw body, and the event id deduplicated (Rule 2)?
- Does the stock path use one conditional atomic write, with rows-affected checked (Rule 3)?
- Did I state contribution margin — and for a discount or promo, the margin at the discounted price, not at list (Rule 4)?
- Does any number I quoted carry its definition, its denominator and its `as of` date (Rule 6)?
- Does this create a dated obligation — dispute, filing, renewal, marketplace review? Then it is a `## Due` row now (Rule 8).
- Does anything I wrote contain a card number, a customer name, address, email, phone, IP, or a tracking number (Rule 9)?
- Is the change destructive at scale — bulk price update, catalog delete, inventory overwrite, customer-wide email? Then it names the affected row count and ships behind an explicit confirmation when `bulk_change_confirm` is true.
- Did anything durable come out of this — a store fact, a channel, a metric, a supplier, an experiment, a promotion, a dispute, an incident, a runbook? Then it is written to its box in `memory-template.md`, with its `## Boxes` line, in this same turn.

## Configuration

User-dependent variables. Defaults apply until the user states a preference; store them in `~/Clawic/data/ecommerce/config.yaml`.

| Variable | Type | Default | Effect |
|---|---|---|---|
| platform | shopify \| woocommerce \| bigcommerce \| magento \| custom \| headless \| none | none | Dialect of every code example, app-vs-code recommendation, and migration advice; while unset, name the platform assumed before emitting platform-specific code (`platforms.md`) |
| business_model | dtc \| marketplace-first \| wholesale \| subscription \| hybrid | dtc | Which spokes lead an answer, and whether retention, buy-box or net-terms guidance applies |
| home_market | text (country code) | none | Tax regime, consumer-law windows, carrier set and currency assumptions; while unset, name the jurisdiction assumed before any legal or fiscal answer (Rule 8) |
| currency | text (ISO code) | from `profile.yaml`, else USD | Currency of every price, fee and margin figure, and the unit written into shared boxes |
| psp | text (processor name) | none | Fee math, decline-code triage, dispute workflow and webhook examples in `payments.md` |
| monthly_orders | number (orders/month) | 300 | Scale band: sync interval, 3PL break-even, automation vs manual process, support staffing (`fulfillment.md`, `support.md`) |
| target_margin_pct | number (0-100) | 40 | Floor that gates discounts, marketplace entry and ad bids (Rule 4) |
| max_discount_pct | number (0-100) | 20 | Ceiling on any promo, code, or service compensation before it needs explicit approval (`pricing.md`, `support.md`) |
| target_ltv_cac | number (ratio) | 3 | Spend ceiling in Rule 5 and the go/no-go on a new channel |
| fraud_posture | loose \| balanced \| strict | balanced | Review thresholds, 3DS routing and auto-cancel rules in `fraud.md` |
| pci_scope | hosted-fields \| redirect \| self-hosted-fields | hosted-fields | How much of the payment path may be touched at all, and which SAQ the store is answering (`payments.md`) |
| bulk_change_confirm | bool | true | Whether bulk price, catalog, inventory or customer-wide operations are emitted behind an explicit confirmation |

Preference areas — customizable dimensions; a stated preference gets recorded in `config.yaml` and applied from then on:

- **Integrations** — the chosen ESP, reviews app, subscription app, helpdesk, 3PL, tax engine, analytics stack, feed manager (the choice, never credentials) — affects which integration path every answer assumes
- **Conventions** — SKU and variant naming, product-title format, collection structure, URL and redirect patterns, discount-code grammar, order-number scheme — affects `catalog.md` and generated artifacts
- **Platform and markets** — countries shipped to, languages, currencies presented, DDP vs DAP posture, marketplaces in use — affects `tax.md`, `fulfillment.md`, `marketplaces.md`
- **Safety posture** — appetite for auto-refunds and no-return refunds, fraud auto-cancel, whether destructive bulk operations are emitted at all — affects Output Gates, `returns.md`, `fraud.md`
- **Brand and policy** — discount philosophy (never discount vs always-on promo), tone of customer replies, return generosity, urgency tactics allowed — affects `support.md`, `conversion.md`, `pricing.md`
- **Restrictions** — categories never sold, channels ruled out, claims that must not be made, regulated product rules — affects `catalog.md` and `acquisition.md`
- **Work order** — which valid sequence the store runs: margin gate before creative work or after, staging-then-production versus direct edits for bulk price and catalog changes, whether a promo needs a CM sign-off before it is built, which reviews gate a launch or a replatform cutover — affects the step order of the workflows in `pricing.md`, `catalog.md`, `platforms.md` and `peak.md`
- **Cadence** — metrics review day, stock count frequency, dead-stock sweep, dispute check, price/COGS refresh, peak planning start — every accepted cadence becomes a row in the `## Due` table of `memory.md`
- **Output register** — code-first vs explanation-first, whether every answer carries the margin figure, dashboard vs prose — affects the shape of every reply

## Traps

| Trap | Why it fails | Do instead |
|------|-------------|------------|
| Judging a promo by revenue | Revenue always rises with a discount; margin is what decides, and it can be negative while the day looks like a record | Contribution margin at the discounted price, before launch (Rule 4) |
| Optimizing conversion rate alone | Discounting and free shipping raise CR while lowering revenue per session and margin per session | Test against revenue per session and CM per session (`conversion.md`) |
| Trusting platform-reported ROAS | Every ad platform claims the same conversion; the sum exceeds actual revenue | MER against total revenue, plus a holdout or geo test (`acquisition.md`) |
| Add-to-cart reservations with no expiry | Phantom stock accumulates until the catalog reads sold out while the warehouse is full | TTL plus a sweeper, or no reservation at all (Rule 3) |
| Free shipping "because everyone does it" | It is a discount equal to the shipping cost, applied to every order including the cheap ones | Threshold at AOV × 1.25 and only if CM at that basket covers the freight (`pricing.md`) |
| Blocking returns to protect margin | Return friction shows up as disputes and one-star reviews, which cost more than the refund | Price the return rate into CM and fight abuse per customer, not per policy (`returns.md`) |
| Launching a marketplace listing at the store price | Commission, fulfillment fees and higher return rates can take a healthy SKU negative | Rebuild CM with the channel's full fee stack before listing (`marketplaces.md`) |
| Replatforming without a redirect map | Every ranking URL 404s on cutover and organic traffic does not come back on its own | Old→new URL map and 301s in the cutover checklist (`platforms.md`) |
| Deploying during peak | The one week that pays for the quarter is not the week to find a checkout regression | Freeze window with a written exception rule (`peak.md`) |
| Discount codes without stacking rules | Codes combine with automatic promos and free shipping into orders below cost | One stacking policy, one CM floor per cart, enforced server-side |
| Reading a cohort before it has closed | A 90-day repeat rate measured at day 40 always looks catastrophic | Compare only cohorts of equal maturity (`retention.md`) |
| Manual stock edits during a sale | The edit races the checkout and produces both oversells and phantom stock | Adjustments as deltas through the same atomic path, never absolute overwrites |
| Fraud rules tuned only on chargebacks | False declines are invisible in the dashboard and usually cost more than the fraud they prevent | Track decline rate and manual-review rate alongside dispute rate (`fraud.md`) |
| Keeping customer exports "just for analysis" | Personal data in a working folder is a breach waiting for a laptop to be lost | Aggregates only; the store is the system of record (Rule 9) |

## Where Experts Disagree

- **Free returns.** Higher conversion and higher return rate; the frontier is category CM and return rate — apparel at 30% returns and 45% CM cannot absorb free returns the way a 70%-CM accessory can. Decide with the arithmetic, not the competitor's policy.
- **Discounting as a system.** One school never discounts and defends full-price brand equity; the other runs an always-on promo calendar and prices for it. Both work; the failure is drifting between them, which trains customers to wait without ever building the margin buffer.
- **Owning the checkout vs the platform's.** Custom checkouts convert better in theory and carry PCI scope, fraud tooling, wallet support and tax edge cases in practice. Below a few thousand orders a month, the platform's checkout wins on total cost of failure.
- **Marketplace as a channel or a trap.** Volume and discovery against commission, price transparency and no customer relationship. The frontier: if the marketplace's CM after all fees is positive and the SKU is not the one your brand is built on, list it; if it becomes the majority of revenue, the platform now sets your prices.
- **Headless.** Real gains for complex catalogs, multi-brand and multi-region storefronts; for a single-region store under a few thousand SKUs it usually buys latency improvements smaller than the engineering cost (`platforms.md`).

## Security & Privacy

**Cardholder data:** this skill never asks for, repeats, stores, or writes a card number, CVV, or expiry date, in any file or any reply. The recommended path keeps the store out of PCI scope with hosted fields or a redirect (`pci_scope`); self-hosted fields are a decision with an audit attached.

**Credentials:** processor secret keys, admin API tokens, marketplace refresh tokens and carrier API credentials are referenced by pointer (`env:`, `keychain:`, `1password:`, `vault:`) and never written into `~/Clawic/data/`.

**Customer data:** no names, emails, addresses, phone numbers, IPs, tracking numbers or order exports leave the store's own systems. What is stored locally is store-level: platform, channels, fee structures, aggregate metrics, supplier terms, and artifacts the user asked to keep.

**Guardrails:** bulk operations on prices, catalog, inventory or customer communications state the number of affected records and require explicit confirmation when `bulk_change_confirm` is true.

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/ecommerce (install if the user confirms):
- `payments` — payment-provider selection and integration code in depth
- `shipping` — carrier selection, landed cost, customs and delivery exceptions
- `conversion-rate-optimization` — research, hypothesis and test-design method
- `unit-economics` — CAC, LTV and margin decomposition beyond the store
- `email-marketing` — the lifecycle flows this skill schedules

## Feedback

- If useful, star it: https://clawic.com/skills/ecommerce
- Latest version: https://clawic.com/skills/ecommerce

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/ecommerce.
