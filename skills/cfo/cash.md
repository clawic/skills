# Cash and Treasury Management

## 13-Week Cash Flow Forecast

The core CFO instrument. Direct method — actual dollars in and out, never the accrual P&L.

- Update every Monday. Track forecast vs actual per week; if collections miss by >10% two weeks running, your DSO assumption is wrong — fix the model, don't annotate the miss.
- Model inflows at historical collection speed, not contract terms. A "net 30" customer base that actually pays in 47 days is a 47-day base.
- Payroll is the anchor outflow: largest, most rigid, and the one whose miss is unrecoverable. Build the forecast around payroll dates.

| Week | Beginning Cash | Inflows | Outflows | Ending Cash |
|------|----------------|---------|----------|-------------|
| 1 | $500K | $100K | $80K | $520K |
| 2 | $520K | $50K | $120K | $450K |

Inflows: customer payments by expected (not invoiced) date, funding wires, credit draws, refunds. Outflows: payroll, rent, vendors, debt service, taxes, one-times.

## Runway and Default Alive

```
Runway (months) = Cash / Forecast monthly net burn
```

Forecast burn includes signed offers, committed contracts, and known one-times over the next 6 months. Worked example: $2.4M cash, trailing burn $150K/mo, three signed hires adding ~$60K/mo fully loaded → forecast burn $210K → runway 11.4 months. The trailing number said 16 — a 5-month error in the direction that kills you.

**Default alive vs default dead** (Paul Graham): project current revenue growth against forecast burn — do you reach cash-flow positive before cash hits zero? If default dead, the only options are raise, cut, or grow faster, and the choice must be made in weeks, not quarters.

**Runway thresholds** (canonical — `fundraising.md` timing keys off this table):

| Runway | State |
|--------|-------|
| 18+ months | Strong; raise only opportunistically |
| 12–17 months | Healthy; prepare the next round |
| 6–11 months | Act now: active raise or cuts that extend past 12 |
| < 6 months | Emergency: cut deep, bridge, or accept dictated terms |

## Working Capital

**Collect faster (AR):**
- Invoice the day of delivery. DSO = AR ÷ revenue × days in period; one day of DSO on $12M annual revenue ≈ $33K of cash ($12M ÷ 365).
- Early-pay discount math: offering 2/10 net 30 costs ~37% annualized (2/98 × 365/20). Offer it only when your cost of capital exceeds that or survival needs the cash — it is expensive money dressed as a courtesy.
- Annual prepay (commonly one to two months free in SaaS) converts 12 months of collection risk into day-one cash; usually the cheapest financing available.
- Credit-check new customers above a size you set; bad debt is a 100% discount.

**Pay strategically (AP):**
- Use full terms by default; paying early without a discount donates your float.
- Take vendor early-pay discounts when annualized return (same formula, reversed) beats your cost of capital.
- Never stretch critical single-source suppliers — the relationship is worth more than the float.

## Credit Facilities

| Type | Use | Cost |
|------|-----|------|
| Revolver | Working capital smoothing | Interest + unused-line fee |
| Term loan | Asset purchase | Fixed payments |
| Venture debt | Extend runway after an equity round | Interest + warrants |
| AR factoring | Immediate cash for receivables | Discount to face value |

- Venture debt is typically sized at ~25–35% of the last equity round and raised within 6–12 months of it, while metrics are fresh.
- Trap: covenants and MAC clauses mean the facility can vanish exactly when you need it. Venture debt extends a good runway; it never rescues a bad one.
- Secure facilities when you don't need them — banks lend to healthy companies. Terms sought under duress are terms you'll regret.

## Treasury Policy

Post-SVB (2023) baseline:
- Operating cash at two or more banks. FDIC insures $250K per depositor per bank; every dollar above that is unsecured credit exposure to your bank.
- Sweep excess into Treasury money-market funds or a T-bill ladder. Policy priority order: preserve capital > liquidity > yield — a CFO who chases yield with runway is confusing jobs.
- Once excess cash exceeds a few months of burn, write a one-page board-approved investment policy: permitted instruments, maturity limits, concentration limits.

## Cash Flow Patterns

- **Subscription**: predictable inflows; churn shows up in cash with a lag — the P&L sees it first, the bank account a quarter later.
- **Transactional**: lumpy; size the cash buffer to your worst historical trough quarter, not the average.
- **Marketplace**: float between collection and payout can make you cash-positive while unprofitable — never confuse float with earnings. Watch processor holdbacks; they appear precisely when volume spikes.
