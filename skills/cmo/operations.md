# Marketing Operations

## Budget: Derive, Then Benchmark

Derivation (canonical):

```
Budget = pipeline target × historical $ spend per $ pipeline, summed by channel
```

The pipeline target comes from the pipeline math in `demand.md`. No spend history yet → budget the channel tests with their kill lines (SKILL.md Rule 6) plus one owned-audience investment, and build the ratio within two quarters.

Sanity benchmarks — never the starting point: Gartner's CMO spend survey has ranged roughly 6-11% of company revenue in recent years for established companies; venture-stage SaaS routinely runs far higher because it buys growth ahead of revenue. Quoting either as "the right number" without the derivation is the tell of a template plan.

Portfolio split heuristic: ~70% proven channels / ~20% emerging / ~10% experiments. This axis is orthogonal to brand vs. activation (60/40 average, B2B ~46/54 — canonical in SKILL.md Rule 3): proven and experimental dollars each carry their own brand and activation share.

## Unit Economics the CMO Owns

- CAC payback (months) = CAC ÷ (monthly revenue per account × gross margin). Example: CAC $6,000, $500/month account, 80% margin → 6,000 ÷ 400 = 15 months.
- Common SaaS screens: payback under ~12 months is strong; beyond ~24 months is alarming outside enterprise ACVs; LTV:CAC ≥ 3 is the conventional filter. These are screens, not goals — you can pass them by cutting all demand-creation spend and starve next year's 95% (SKILL.md Rule 4).
- Blended CAC (all customers ÷ all spend) vs. paid CAC (paid-sourced only): blended hides paid inefficiency behind word of mouth. Report both; make paid decisions on paid CAC.

## Dashboards

- Executive (monthly): pipeline created and influenced by channel, CAC and payback trend, spend vs. plan, one brand-health indicator if tracked.
- Operational (weekly): stage-by-stage conversion per channel, campaign performance vs. kill lines, lead SLA compliance.
- Every metric needs a named owner and a threshold that triggers action; a metric nobody acts on is decoration.
- Brand metrics get quarterly review, not weekly — they move on brand timescales (`brand.md`).

## Team Structure

| Stage | Team | Rule |
|-------|------|------|
| Seed | One generalist doer + freelancers | Match the hire to your one proven or likeliest channel; do NOT hire a VP to go find the channel |
| Series A | Head of marketing + 2-3 specialists | Specialists in the working motion, not full functional coverage |
| Series B | CMO + managers per function (demand, content, product marketing, ops) | Product marketing is usually the most under-hired function at this stage |
| Series C+ | Full teams per function | Ops/analytics becomes its own discipline |

Interfaces: with sales — shared lead definitions (MQL contract in `demand.md`), shared pipeline number, marketing sits in pipeline reviews. With product — product marketing owns launches and win/loss; feed win/loss into positioning (`brand.md`) quarterly.

## Stack

Buy in this order: CRM → analytics → email/automation → attribution and enrichment last. Selection rules:
- Integration with the CRM beats feature lists.
- Total cost includes implementation plus the human who runs it; every tool needs a named owner or it becomes shelfware.
- Tool markets churn — evaluate current options at purchase time instead of trusting any static vendor list.

## Agencies

Use agencies for: specialized skills (paid media at scale, technical SEO audits), spike capacity, testing a channel before hiring for it. Keep in-house: strategy, positioning, daily creative voice, anything requiring tribal knowledge.

Working rules:
- The pitch team is senior; the delivery team often isn't. Bait-and-switch is the norm — name the delivery team in the contract.
- Agencies paid a % of ad spend are structurally incentivized to grow spend, not efficiency; tie fees to pipeline metrics or go flat-fee.
- Agencies run on your briefs; a vague brief buys expensive guessing (use the campaign brief in `demand.md`).
- Quarterly performance review against pre-agreed metrics, with a replacement trigger written down in advance.
