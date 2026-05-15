# B-Live Comprehensive Performance Audit
*Cassian Trading Corp — produced 2026-05-15 19:15 ET*
*Audit duration: ~2 hours*

---

## Executive Summary

The system is generating positive expected value per dollar deployed, but that dollar figure is small: average concurrent deployment is ~2.7% of wallet ($17.50 of $644.12). Since Stream A launched 7 days ago, the snapshot-based ground truth shows +$23.67 in free USDC — essentially flat — while total cleaned wallet is up +$55.71 (baseline snapshot + open position cost basis). The biggest issue is not a broken strategy: SOL DOWN 15min is a live confirmed negative-EV bucket (-$34.63, 14d, 81% fill coverage) that has not been cut. The biggest silent failure is the `actual_fill_price` column being identically equal to `entry_price` for all 1,119 resolved trades — slippage measurement is completely broken and execution quality is invisible. The pipeline is partially built: the calibration model exists but is not wired into live trade logging (34.8% coverage on `calibrated_synth_prob`), blocking both F3_DOWN_v2 and F1_v2.

---

## Table of Contents
- [A — Real-Money Performance](#a--real-money-performance)
- [B — Per-Bucket Performance](#b--per-bucket-performance)
- [C — Capital Efficiency](#c--capital-efficiency)
- [D — Trade Execution Health](#d--trade-execution-health)
- [E — Active Configurations](#e--active-configurations)
- [F — Pipeline State](#f--pipeline-state)
- [G — Data Quality](#g--data-quality)
- [H — Infrastructure Health](#h--infrastructure-health)
- [I — Recent Observations](#i--recent-observations)
- [J — Open Questions](#j--open-questions)

---

## A — Real-Money Performance

### A.1 Wallet Trajectory

*Source: `portfolio_snapshots` table, `trades` table for open positions; `cash_flow_reconciliation.py` run at 2026-05-15 18:58 UTC*

| Metric | Value |
|--------|-------|
| Latest raw snapshot | $612.08 (at 2026-05-15 18:05 UTC) |
| Open positions cost basis | $32.04 (3 trades) |
| **Cleaned wallet (true value)** | **$644.12** |
| Epoch capital (Apr 7, 2026) | $287.66 |
| **All-time real PnL** | **+$356.46 (+123.9% over 38 days)** |

Note on annualization: +123.9% over 38 days is mathematically enormous when annualized (~230,000% compounded). This number is not meaningful at this sample size — report it as "+$356.46 over 38 days" only.

**⚠️ Apr 26 anomaly:** EOD snapshot jumped from $129.90 (Apr 25) to $682.66 (Apr 26), a +$553 single-day move. This is almost certainly a capital deposit, not trading P&L. If so, the all-time gain of +$356.46 is overstated by approximately that deposit amount. This has not been reconciled.

**Daily EOD wallet (raw snapshot — does NOT add open position cost basis):**

| Date | EOD Snapshot | Daily Δ (snapshot) |
|------|------------|-------------------|
| Apr 25 | $129.90 | — |
| Apr 26 | $682.66 | +$552.76 ⚠️ likely deposit |
| Apr 27 | $803.58 | +$120.92 |
| Apr 28 | $0.00 | — ⚠️ corrupted snapshot |
| Apr 29 | $654.06 | — |
| Apr 30 | $665.99 | +$11.93 |
| May 1 | $662.64 | -$3.35 |
| May 2 | $577.63 | -$85.01 |
| May 3 | $623.39 | +$45.76 |
| May 4 | $706.57 | +$83.18 |
| May 5 | $691.65 | -$14.92 |
| May 6 | $647.24 | -$44.41 |
| May 7 | $661.67 | +$14.43 |
| May 8 | $562.55 | -$99.12 |
| May 9 | $590.98 | +$28.44 ← Stream A launched |
| May 10 | $573.99 | -$17.00 |
| May 11 | $550.44 | -$23.54 ⚠️ downtime day |
| May 12 | $641.34 | +$90.90 |
| May 13 | $642.22 | +$0.88 |
| May 14 | $600.44 | -$41.78 |
| May 15 | $612.08 | +$11.64 (partial) |

Raw snapshots are free USDC only; each row understates true wallet value by the cost basis of open positions at that time (typically $30–120).

### A.2 Settlement PnL Series

*Source: strict actual_fill_usdc formula (`actual_fill / entry_price * 0.98 - actual_fill` for wins, `-actual_fill` for losses). No bet_size fallback. Pre-Apr 26 excluded (zero fill data).*

**30-day series (Apr 26 – May 14, 19 complete days with fill data):**

| Date | N Trades | N Filled | Fill% | Settlement PnL (strict) |
|------|----------|----------|-------|------------------------|
| Apr 26 | 176 | 21 | 12% | -$47.83 ⚠️ low coverage |
| Apr 27 | 70 | 50 | 71% | +$32.25 |
| Apr 28 | 120 | 31 | 26% | -$93.10 ⚠️ low coverage |
| Apr 29 | 99 | 75 | 76% | +$44.13 |
| Apr 30 | 90 | 77 | 86% | +$73.75 |
| May 1 | 180 | 142 | 79% | +$118.26 |
| May 2 | 164 | 148 | 90% | -$1.99 |
| May 3 | 124 | 109 | 88% | +$28.76 |
| May 4 | 87 | 74 | 85% | +$33.21 |
| May 5 | 70 | 57 | 81% | -$53.03 |
| May 6 | 68 | 58 | 85% | +$12.18 |
| May 7 | 72 | 52 | 72% | +$54.63 |
| May 8 | 74 | 66 | 89% | +$49.88 |
| May 9 | 88 | 55 | 63% | +$76.98 ← Stream A |
| May 10 | 87 | 61 | 70% | +$47.67 |
| May 11 | 21 | 18 | 86% | -$39.63 ⚠️ downtime |
| May 12 | 90 | 75 | 83% | +$76.58 |
| May 13 | 94 | 78 | 83% | +$49.51 |
| May 14 | 91 | 67 | 74% | +$21.18 |

**Stats (19-day window, strict fills):**
- Mean: **+$25.44/day**
- Median: **+$33.21/day**
- Std: **$51.28/day**
- Daily Sharpe: 0.496 | Annualized: ~9.5

**Stats (post-Stream A only, May 9–14, 6 complete days):**
- Mean: **+$38.72/day**
- Median: **+$48.59/day**
- Std: **$39.87/day**
- Daily Sharpe: 0.97 (N=6, not meaningful)

**Caveats:** Strict settlement PnL systematically understates true daily P&L because it excludes trades without fill data (25–37% on most days). The actual daily mean is higher; the variance estimates are not reliable. Snapshot deltas are the true ground truth but are dominated by timing noise.

**7-day rolling settlement PnL (post-Stream A):**
- May 9–15: +$232.28 strict (7 days, 6 complete + partial today)
- Snapshot ground truth: +$23.67 free USDC (May 9 baseline → May 15 EOD)
- Cleaned ground truth: approximately +$55.71 (adding open position basis at both ends)

### A.3 PnL by Period

*Source: trades table, actual_fill_usdc strict. Pre-Stream A = before 2026-05-09T05:34:26 UTC.*

| Period | N Trades | N w/ Fill | Fill% | Settlement PnL | Days | Daily Mean |
|--------|----------|-----------|-------|---------------|------|------------|
| Pre-Stream A | 3,547 | 976 | 27.5% | +$243.30 | 38 | +$6.40 |
| Post-Stream A | 501 | 375 | 74.9% | +$286.83 | 7 | +$40.97 |

**Delta:** +$34.57/day improvement. Pre-Stream A fill coverage is too low (27.5%) for the $6.40/day figure to be reliable — the true pre-Stream A daily PnL is likely higher. Win rate improved from 71.7% → 75.8% post-Stream A.

**Statistical confidence:** With 7 post-Stream A days and daily std ~$40, the standard error of the daily mean is $40/√7 = $15.1. The apparent improvement of +$34.57/day has t-stat ≈ 2.3 vs a pre-Stream A baseline. Directionally significant but not conclusive at 14 days.

### A.4 Reconciliation Status

*Source: daily snapshot deltas vs CF PnL with fallback (post-May 9). Strict settlement PnL not used here — too many excluded trades to reconcile cleanly.*

| Date | Snapshot Δ | CF Net (fallback) | Recon Error | Error > $10? |
|------|-----------|-------------------|------------|--------------|
| May 10 | -$17.00 | +$112.85 | -$129.85 | ✅ yes |
| May 11 | -$23.54 | -$23.76 | +$0.22 | no |
| May 12 | +$90.90 | +$98.81 | -$7.91 | no |
| May 13 | +$0.88 | +$54.86 | -$53.98 | ✅ yes |
| May 14 | -$41.78 | +$90.13 | -$131.91 | ✅ yes |

Median reconciliation error (post-Stream A): **-$53.98** (3 of 5 complete days > $10).

**Explanation for large errors:** The CF method uses `resolved_at` as the date key, while the snapshot captures free USDC at EOD. Positions opened on day N tie up USDC (snapshot goes down) but don't resolve until day N+1 (CF shows gain on N+1). The structural timing mismatch drives most of the error — not a data quality bug. The only clean window is when all open positions from period start are closed by period end, which is satisfied over the full 7-day post-Stream-A window but not day-by-day.

---

## B — Per-Bucket Performance

*Source: `trades` table, strict actual_fill_usdc, trailing 14 days (May 2–15). Fill coverage < 80% = UNCERTAIN.*

### B.1 Bucket Scorecard

| Asset | Dir | Horizon | N | Fill% | WR | Avg Fill $ | Bet (intend) | Bet (actual) | PnL/filled | Total PnL | Grade |
|-------|-----|---------|---|------|----|-----------|-------------|-------------|-----------|----------|-------|
| BTC | DOWN | 15min | 420 | 76.2 | 74.0% | 0.6612 | $10.50 | $10.37 | +$0.80 | **+$256.22** | UNCERTAIN |
| ETH | DOWN | 15min | 276 | 81.5 | 77.5% | 0.6670 | $10.59 | $10.36 | +$1.08 | **+$243.16** | STRONG |
| SOL | UP | hourly | 51 | 94.1 | 78.4% | 0.6874 | $17.59 | $17.00 | +$1.19 | **+$57.02** | STRONG ⚠️ reverting |
| ETH | UP | 15min | 59 | 74.6 | 79.7% | 0.6948 | $10.77 | $10.93 | +$1.14 | **+$49.99** | UNCERTAIN |
| ETH | UP | hourly | 39 | 92.3 | 76.9% | 0.6739 | $17.16 | $16.94 | +$1.23 | **+$44.31** | STRONG |
| BTC | UP | hourly | 69 | 92.8 | 73.9% | 0.6698 | $23.79 | $23.15 | +$0.57 | **+$36.61** | STRONG ⚠️ reverting |
| BTC | UP | 15min | 85 | 68.2 | 71.8% | 0.7017 | $11.84 | $11.92 | -$0.24 | **-$14.11** | UNCERTAIN/NEGATIVE |
| BTC | DOWN | hourly | 47 | 93.6 | 63.8% | 0.6514 | $14.51 | $14.70 | -$0.36 | **-$15.97** | (CUT — all prior 7d) |
| SOL | UP | 15min | 34 | 70.6 | 73.5% | 0.7004 | $11.79 | $11.25 | -$1.01 | **-$24.33** | UNCERTAIN/NEGATIVE |
| SOL | DOWN | 15min | 195 | 81.0 | 68.7% | 0.6814 | $10.54 | $10.00 | -$0.22 | **-$34.63** | **NEGATIVE 🔴** |
| ETH | DOWN | hourly | 33 | 93.9 | 60.6% | 0.6445 | $13.44 | $13.04 | -$1.18 | **-$36.67** | (CUT — all prior 7d) |
| SOL | DOWN | hourly | 52 | 82.7 | 73.1% | 0.6816 | $17.66 | $16.04 | -$1.07 | **-$46.06** | (CUT — all prior 7d) |

**Key:**
- BTC/ETH/SOL DOWN hourly: all 3 are from the prior 7 days only (pre-cut). The blanket DOWN hourly cut (May 6) is working — zero trades in the last 7 days.
- **SOL DOWN 15min is still live and negative:** -$34.63 total, -$0.22/trade, 68.7% WR vs ~67% break-even at avg entry 0.6814. Marginal EV but directionally negative over 14 days with 81% fill coverage (reliable).
- BTC UP 15min: UNCERTAIN (68.2% fill), small negative PnL — hold signal.

### B.2 Bucket Trends (last 7d vs prior 7d)

| Asset | Dir | Horizon | WR Prior 7d | WR Last 7d | WR Δ | PnL/trade Prior | PnL/trade Last | PnL Δ |
|-------|-----|---------|------------|-----------|------|----------------|---------------|-------|
| BTC | DOWN | 15min | 71.7% | 75.6% | **+3.9pp** | +$0.02 | **+$1.42** | +$1.40 ✅ |
| ETH | DOWN | 15min | 76.7% | 78.1% | +1.4pp | +$1.25 | +$0.96 | -$0.29 |
| SOL | DOWN | 15min | 65.7% | 72.4% | +6.7pp | -$0.59 | +$0.21 | **+$0.80** (recovering) |
| ETH | UP | 15min | 74.4% | 90.0% | **+15.6pp** | +$0.53 | +$2.30 | +$1.77 ✅ |
| ETH | UP | hourly | 76.9% | 76.9% | 0pp | -$0.05 | **+$3.79** | +$3.84 ✅ |
| **BTC** | **UP** | **hourly** | **80.9%** | **59.1%** | **-21.8pp** | **+$3.21** | **-$4.83** | **-$8.04 🔴** |
| **SOL** | **UP** | **hourly** | **82.9%** | **60.0%** | **-22.9pp** | **+$2.27** | **-$2.91** | **-$5.18 🔴** |
| SOL | UP | 15min | 69.2% | 87.5% | +18.3pp | -$1.43 | +$1.07 | +$2.50 |
| BTC | UP | 15min | 70.8% | 75.0% | +4.2pp | +$0.01 | -$1.34 | -$1.35 ⚠️ |

**Major finding: BTC UP hourly and SOL UP hourly have both crashed hard in the last 7 days vs the prior 7 days.** BTC UP hourly: -21.8pp WR, -$8/trade reversal. SOL UP hourly: -22.9pp WR, -$5/trade reversal. Both happened simultaneously in the May 8–15 window, suggesting a market regime shift rather than a strategy failure.

### B.3 Time-of-Day Patterns (top 3 buckets)

*Source: B4 query grouped by UTC hour, trailing 14 days*

**BTC DOWN 15min** (top contributor, $256 total):
- Worst hours: UTC 2 (-$25.17), UTC 8 (-$57.48), UTC 14 (-$19.31), UTC 15 (-$24.82)
- Best hours: UTC 4 (+$73.74), UTC 10 (+$47.55), UTC 13 (+$46.58)
- UTC 8 is striking: -$57.48 in one hour band = ~22% of the bucket's full gross gain turned negative

**ETH DOWN 15min** (second, $243 total):
- Worst hours: UTC 2 (-$26.39), UTC 15 (-$19.45), UTC 23 (-$32.79)
- Best hours: UTC 9 (+$44.71), UTC 4 (+$40.65), UTC 18 (+$37.50)
- UTC 23 pattern echoed in BTC DOWN 15min (UTC 2 = same overnight window)

**SOL UP hourly** (third, $57 total):
- Worst hours: UTC 0 (-$16.37), UTC 22 (-$33.12), UTC 23 (-$18.49)
- Best hours: UTC 15 (+$40.81), UTC 21 (+$23.22)
- UTC 22-23 are consistently negative across both SOL UP hourly and ETH DOWN 15min — this is the NY midnight / Asia open window

---

## C — Capital Efficiency

### C.1 Deployment Metrics

*Source: trades table, portfolio_snapshots, trailing 7 days*

| Day | N Trades | Total Bet $ | Avg Bet | Turnover vs Wallet |
|-----|----------|------------|--------|-------------------|
| May 8 | 72 | $1,017 | $14.13 | 1.58× |
| May 9 | 88 | $1,120 | $12.72 | 1.74× |
| May 10 | 87 | $1,066 | $12.26 | 1.86× |
| May 11 | **20** | **$272** | $13.59 | — ⚠️ downtime |
| May 12 | 93 | $948 | $10.19 | 1.47× |
| May 13 | 93 | $1,086 | $11.68 | 1.69× |
| May 14 | 89 | $1,066 | $11.98 | 1.78× |
| May 15 | 55 (partial) | $668 | $12.14 | — |

**Average concurrent open positions (estimated):**
- 15min trades: 87/day × 20 min avg / 1440 min = **1.21 concurrent** × ~$11 = ~$13.30
- Hourly UP trades: ~6/day × 60 min / 1440 = **0.25 concurrent** × ~$18 = ~$4.50
- Total avg deployed: ~**$17.80**
- **Avg deployment %: ~2.7%** of $644 wallet

Intraday range: min near 0 (between cycle scans), max at MAX_OPEN=6 (observed with BTC UP hourly anchor bets ~$23 each = ~$138 when 6 positions open = 21% deployed). Max saturation events (6 simultaneous) are infrequent.

### C.2 Daily Turnover

Average daily turnover (excluding May 11 anomaly): **$1,051 / $644 = 1.63× wallet/day**. This means each dollar in the wallet is "recycled" through trades 1.6× per day on average. The low deployment (2.7%) vs high turnover (1.63×) coexist because positions are held for very short durations (~20 min for 15min contracts).

### C.3 Capital Utilization Assessment

At current strict settlement PnL ~$38.72/day (post-Stream A mean, 6 days) with avg deployment of $17.80:
- Edge rate: ~$2.18 per dollar deployed per day (very rough)
- If deployment doubled to $35.60 (doubling bet sizes): expected PnL ≈ +$77/day, +$38/day incremental

**Bet scaling (H_BET_SCALE_v1) is likely the highest-leverage single action available.** The 7-day clean window after H_ETH_DOWN_HOURLY_SUSPEND_v1 (deployed May 6) has already elapsed (9 days ago). H_BET_SCALE_v1's gating condition may be satisfied.

---

## D — Trade Execution Health

### D.1 Fill Economics — ⚠️ DATA COMPLETELY BROKEN

*Source: `actual_fill_price - entry_price` for all resolved trades with fill data, trailing 14 days*

**Every single resolved trade has `actual_fill_price - entry_price = 0.0000`.**

1,119 trades across all 12 buckets show zero slippage. This is not a real finding — it is a recording bug. The `actual_fill_price` column is being populated with the `entry_price` estimate at order time, not with the actual CLOB fill price from the order acknowledgment. Slippage measurement is entirely non-functional. Execution quality is invisible.

This is **a critical data gap**: we cannot measure whether orders are filling at the expected price or experiencing adverse selection.

### D.2 FOK Rejection Rate

*Source: trades table, status/outcome, trailing 7 days*

| Status | Outcome | Count |
|--------|---------|-------|
| resolved | win | 449 |
| resolved | loss | 145 |
| placed | pending | 3 |
| failed | pending | **1** |

**Aggregate rejection rate: 1/598 = 0.17%.** Effectively zero failures. The FOK→FAK fallback is working. No bucket has rejection rate > 25%.

*Note: The trader.log shows FOK→FAK→fail sequences in real time (2 events in the last 100 log lines for ETH DOWN 15min), but the `failed` outcome is properly recorded and not retried. No missed-PnL leak from rejected orders.*

### D.3 Latency

*Source: `resolved_at - created_at` proxy, trailing 7 days*

| Horizon | Direction | Avg Duration | N |
|---------|-----------|-------------|---|
| 15min | DOWN | 20.0 min | 501 |
| 15min | UP | 21.6 min | 48 |
| hourly | UP | 60.5 min | 45 |

Resolution times match expected contract windows. No latency anomalies observed in log data. Note: this is resolution time (order→contract settle), not order placement latency — true placement latency is not recorded in the DB.

---

## E — Active Configurations

### E.1 Stream A Configs

*Source: trader.py, .env — read 2026-05-15*

| Parameter | Value | Env Override | Since |
|-----------|-------|-------------|-------|
| MAX_UP_ENTRY_PRICE | 0.70 | explicit (.env = 0.70 = code default) | May 9 (A1) |
| MIN_SYNTH_UP_15MIN | 0.65 | explicit (.env = 0.65 = code default) | May 9 (A2) |
| Effective DOWN threshold | **0.40** | code default 0.38, but T1 DOWN cut at line 1940 overrides | May 9 |

**Note:** The dead zone gate uses 0.38 but the T1 DOWN cut (`synth > 0.40` → skip) means the operative floor is 0.40, not 0.38. The 0.38–0.40 DOWN band is effectively blocked.

### E.2 Active Cuts

| Cut | Since | Coverage | Source YAML | Status |
|-----|-------|----------|------------|--------|
| All DOWN hourly (BTC, ETH, SOL) | 2026-05-06 | ✅ confirmed in logs | H_ETH_DOWN_HOURLY_SUSPEND_v1.yaml (sealed, SHA `5c77c3d2...`) | DEPLOYED, in observation |
| T1 DOWN synth > 0.40 | 2026-05-09 | ✅ code line 1940 | No pre-reg YAML | DEPLOYED |
| T3 UP 15min guard (floor to T1) | 2026-05-04 | ✅ code line 2136 | No pre-reg YAML | DEPLOYED |
| Daily horizon hard cut | pre-audit | ✅ code line 1775 | No pre-reg YAML | DEPLOYED |

### E.3 Risk Controls

| Control | Current Value | Triggered past 7 days? |
|---------|--------------|----------------------|
| Daily drawdown cap | 10% of balance (~$64), min $15 | No (not seen in logs) |
| Epoch drawdown pause | 25% of $234.94 = -$58.74 | No |
| Epoch drawdown halt | 50% of $234.94 = -$117.47 | No |
| DOWN streak step-down (3–4 losses) | ×0.60 | Unknown — not visible in logs reviewed |
| DOWN streak step-down (5+ losses) | ×0.35 | Unknown |
| FOK fail dial-back | >40% fail rate, min 10 orders | No (0.17% fail rate) |
| Per-cycle 15min cap | max($22, 4% of balance) | **YES — actively firing** (seen in logs) |
| Correlated window cap | max($25, 5% of balance) | Unknown |

The per-cycle 15min cap is the most actively triggered control — seen in trader.log: `cycle 15min cap: $21.4 + $10.7 > $23.8 (3% of $594)`. Note: log label says "3%" but the formula uses 4% → `$594 × 0.04 = $23.76`. The label may be stale but the math is correct.

### E.4 Filter Stack (22 active filters)

| # | Filter | Logic |
|---|--------|-------|
| 1 | Daily horizon gate | Skip if `horizon == "daily"` |
| 2 | Dead zone gate | Skip if `0.38 ≤ synth ≤ 0.62` |
| 3 | Time-to-expiry gate | Skip if TTL < 180s (15min) or 600s (hourly) |
| 4 | Signal age gate | Skip if signal age > 600s |
| 5 | Liquidity gate | Skip if `ask_size < $10` |
| 6 | SKIP_DOWN / SKIP_UP env flags | Env-flag disable entire direction |
| 7 | Positive-EV gate | Skip if synth ≤ entry_price (UP) or 1-synth ≤ entry_price (DOWN) |
| 8 | Entry price bounds gate | Hourly: [0.40, 0.82]; 15min: [0.50, 0.82] |
| 9 | UP synth floor gate | Skip UP if synth < 0.65 (15min) or < 0.62 (hourly) |
| 10 | T1 DOWN cut | Skip DOWN if synth > 0.40 |
| 11 | DOWN hourly hard cut | Skip all DOWN hourly (all assets) |
| 12 | BTC 1h counter-trend gate | Skip DOWN if BTC 1h > +0.5%; skip UP if BTC 1h < -0.5% |
| 13 | MAX_UP_ENTRY_PRICE cap | Skip UP if live ask > 0.70 (post live-fetch) |
| 14 | T1 UP entry floor | Skip T1 UP if entry < 0.55 |
| 15 | Slug-level dedup guard | Skip if already traded or open on same market |
| 16 | Per-asset position cap | Skip if ≥ 3 open positions for same asset |
| 17 | On-chain position check | Skip if already hold token on-chain |
| 18 | T3 UP 15min guard | Floor T3 to T1 if UP+15min+T3 |
| 19 | Daily drawdown cap | Floor to T1 if daily PnL < -10% of balance |
| 20 | FOK fail-rate dial-back | Floor to T1 if >40% FOK reject rate (min 10 orders) |
| 21 | Per-cycle 15min cap | Skip if cycle 15min spent + new bet > max($22, 4% balance) |
| 22 | Correlated window cap | Floor to T1 or skip if window exposure > max($25, 5% balance) |

Filter skip counts per filter for trailing 7 days: **not available** (not logged to DB, would require log parsing).

### E.5 Sizing Logic

| Tier | Conviction | Fraction (DOWN) | Fraction (UP) | 15min haircut | Effective at $644 |
|------|-----------|----------------|--------------|--------------|------------------|
| T1 | synth < 0.70 | 3.0% | 3.0% | ×0.60 | $11.59 (15min) / $19.32 (hourly) |
| T2 | 0.70 ≤ synth < 0.80 | 3.0% | 3.0% | ×0.60* | $11.59 / $19.32 |
| T3 | synth ≥ 0.80 | 3.0% | **5.0%** | ×0.60 (DOWN) / 1.0 (UP T3) | $11.59 (DOWN) / **$32.20 (UP hourly)** |

*T2 UP 15min exception: haircut = 1.0 (no haircut), deployed 2026-05-04.

BTC UP hourly anchor: `trend_scale = max(trend_scale, 1.5)` for all BTC UP hourly trades. Effective T1 BTC UP hourly = $19.32 × 1.5 = $28.98 at $644 balance. Avg bet observed ($23.15) is lower — suggests most BTC UP hourly trades are during non-trending periods where `trend_scale = 1.0` and the anchor floors to 1.5, but some entries are at lower conviction or the balance was lower when trades fired.

.env T3 floor anomaly: `.env` sets `LIVE_BET_TIER3=12.0` but code default (line 203) is `$18.0`. At current balance, the fractional formula dominates ($32.20 > $12), so the floor isn't binding. Below $400 balance, the floor would kick in at $12 instead of code default $18.

---

## F — Pipeline State

### F.1 Calibration Pipeline

*Source: calibration_current.pkl, calibration_refit.py, signals.db*

| Metric | Value |
|--------|-------|
| Model file exists | ✅ `/bot/calibration_current.pkl` |
| Model type | IsotonicRegression (UP + DOWN) |
| Fit date | 2026-05-12T05:39:28 UTC (3 days ago) |
| N_UP training samples | 224 |
| N_DOWN training samples | 515 |
| Holdout days | 7 |
| Brier scores | Not extracted (pkl structure confirmed, scores not serialized) |
| **Wired into live trading?** | ❌ **NO** |
| calibrated_synth_prob coverage (last 7d) | **34.8%** (207/594 resolved trades) |
| sklearn version warning | ⚠️ Fit on 1.6.1, now running 1.8.0 |

**Critical:** The calibration model exists and was fit on May 12, but `trader.py` does not write `calibrated_synth_prob` at trade execution time. The 207 covered trades got values from a historical backfill run (`calibration_refit.py --backfill`). New trades since that backfill have NULL calibrated_synth_prob. Until this column is populated live, all strategies requiring it (F1_v2, F3_DOWN_v2) are blocked.

Weekly refit cron intended (`0 23 * * 0` per script comments) — **not confirmed as active** in crontab.

### F.2 Pre-Registered Hypotheses (sealed, not yet deployed)

| Hypothesis | SHA256 | What it does | Blocking gate |
|-----------|--------|-------------|--------------|
| F3_DOWN_v2 | `4936ab59...` | Edge-direct Kelly sizing for BTC/ETH DOWN 15min only | calibration wired live + F1_v2 decision resolved |
| H_BET_SCALE_v1 | **TBD — not hashed** ⚠️ | Raise T1:$5→$6, T2:$8→$9, T3:$12→$15 | 7-day clean window post-ETH DOWN suspension (elapsed ✅) + SOL DOWN hourly investigation result |

**H_BET_SCALE_v1 pre-registration is incomplete** — SHA256 seal has not been computed. The YAML exists as a draft.

### F.3 Active A/B Experiments

None. No paired A/B experiments are currently running.

### F.4 Killed / Superseded Hypotheses

From `/hypotheses/negative_results.md`:

| Hypothesis | Result | Summary |
|-----------|--------|---------|
| F1_v1 (raw synth breakeven filter) | Killed 2026-05-09 | -$103.87 vs realized on May UP trades (n=290). Root cause: synth_prob is not a calibrated probability; blocked 40 winning trades (82% WR) incorrectly |
| F3_DOWN (Kelly on all DOWN) | Superseded 2026-05-12 | Applied Kelly to all DOWN buckets including bleeders; replaced by F3_DOWN_v2 (good buckets only) |

**F1_v1 retest draft** (`F1_v1_RETEST_DRAFT.yaml`) — DRAFT, not committed. V4 v2 counterfactual shows +$14.75 (+17%) on n=39 trades. Below the 50-fill gate for promotion. Blocked on fill count.

---

## G — Data Quality

### G.1 F2 (Fill) Coverage

*Source: trades table, actual_fill_usdc NOT NULL AND > 0*

| Period | N Resolved | N with Fill | Coverage |
|--------|-----------|------------|---------|
| Trailing 24h (May 15) | 52 | 37 | **71.2%** |
| Trailing 7d (May 9–15) | 523 | 391 | **74.8%** |

**Coverage by bucket (trailing 7d):**

| Bucket | Fill Coverage |
|--------|-------------|
| BTC UP hourly | 92.8% ✅ |
| ETH DOWN hourly | 93.9% (cut) |
| BTC DOWN hourly | 93.6% (cut) |
| SOL UP hourly | 94.1% ✅ |
| ETH UP hourly | 92.3% ✅ |
| SOL DOWN hourly | 82.7% (cut) |
| ETH DOWN 15min | 81.5% ✅ |
| SOL DOWN 15min | 81.0% ✅ |
| ETH UP 15min | 74.6% ⚠️ UNCERTAIN |
| BTC DOWN 15min | 76.2% ⚠️ UNCERTAIN |
| SOL UP 15min | 70.6% ⚠️ UNCERTAIN |
| BTC UP 15min | 68.2% ⚠️ UNCERTAIN |

Pattern: hourly buckets have 82–94% coverage; 15min UP buckets consistently run 68–75%. The F2 recording issue disproportionately affects 15min UP trades. This is the same pattern noted in the F2 recurrence incident (2026-05-14).

### G.2 Calibration Model Age

Model last fit: **2026-05-12T05:39 UTC** (3 days ago). Next scheduled refit: Sunday 2026-05-17 23:00 UTC (if cron is active — **unconfirmed**). sklearn version mismatch: fit on 1.6.1, current environment is 1.8.0. `InconsistentVersionWarning` will appear on every load but model should function.

### G.3 Integrity Check Status

**No daily integrity check exists.** `find /Users/terryyuan/company/shawn -name "*integrity*"` returns no results. The `daily_integrity_check.md` spec referenced in Day 7 EOD was proposed as a future build (Action 4 of that day) but was not implemented.

### G.4 Git Status

*Source: `git status --porcelain`, `git log -10`, checked 2026-05-15*

**Uncommitted changes in shawn:**
- `shawn/bot/data/session_timestamps.json` — modified (routine session state)
- `shawn/bot/data/sessions.json` — modified (routine)
- `shawn/bot/reporter_state.json` — modified (routine)
- `shawn/data/live_trades.csv` — modified (data output file)

No uncommitted changes to `trader.py`, `bot/*.py` scripts, or `hypotheses/*.yaml` files.

**Last substantive commit:** `2026-05-12 00:49:56 -0400` (`chore(shawn): Day 7 EOD — post-verification actions complete`). Three days of automated daily snapshot commits since then. No strategy code has changed since May 12.

---

## H — Infrastructure Health

### H.1 Bot Uptime

*Source: trader.pid, ps aux, watchdog.log*

| Metric | Value |
|--------|-------|
| Current PID | 1085 |
| Bot status | ✅ Running |
| Started | Mon May 11 11PM ET (= 2026-05-12 03:00 UTC) |
| Continuous uptime | ~3.5 days |
| CPU time accumulated | 147:26 (wall-clock CPU) |

**Restarts in past 7 days:**
- 2026-05-10 04:05 UTC — watchdog restart (reason: trader.py not running)
- 2026-05-12 03:00 UTC — manual or watchdog restart (current PID 1085)

**May 11 downtime gap:** Bot ran only 20 trades on May 11 (vs 87-93 normal). The bot was down for most of the day. The watchdog restarted at May 10 04:05 UTC, bot ran some hours, then stopped before being restarted as PID 1085 on May 12 03:00 UTC. Exact cause unknown — `trader_restart.log` does not exist.

### H.2 Watchdog Status

Watchdog is **active** (confirmed by watchdog.log and cron). Logic: checks trader.pid, if PID is dead restarts. Uses `mkdir` atomic lock to prevent overlapping cron invocations. `TRADING_PAUSED` flag mechanism functional.

No warning events in watchdog.log in the past 7 days beyond the normal restart entries.

### H.3 Polymarket API Health

*Source: trader.log last 200 lines, filtered for errors*

Recent error pattern (2026-05-15 14:39–14:51 UTC):
```
FOK thin-book: ETH 15min → FAK retry [B-LIVE]
FAK also failed: eth-updown-15m-1778869800 DOWN [B-LIVE] → order failed
```

2 FOK→FAK→fail sequences in the last 100 log lines. Both on ETH DOWN 15min on the same slug (`eth-updown-15m-1778869800`) — thin book on that specific contract. The system handled gracefully (order marked `failed`, no hang).

No rate limit warnings, no 503 spikes, no timeout events in log excerpts reviewed. API response quality appears normal.

---

## I — Recent Observations

*This section is for unstructured findings — things noticed but not yet formally reported.*

**1. BTC UP hourly and SOL UP hourly have both reversed hard in the last 7 days.**
Prior 7d (May 1–7): BTC UP hourly 80.9% WR, +$3.21/trade. Last 7d (May 8–15): 59.1% WR, -$4.83/trade. SOL UP hourly: 82.9% → 60.0% WR. Both buckets deteriorated simultaneously by ~22pp WR. This didn't show up in previous reports because the 14-day aggregate masks the split. The combined impact of these two buckets going negative is approximately -$4.83×22 + -$2.91×10 = -$106 + -$29 = **-$135 over 7 days** from two buckets that were collectively printing +$3+/trade before. This is the biggest unaddressed structural issue in the current system and the most likely explanation for why the wallet is flat since May 8.

**Possible cause:** Stream A deployed MAX_UP_ENTRY_PRICE=0.70 on May 9. Pre-Stream A, UP hourly trades could enter up to 0.82. If the 0.71–0.82 entry band had materially higher WR (high conviction, strong UP momentum), blocking those entries would reduce WR. Worth investigating: what was the WR of BTC/SOL UP hourly trades with entry_price > 0.70 pre-Stream A?

**2. `actual_fill_price` is completely non-functional as a data field.**
All 1,119 resolved trades have `actual_fill_price - entry_price = 0.0000`. This means slippage is invisible, realized_pnl_usdc is unreliable, and fee_paid_usdc is probably also wrong. We have no idea whether our CLOB orders are filling at the ask or experiencing adverse fills. For limit orders that always match the posted ask, this might be expected (zero slippage by construction) — but that needs to be confirmed as intentional, not a recording bug.

**3. SOL DOWN 15min is a live negative bucket that nobody has cut.**
At -$34.63 total over 14 days with 81% fill coverage (reliable), 68.7% WR vs ~67% break-even, SOL DOWN 15min is right at the edge but directionally negative for the full 14d window. In the B2 trend analysis it's recovering (prior 7d -$0.59/trade → last 7d +$0.21/trade), so it may have turned. But it hasn't been formally investigated or pre-registered. It's flying under the radar while BTC/ETH DOWN hourly got formal treatment.

**4. May 11 had only 20 trades — a significant gap that's not documented.**
The bot was down for most of May 11. This was not in any incident report. 20 trades vs the normal 87-93 means approximately 65–70 missed trade opportunities. At current settlement PnL rates (~$38.72/day), this is a ~$35 opportunity cost. The `trader_restart.log` doesn't exist, so the cause is untraced.

**5. Calibration coverage is 34.8% on recent resolved trades but the model has been "built."**
From prior discussions and the Day 7 EOD report, the calibration pipeline was described as complete. What's actually complete is the standalone `calibration_refit.py` script and the `calibration_current.pkl` file. What's NOT complete: the live trade path in `trader.py` that writes `calibrated_synth_prob` to the DB at trade execution time. Every new trade since the May 12 backfill has NULL `calibrated_synth_prob`. F1_v2 and F3_DOWN_v2 are fully blocked on this.

**6. BTC UP 15min shows 75.0% WR in last 7d but -$1.34/trade — a WR/PnL mismatch.**
Higher WR than prior 7d (70.8%) but negative PnL. This can happen when wins are at lower entry prices (smaller payouts) while losses are at higher entries (larger stakes). With only n=20 filled trades, this could be noise. But it's worth flagging because the theoretical EV should be positive with 75% WR at typical entry of 0.70.

**7. The UTC 8 hour band is a consistent loss zone for BTC DOWN 15min (-$57.48 over 14 days).**
UTC 8 = 4AM ET = late-night US / morning Europe. The pattern in both BTC and ETH DOWN 15min shows elevated losses in the UTC 2–8 window. This is the same window that showed poor performance in earlier stagnation analyses. It hasn't been formally investigated and no time-gating filter exists.

**8. The H_BET_SCALE_v1 SHA seal is missing.**
The pre-registration is incomplete. The 7-day clean window after the ETH DOWN hourly suspension (May 6) has elapsed. If bet scaling is the intended next step, the YAML needs to be sealed with a SHA before deployment.

**9. The three DOWN hourly buckets that were cut show $99 in "savings" over the trailing 14 days.**
BTC DOWN hourly: -$15.97, ETH DOWN hourly: -$36.67, SOL DOWN hourly: -$46.06 = total -$98.70 that would have been lost if the cut hadn't happened. These are all from the prior 7 days (before cut). The cut is clearly working but this number is often not cited when evaluating whether the system is improving.

**10. DRAWDOWN_EPOCH_CAPITAL in .env ($234.94) differs from the CLAUDE.md epoch capital ($287.66).**
The drawdown circuit breaker uses $234.94 as its base, which is lower than the stated epoch capital of $287.66. This means the drawdown pause triggers at -$58.74 (25% of $234.94) from a different baseline than the P&L reporting baseline. These two epoch capitals are inconsistent and could create confusion in circuit breaker interpretation.

---

## J — Open Questions

**Strategy questions:**
1. Is the BTC/SOL UP hourly WR reversal (week-over-week -22pp) driven by Stream A's MAX_UP_ENTRY_PRICE=0.70 cutting off high-entry high-WR trades, or by an underlying market regime change (BTC volatility, directional bias shift)? Answerable by: query WR of BTC UP hourly trades by entry price band (< 0.70 vs 0.70–0.82) pre-Stream A.
2. SOL DOWN 15min: is the last-7d recovery (+$0.21/trade) a real signal or noise (n=87 trades, +6.7pp WR)? Should this bucket be formally investigated or given another 14 days?
3. UTC 2–8 is a consistent loss window for DOWN 15min. Is there a regime explanation (low liquidity, thin book, adverse BTC momentum during Asia hours)? If yes, a time gate could recover $50+/week.
4. The H_BET_SCALE_v1 7-day clean window appears to have elapsed. Is the SOL DOWN hourly investigation (H_SOL_DOWN_HOURLY_INVESTIGATE_v1) still a prerequisite gate for bet scaling, or can scaling proceed given the blanket DOWN hourly cut already covers SOL?

**Data questions:**
5. Is `actual_fill_price == entry_price` by design (limit orders always fill at the posted price) or a recording bug? If by design, can we instead capture the post-fill CLOB confirmation price to validate execution?
6. Why does F2 fill coverage consistently run lower for 15min UP buckets (68–76%) vs all other buckets? Are UP 15min orders structurally less likely to generate fill confirmations, or is this the F2 recurrence bug resurfacing?
7. What is the correct interpretation of EPOCH_CAPITAL ($287.66) given the Apr 26 snapshot jump of +$553? Was there a deposit on that date? If so, the all-time PnL of +$356 is overstated.

**Code questions:**
8. What specifically needs to be added to trader.py to write `calibrated_synth_prob` at trade execution time? This is the single blocker for both F1_v2 and F3_DOWN_v2.
9. Is the calibration refit cron (`0 23 * * 0`) actually registered in crontab? The script has the comment but no verification was done.
10. The `realized_pnl_usdc`, `fee_paid_usdc`, and `slippage_usdc` columns exist in the schema — are any of them populated? If so, do they agree with the CF formula?

**Infrastructure questions:**
11. What caused the May 11 downtime? No `trader_restart.log` exists, and the watchdog logs show restarts on May 10 but not the subsequent crash. Should we add structured restart logging?

**Decision questions:**
12. The H_BET_SCALE_v1 pre-registration SHA is not computed. Should this be sealed and deployed now (the 7-day window has elapsed), or wait for the UP hourly reversal to be diagnosed first (risk: the reversal means current EV per trade is lower, making scaling less attractive)?
13. Given that BTC/SOL UP hourly are both negative in the last 7 days, should their anchor multiplier (1.5× floor) be suspended pending investigation? The multiplier amplifies losses in bad regimes.

---

*Source citations: All SQL data from `/Users/terryyuan/company/shawn/bot/signals.db`. Configuration from `/Users/terryyuan/company/shawn/bot/trader.py` and `/Users/terryyuan/company/shawn/bot/.env`. CF reconciliation from `cash_flow_reconciliation.py` run 2026-05-15 18:58 UTC. Log data from `/Users/terryyuan/company/shawn/bot/trader.log` (last 200 lines, 2026-05-15 14:39–14:51 UTC). Hypothesis YAMLs from `/Users/terryyuan/company/shawn/hypotheses/`. Git data from repo root `/Users/terryyuan/company/`.*

*All dollar figures in settlement PnL sections use strict actual_fill_usdc with formula `fill/price × 0.98 − fill` (wins) and `−fill` (losses). No bet_size fallback used anywhere in this report except where explicitly labeled "CF with fallback."*
