# Cassian Trading Corp — Trading Systems Overview
*Updated: May 18, 2026 | Audience: Non-technical readers*

---

## What Is Polymarket?

Polymarket is a prediction market platform where you trade on the outcomes of real-world events using real money (USDC, a dollar-pegged crypto stablecoin). Every market is a Yes/No question:

- "Will BTC be higher in the next 15 minutes?"
- "Will the Fed cut rates in June?"
- "Will ETH close above $2,500 today?"

Each contract is worth exactly $1 if the event happens, and $0 if it doesn't. If a contract is trading at $0.65, it means the market collectively believes there's a **65% chance** the event occurs. You buy it at 65¢ and get paid $1 if you're right — a 54% return. If you're wrong, you lose your 65¢.

---

## How Our Strategy Makes Money

We trade short-term crypto price direction contracts on Polymarket — specifically whether BTC, ETH, or SOL will be higher or lower over the next **15 minutes** or **1 hour**.

### The Core Idea: Be More Accurate Than the Crowd

The market price is the crowd's best guess. We make money when **our probability estimate is more accurate than theirs**. That gap — our estimate minus the market price — is our *edge*.

We use a neural network called **Synth** to generate those estimates. Synth reads real-time crypto price momentum data and outputs a probability — for example, "BTC has a 74% chance of going up in the next 15 minutes." If Polymarket is only charging 63¢ for that contract, we have 11 percentage points of edge. We buy.

### The Math of Why This Works

These contracts have a simple payoff structure:

- **You pay:** the market price (e.g., 65¢)
- **You collect:** $1 if right, $0 if wrong
- **Breakeven win rate:** whatever you paid (pay 65¢ → need to win 65% of the time just to break even)

Our DOWN 15-minute contracts — our highest-volume bucket — average an entry price of ~65¢ and a win rate of **76–80%**. At 65¢ entry with a 78% win rate:

```
Expected value per $1 bet:
  78% × ($1 - $0.65)  =  +$0.273  (when you win)
  22% × (-$0.65)      =  -$0.143  (when you lose)
  Net EV              =  +$0.13 per dollar wagered
```

That's a **13% edge per dollar bet**. Across ~100 trades/day at $5–18/trade, it compounds meaningfully.

### Why We Focus on DOWN Contracts

Synth is better at predicting downward moves in the short term. This isn't unusual — crypto tends to drop faster than it rises (panic sells move quicker than accumulation), making downward momentum more persistent and predictable. Our DOWN 15-minute bucket is the workhorse: high volume, consistent 76–80% win rates across all three assets.

### Why We Don't Just Bet Everything

Three constraints cap our position sizes:

1. **Thin order books.** Polymarket markets don't have deep liquidity. A large order moves the price against you before it fills, eating into your edge. We size to the available liquidity.
2. **Diversification.** We spread across BTC, ETH, and SOL, across multiple time horizons, so one bad run doesn't blow up the portfolio.
3. **Uncertainty about edge.** Win rates fluctuate. We use conservative Kelly sizing — bet a fraction of what pure Kelly math suggests, to survive variance.

---

## The Two Systems

### 1. B-Live — Real Money Bot

| | |
|--|--|
| **Status** | Live — trading real USDC on Polymarket |
| **Wallet** | ~$718 total value |
| **Started** | April 7, 2026 at $287.66 |
| **Real PnL** | ~+$430 since start |
| **Resolved trades** | 3,557 (last 30 days) |
| **Daily volume** | ~90–130 trades/day |

**How it works step by step:**

