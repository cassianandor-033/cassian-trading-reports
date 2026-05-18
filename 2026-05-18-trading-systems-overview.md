# Cassian Trading Corp — Trading Operations Briefing
*Prepared for: Incoming CEO | Date: May 18, 2026 | Classification: Internal*

---

## Executive Summary

Cassian Trading Corp runs automated algorithmic trading bots on Polymarket, a prediction market platform where real money is wagered on real-world event outcomes. Our edge comes from a third-party AI signal provider called **Synth**, which generates short-term crypto price direction probabilities that are more accurate than what Polymarket's crowd prices in. When Synth disagrees with the market by a meaningful margin, our bots place bets automatically.

**Current state:**
- **B-live** (our primary live strategy): ~$718 wallet, +$430 real profit since April 7, 74.3% win rate across 3,557 resolved trades
- **D1 paper** (research simulator): 72.7% win rate across 6,059 simulated trades — confirms the strategy has statistical edge
- **Active issue:** B-live wallet growth has stalled. Under active diagnosis — likely caused by an overly restrictive filter we deployed in early May

---

## Part 1: What Is Polymarket and How Do We Make Money?

### The Platform

Polymarket is a prediction market where you trade on the probability of real-world events. Every market is a binary question with a Yes/No outcome:

- "Will BTC be higher in 15 minutes?"
- "Will ETH close above $2,500 today?"
- "Will the Fed cut rates in June?"

Each contract pays out exactly **$1 if the event happens, $0 if it doesn't.** If a contract is trading at $0.65, the crowd collectively believes there is a 65% chance the event occurs. You pay 65¢ today; if you're right, you collect $1. If you're wrong, you lose your 65¢.

All settlement is in **USDC** — a dollar-pegged stablecoin on the Polygon blockchain. There is no counterparty risk in the traditional sense; the contracts are trustless smart contracts. Payouts happen automatically on-chain when a market resolves.

### Our Edge: Better Probability Estimates Than the Crowd

The market price at any moment is the crowd's best estimate of the true probability. We make money when **our estimate is more accurate than the crowd's**. The gap between our estimate and the market price is our *edge*.

We don't try to generate our own estimates from scratch. Instead, we license signals from **Synth** — a specialist AI model that has a track record of estimating short-term crypto price direction better than Polymarket's crowd.

**The math:** These contracts have a simple payoff structure. If we pay 65¢ for a contract and our model says the true probability is 75%:

```
Expected value per $1 bet:
  75% × ($1.00 - $0.65) = +$0.2625   (collect $1, spent $0.65)
  25% × (-$0.65)        = -$0.1625   (lose the 65¢)
  Net EV                = +$0.10 per dollar wagered  →  10% edge
```

Across ~100 trades per day at $5–18 per trade, a 10–13% edge compounds into meaningful returns. Our realized win rate of 74.3% against an average entry price of ~0.65¢ puts us consistently above breakeven (which would be 65%).

### Why We Trade Crypto Price Direction Contracts

We focus specifically on **15-minute and 1-hour BTC/ETH/SOL price direction markets** for three reasons:

1. **High liquidity** — These markets have sufficient book depth to absorb our order sizes without moving the price against us.
2. **Fast resolution** — A 15-minute contract resolves in 20 minutes. Capital turns over quickly, compounding returns rather than sitting locked up.
3. **Synth's specialization** — Synth is explicitly built for short-term crypto momentum prediction. Its edge is strongest here.

We focus more heavily on **DOWN contracts** (betting the price will fall). Crypto tends to drop faster than it rises — panic selling is more persistent and predictable than accumulation — which makes Synth's downward momentum signals more reliable. Our DOWN 15-min bucket runs at 76–80% win rate consistently.

---

## Part 2: Synth — Our Signal Provider

### What Synth Is

Synth is a third-party AI signal service we subscribe to. It runs a neural network trained on crypto price data that outputs a single number per market, per asset, per time horizon: **the probability that the asset will be higher at contract resolution**. We don't have visibility into Synth's model internals — we consume it as a black box through their API.

