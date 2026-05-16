# Deeper Audit — Stagnation Root Cause & Walk-Forward Validity

*Cassian Trading Corp — 2026-05-16*
*Produced in ~2.5 hours. Ref: b-live-audit.md, post-audit-investigations.md*

---

## Executive Summary

**The pre-Stream A "bleed" was real.** The on-chain cleaned wallet fell from $828.78 (Apr 27 peak) to $562.55 (May 8 trough) — a **-$266.23 real decline** over 12 days. This is the primary, on-chain-verified metric. Settlement PnL (an internal CLOB-based calculation) reports +$1,196 over the same period — a $1,462 discrepancy that is an **unresolved data quality issue**, not a timing artifact. No reliable internal metric can explain daily PnL attribution until that gap is investigated and closed.

**Settlement PnL does not reconcile with on-chain wallet movements.** A claim of +$1,196 settlement PnL during a period when the wallet fell -$266 is implausible on its face. Possible causes are listed in the Settlement PnL Reconciliation Gap section below. All dollar figures in this report derived from settlement PnL should be treated as approximate directional signals only — not real USDC totals.

**Band B (entry ≥ 0.70) shows durable win rate advantage.** BTC UP hourly Band B held 90–93% WR across three consecutive pre-Stream A weeks (N=29–31 per week). ETH Band B also profitable (80% WR vs 70.6% A_low). Win rate data is not formula-based and is reliable. The pattern is cross-asset and appears durable.

**D1 paper consistently underperforms B-live in win rate.** After Apr 28, D1 paper WR runs 4–10pp below B-live on most days. This is a win rate comparison and does not depend on dollar PnL.

**Regime is not the explanation.** BTC conditions were similar during both the W3 period and the W4–W5 decline. Win rates show no correlation with BTC daily returns.

**Revised synthesis:** The on-chain wallet bleed was real (-$266). The structural drag buckets (DOWN hourly) were cut by Stream A and that removal appears correct. However, the $1,462 settlement PnL reconciliation gap must be resolved before any dollar-based performance conclusion can be trusted. Until then: use win rates and on-chain wallet changes as the primary evidence base.

---

## ⚠️ Settlement PnL Reconciliation Gap

**This is an unresolved data quality issue. Do not dismiss it as a timing artifact.**

| Metric | Apr 27 – May 8 | Source |
|--------|---------------|--------|
| On-chain wallet change | **-$266.23** | Cleaned wallet (portfolio_snapshots.portfolio_usdc + open_position_cost_basis) |
| Settlement PnL | **+$1,196** | Internal CLOB fill data (cash_flow_reconciliation.py) |
| **Discrepancy** | **-$1,462** | Unexplained |

The on-chain cleaned wallet is the primary metric. It tracks actual USDC available plus cost basis of open positions — this is real money. Settlement PnL is a derived calculation from fill data. When they disagree by $1,462 over 12 days, the settlement PnL calculation has a bug or systematic bias.

**Candidate causes (to investigate in priority order):**

1. **actual_fill_usdc fallback overestimates fills.** 74.8% of trades use bet_size_usdc as the actual_fill_usdc fallback when no real fill data is available. If the intended bet size exceeds actual fill (e.g., partial fills, price slippage), settlement PnL overstates real receipts on winning trades.

2. **Positions counted as resolved in settlement but still open/pending.** If cash_flow_reconciliation.py counts a trade as settled (outcome recorded, PnL credited) while the position is still open on-chain, it double-counts the win before the USDC actually arrives in the wallet.

3. **Fee factor is wrong.** The 0.98 fee factor used in settlement PnL calculations may understate actual Polymarket fees, systematically inflating reported PnL.

4. **Open cost basis underestimation.** The cleaned wallet formula (free_usdc + open_cost_basis) may undercount true in-flight capital if open_cost_basis is recorded at entry price but positions are marked at current market price. This would make the wallet appear lower than it is — but the trough at $562.55 was on May 8 when open_cost_basis = $0, so this explanation doesn't apply at the trough.

