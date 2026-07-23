# Financial Planning and Forecasting

## Planning Rhythm

| Cadence | Activity |
|---------|----------|
| Weekly | 13-week cash update (`cash.md`), AR/AP review |
| Monthly | Close, variance analysis, rolling forecast refresh |
| Quarterly | Board prep, scenario triggers reviewed, re-forecast |
| Annually | Budget, long-range plan |

## Rolling Forecast

- Always 12–18 months ahead; refresh monthly. A static annual budget is wrong by February — keep the budget as the board contract and the rolling forecast as the truth, and report both.
- Split by horizon: near months (1–3) bottom-up from signed contracts and pipeline; far months driver-based. Mixing methods per line item is how forecasts go stale invisibly.

## Driver-Based Modeling

Three to five drivers connect operations to money; more than that and nobody can falsify the model.

| Driver | Financial impact |
|--------|------------------|
| Sales reps | Revenue = ramped reps × quota × trailing attainment |
| Customers | Revenue, support cost, churn exposure |
| Engineers | Delivery capacity, payroll |
| Usage/servers | Infrastructure cost, gross margin |

- Use *trailing actual* attainment, never 100% of quota — planning at full quota is fantasy revenue with a spreadsheet's authority.
- Count only *ramped* reps: a rep hired in March produces little before summer; headcount ≠ capacity.
- Fully loaded headcount cost ≈ 1.25–1.4× base salary (taxes, benefits, equipment). Modeling raw salaries understates burn ~30%.

## Scenario Planning

| Scenario | Assumptions | Use |
|----------|-------------|-----|
| Best case | Everything works | Stretch targets, hiring ceiling |
| Base case | Realistic, some misses | Operating plan |
| Worst case | Key risks land | Contingency with triggers |

A scenario without a pre-committed trigger is a document, not a plan. Format: "If [metric] < [threshold] by [date], then [action]" — e.g. "If Q1 net-new ARR < 60% of plan by March 15, freeze hiring and cut discretionary spend." Agree on the trigger *now*, while everyone is calm; deciding during the miss guarantees a quarter of debate you can't afford.

## Budget Process

Calendar-year timeline: September strategy → October department inputs → November consolidation and trade-offs → December board approval. Starting in November produces a top-down decree with bottom-up resentment.

- Run top-down targets and bottom-up builds in parallel, then reconcile — either alone fails (no strategy vs no reality).
- Zero-base every 2–3 years, not annually (exhausting) and not never (bloat compounds silently).
- The budget is a contract with the board; the rolling forecast is your operating truth. Confusing them creates either sandbagging or credibility loss.

## Variance Analysis

Explain every variance beyond materiality — set the policy once (e.g. ±5% or a fixed dollar floor, whichever is greater) so "significant" isn't renegotiated monthly.

```
Revenue: $950K actual vs $1M plan (-5%)
Cause: Deal slipped to next month (timing)
Impact: Recovers in Q2
Action: None
```

Categories: timing (will reverse) · volume (more/less activity) · rate (price/cost change) · mix (different products/segments). Invisible distinction: a "timing" variance that repeats two months running is a volume variance in denial — reclassify it and cut the forecast.

## Key Metrics

| Metric | Formula | Read |
|--------|---------|------|
| Gross margin | (Revenue − COGS) ÷ Revenue | SaaS 70–80%+; below 70% caps your multiple |
| Burn multiple (Sacks) | Net burn ÷ net new ARR | <1.5 efficient; 1.5–2 acceptable; >2 fix before scaling spend |
| Magic number | Net new ARR (annualized) ÷ prior-quarter S&M | >0.75 scale GTM spend; <0.5 fix GTM first |
| CAC payback | CAC ÷ (monthly ARPA × gross margin) | <12 mo SMB; <18–24 mo enterprise |
| LTV:CAC | LTV ÷ CAC | >3x — but see caveat below |
| NDR | (Start ARR + expansion − churn) ÷ start ARR | >100% floor; 110–120% best-in-class |
| Rule of 40 | Growth % + FCF margin % | ≥40 healthy; trade growth for margin as you scale |

Caveat: LTV requires churn history young companies don't have — pre-Series A, LTV is extrapolated fiction; use CAC payback, which needs only months of data. Worked example: CAC $6K, ARPA $500/mo, 75% gross margin → payback = 6,000 ÷ (500 × 0.75) = 16 months — fine for enterprise, alarming for SMB.