### What Synth Provides

Every 5 minutes, we poll Synth's API for each combination of asset (BTC, ETH, SOL) and horizon (15-minute, 1-hour). Each API response contains:

| Field | Meaning |
|-------|---------|
| `synth_probability_up` | Synth's model probability the asset ends UP (0.0 – 1.0) |
| `polymarket_probability_up` | Polymarket's current market price for the UP outcome |
| `best_ask_price` / `best_bid_price` | Live CLOB order book prices |
| `best_ask_size` / `best_bid_size` | Available liquidity depth in USDC |
| `event_end_time` | Exact resolution timestamp |
| `slug` | Market identifier for order placement |

From these, we compute two derived signals:
- **Direction:** UP if Synth's probability exceeds the market price; DOWN if it's below
- **Gross edge:** `synth_prob − market_price` (the raw disagreement)

### How We Evaluate Synth's Signal Quality

We track Synth's calibration over time — does a 70% Synth signal actually win 70% of the time? Our 30-day data shows strong calibration at the high end: trades where Synth is very confident (>0.75 synth_prob) win at 80–86%, which matches what a well-calibrated model should do. Lower-conviction signals are noisier.

We've built a standalone **calibration model** (isotonic regression fit on historical outcomes) that corrects Synth's raw probabilities toward observed win rates. This model is built and tested but not yet wired into live trading — when it ships, it will improve our edge estimation and help us filter out signals where Synth's raw probability overstates true confidence.

---

## Part 3: The Trading and Execution Engine

### Architecture Overview

The entire trading system lives in a single Python file: `trader.py` (~6,500 lines). It runs as a persistent background process on a dedicated machine, polling Synth every 5 minutes and placing orders on Polymarket's order book (the CLOB — Central Limit Order Book).

The main loop:

```
Every 5 minutes:
  1. Resolve all pending trades (check which markets have settled)
  2. Run safety checks (drawdown limits, paused state, USDC balance)
  3. For each asset × horizon: fetch Synth → run filter stack → place order if edge found
  4. Sleep until next cycle
```

A single **file lock** (`fcntl.flock`) serializes the entire check-place-log sequence to prevent duplicate orders from concurrent processes.

A separate subprocess (`phase0_collect.py`) continuously polls Polymarket for market resolutions and confirms outcomes directly against the **Polygon blockchain** (CTF smart contract `payoutNumerators`). This makes resolution tracking immune to database errors — we verify on-chain, not just from API responses.

### The Filter Stack — 23 Gates Before Any Order Is Placed

Every signal that comes from Synth passes through 23 sequential gates. The first failure kicks the trade out. This is the full stack, in order:

| # | Gate | What it does |
|---|------|-------------|
| 1 | Daily horizon kill | Daily-resolution markets excluded (57% WR — not worth it) |
| 2 | Synth conviction | UP requires synth_prob ≥ 0.62; DOWN requires synth_prob ≤ 0.40 |
| 3 | Time-to-resolution | Skip if < 3 min left (15min) or < 10 min left (hourly) |
| 4 | Signal freshness | Skip if Synth response is > 10 min old |
| 5 | Liquidity | Skip if book depth < $10 on the relevant side |
| 6 | Direction kill switches | Emergency env-var toggles: `SKIP_DOWN=true` or `SKIP_UP=true` |
| 7 | UTC hour gate | Skip UTC hours 10 and 20 (US market open/close — structural noise spikes confirmed) |
| 8 | DOWN hourly ban | All DOWN hourly trades disabled since May 6 (all below breakeven) |
| 9 | BTC momentum gate | Skip DOWN if BTC rising > 0.5%/h; skip UP if BTC falling > 0.5%/h |
| 10 | Duplicate trade check | Skip if we already have a pending order on this market |
| 11 | Cross-direction check | Skip if we already hold a position in the opposite direction on same market |
| 12 | Per-asset position cap | Max 3 open positions per asset |
| 13 | On-chain position check | Query Polymarket API to confirm wallet doesn't already hold token (restart-safe) |
| 14 | Live CLOB re-fetch | Immediately before order: re-fetch live ask price; reject if it's drifted |
| 15 | UP entry ceiling | UP entry > 0.70 skipped (Stream A — market agreement too high, edge gone) |
| 16 | T1 floor on cheap UP | T1 UP trades at entry < 0.55 skipped (razor-thin margin) |
| 17 | Positive-EV gate | `synth_prob > entry_price` for UP; `(1 − synth_prob) > entry_price` for DOWN |
| 18 | Crash regime | If BTC and ETH hourly UP both < 0.50: halve all UP bet sizes |
| 19 | Daily drawdown cap | If today's realized P&L < −10% of portfolio: floor all bets to T1 |
| 20 | Consecutive loss floor | If 10+ consecutive losses: floor all bets to T1 |
| 21 | FOK fail-rate dial-back | If > 40% of today's orders rejected: floor all bets to T1 |
| 22 | Per-cycle exposure cap | Max 4% of portfolio in 15-min bets per single scan cycle |
| 23 | Correlated window cap | Max 5% of portfolio in same resolution window + direction |