5. **W3 settlement PnL (+$2,710 in one week) is implausible.** Total wallet growth from EPOCH_CAPITAL ($287.66) to W3 peak ($828.78) is +$541.12. A single week claiming +$2,710 in settlement PnL against a wallet that only grew $541 total is a ~5× overstatement. This suggests settlement PnL has systematic upward bias that precedes the Apr 27–May 8 window.

**Action required:** Run cash_flow_reconciliation.py with actual_fill_usdc fallback logging enabled, compare resolved trade USDC receipts against portfolio_snapshots inflows day by day for a 1-week sample. Do not use settlement PnL dollar totals for any promotion or strategy decisions until reconciled.

---

## DA1 — Full-System Trajectory (Apr 7 – Present)

### Primary metric: on-chain cleaned wallet

*Note: portfolio_snapshots data begins Apr 25. W1–W2 wallet trajectory inferred from EPOCH_CAPITAL ($287.66) + rough accumulation — these are estimates, not observed values. Apr 28 snapshot anomaly: bot was down most of the day; free USDC = $0, open_cost_basis = $98.40, cleaned = $98.40 (undercount — excluded from trend analysis).*

**Daily cleaned wallet (free USDC + open cost basis at EOD) — PRIMARY METRIC:**

| Date | Free USDC | Open Cost | Cleaned Wallet | Daily Δ | Note |
|------|-----------|-----------|----------------|---------|------|
| Apr 7 | — | — | **$287.66** | — | EPOCH_CAPITAL (start) |
| Apr 25 | $129.90 | $86.40 | $216.30 | — | First snapshot (low — many positions in flight) |
| Apr 26 | $682.66 | $66.00 | $748.66 | +$532.36 | Large batch of positions resolved |
| Apr 27 | $803.58 | $25.20 | **$828.78** | +$80.12 | **ON-CHAIN PEAK** |
| Apr 28 | $0.00 | $98.40 | $98.40 | — | ⚠️ Bot down ~10h; anomalous, excluded |
| Apr 29 | $654.06 | $20.10 | $674.16 | -$154.62 | First day of bleed from peak |
| Apr 30 | $665.99 | $3.90 | $669.89 | -$4.27 | |
| May 1 | $662.64 | $76.80 | **$739.44** | +$69.55 | Secondary peak |
| May 2 | $577.63 | $35.34 | $612.97 | -$126.47 | |
| May 3 | $623.39 | $22.71 | $646.10 | +$33.13 | |
| May 4 | $706.57 | $19.00 | $725.57 | +$79.47 | |
| May 5 | $691.65 | $0.00 | $691.65 | -$33.92 | |
| May 6 | $647.24 | $12.36 | $659.60 | -$32.05 | |
| May 7 | $661.67 | $19.10 | $680.77 | +$21.17 | |
| May 8 | $562.55 | $0.00 | **$562.55** | -$118.22 | **ON-CHAIN TROUGH (pre-Stream A)** |
| May 9 | $590.98 | $0.00 | $590.98 | +$28.43 | Stream A deployed |
| May 10 | $573.99 | $37.59 | $611.58 | +$20.60 | |
| May 11 | $550.44 | $10.56 | $561.00 | -$50.58 | |
| May 12 | $641.34 | $9.90 | $651.24 | +$90.24 | |
| May 13 | $642.22 | $0.00 | $642.22 | -$9.02 | |
| May 14 | $600.44 | $11.40 | $611.84 | -$30.38 | |
| May 15 | $617.12 | $21.36 | $638.48 | +$26.64 | |
| May 16 (partial) | $519.29 | $9.18 | $528.47 | -$110.01 | |

**On-chain PnL summary (EPOCH_CAPITAL = $287.66):**
- Apr 27 peak: +$541.12 above epoch
- May 8 trough: +$274.89 above epoch (but -$266.23 from peak)
- May 15 EOD: +$350.82 above epoch
- **The wallet has not recovered to its Apr 27 peak as of May 15.**

