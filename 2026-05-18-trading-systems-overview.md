# Cassian Trading Corp — Trading Systems Overview
*Updated: May 18, 2026 | Audience: Non-technical readers*

---

## What We're Actually Doing

We run automated trading bots on **Polymarket** — a prediction market platform where you bet on real-world events with real money. Every market is a Yes/No question: "Will BTC be higher in 15 minutes?" "Will ETH close above $2,500 today?" The market price is the crowd's probability — if something trades at $0.65, the crowd thinks it has a 65% chance of happening.

**How we make money:** We use a neural network called **Synth** that reads short-term crypto price momentum and generates its own probability estimate. When Synth's estimate meaningfully disagrees with what the market is charging, we bet. If the market says 65% but Synth says 75%, that gap is our edge. We size our bet based on how confident Synth is.

**What we're trading:** Polymarket event contracts on whether BTC, ETH, or SOL will be higher or lower over the next 15 minutes or 1 hour. These resolve quickly — a 15-minute contract pays out 20 minutes later — so we cycle through ~100 trades per day.

---

## The Two Systems

### 1. B-Live — Real Money Bot

| | |
|--|--|
| **Status** | Live — trading real USDC on Polymarket |
| **Wallet** | ~$718 total value |
| **Epoch start** | April 7, 2026 (at $287.66) |
| **Real PnL** | ~+$430 since epoch start |
| **Trades placed** | 3,557 resolved (last 30 days) |
| **Daily volume** | ~90–130 trades/day |

**How it works, step by step:**