This stack reflects ~6 weeks of live refinement — every gate was added in response to an observed failure mode or a diagnosed negative-EV pattern.

### Bet Sizing

Conviction-based tiered sizing. Base tier is determined by Synth's probability score:

| Tier | UP threshold | DOWN threshold | Portfolio % | Floor |
|------|-------------|----------------|-------------|-------|
| T1 | synth_prob 0.62–0.70 | synth_prob 0.30–0.40 | 3.0% | $5 |
| T2 | synth_prob 0.70–0.80 | synth_prob 0.20–0.30 | 3.0% | $8 |
| T3 UP | synth_prob ≥ 0.80 | — | 5.0% | $12 |
| T3 DOWN | — | synth_prob ≤ 0.20 | 3.0% | $12 |

Multipliers applied on top:

| Modifier | Effect |
|----------|--------|
| 15-min haircut | × 0.60 (thinner books on short horizon) |
| BTC trend alignment | × 1.50 when 4h BTC momentum matches trade direction |
| BTC UP hourly anchor | × 1.50 minimum (structural UP boost) |
| Crash regime | × 0.50 on all UP bets |
| DOWN loss streak | × 0.60 (3–4 losses) or × 0.35 (5+ losses) |
| Drawdown floor | Forces T1 regardless of signal |

At the current ~$650 wallet, the dollar amounts work out to roughly T1 = ~$19, T2 = ~$19, T3 UP = ~$32, T3 DOWN = ~$19 before multipliers — though the $5/$8/$12 floors still apply if the wallet drops significantly.

### Order Execution

**Order type:** FOK (Fill-or-Kill) — the order must fill completely at our price or cancel immediately. No partial fills sitting on the book.

**Pre-order:** Immediately before placing an order, the bot calls Polymarket's CLOB to get the live clearing price. This prevents stale price execution if the book moved during the filter evaluation.

**Retry and fallback logic:**

| Scenario | Response |
|----------|----------|
| 425 "Too Early" (new market not ready) | Wait 8s, retry once; 15-min cooldown if persists |
| 403 geoblock (at market open) | Wait 30s, retry once |
| FOK "couldn't be fully filled" (thin book) | Retry as FAK (partial fill accepted) |
| `order_version_mismatch` (stale signed order) | Up to 3 retries with 0.5s backoff |
| 503 / maintenance mode | Skip cleanly |

**Fill confirmation:** After every order, the bot queries the CLOB `get_order(order_id)` up to 3 times (2s delay each) to get the actual fill price. If that fails, it falls back to `get_trades()`. This fill confirmation is logged to the database and used in settlement PnL calculations.

---

## Part 4: Live Performance

### B-Live — Primary Real-Money Strategy

B-live is our flagship strategy. It trades UP and DOWN contracts on BTC, ETH, and SOL across 15-minute and 1-hour horizons. All orders are real CLOB orders with real USDC.