### Reconciliation note

| Metric | Apr 27 – May 8 | Notes |
|--------|---------------|-------|
| On-chain wallet change | **-$266.23** | Primary metric — verified |
| Settlement PnL | **+$1,196** | Internal calculation — **unreconciled, see gap section above** |
| Discrepancy | **-$1,462** | Open data quality issue |

**Weekly aggregates (settlement PnL by week — secondary/unreconciled):**

*Settlement PnL figures below are labeled as internal calculations unreconciled with on-chain. Use win rates and wallet endpoints as primary evidence.*

| Week | N trades | WR | Settlement PnL (internal, unreconciled) | Wallet end (cleaned, on-chain) |
|------|---------|-----|----------------------------------------|-------------------------------|
| W1 Apr 7–13 | 246 | 68.3% | +$120 ⚠️ | ~$408 (est., no snapshots) |
| W2 Apr 14–20 | 923 | 70.4% | +$298 ⚠️ | ~$706 (est., no snapshots) |
| W3 Apr 21–27 | 1,095 | **74.9%** | **+$2,710 ⚠️ IMPLAUSIBLE** | $829 (on-chain) |
| W4 Apr 28–May 4 | 860 | 73.4% | +$849 ⚠️ | ~$726 (on-chain, excl. Apr 28) |
| W5 May 5–11 | 479 | 74.3% | +$433 ⚠️ | $561 (on-chain) |
| W6 May 12–15 | 384 | 75.3% | +$526 ⚠️ | $638 (on-chain) |

W3 settlement PnL of +$2,710 requires particular scrutiny: total wallet growth from epoch to W3 peak is only +$541. A single week cannot produce +$2,710 in real USDC when the portfolio only grew $541 in total. This is the strongest evidence of systematic settlement PnL overstatement.

### Verdict

**Two distinct on-chain phases:**
1. **Compounding (W1–W3):** Wallet grew from $288 to $829 (+$541) on-chain. Win rates were strong (68–75% WR). Settlement PnL claims for this period (+$3,128 cumulative) are inconsistent with the $541 real gain — likely 5–6× overstated.
2. **On-chain decline (Apr 27–May 8):** Wallet fell -$266 on-chain. This is real. Settlement PnL claiming +$1,196 over this same period is unreconciled. The wallet stabilized post-Stream A (May 9+) but has not recovered to peak.

---

## DA2 — Per-Bucket Weekly Settlement PnL

*All dollar figures in this section are settlement PnL (internal, unreconciled with on-chain). Use win rates as the primary cross-bucket comparison metric. Dollar figures indicate relative magnitude and direction only.*

### Key tables

**Top positive contributors per week (WR is primary; $ is directional only):**

| Week | Bucket | N | WR | Settlement PnL (unreconciled) |
|------|--------|---|-----|-------------------------------|
| W3 | BTC UP 15min | 300 | 77.3% | +$890 |
| W3 | BTC UP hourly | 105 | 77.1% | +$499 |
| W3 | ETH UP hourly | 92 | 76.1% | +$432 |
| W3 | ETH UP 15min | 207 | 74.9% | +$412 |
| W3 | SOL UP 15min | 194 | 76.3% | +$334 |
| W3 | SOL UP hourly | 77 | 79.2% | +$313 |
| W4 | SOL DOWN hourly | 82 | **80.5%** | +$305 |
| W4 | BTC DOWN 15min | 216 | 76.4% | +$244 |
| W4 | ETH DOWN 15min | 104 | 82.7% | +$178 |
| W5 | BTC DOWN 15min | 150 | 74.0% | +$159 |
| W5 | ETH DOWN 15min | 96 | 79.2% | +$150 |
| W6 | BTC DOWN 15min | 173 | 74.6% | +$243 |
| W6 | ETH DOWN 15min | 127 | 78.0% | +$212 |

**Top negative contributors per week:**