1. Every 15 minutes, Synth generates a probability score for each asset and direction.
2. If conviction is high enough (above a tuned threshold), the bot places a limit order on Polymarket's order book.
3. The order either fills (we're in the trade) or gets rejected if the price moved before we got there.
4. The contract resolves automatically — we collect winnings or take the loss. No manual intervention.

**Bet sizing — three tiers:**

| Tier | Conviction level | Bet size |
|------|-----------------|----------|
| T1 | Moderate | ~$5 |
| T2 | High | ~$8–12 |
| T3 | Highest | ~$18 |

**Guardrails built in:**
- Two hours of the day are skipped entirely (UTC 10 and 20) — these overlap US market open/close and historically generate more noise than signal
- DOWN hourly bets are suspended — identified as negative expected-value and cut on May 6
- UP trades capped at 70¢ entry — above this the market agrees with Synth too much and edge disappears
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

**Real PnL:** ~+$430 since April 7. Sourced from portfolio snapshots (on-chain USDC balance) — our ground-truth measurement. More on this below.

---

### 2. D1 Paper — Research Simulator

| | |
|--|--|
| **Status** | Simulation — no real money |
| **Started** | April 8, 2026 |
| **Total simulated trades** | 6,059 all-time |
| **Daily volume** | ~150–160 trades/day |

D1 paper runs the same Synth signals as B-live but places no real orders. It's a research sandbox — a safe place to test configuration changes before they touch real money. If a proposed change degrades D1 paper's win rate, we don't ship it to B-live.

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

**⚠️ D1 paper dollar figures are not real — use win rate only.**

D1 paper's simulated bankroll has ballooned from $100 to several million dollars on paper. This number should be ignored entirely. The simulation assumes perfect fills, zero slippage, zero rejected orders, and compounds bet sizes as the "bankroll" grows. In reality, none of those assumptions hold. The **win rate (72.7%)** is the only number worth taking from D1 paper.

**How D1 paper differs from B-live:**

| Feature | D1 Paper | B-Live |
|---------|----------|--------|
| Real money | ❌ No | ✅ Yes |
| DOWN hourly bets | ✅ Still fires | ❌ Cut (negative EV) |
| UP entry price cap | ❌ None | ✅ Capped at 70¢ |
| Bet sizing | % of compounding paper bankroll | Fixed dollar tiers |
| Slippage / rejections | None (perfect fills assumed) | Real CLOB friction |

**B-live outperforms D1 paper on win rate (74.3% vs 72.7%)** because B-live has already cut the negative-EV buckets — DOWN hourly runs at 61–65% WR, well below breakeven. D1 paper still fires those, dragging its overall average down.

---

## Next Steps

### 1. Diagnosing B-Live Wallet Stagnation

The wallet has been roughly flat for several weeks. Real PnL is +$430 over ~6 weeks, but most of those gains came in a single good week early on. Since then: the win rate is healthy but the wallet isn't growing.

**What we believe is happening:**

In early May we deployed a filter called Stream A that caps UP trade entries at 70¢. This was intended to avoid overpaying — but it had a side effect: it blocked 71–93% of our UP hourly trade volume. The trades it blocked (entries at 71–82¢) were historically our *best* performers — Band B trades ran at 87–90% win rate before the cap.

Net result: far fewer trades per day, with the high-conviction UP hourly bucket nearly eliminated. The wallet isn't losing money, but it has far fewer opportunities to grow.

**What we're doing about it:**
- Evaluating whether to raise or remove the 70¢ entry cap — this is the leading candidate change
- Running one change at a time, 7-day windows minimum, so we can isolate cause and effect
- Monitoring whether Synth's UP signals have genuinely weakened (regime change) or if it's purely a configuration problem

### 2. PnL Recording — Partially Solved, One Gap Remains

**The problem we found:** The bot internally tracks a column called `pnl_usdc` using a theoretical formula. For live traders, this formula can be 2–8× inflated vs actual realized USDC — it assumes ideal fills at theoretical prices. We caught this discrepancy and banned formula PnL from all operator-facing reporting.

**What's working:**
- ✅ **Portfolio snapshots** — The bot polls Polymarket hourly and records the wallet's USDC balance. This is our ground-truth "how much do we have" number.
- ✅ **Settlement PnL script** (`cash_flow_reconciliation.py`) — Computes real per-trade PnL from actual fill amounts recorded by the CLOB. This is the right approach and what we use in all reports.

**What's still open:**
- ⚠️ Settlement PnL only covers ~74.8% of trades. The remaining ~25% are missing actual fill data — the bot falls back to bet_size when the CLOB fill lookup fails. This creates a systematic gap in per-trade accuracy.
- ⚠️ Portfolio snapshots capture free USDC only. Open positions are not included — the snapshot undercounts true wallet value by the cost basis of any currently-open positions. We have a cleaning formula for this, but it requires historical reconstruction.

**Bottom line:** We have the right infrastructure. The cleaned portfolio snapshot is the most reliable single number. The settlement PnL script is the right per-trade tool. The last open item is getting fill data coverage from 75% to 100% — that's an engineering task.

---

*Data sourced from signals.db, queried May 18, 2026. Win rates are from resolved trades only. All dollar figures use on-chain portfolio snapshots, not formula estimates. D1 paper dollar figures are excluded from this report as unreliable.*