1. Every 15 minutes, Synth scores BTC, ETH, and SOL — generating a probability that each will go UP in the next 15 min or 1 hour.
2. If Synth's conviction is high enough (above a threshold we've tuned), the bot places a limit order on Polymarket's order book.
3. The order either fills (we're in the trade) or gets rejected (market moved too fast).
4. The contract resolves automatically — we collect winnings or take the loss.

**Bet sizing — three tiers:**
- **T1** (moderate conviction): ~$5/trade
- **T2** (higher conviction): ~$8–12/trade
- **T3** (highest conviction): ~$18/trade

**Guardrails built in:**
- Two hours of the day are skipped entirely (UTC 10 and 20) — historically lossy around US market open/close
- DOWN hourly bets are cut — identified as negative expected-value on May 6
- UP trades are capped at 70¢ entry — we won't pay the market more than 70¢ to win $1
- Drawdown protection — bot pauses if losses exceed 25% of capital in a rolling window

**Current performance (last 14 days, resolved trades only):**

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

**Why win rate matters:** These contracts are priced around 65–68¢ to win $1. Breakeven is ~66% win rate. At 74%, we're 8 percentage points above breakeven — that's real statistical edge.

**Real PnL note:** ~+$430 since April 7. This figure comes from portfolio snapshots (on-chain USDC balance), which is our ground-truth measurement. More on this in the Next Steps section.

---

### 2. D1 Paper — Research Simulator

| | |
|--|--|
| **Status** | Paper trading — simulated, no real money |
| **Started** | April 8, 2026 |
| **Total trades simulated** | 6,059 all-time |
| **Daily volume** | ~150–160 trades/day |

**What it is:**

D1 paper is a simulation that runs the same Synth signals as B-live but places no real orders. It exists so we can test new configurations safely — if a change degrades D1 paper's win rate, we don't ship it to B-live.

**Performance (last 14 days):**

| Asset | Direction | Timeframe | Win Rate | # Trades |
|-------|-----------|-----------|----------|-----------|
| SOL | UP | hourly | **83.0%** | 53 |
| ETH | UP | hourly | **78.6%** | 56 |
| ETH | DOWN | 15-min | 77.3% | 431 |
| SOL | DOWN | 15-min | 77.3% | 255 |
| ETH | UP | 15-min | 77.4% | 62 |
| BTC | DOWN | 15-min | 73.7% | 609 |
| BTC | DOWN | hourly | 61.6% | 224 |
| ETH | DOWN | hourly | 64.8% | 193 |

**Overall 30-day win rate: 72.7%** across 4,789 resolved trades.

**⚠️ Why D1 paper's dollar figures are meaningless:**

D1 paper shows a paper bankroll that has grown from $100 to several million dollars. **This number is not real and should be ignored entirely.** Here's why it happens:

The simulation uses a "compounding" bet model — each bet is a fixed percentage of the current paper bankroll. When a trade wins, the bankroll grows, and the next bet is larger. Over thousands of trades at 72% win rate with no fees, no slippage, and no rejected orders, this compounds exponentially.

Real trading doesn't work this way. B-live faces real CLOB slippage, orders that get rejected, and transaction friction. D1 paper assumes every trade fills perfectly at the theoretical price. The **win rate (72.7%)** is the only number worth taking from D1 paper. The dollar figures are an artifact of the simulation design.

**How D1 paper differs from B-live:**

| Feature | D1 Paper | B-Live |
|---------|----------|--------|
| Real money | ❌ No | ✅ Yes |
| DOWN hourly bets | ✅ Still fires | ❌ Cut (negative EV) |
| UP entry price cap | ❌ None | ✅ Capped at 70¢ |
| Bet sizing | % of compounding bankroll | Fixed dollar tiers |
| Slippage / rejections | None (perfect fills assumed) | Real CLOB friction |

**B-live outperforms D1 paper on win rate (74.3% vs 72.7%)** — because B-live has cut the negative-EV buckets (DOWN hourly runs ~61–65% WR) that D1 paper still fires. D1 paper's higher trade count comes from running those extra buckets.

---

## Next Steps

### 1. Diagnosing B-Live Wallet Stagnation

The B-live wallet has been roughly flat for the last several weeks despite a positive win rate. This is the team's primary focus right now.

**What we know:**
- The wallet grew well in an early "golden week" (W3), then plateaued. Real PnL is +$430 total over ~6 weeks, but most of that came in one good week.
- In early May, we deployed a filter called Stream A that caps UP trade entries at 70¢. This cut 71–93% of UP hourly trade volume (we had been seeing high win rates on UP trades at 71–82¢ entries that are now blocked).
- The DOWN hourly bucket was cut on May 6 — it was losing money.

**What we're investigating:**
- Is Stream A's 70¢ entry cap too restrictive? Band B trades (71–82¢ entries) had 87–90% win rates pre-cap, but Stream A blocks all of them. Removing or raising the cap is the leading candidate intervention.
- Are there other filters in the stack that are blocking profitable trades we'd otherwise take?
- Is this a genuine market regime change (Synth's signals are weaker now), or a configuration issue?

**What we're not doing:** We're not stacking multiple changes at once — one change per 7-day window, so we can isolate what moves the needle.

---

### 2. PnL Recording — Partially Solved, One Gap Remains

**The problem:** The bot internally tracks a column called `pnl_usdc` using a theoretical formula. For live traders, this formula is ~2–8× inflated vs actual realized USDC (it assumes ideal fills at cheap prices). We caught this discrepancy and now treat `pnl_usdc` as unreliable for any operator-facing number.

**What's in place:**
- ✅ **Portfolio snapshots** — The bot polls the Polymarket API hourly and records the wallet's free USDC balance. This is on-chain adjacent and is our primary ground-truth number for "how much money do we have."
- ✅ **Settlement PnL script** (`cash_flow_reconciliation.py`) — Computes real per-trade PnL using actual fill amounts from CLOB confirmations, not the formula. This is the right approach.

**The remaining gap:**
- ⚠️ Settlement PnL only covers ~74.8% of trades. The other ~25% are missing actual fill data — the bot falls back to bet_size when the fill lookup fails. This means the settlement script still has a systematic gap.
- ⚠️ Portfolio snapshots capture free USDC only. When positions are open, the snapshot undercounts true wallet value by the cost basis of those positions. We have a cleaning formula for this, but it requires historical reconstruction to be accurate.

**Bottom line:** We have the right measurement infrastructure in place. The portfolio snapshot (cleaned for open positions) is the most reliable single number. Settlement PnL is the right per-trade metric but needs better fill data coverage. Formula `pnl_usdc` is banned from all operator-facing reporting.

The fill data coverage gap is the last open item — fixing it requires ensuring the CLOB fill lookup never falls back to bet_size. That's an engineering task, not a strategy one.

---

*Report generated from signals.db queries run May 18, 2026. Win rates are from resolved trades only. Real PnL is from portfolio snapshots (on-chain USDC), not formula estimates. Do not quote D1 paper dollar figures as performance — use win rate only.*
