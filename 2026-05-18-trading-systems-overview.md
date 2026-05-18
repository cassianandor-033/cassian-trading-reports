# Cassian Trading Corp — Trading Systems Overview
*Updated: May 18, 2026 | Audience: Non-technical*

---

## What We're Actually Doing

We run automated trading bots on **Polymarket** — a prediction market platform where you bet on real-world events. Every market is a question with a Yes/No answer: "Will BTC be above $80k by end of May?" "Will the Fed cut rates in June?" The market price is the crowd's probability — if something trades at $0.65, the market thinks it has a 65% chance of happening.

**How we make money:** We use a neural network called **Synth** that reads short-term crypto price momentum and generates its own probability estimate. When Synth's estimate meaningfully disagrees with what the market is charging, we bet. If the market says 65% but Synth says 75%, there's edge. We size our bet based on how confident Synth is.

**What we're trading:** Polymarket event contracts on BTC, ETH, and SOL price movements over 15-minute and 1-hour windows. Example: "Will BTC be higher in 15 minutes?" These resolve quickly — a 15-minute contract pays out 20 minutes later — so we cycle through a lot of trades.

---

## The Two Systems

### B-Live — The Real Money Bot

| | |
|--|--|
| **Status** | Live — trading real USDC on Polymarket |
| **Wallet** | ~$718 total value |
| **Started** | April 18, 2026 |
| **Trades placed** | 3,557 (last 30 days) |
| **Daily volume** | ~90–130 trades/day on normal days |

**How it works:**