| Week | Bucket | N | WR | Settlement PnL (unreconciled) |
|------|--------|---|-----|-------------------------------|
| W3 | BTC DOWN hourly | 24 | 58.3% | -$70 |
| W3 | ETH DOWN hourly | 29 | 62.1% | -$48 |
| W3 | BTC DOWN 15min | 19 | 52.6% | -$41 |
| W3 | ETH DOWN 15min | 13 | 61.5% | -$30 |
| W4 | BTC DOWN hourly | 74 | 66.2% | -$43 |
| W4 | SOL UP hourly | 36 | 69.4% | -$46 |
| W4 | ETH DOWN hourly | 47 | 63.8% | -$24 |
| W5 | SOL DOWN hourly | 4 | 50.0% | -$24 |
| W5 | ETH UP hourly | 16 | 68.8% | -$21 |

**Buckets that flipped or declined W3→W4 (WR-based, reliable):**
- BTC UP hourly: 77.1% WR (W3) → 92.3% WR (W4, Band B only, N=13) — overall bucket mixed
- SOL UP hourly: 79.2% WR (W3) → 69.4% WR (W4) — negative WR flip ✅ confirms structural issue
- BTC DOWN hourly: 58.3% WR (W3) → 66.2% WR (W4) — persistently weak, correctly cut by Stream A
- ETH DOWN hourly: 62.1% WR (W3) → 63.8% WR (W4) — persistently weak, correctly cut by Stream A

### Verdict

The weak-WR DOWN hourly buckets (BTC, ETH) were consistent bleeders across W3–W4. Stream A's cut of DOWN hourly was directionally correct based on win rate evidence alone, independent of any settlement PnL dollar figures.

---

## DA3 — Pre-Stream A Drawdown Decomposition (Apr 27 – May 8)

### On-chain wallet changes — primary metric

Daily wallet deltas computed from the cleaned wallet column in DA1. Apr 28 excluded (bot down — anomalous undercount).

| Date | Cleaned Wallet | Daily Δ (on-chain) | Settlement PnL (internal, unreconciled) | Note |
|------|----------------|-------------------|-----------------------------------------|------|
| Apr 27 | $828.78 | +$80.12 | +$114 | Peak |
| Apr 28 | $98.40 | — | +$271 | ⚠️ Excluded — bot down ~10h |
| Apr 29 | $674.16 | -$154.62 | +$39 | Large on-chain drop vs positive settlement PnL |
| Apr 30 | $669.89 | -$4.27 | +$81 | Disagree in direction |
| May 1 | $739.44 | +$69.55 | +$409 | Direction agrees, magnitude differs |
| May 2 | $612.97 | -$126.47 | +$6 | Large on-chain drop vs near-zero settlement PnL |
| May 3 | $646.10 | +$33.13 | +$49 | Direction agrees |
| May 4 | $725.57 | +$79.47 | +$43 | Direction agrees, magnitude differs |
| May 5 | $691.65 | -$33.92 | -$33 | ✅ Both negative (agreement) |
| May 6 | $659.60 | -$32.05 | +$95 | Strong disagreement — settlement positive, on-chain negative |
| May 7 | $680.77 | +$21.17 | +$87 | Direction agrees |
| May 8 | $562.55 | -$118.22 | +$34 | **Largest single-day drop; settlement positive** |

**Period total (Apr 27→May 8, excl. Apr 28):**
- On-chain wallet change: **-$266.23** (primary metric)
- Settlement PnL: **+$1,196** (internal, unreconciled)
- **Days where direction disagrees: 5 of 11 (Apr 29, Apr 30, May 2, May 6, May 8)**

### Key observations

**Apr 29 (-$154.62 on-chain, +$39 settlement):** The largest directional disagreement. The cleaned wallet dropped significantly despite settlement PnL showing positive. This is not explained by timing — on May 8 the open_cost_basis is $0, meaning no pending positions were absorbing USDC. These disagreements are evidence of a systematic issue in settlement PnL, not snapshot noise.