**30-day summary:**

| Metric | Value |
|--------|-------|
| Resolved trades | 3,557 |
| Win rate | **74.3%** |
| Assets | BTC, ETH, SOL |
| Wallet (current) | ~$718 |
| Epoch capital (Apr 7) | $287.66 |
| Real PnL | **~+$430** |

**Performance by bucket (last 14 days):**

| Asset | Direction | Timeframe | Win Rate | Trades |
|-------|-----------|-----------|----------|--------|
| ETH | DOWN | 15-min | **79.7%** | 349 |
| BTC | DOWN | 15-min | **76.3%** | 486 |
| ETH | UP | 15-min | **86.8%** | 38 |
| SOL | UP | 15-min | 78.3% | 23 |
| SOL | UP | hourly | 75.0% | 32 |
| BTC | UP | 15-min | 75.5% | 53 |
| BTC | UP | hourly | 68.8% | 48 |
| SOL | DOWN | 15-min | 70.6% | 170 |

**PnL measurement:** Real PnL is tracked via hourly portfolio snapshots — the on-chain USDC balance in the trading wallet, adjusted for open position cost basis. Formula-based PnL estimates in the database are explicitly excluded from all reporting.

---

### D1 Paper — Research and Validation Simulator

D1 paper runs the same Synth signals as B-live but places no real orders. Its purpose is to validate configuration changes before they go live. If a proposed filter change improves D1 paper's win rate, it's a candidate for B-live. If it degrades it, we don't ship it.

**30-day summary:**

| Metric | Value |
|--------|-------|
| Resolved trades | 4,789 |
| Win rate | **72.7%** |
| All-time trades | 6,059 |

**Performance by bucket (last 14 days):**

| Asset | Direction | Timeframe | Win Rate | Trades |
|-------|-----------|-----------|----------|--------|
| SOL | UP | hourly | **83.0%** | 53 |
| ETH | UP | hourly | **78.6%** | 56 |
| ETH | DOWN | 15-min | 77.3% | 431 |
| SOL | DOWN | 15-min | 77.3% | 255 |
| ETH | UP | 15-min | 77.4% | 62 |
| BTC | DOWN | 15-min | 73.7% | 609 |
| BTC | DOWN | hourly | 61.6% | 224 |
| ETH | DOWN | hourly | 64.8% | 193 |

**Important caveat on D1 paper dollar figures:** D1 paper tracks a compounding paper bankroll that has ballooned from $100 to several million dollars on paper. **This number is not real and must be ignored.** The simulation assumes perfect fills at theoretical prices with zero fees and zero rejected orders, then compounds bet sizes as the paper bankroll grows. Real trading faces slippage, FOK rejections, and CLOB friction. The only reliable metric from D1 paper is **win rate**. Dollar figures from the simulator are banned from operator-facing reporting.

**Why B-live outperforms D1 paper on win rate (74.3% vs 72.7%):** B-live has cut the negative-EV buckets. DOWN hourly trades run at only 61–65% WR — below breakeven — and were suspended in B-live on May 6. D1 paper still fires them, dragging its average down. B-live's tighter configuration is the better-performing one.

---

## Part 5: Additional Live Strategies (Smaller Scale)

Beyond B-live and D1 paper, we run several smaller strategies in parallel — all sharing the same Synth signal feed and execution infrastructure:

| Strategy | Type | Purpose |
|---------|------|---------|
| **D1 live** | Real money (~$200 wallet) | Live execution of D1 paper parameters for direct comparison |
| **D4 live** | Real money | DOWN-only at very high conviction (synth ≤ 0.35) |
| **D2 paper** | Simulated | 2.5× scaling stress test — how does the strategy perform at higher bet sizes? |
| **D3 paper** | Simulated | HYPE-only variant (tests a different asset) |
| **D4 paper** | Simulated | DOWN-only paper version |
| **D5 / D6 paper** | Simulated | Ultra-conviction and anchor variants under evaluation |

These run in the same `trader.py` main loop, evaluated on the same 5-minute cycle.

