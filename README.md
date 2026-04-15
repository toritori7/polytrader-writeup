# Polytrader — Coding Agent Writeup

## What I built

**Polytrader** is an automated quantitative trading agent for Polymarket prediction markets. It crawls markets / price history / trade history into SQLite, scans for strategy signals, validates them through a backtest engine, and places live orders via `py-clob-client`.

The core strategy targets **low-probability, mid-volume** markets: detect abnormal price spikes (likely large market orders eating the book) and take the opposite side, waiting for mean reversion. A second **hazard-rate** strategy was added later for long-horizon low-probability events (e.g. "Will China invade Taiwan before 2027").

## How I used the coding agent

I used a coding agent to upgrade a read-only crawler project into a fully working live-trading system. I gave the agent a 5-phase `PLAN.md` (infra fixes → strategy module → backtest framework → execution layer → CLI integration) and made it advance through gated steps — the backtest pass criteria were a hard gate before any live-trading code was allowed to run.

## Key iterations

**v1 — Crawler wouldn't run**
- Problems: corporate Zscaler proxy broke every HTTPS request with SSL errors; Trades API rejected `limit=10000` with 400.
- Fix: added `polytrader/ssl_patch.py` to monkey-patch `requests` and `py-clob-client`'s httpx client; dropped `get_trades` limit to the official max of 500; imported the patch at the top of `main.py`.

**v2 — Strategy + backtest**
- Added `strategies/` (`base.py` / `signals.py` / `mean_reversion.py`) and `backtesting/` (walk-forward engine).
- Pass criteria: win rate > 55%, Sharpe > 1.0, profit factor > 1.5, trade count > 30. Fail the backtest → no live trading.
- Tightened selection filters over several rounds: YES price < 15¢, volume > $50K, days to expiry > 30.

**v3 — Live trading, lots of landmines** (all documented in `NOTE.md`)
- Proxy-wallet signing kept returning `invalid signature` → switched to `signature_type=0` EOA wallet.
- Mixed up USDC vs USDC.e once on withdrawal → locked the flow to USDC.e only (the bridged version is what Polymarket actually uses).
- Approving `max_uint256` failed → approving a fixed amount ($10) worked.
- Selling kept failing with "allowance is not enough" → discovered the CTF contract needs `setApprovalForAll` against all three exchange contracts (one-time, applies to all markets).
- Wrapped it all in `execution/safety.py`: per-order cap $5, total exposure $20, ≤10 orders/hour, spread < 3¢.

## Final result

- Crawler pulls 3 months of data — markets / price curves / trades in SQLite.
- Backtest engine produces full metrics and an interactive web dashboard (`visualize.py explorer`).
- **Full live BUY → SELL round-trip verified on 2026-04-06** (min $1 / 5 shares, ~1¢ round-trip spread).
- One-command CLI flow: `python main.py strategy backtest` / `trade execute --live`.