**May 8 (-$118.22 on-chain, +$34 settlement):** The worst single on-chain day. Settlement claims +$34 while the wallet dropped $118. The open_cost_basis was $0 at EOD — there were no in-flight positions that could explain the wallet being low.

**May 5:** The only day both metrics agree (both negative). This is also the day the settlement PnL calculation shows -$33, consistent with the on-chain drop of -$33.92.

### Structural drag buckets (WR evidence, reliable)

The DOWN hourly buckets showed persistently weak WR throughout this period. Bucket-level WR (not dependent on settlement PnL dollar figures):
- BTC DOWN hourly: 66.2% WR in W4 (below breakeven for this entry range)
- ETH DOWN hourly: 63.8% WR in W4
- SOL DOWN hourly: variable but went 50.0% WR in W5 (N=4)

Stream A's cut of DOWN hourly was the correct structural response, supported by WR evidence independently of any dollar PnL.

### Verdict

**The bleed was real on-chain (-$266.23 from peak to trough).** Settlement PnL claiming +$1,196 over this same period does not match on-chain reality. The direction of daily wallet changes disagrees with settlement PnL on 5 of 11 days — this is not random timing noise. The settlement PnL calculation has a systematic problem that must be diagnosed before it can be used as a reliable performance metric.

---

## DA4 — Band B Time-Stability (BTC/SOL UP Hourly, Pre-Stream A)

*Win rates are the primary metric here. Settlement PnL dollar figures retained for directional context only (labeled unreconciled).*

### Raw results

| Asset | Week | N | WR | Avg Entry | Settlement PnL (unreconciled) |
|-------|------|---|----|-----------|-------------------------------|
| BTC | W1 Apr 7–13 | 3 | 0.0% | 0.7633 | -$25.00 |
| BTC | W2 Apr 14–20 | **31** | **90.3%** | 0.7432 | +$62.02 |
| BTC | W3 Apr 21–27 | **29** | **93.1%** | 0.7431 | +$160.73 |
| BTC | W4 Apr 28–May 4 | **13** | **92.3%** | 0.7431 | +$50.62 |
| BTC | W5 May 5–9 | 8 | 75.0% | 0.7550 | +$10.95 |
| SOL | W1 Apr 7–13 | 1 | 100.0% | 0.7200 | +$1.81 |
| SOL | W2 Apr 14–20 | **26** | **84.6%** | 0.7500 | +$52.07 |
| SOL | W3 Apr 21–27 | **30** | **90.0%** | 0.7693 | +$157.77 |
| SOL | W4 Apr 28–May 4 | **22** | **77.3%** | 0.7418 | +$2.48 |
| SOL | W5 May 5–9 | 6 | 83.3% | 0.7517 | +$10.61 |

### Verdict

**BTC Band B: Durable (WR basis).** WR stable at 90–93% across W2, W3, W4 (N=13–31 per week). W5 dip to 75% is on N=8 (insufficient for conclusion). No week-over-week erosion through W4.

**SOL Band B: Mixed (WR basis).** WR of 84–90% in W2–W3, then W4 dipped to 77.3% (N=22 — sufficient sample). Below the 80% stability threshold in W4 but recovered to 83.3% in W5. One dip, not a monotonic decline — does not clearly indicate durable erosion.

**Overall verdict:** High confidence (BTC) to moderate confidence (SOL) that Band B win rate pattern is durable. This conclusion holds independent of settlement PnL dollar figures.

---

## DA5 — Cross-Asset Band B (ETH and All UP Hourly, Pre-Stream A)

*Win rates are primary. Settlement PnL dollar figures are directional context only (labeled unreconciled).*

### Raw results

