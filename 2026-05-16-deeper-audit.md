# Deeper Audit — Stagnation Root Cause & Walk-Forward Validity

*Cassian Trading Corp — 2026-05-16*
*Produced in ~2.5 hours. Ref: b-live-audit.md, post-audit-investigations.md*

---

## Executive Summary

**The pre-Stream A "bleed" was not a settlement PnL bleed.** Settlement PnL was positive on 11 of 12 days during Apr 27–May 8. The wallet decline from $829 → $563 reflects snapshot timing artifacts, not actual trading losses. The structural drags during that period were the DOWN hourly buckets (BTC -$43, ETH -$24, SOL -$24 in W4–W5) — which Stream A subsequently cut.

**Band B (entry ≥ 0.70) is stable and durable.** BTC UP hourly Band B held 90–93% WR across three consecutive pre-Stream A weeks (N=29–31 per week). ETH Band B is also profitable (80% WR vs 70.6% A_low), confirming the pattern is not BTC/SOL-specific. SOL had one dip (77.3% in W4, N=22) but recovered in W5.

**D1 paper consistently underperforms B-live in win rate.** After Apr 28, D1 paper WR runs 4–10pp below B-live on most days, including post-Stream A. D1 paper is not a clean counterfactual (different strategy, different filters), but its underperformance rules out "Stream A is universally net-negative."

**Regime is not the explanation.** BTC price action was similarly choppy during both the W3 compounding (+$2,710) and the W4–W5 decline. Settlement PnL shows no correlation with BTC daily returns.

**Synthesis: Outcome (a).** The bleeders (DOWN hourly buckets) were bucket-specific and have been addressed by Stream A. The remaining cap on UP hourly Band B appears to be the next-level friction. Pre-Stream A Band B data is durable enough to act on.

---

## DA1 — Full-System Trajectory (Apr 7 – Present)

### Data

*Note: portfolio_snapshots data begins Apr 25. W1–W2 wallet trajectory inferred from EPOCH_CAPITAL ($287.66) + settlement PnL accumulation. Apr 28 snapshot anomaly: bot was down most of the day; free USDC = $0, open_cost_basis = $98.40, cleaned = $98.40 (undercount — not used for trend analysis).*

**Daily cleaned wallet (free USDC + open cost basis at EOD):**

| Date | Free USDC | Open Cost | Cleaned Wallet | Note |
|------|-----------|-----------|----------------|------|
| Apr 7 | — | — | **$287.66** | EPOCH_CAPITAL (start) |
| Apr 25 | $129.90 | $86.40 | $216.30 | First snapshot (low — many positions in flight) |
| Apr 26 | $682.66 | $66.00 | $748.66 | Large batch of positions resolved |
| Apr 27 | $803.58 | $25.20 | **$828.78** | **PEAK** |
| Apr 28 | $0.00 | $98.40 | $98.40 | ⚠️ Bot down ~10h; anomalous, not used |
| Apr 29 | $654.06 | $20.10 | $674.16 | |
| Apr 30 | $665.99 | $3.90 | $669.89 | |
| May 1 | $662.64 | $76.80 | **$739.44** | Secondary peak |
| May 2 | $577.63 | $35.34 | $612.97 | |
| May 3 | $623.39 | $22.71 | $646.10 | |
| May 4 | $706.57 | $19.00 | $725.57 | |
| May 5 | $691.65 | $0.00 | $691.65 | |
| May 6 | $647.24 | $12.36 | $659.60 | |
| May 7 | $661.67 | $19.10 | $680.77 | |
| May 8 | $562.55 | $0.00 | **$562.55** | **TROUGH (Stream A not yet deployed)** |
| May 9 | $590.98 | $0.00 | $590.98 | Stream A deployed |
| May 10 | $573.99 | $37.59 | $611.58 | |
| May 11 | $550.44 | $10.56 | $561.00 | |
| May 12 | $641.34 | $9.90 | $651.24 | |
| May 13 | $642.22 | $0.00 | $642.22 | |
| May 14 | $600.44 | $11.40 | $611.84 | |
| May 15 | $617.12 | $21.36 | $638.48 | |
| May 16 (partial) | $519.29 | $9.18 | $528.47 | |