---

## Part 6: Next Steps and Open Issues

### Issue 1 — B-Live Wallet Stagnation

**The problem:** B-live has been roughly flat for several weeks. The win rate is healthy (74.3%) but the wallet isn't growing meaningfully past its current level.

**Root cause (leading hypothesis):** In early May we deployed **Stream A** — a filter capping UP trade entries at 70¢. This was intended to prevent overpaying when the market already largely agrees with Synth. However, it had an unintended consequence: it blocked 71–93% of UP hourly trade volume. The trades it blocked — entries at 71–82¢ — were historically our *best* performers, running at 87–90% win rate before the cap was imposed.

Net effect: far fewer trades per day, with high-conviction UP hourly opportunities nearly eliminated. The wallet isn't losing money, but it has far fewer chances to grow.

**Planned action:** Evaluate raising or removing the 70¢ cap. Pre-registration required before any change. One change per 7-day window — no stacking changes, because stacking destroys our ability to attribute cause and effect.

**Secondary investigation:** Whether Synth's UP signals have genuinely weakened in the current market regime, or whether this is entirely a configuration problem.

### Issue 2 — PnL Measurement Gap

**The problem:** The bot internally calculates a theoretical `pnl_usdc` using a formula that is 2–8× inflated vs realized USDC for live traders. This was discovered in early May and triggered a measurement overhaul.

**Current status:**

| Tool | Status | Notes |
|------|--------|-------|
| Portfolio snapshots (on-chain USDC) | ✅ Running | Hourly, ground-truth wallet balance |
| Settlement PnL script | ✅ Built | Computes real per-trade PnL from CLOB fill data |
| Formula `pnl_usdc` | ❌ Banned | Excluded from all operator-facing reports |

**Remaining gap:** Settlement PnL only covers ~74.8% of trades. The other ~25% are missing actual fill data — the CLOB fill lookup fails intermittently and falls back to bet_size, which is less accurate. This is an engineering task: ensure the fill lookup never silently degrades to a fallback.

Additionally, portfolio snapshots capture free USDC only — open positions are not counted until they resolve. We have a cleaning formula that adds back open position cost basis to get true wallet value, but it requires careful historical reconstruction to be precise.

**Bottom line:** We have the right measurement infrastructure. The last open item is getting fill data coverage from 75% to 100%.

### Issue 3 — Calibration Model Not Yet Live

We've built and validated an isotonic regression calibration model that corrects Synth's raw probabilities toward historically observed win rates. It's been tested on holdout data and shows improvement in edge estimation. However, it is not yet integrated into the live `trader.py` execution path.

When it ships, the calibration model will make the positive-EV gate (filter #17) more precise — we'll be comparing calibrated probabilities against market prices rather than raw Synth scores. This should modestly improve both trade selection and bet sizing accuracy.

---

## Appendix: Key Files and Infrastructure

| File / System | Purpose |
|--------------|---------|
| `bot/trader.py` | Main trading bot — all strategy logic, execution, main loop (~6,500 lines) |
| `bot/.env` | Live configuration: API keys, tier floors, direction toggles, drawdown epoch |
| `bot/signals.db` | SQLite database: all trades, signals, portfolio snapshots (~83MB) |
| `bot/calibration_refit.py` | Standalone calibration model fitting and management |
| `cash_flow_reconciliation.py` | Settlement PnL computation from CLOB fill data |
| `phase0_collect.py` | Subprocess that polls Polymarket for market resolutions, verifies on Polygon |
| GitHub (public) | Audit reports, system overviews: `cassianandor-033/cassian-trading-reports` |

**Infrastructure:** Single Mac Mini running `trader.py` as a persistent process. Polygon blockchain for on-chain settlement verification. Polymarket CLOB API for order placement. Synth API for signal data.

---

*Data current as of May 18, 2026. Win rates sourced from signals.db resolved trades. Real PnL sourced from portfolio_snapshots (on-chain USDC). Formula pnl_usdc excluded from all figures in this report.*