1. Every 15 minutes, Synth scores the market for BTC, ETH, and SOL in both directions (UP and DOWN).
2. If Synth's conviction is high enough, the bot places a limit order on Polymarket's order book.
3. The order either fills (we're in the trade) or gets rejected (market moved, we skip it).
4. We hold until the contract resolves, then collect winnings or accept the loss.

**Bet sizing — three tiers:**
- **T1** (moderate conviction): ~$5/trade
- **T2** (higher conviction): ~$8–12/trade  
- **T3** (highest conviction): ~$18/trade

**Guardrails built in:**
- Skip two specific hours of the day (UTC 10 and 20) — historically lossy due to US market open/close volatility
- No DOWN hourly bets (these were identified as negative-edge and cut on May 6)
- Cap on UP entry price at 70¢ — won't buy if the market already agrees too strongly
- Drawdown protection — bot pauses if losses exceed 25% of starting capital in a period

**Performance (last 14 days):**

| Asset | Direction | Timeframe | Win Rate | # Trades |
|-------|-----------|-----------|----------|-----------|
| ETH | DOWN | 15-min | **79.7%** | 349 |
| BTC | DOWN | 15-min | **76.3%** | 486 |
| ETH | UP | 15-min | **86.8%** | 38 |
| SOL | UP | 15-min | 78.3% | 23 |
| SOL | UP | hourly | 75.0% | 32 |
| BTC | UP | 15-min | 75.5% | 53 |
| BTC | UP | hourly | 68.8% | 48 |
| SOL | DOWN | 15-min | 70.6% | 170 |

**Overall 30-day win rate: 74.3%** across 3,557 resolved trades.

**Real PnL:** ~+$430 since epoch start (Apr 7). Source: portfolio snapshots (on-chain USDC), the only reliable measure. All dollar figures in strategy reports use this as ground truth — not the formula estimates the bot also tracks internally.

---

### D1 Paper — The Research Simulator

| | |
|--|--|
| **Status** | Paper trading (simulated — no real money) |
| **Bankroll** | Starts at $100, compounds via formula |
| **Started** | April 8, 2026 |
| **Trades simulated** | 6,059 all-time |
| **Daily volume** | ~150–160 trades/day |

**What it is:**

D1 paper is a simulation that runs the same Synth signals as B-live but with two key differences: it places no real orders, and its bet sizes grow as it "wins" (fixed-fractional compounding). It's a research tool — a way to test whether the underlying strategy has edge before committing real money to a configuration change.

**Why it exists:**

Before deploying any new filter or threshold to B-live, we can run it in D1 paper first. If D1 paper's win rate holds, it's evidence the change is valid. If it degrades, we don't ship it.

**Key differences from B-live:**

| Feature | D1 Paper | B-Live |
|---------|----------|--------|
| Money | Simulated (no real USDC) | Real USDC |
| DOWN hourly bets | ✅ Still fires | ❌ Cut (negative EV) |
| UP entry price cap | None — bets up to 82¢ | Capped at 70¢ |
| Bet sizing | % of compounding bankroll | Fixed dollar tiers |
| Slippage / rejections | None (assumes perfect fills) | Real CLOB friction |

**Performance (last 14 days):**

| Asset | Direction | Timeframe | Win Rate | # Trades |
|-------|-----------|-----------|----------|-----------|
| SOL | UP | hourly | **83.0%** | 53 |
| ETH | UP | hourly | **78.6%** | 56 |
| ETH | DOWN | 15-min | 77.3% | 431 |
| SOL | DOWN | 15-min | 77.3% | 255 |
| ETH | UP | 15-min | 77.4% | 62 |
| BTC | DOWN | 15-min | 73.7% | 609 |
| BTC | UP | hourly | 70.4% | 54 |
| BTC | DOWN | hourly | 61.6% | 224 |
| ETH | DOWN | hourly | 64.8% | 193 |

**Overall 30-day win rate: 72.7%** across 4,789 resolved trades.

**⚠️ Important caveat on D1 paper dollar figures:** D1 paper's formula bankroll has grown from $100 to several million dollars on paper. This number is **not real money and cannot be taken at face value.** Because D1 paper assumes perfect fills at theoretical prices with no fees, its dollar figures compound unrealistically. The win rate (72.7%) is the only reliable metric from D1 paper — the dollar figures are artifacts of the simulation design.

---

## How the Two Systems Compare

| Metric | B-Live | D1 Paper |
|--------|--------|----------|
| Win rate (30d) | **74.3%** | 72.7% |
| Trades/day | ~100 | ~155 |
| Real money | ✅ Yes | ❌ No |
| Hourly DOWN bets | ❌ Cut | ✅ Still running |
| UP entry cap | ✅ 70¢ max | ❌ None |

**B-live is outperforming D1 paper on win rate.** This makes sense: B-live has been refined with filters that cut the negative-EV buckets (hourly DOWN, which runs 60–65% WR vs B-live's threshold of ~74%). D1 paper still fires those trades, dragging its overall WR below B-live.

D1 paper's higher trade count comes from the same filters — it runs more buckets that B-live has excluded.

---

## What the Numbers Mean

A 74–76% win rate on binary contracts priced around 65–68¢ generates positive expected value per trade. To see why:

- You pay ~66¢ to win $1 if right, lose 66¢ if wrong.
- Breakeven win rate = 66% (you need to win 66% just to break even).
- At 74% WR, you're winning 8 percentage points above breakeven — that's real edge.

**Why we don't just bet everything:** Polymarket markets are thin. Large orders move the price and eat into the edge. We size to the book depth. At current bet sizes ($5–18/trade), slippage is minimal.

---

## Current State and Open Questions

**Working well:**
- DOWN 15-minute bucket is the workhorse — high volume, consistently 75–80% WR across all assets
- ETH UP 15-min is the strongest bucket right now (86.8% WR over 38 trades)
- Cutting DOWN hourly on May 6 was the right call — those buckets ran 60–65% WR, below breakeven

**Active investigations:**
- BTC UP hourly degraded in the prior audit window — was partly caused by an entry price cap cutting high-WR trades. Under ongoing review.
- Calibration model exists but isn't wired into live trading yet — when it ships, it should improve the accuracy of "is there real edge here" decisions.

**What we're not doing yet:**
- Scaling bet sizes — wallet has grown but bets are still at initial tier sizes. A bet scaling experiment (H_BET_SCALE_v1) is pre-registered but waiting on stability confirmation.

---

*Report generated from signals.db queries run May 18, 2026. Win rates are from resolved trades only. Real PnL is from portfolio snapshots (on-chain USDC), not formula estimates.*