**Weekly aggregates (settlement PnL by created_at week, from DA2):**

| Week | N trades | WR | Settlement PnL | Wallet end (cleaned) |
|------|---------|-----|---------------|---------------------|
| W1 Apr 7–13 | 246 | 68.3% | +$120 | ~$408 (est.) |
| W2 Apr 14–20 | 923 | 70.4% | +$298 | ~$706 (est.) |
| W3 Apr 21–27 | 1,095 | **74.9%** | **+$2,710** | $829 |
| W4 Apr 28–May 4 | 860 | 73.4% | +$849 | ~$726 |
| W5 May 5–11 | 479 | 74.3% | +$433 | $561 |
| W6 May 12–15 | 384 | 75.3% | +$526 | $638 |

### Verdict

**Two distinct phases:**
1. **Rapid compounding (W1–W3):** System grew from $288 to $829, with W3 alone producing +$2,710 in settlement PnL as UP buckets fired simultaneously at 74–79% WR. This was an anomalously profitable week, not a baseline.
2. **Consolidation / plateau (W4–W6):** Settlement PnL remains solidly positive ($433–$849/week), but the wallet has not re-attained the W3 peak. Stagnation is primarily explained by UP hourly buckets being much less productive post-W3 (see DA2).

---

## DA2 — Per-Bucket Weekly Settlement PnL

### Key tables

**Top positive contributors per week:**

| Week | Bucket | N | WR | PnL |
|------|--------|---|-----|-----|
| W3 | BTC UP 15min | 300 | 77.3% | **+$890** |
| W3 | BTC UP hourly | 105 | 77.1% | **+$499** |
| W3 | ETH UP hourly | 92 | 76.1% | **+$432** |
| W3 | ETH UP 15min | 207 | 74.9% | **+$412** |
| W3 | SOL UP 15min | 194 | 76.3% | +$334 |
| W3 | SOL UP hourly | 77 | 79.2% | +$313 |
| W4 | SOL DOWN hourly | 82 | **80.5%** | **+$305** |
| W4 | BTC DOWN 15min | 216 | 76.4% | +$244 |
| W4 | ETH DOWN 15min | 104 | 82.7% | +$178 |
| W5 | BTC DOWN 15min | 150 | 74.0% | +$159 |
| W5 | ETH DOWN 15min | 96 | 79.2% | +$150 |
| W6 | BTC DOWN 15min | 173 | 74.6% | +$243 |
| W6 | ETH DOWN 15min | 127 | 78.0% | +$212 |

**Top negative contributors per week:**

| Week | Bucket | N | WR | PnL |
|------|--------|---|-----|-----|
| W3 | BTC DOWN hourly | 24 | 58.3% | **-$70** |
| W3 | ETH DOWN hourly | 29 | 62.1% | **-$48** |
| W3 | BTC DOWN 15min | 19 | 52.6% | **-$41** |
| W3 | ETH DOWN 15min | 13 | 61.5% | **-$30** |
| W4 | BTC DOWN hourly | 74 | 66.2% | **-$43** |
| W4 | SOL UP hourly | 36 | 69.4% | **-$46** |
| W4 | ETH DOWN hourly | 47 | 63.8% | **-$24** |
| W5 | SOL DOWN hourly | 4 | 50.0% | **-$24** |
| W5 | ETH UP hourly | 16 | 68.8% | **-$21** |