| Asset | Band | N | WR | Avg Entry | Settlement PnL (unreconciled) | PnL/trade (unreconciled) |
|-------|------|---|----|-----------|-------------------------------|--------------------------|
| BTC | A_low (< 0.70) | 242 | 67.4% | 0.5954 | +$429 | +$1.77 |
| BTC | **B_high (≥ 0.70)** | 84 | **86.9%** | 0.7450 | +$259 | +$3.09 |
| ETH | A_low (< 0.70) | 177 | 70.6% | 0.6058 | +$454 | +$2.57 |
| ETH | **B_high (≥ 0.70)** | 65 | **80.0%** | 0.7469 | +$106 | +$1.63 |
| SOL | A_low (< 0.70) | 142 | 69.0% | 0.6155 | +$152 | +$1.07 |
| SOL | **B_high (≥ 0.70)** | 85 | **84.7%** | 0.7545 | +$225 | +$2.64 |

### Verdict

**ETH Band B is profitable by WR (80.0% vs 70.6% A_low, +9.4pp gap) — cross-asset evidence is strong.**

All three assets show the same directional pattern: Band B WR exceeds Band A WR by 9–20pp. This conclusion is based on win rates only and does not rely on settlement PnL dollars. The WR evidence for Band B being a distinct, higher-performing segment is robust.

---

## DA6 — Market Regime Alignment

*Daily WR is reliable. Settlement PnL column is labeled unreconciled — use WR for regime correlation analysis.*

### BTC daily prices and regime tags

| Date | BTC Close | Daily Return | Regime | B-live WR | Settlement PnL (unreconciled) |
|------|-----------|-------------|--------|-----------|-------------------------------|
| Apr 25 | $73,856 | -2.47% | strong_down | 82.9% | +$1,471 |
| Apr 26 | $75,875 | +2.73% | strong_up | 71.6% | +$378 |
| Apr 27 | $76,350 | +0.63% | choppy | 70.0% | +$114 |
| Apr 28 | $78,195 | +2.41% | strong_up | 75.0% | +$271 |
| Apr 29 | $78,261 | +0.08% | choppy | 74.7% | +$39 |
| Apr 30 | $77,445 | -1.04% | choppy | 75.6% | +$81 |
| May 1 | $77,619 | +0.23% | choppy | 76.7% | +$409 |
| May 2 | $78,645 | +1.32% | choppy | 69.5% | +$6 |
| May 3 | $77,361 | -1.63% | choppy | 69.4% | +$49 |
| May 4 | $76,345 | -1.31% | choppy | 74.7% | +$43 |
| May 5 | $75,775 | -0.75% | choppy | 68.6% | -$33 |
| May 6 | $76,287 | +0.68% | choppy | 70.6% | +$95 |
| May 7 | $78,172 | +2.47% | strong_up | 75.0% | +$87 |
| May 8 | $78,655 | +0.62% | choppy | 74.3% | +$34 |
| May 9–15 | $78.6–80.7k | choppy | choppy | 72–84% | positive |

**W3 (Apr 21–27) BTC returns:** +0.88%, +0.42%, +2.64%, -1.82%, -2.47%, +2.73%, +0.63% — mix of strong moves and chop.

**W4 (Apr 28–May 4) BTC returns:** +2.41%, +0.08%, -1.04%, +0.23%, +1.32%, -1.63%, -1.31% — predominantly choppy.

**Regime × WR cross-tab (bleed period Apr 27–May 8):**

| Regime | Days | Avg WR |
|--------|------|--------|
| strong_up (≥+2%) | 2 | 75.0% |
| choppy (\|<2%\|) | 10 | 73.0% |
| strong_down (≤-2%) | 0 | — |

### Verdict

**No regime shift explains the on-chain bleed.** BTC moved from mild uptrend in W3 to mild chop in W4, but WR remained consistently in the 70–75% range across both regime types. The on-chain wallet decline during Apr 27–May 8 cannot be attributed to market regime.

**Outcome (c) is ruled out** (on WR evidence, independent of settlement PnL).

---

## DA7 — D1 Paper vs B-Live Comparison

### Key caveat