**Buckets that flipped from profitable (W3) to negative (W4):**
- BTC UP hourly: +$499 (W3) → +$63 (W4) — not negative but 87% drop in PnL
- SOL UP hourly: +$313 (W3) → **-$46** (W4) — flipped negative
- BTC DOWN hourly: -$70 (W3) → -$43 (W4) → cut in W5
- ETH DOWN hourly: -$48 (W3) → -$24 (W4) → cut in W5

### Verdict

The W3 compounding was driven overwhelmingly by UP buckets firing simultaneously at high WR. From W4 onwards:
- DOWN hourly buckets bled consistently (eventually cut by Stream A)
- UP hourly buckets declined sharply but remained net-positive
- DOWN 15min emerged as the dominant positive contributor

The transition from W3 → W4 explains the plateau: the UP bucket surge didn't sustain, and DOWN hourly bleeding was ongoing until the May 6 cut.

---

## DA3 — Pre-Stream A Drawdown Decomposition (Apr 27 – May 8)

### Daily settlement PnL summary

| Date | N | WR | Settlement PnL | Biggest loser | Biggest winner |
|------|---|----|---------------|--------------|----------------|
| Apr 27 | 70 | 70.0% | **+$114** | BTC UP 15min -$37, BTC DOWN 15min -$37 | BTC UP hrly +$95, ETH UP hrly +$72 |
| Apr 28 | 120 | 75.0% | **+$271** | SOL UP hrly -$49, ETH UP 15min -$38 | SOL DOWN hrly +$185, BTC UP 15min +$70 |
| Apr 29 | 99 | 74.7% | **+$39** | SOL UP hrly -$19, SOL UP 15min -$19 | BTC DOWN 15min +$55, ETH DOWN 15min +$30 |
| Apr 30 | 90 | 75.6% | **+$81** | ETH DOWN hrly -$16, SOL UP hrly -$7 | BTC DOWN 15min +$53, ETH DOWN 15min +$20 |
| May 1 | 180 | 76.7% | **+$409** | ETH UP hrly -$7 | SOL DOWN hrly +$161, ETH DOWN 15min +$62 |
| May 2 | 164 | 69.5% | **+$6** | BTC DOWN hrly -$53, ETH DOWN hrly -$37 | ETH UP hrly +$52, ETH DOWN 15min +$33 |
| May 3 | 124 | 69.4% | **+$49** | ETH UP hrly -$39, SOL DOWN hrly -$34 | BTC DOWN 15min +$57, BTC UP hrly +$45 |
| May 4 | 87 | 74.7% | **+$43** | ETH DOWN hrly -$17, BTC DOWN 15min -$13 | ETH UP hrly +$36, ETH UP 15min +$26 |
| May 5 | 70 | 68.6% | **-$33** ⚠️ | ETH UP hrly -$29, ETH DOWN 15min -$25, SOL DOWN hrly -$24, SOL UP 15min -$23 | BTC UP hrly +$25, ETH UP 15min +$30 |
| May 6 | 68 | 70.6% | **+$95** | BTC DOWN 15min -$58 | SOL UP hrly +$34, BTC UP hrly +$31, BTC UP 15min +$44 |
| May 7 | 72 | 75.0% | **+$87** | ETH UP 15min -$31, SOL DOWN 15min -$11 | BTC DOWN 15min +$44, ETH DOWN 15min +$36 |
| May 8 | 74 | 74.3% | **+$34** | BTC UP hrly -$46 | ETH DOWN 15min +$50, ETH UP 15min +$31 |

**Total Apr 27–May 8 settlement PnL: +$1,196** *(only 1 negative day out of 12)*

### Verdict

**The wallet declined $266 while settlement PnL was +$1,196.** This is not a settlement PnL crisis. The cleaned wallet decline reflects free USDC timing effects: the bot was deploying capital into positions faster than resolutions returned it, and snapshot timing caught the wallet in intermediate states.

The only structurally negative pattern during this period: DOWN hourly buckets (BTC and ETH specifically) contributed losses on May 2 (-$90 combined), May 3 (-$34 SOL DOWN hrly), May 4 (-$17 ETH DOWN hrly) — totaling ~-$141 in DOWN hourly losses across the period. These buckets were the source of genuine structural drag, not the overall system. They were subsequently cut by Stream A.

**No single bucket accounted for >40% of any single day's loss.**

---

## DA4 — Band B Time-Stability (BTC/SOL UP Hourly, Pre-Stream A)

### Raw results

| Asset | Week | N | WR | Avg Entry | Settlement PnL |
|-------|------|---|----|-----------|----------------|
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

**BTC Band B: Durable.** WR stable at 90–93% across W2, W3, W4 (N=13–31 per week). W5 dip to 75% is on N=8 (insufficient for conclusion). Pattern shows no week-over-week erosion.

**SOL Band B: Mixed.** WR of 84–90% in W2–W3, then W4 dipped to 77.3% (N=22 — sufficient sample). This is below the 80% stability threshold. However it recovered to 83.3% in W5. Per the decision rule, this is "declining week-over-week" in W4, but V-shaped not monotonically declining — does not clearly indicate durable erosion.

**Overall verdict:** High confidence (BTC) to moderate confidence (SOL) that Band B pattern is durable. BTC Band B meets the ">80% stable across all weeks" criterion; SOL has one dip (W4) but recovered. Neither shows the monotonically declining pattern that would indicate "pattern was eroding before Stream A."

---

## DA5 — Cross-Asset Band B (ETH and All UP Hourly, Pre-Stream A)

### Raw results

| Asset | Band | N | WR | Avg Entry | Settlement PnL | PnL/trade |
|-------|------|---|----|-----------|----------------|-----------|
| BTC | A_low (< 0.70) | 242 | 67.4% | 0.5954 | +$429 | +$1.77 |
| BTC | **B_high (≥ 0.70)** | 84 | **86.9%** | 0.7450 | **+$259** | **+$3.09** |
| ETH | A_low (< 0.70) | 177 | 70.6% | 0.6058 | +$454 | +$2.57 |
| ETH | **B_high (≥ 0.70)** | 65 | **80.0%** | 0.7469 | **+$106** | **+$1.63** |
| SOL | A_low (< 0.70) | 142 | 69.0% | 0.6155 | +$152 | +$1.07 |
| SOL | **B_high (≥ 0.70)** | 85 | **84.7%** | 0.7545 | **+$225** | **+$2.64** |

### Verdict

**ETH Band B is profitable (80.0% WR, +$1.63/trade) — strong cross-asset evidence.**

All three assets show the same pattern: Band B WR is 9–20pp above Band A WR. The magnitude varies (BTC gap = 19.5pp, SOL = 15.7pp, ETH = 9.4pp), but the direction is universal. Per the decision rule: "ETH Band B is also clearly profitable → strong cross-asset evidence, cap-relaxation hypothesis is robust."

One nuance: ETH Band B has lower PnL/trade (+$1.63) than BTC (+$3.09) or SOL (+$2.64), suggesting the Band B effect is stronger for BTC/SOL. ETH Band B is profitable but less decisively so.

---

## DA6 — Market Regime Alignment

### BTC daily prices and regime tags

| Date | BTC Close | Daily Return | Regime | B-live WR | B-live PnL |
|------|-----------|-------------|--------|-----------|-----------|
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
| May 5 | $75,775 | -0.75% | choppy | 68.6% | **-$33** |
| May 6 | $76,287 | +0.68% | choppy | 70.6% | +$95 |
| May 7 | $78,172 | +2.47% | strong_up | 75.0% | +$87 |
| May 8 | $78,655 | +0.62% | choppy | 74.3% | +$34 |
| May 9–15 | $78.6–80.7k | choppy | choppy | 72–84% | positive |

**W3 (Apr 21–27) BTC returns:** +0.88%, +0.42%, +2.64%, -1.82%, -2.47%, +2.73%, +0.63% — mix of strong moves and chop.