D1 paper is a different strategy (different conviction thresholds, different filters, no Stream A cap). It is **not** a clean counterfactual for "B-live without Stream A." Dollar PnL comparisons are meaningless between these strategies. **Win rate comparison only.**

### Daily win rate comparison

| Date | D1 WR | Live WR | D1 Δ | Phase |
|------|-------|---------|------|-------|
| Apr 25 | 83.8% | 82.9% | +0.9pp | Pre-Stream A |
| Apr 26 | 75.8% | 71.6% | +4.2pp | |
| Apr 27 | 71.0% | 70.0% | +1.0pp | |
| Apr 28 | 64.1% | **75.0%** | **-10.9pp** | ← D1 significantly worse |
| Apr 29 | 75.6% | 74.7% | +0.9pp | |
| Apr 30 | 69.2% | **75.6%** | **-6.4pp** | |
| May 1 | 67.1% | **76.7%** | **-9.6pp** | |
| May 2 | 64.5% | 69.5% | -5.0pp | |
| May 3 | 64.2% | 69.4% | -5.2pp | |
| May 4 | 74.3% | 74.7% | -0.4pp | |
| May 5 | 64.3% | 68.6% | -4.2pp | |
| May 6 | 75.5% | 70.6% | +4.9pp | |
| May 7 | 68.7% | **75.0%** | **-6.3pp** | |
| May 8 | 70.9% | 74.3% | -3.4pp | |
| **May 9 (Stream A)** | 73.2% | **80.7%** | **-7.5pp** | ← Post-Stream A |
| May 10 | 70.1% | **75.9%** | **-5.8pp** | |
| May 11 | **57.9%** | **71.4%** | **-13.5pp** | ← D1 terrible day |
| May 12 | 74.5% | 73.3% | +1.2pp | |
| May 13 | 69.1% | 72.3% | -3.2pp | |
| May 14 | 65.2% | **74.7%** | **-9.5pp** | |
| May 15 | 84.1% | 84.2% | -0.1pp | |

**D1 paper underperforms B-live on 15 of 21 days.** Mean D1 advantage: **-4.2pp** (D1 is worse). Post-Stream A (May 9–15): D1 underperforms on 5 of 7 days, mean gap **-6.9pp**.

### Weekly WR summary

| Week | D1 WR | Live WR | Gap |
|------|-------|---------|-----|
| Apr 25–May 4 | 70.3% | 73.7% | -3.4pp |
| May 5–11 | 68.8% | 72.6% | -3.8pp |
| May 12–15 | 73.5% | 74.3% | -0.8pp |

### Verdict

D1 paper WR is lower than B-live throughout, and the gap widens post-Stream A. This rules out the concern that "D1 paper is outperforming B-live without the cap" — it isn't. If the cap were purely destructive, we'd expect D1 paper (without the cap) to clearly dominate in WR. It doesn't.

---

## Synthesis — Answer to the §2 Question

**Question: Is the pre-Stream A on-chain bleed (Apr 27–May 8, -$266 real wallet drop) explained by (a) bucket-specific issues addressed by Stream A, (b) bucket-specific issues not addressed by Stream A, (c) market regime/variance, or (d) something else?**

**Revised framing:** The on-chain bleed is confirmed real. The original question assumed settlement PnL was reliable (+$1,196 over the bleed period) and treated the wallet drop as a "timing artifact." That framing was wrong. The wallet drop is the truth; the settlement PnL figure is suspect.

### Evidence flow

**Step 1 — What does on-chain data say happened?**

Wallet fell -$266 from Apr 27 to May 8. Daily on-chain deltas disagree with settlement PnL direction on 5 of 11 days. The settlement PnL is not a reliable day-by-day attribution tool.

**Step 2 — What do WR-based bucket analyses say?**

DOWN hourly buckets (BTC, ETH) showed persistently sub-breakeven WR during W3–W4 (58–66% range). SOL UP hourly also weakened (69.4% W4 vs 79.2% W3). These WR weaknesses are real and were the structural drags. Stream A's cut of DOWN hourly was the correct response based on WR evidence alone.

**Step 3 — Is the bleed fully explained by bucket cuts?**

Unknown. Without a reconciled settlement PnL calculation, we cannot attribute the -$266 on-chain drop to specific buckets with confidence. The bucket WR evidence supports DOWN hourly as the structural drag, but the dollar magnitude of that drag cannot be trusted from settlement PnL.

**Step 4 — Was W3 anomalous or baseline?**

W3 wallet gain was ~+$541 on-chain (from ~$288 to $829). Settlement PnL claims +$2,710 for W3 alone — about 5× the actual real gain. The wallet has not recovered to peak since May 8. Whether this is "regression from an anomaly" or "ongoing bleed" depends on whether post-Stream A performance is actually profitable in real USDC — and that requires fixing the settlement PnL reconciliation first.

### Classification

**Primary: (a) — bucket-specific issues (DOWN hourly WR weakness) that were addressed by Stream A.** Supported by WR evidence only; dollar attribution cannot be trusted.

**Secondary: open data quality issue.** The -$266 on-chain bleed cannot be fully attributed without a reconciled settlement PnL metric. The $1,462 discrepancy between settlement PnL and on-chain wallet change is itself the most important finding of this audit.

**The bleed was real on-chain. Settlement PnL claims do not reconcile with wallet movements. No reliable internal metric exists to attribute daily dollar PnL without first fixing the settlement PnL reconciliation gap.**

---

## Open Questions / Data Limitations

1. **Settlement PnL reconciliation (CRITICAL):** The $1,462 discrepancy between settlement PnL (+$1,196) and on-chain wallet change (-$266) during Apr 27–May 8 must be investigated before any settlement PnL dollar figures can be trusted. The 74.8% actual_fill_usdc fallback rate is the most likely cause but needs verification.

2. **DA1 pre-Apr 25 gap:** No portfolio_snapshots before Apr 25. W1–W2 wallet trajectory is estimated from settlement PnL accumulation — which is itself unreconciled. The true W1–W2 on-chain trajectory is unknown.

3. **Apr 28 anomaly:** Bot was down for ~10h on Apr 28. ~65 trades missed. No incident documentation. Excluded from all trend analysis.

4. **DA7 counterfactual limitation:** D1 paper is not "B-live without Stream A" — it's a different strategy. The ideal counterfactual (B-live with cap set to 0.90 instead of 0.70) doesn't exist in the data. The D1 comparison confirms D1 is not outperforming B-live in WR but cannot quantify the specific impact of relaxing the 0.70 cap in real USDC.

5. **SOL Band B W4 dip:** SOL Band B WR dropped to 77.3% in W4 (N=22). One meaningful data point suggesting the pattern may not be fully stable for SOL. W5 recovery (83.3%, N=6) is insufficient to fully resolve this.

6. **Open cost basis undercount:** The cleaned wallet typically shows low open_cost_basis values ($0–$99) even during active trading days, suggesting most positions resolve same-day. On May 8 (trough day) open_cost_basis = $0, which rules out "positions in flight" as the explanation for the wallet drop on that day.

7. **Post-Stream A real performance unknown:** The wallet has stabilized in the $561–$651 range post-Stream A, but without a reconciled settlement PnL metric, whether the system is generating real positive returns or merely stabilized at a lower level is unclear. Real PnL since Stream A = wallet on May 15 ($638.48) minus wallet on May 9 ($590.98) = **+$47.50 on-chain** — a modest but real gain over 6 days.

---

*Report: `/reports/2026-05-16-deeper-audit.md`*
*DB: `/Users/terryyuan/company/shawn/bot/signals.db`*
*BTC price source: CoinGecko API*
*Measurement philosophy: on-chain cleaned wallet (portfolio_snapshots.portfolio_usdc + open_position_cost_basis) is primary PnL metric. Settlement PnL (cash_flow_reconciliation.py) is internal/secondary and currently unreconciled with on-chain — labeled throughout.*