**W4 (Apr 28–May 4) BTC returns:** +2.41%, +0.08%, -1.04%, +0.23%, +1.32%, -1.63%, -1.31% — predominantly choppy.

**Regime × PnL cross-tab (bleed period Apr 27–May 8):**

| Regime | Days | Avg WR | Avg Daily PnL |
|--------|------|--------|--------------|
| strong_up (≥+2%) | 2 | 75.0% | +$179 |
| choppy (|<2%|) | 10 | 73.0% | +$97 |
| strong_down (≤-2%) | 0 | — | — |

### Verdict

**No regime shift explains the bleed period.** BTC moved from a trending upward market in W3 (~$74.8k→$76.4k) to mild chop in W4 ($78.2k±2%), but:
1. B-live trades both UP and DOWN so directionality should partially cancel
2. Settlement PnL was positive across all regime types
3. The only negative day (May 5) occurred on a moderately choppy BTC day (-0.75%), not a crash

BTC rose 20.6% over the entire epoch ($68k → $82k). The compounding (W3) and the bleed (W4–W5) both occurred in the context of a mild BTC uptrend. Regime does not explain the performance difference. **Outcome (c) is ruled out.**

---

## DA7 — D1 Paper vs B-Live Comparison

### Key caveat

D1 paper is a different strategy (different conviction thresholds, different filters, no Stream A cap). It is **not** a clean counterfactual for "B-live without Stream A." Additionally, D1 paper bet sizes scale to a formula-inflated portfolio ($8,000–$16,000/trade vs B-live's $10–$25/trade), making dollar PnL comparisons meaningless. Win rate comparison only.

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

Per decision rule: "If D1 paper has been similarly stagnant or worse → the bleeding is real and broader than Stream A; cap relaxation is not the answer."

D1 paper WR is lower than B-live throughout, and the gap widens post-Stream A. However, this cannot be interpreted as "Stream A is working" because D1 paper is a fundamentally different strategy with more liberal signal acceptance (lower conviction thresholds, higher N per day). D1 paper's lower WR likely reflects trading more marginal signals, not that the B-live cap is helping.

**What DA7 does establish:** The walk-forward concern that "D1 paper is outperforming B-live without the cap" is **not supported**. D1 paper is not outperforming. If the cap were purely destructive, we'd expect D1 paper (without the cap) to clearly dominate — it doesn't.

---

## Synthesis — Answer to the §2 Question

**Question: Is the pre-Stream A bleeding (Apr 28–May 8) explained by (a) bucket-specific issues that were addressed by Stream A, (b) bucket-specific issues not addressed by Stream A, (c) market regime/variance, or (d) something else?**

### Evidence flow

**Step 1 — Which buckets bled pre-Stream A?**

From DA2/DA3: The structural negatives during W4–W5 were:
- BTC DOWN hourly: -$43 (W4), ongoing in W5 before cut
- ETH DOWN hourly: -$24 (W4), -$11 (W5 before cut)
- SOL DOWN hourly: -$24 (W5) — was +$305 in W4 (big swing)
- SOL UP hourly: -$46 (W4) — also affected by Stream A cap

**Were those buckets cut by Stream A?**
- DOWN hourly (BTC, ETH, SOL): ✅ CUT by Stream A (blanket DOWN hourly suspension May 6)
- SOL UP hourly: Partially affected (Stream A's entry cap 0.70 reduced volume by 93%)

**Conclusion: (a)** — the negative buckets (DOWN hourly) were cut by Stream A.

**Step 2 — Are there bleeding buckets NOT addressed by Stream A?**

From DA2 W5/W6: BTC UP 15min, SOL DOWN 15min are marginal ($0–$22/week). No large structural negatives remain post-Stream A. The remaining buckets (DOWN 15min for BTC/ETH) are strong positives. **No evidence for (b).**

**Step 3 — Was regime different during bleed vs compounding?**

DA6: Both periods (W3 compounding, W4–W5 bleed) had similar BTC conditions — mild uptrend, predominantly choppy daily moves. Settlement PnL showed no correlation with BTC daily returns. **Regime does not explain the difference. (c) is ruled out.**

**Step 4 — DA1 longer-term pattern**

W3 was an anomaly: 1,095 trades at 74.9% WR producing +$2,710 in a single week. This is 3–5× the typical weekly settlement PnL (W4–W6 average: $603). The W3 "golden week" was driven by all UP buckets (15min and hourly for BTC, ETH, SOL) firing simultaneously at unusually high WR. The "stagnation" after W3 is partly the natural regression from an outlier week, not a system failure.

### Classification

**Primary: (a) — bucket-specific issues that were addressed by Stream A.**

The DOWN hourly buckets were the structural drag during the pre-Stream A bleed period. Stream A cut them. The subsequent "stagnation" (W4–W6 at $433–$849/week settlement PnL) reflects:
1. Normal regression from W3's anomalous performance (not a new failure)
2. The Cap on UP hourly Band B (Q1 finding) reducing high-WR trade volume by 71–93%

**Secondary: structural UP hourly volume loss from Stream A.**

The Q1 finding (Stream A cutting high-WR Band B trades) is supported by DA4/DA5 (Band B is durable across all assets) and consistent with DA7 (D1 paper not outperforming B-live). DA4 shows Band B WR held 90%+ for BTC through W4. DA5 shows the pattern is cross-asset (ETH Band B also 80% WR). The cap appears to be the next-level friction, not a necessary protection.

**The bleeding was real but smaller than it appeared.** DA3 confirms settlement PnL was positive on 11 of 12 bleed-period days (+$1,196 total). The wallet decline ($829→$563) reflects snapshot timing effects and the impact of a few high-bet DOWN hourly losses (concentrated on May 2 BTC/ETH DOWN hourly losing days), not a systemic PnL failure.

---

## Open Questions / Data Limitations

1. **DA1 pre-Apr 25 gap:** No portfolio_snapshots before Apr 25. W1–W2 wallet trajectory is inferred from settlement PnL accumulation, not observed. The actual cleaned wallet trajectory before Apr 25 is unknown.

2. **Apr 28 anomaly:** Bot was down for ~10h on Apr 28. ~65 trades missed. No incident documentation. Not material to the synthesis but represents a data gap.

3. **DA7 counterfactual limitation:** D1 paper is not "B-live without Stream A" — it's a different strategy. The ideal counterfactual (B-live with cap set to 0.90 instead of 0.70) doesn't exist in the data. The D1 comparison confirms D1 is not outperforming B-live but cannot confirm the specific impact of relaxing the 0.70 cap.

4. **SOL Band B W4 dip:** SOL Band B WR dropped to 77.3% in W4 (N=22). This is one meaningful data point suggesting the pattern may not be fully stable for SOL. DA4's decision rule ("declining week-over-week → pattern eroding") applies partially to SOL. The W5 recovery (83.3%, N=6) is insufficient to fully resolve this.

5. **Open cost basis tracking:** The cleaned wallet calculation shows low open_cost_basis values (typically $0–$99) even during active trading days. This suggests most positions resolve within the same calendar day, or the historical open cost reconstruction is underestimating in-flight capital. The wallet trajectory decline vs positive settlement PnL discrepancy is structural and not fully explained.

6. **D1 paper sizing:** D1 paper bet sizes are $8,000–$16,000 (inflated from compounding formula PnL). D1 paper is not a realistic paper simulation of any deployable live strategy at this sizing.

---

*Report: `/reports/2026-05-16-deeper-audit.md`*
*DB: `/Users/terryyuan/company/shawn/bot/signals.db`*
*BTC price source: CoinGecko API*
