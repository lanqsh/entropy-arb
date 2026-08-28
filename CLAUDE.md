# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**entropy-arb** is a two-venue perpetual futures arbitrage bot written in Python. It trades the spread between **Entropy** (the `io` builder DEX on Hyperliquid) and one of three hedge venues: Lighter mainnet, Lighter Robinhood chain, or trade.xyz. The bot simultaneously buys on one venue and sells on the other when the price premium crosses configured thresholds, carrying delta-neutral positions until mean reversion.

The strategy is entirely data-driven: users collect order book data with `--record-only`, analyze it with `tools/analyze.py` to derive the premium's natural midline and entry bands, then configure those thresholds in `config.yaml`.

## Development Commands

### Setup
```bash
python3 -m venv .venv && source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install -r requirements.txt          # data collection only
pip install -r requirements-live.txt     # live trading (includes signing SDKs)
cp config.example.yaml config.yaml       # configure strategy
cp .env.example .env                     # fill in credentials for live trading
```

### Running
```bash
# Data collection (no credentials needed, no trading)
python3 main.py --record-only --symbol SNDK --hedge lighter-rh

# Analyze collected data and get threshold suggestions
python3 tools/analyze.py

# Live trading (requires .env credentials)
python3 main.py --symbol SNDK --hedge lighter-rh

# Chinese dashboard
python3 main.py --symbol SNDK --hedge lighter-rh --cn

# Plain logs (no Rich dashboard)
python3 main.py --symbol SNDK --hedge lighter-rh --no-dashboard
```

Markets (`--symbol` and `--hedge`) are **always** command-line arguments, never config defaults. This is intentional: which markets to trade is an explicit decision every time.

### Testing
```bash
python3 -m pytest tests/              # run all tests
python3 -m pytest tests/test_book.py  # single test file
python3 -m pytest -v                  # verbose output
```

No test configuration files exist; pytest discovers tests by naming convention (`test_*.py`).

## Architecture

### Signal Model
The entire strategy is a fixed-band threshold model around a measured midline:

```
premium_bps = (Entropy_price / hedge_price - 1) × 10_000

SELL entropy + BUY hedge   when premium ≥ midline + upper  (net of fees)
BUY entropy + SELL hedge   when premium ≤ midline - lower  (net of fees)
```

- **`midline_bps`**: The premium's natural center. Cross-venue premiums are rarely zero due to different oracles, quote assets, and listing premia. A wrong midline causes the bot to trade one direction only and lose money.
- **`upper_bps`** / **`lower_bps`**: Entry bands around the midline. Both are applied to **executable prices** (bid vs ask) and are **net of taker fees** on both legs, so one full round trip always nets ≥ `upper + lower` bps after fees.

These three numbers are derived by the user from recorded data (`tools/analyze.py`) and configured in `config.yaml`.

### Core Components

**[main.py](main.py)** — Entry point. Parses arguments (`--symbol`, `--hedge`, `--record-only`, `--cn`, `--no-dashboard`), loads config, sets up logging, and launches the engine with an optional Rich dashboard.

**[entropy_arb/config.py](entropy_arb/config.py)** — Configuration contract: strategy from `config.yaml` (thresholds, sizing, risk), credentials from `.env`, markets from CLI args. Every YAML key is validated against a schema; unknown keys are startup errors. The split is deliberate: `config.yaml` is the strategy and can be committed; `.env` is secrets-only.

**[entropy_arb/engine.py](entropy_arb/engine.py)** — The two-venue arbitrage loop. Evaluates both directions (buy entropy / sell hedge, and vice versa) on every book update, applies persistence gates and inventory ladders, sends concurrent taker orders, settles fills, hedges net-delta imbalances, and reconciles positions against the chain. Failure containment: rate-limited venues pause briefly, unreachable venues trigger outage mode with periodic probing, and consecutive execution failures halt the engine entirely.

**[entropy_arb/book.py](entropy_arb/book.py)** — Order book state and fee-aware crossing math. One `OrderBook` class handles both feed protocols: zkLighter (snapshot + diffs) and Hyperliquid (full snapshots). `plan_arb()` walks both books level-by-level, respecting fees and thresholds, and returns the maximum crossable quantity and expected edge.

**[entropy_arb/feeds.py](entropy_arb/feeds.py)** — WebSocket feed adapters: Hyperliquid's official `l2Book` subscription and zkLighter's diff-based book feed. Each runs as an async task and updates the venue's `OrderBook` on every message. Freshness is connection-based: any inbound frame touches `alive_ts`, so a quiet market is not stale, only a dead feed is.

**[entropy_arb/venue_hl.py](entropy_arb/venue_hl.py)** — Hyperliquid venue adapter (Entropy and trade.xyz). Handles market metadata loading, taker order sending (IOC limits with synchronous settle), position/equity polling, and signing via `hyperliquid-python-sdk`. Two venues on the same account share a nonce sequence (internal state, no coordination overhead).

**[entropy_arb/venue_lighter.py](entropy_arb/venue_lighter.py)** — zkLighter adapter (mainnet and Robinhood chain). Market orders with average-price protection settle asynchronously on the authenticated account WebSocket. Endpoint profiles for both deployments are in `config.py`.

**[entropy_arb/recorder.py](entropy_arb/recorder.py)** — Minute-bar recorder. Samples both live books every second; writes one CSV row per minute with bid/ask, premium open/high/low/close/mean/std, executable edges (pre-fee), and sample count. Runs automatically in all modes when `recorder.enabled: true`.

**[entropy_arb/dashboard.py](entropy_arb/dashboard.py)** — Live Rich terminal dashboard. Displays both books with age/spread, positions and caps, equity and session PnL, executable premium of each direction vs full hurdles (fees + inventory surcharge), recorder progress, last executions, and a scrolling log tail. Works in `--record-only` too. Falls back to plain logs on non-TTY runs. Add `--cn` for Chinese labels.

**[tools/analyze.py](tools/analyze.py)** — Threshold analyzer. Reads `logs/minutes.csv`, prints premium distribution and candidate band firing frequencies, and outputs a ready-to-paste `thresholds:` block. Pass `--hours 24` to restrict to recent data (premiums drift), and `--fees-bps` (sum of both venues' taker fees) to subtract fees before counting firings.

### Execution Flow

1. **Book updates** arrive via WebSocket feeds and set `engine._update_evt`.
2. **Strategy loop** (`_strategy_loop`) evaluates both directions via `_scan()`:
   - Checks both books are fresh, venues are ready, no locks held, rate budgets OK.
   - Plans the arb with `_plan()` (walks books, applies fees + threshold + inventory ladder).
   - Arms the direction if edge is present; fires only if armed and edge persists for `premium_persist_sec`.
3. **Execution** (`_execute`) sends both legs concurrently as taker orders (Lighter market orders, Hyperliquid IOC limits), awaits settlement, updates local positions/cash, logs fills and edge.
4. **Delta hedge** (`_maybe_hedge`) checks net position after each execution; if `abs(net) > net_tolerance_base`, reduces the imbalanced venue with a reduce-only taker order.
5. **Reconcile** (`_reconcile_loop`) polls on-chain positions every `reconcile_sec` (or on demand after unresolved outcomes), adopting chain state when drift exceeds tolerance. A grace period prevents overwriting positions that just traded (Lighter's REST lags its ws settlements).
6. **Failure handling**: Rate-limited venues pause for `rate_limit_pause_sec`; unreachable venues enter outage mode and are probed every `venue_probe_sec`; `max_consecutive_errors` execution failures halt the engine.

### Concurrency & Locking

Each venue has an `asyncio.Lock`. An execution holds **both** locks (prevents concurrent trades on the same venue); a reconcile holds **one** lock (chain read cannot race an in-flight order). The strategy loop checks locks are free before planning; lock acquisition after verification is the fast path (no suspension). Executions run as shielded tasks so shutdown cancels the strategy loop's await, never the in-flight settlement.

### Inventory Management

**Position caps** (`max_position_usd`) are per-venue hard limits. **Inventory ladder** (`inventory_scale_bps`, `inventory_floor_frac`) adds a surcharge once a venue's position passes `floor_frac` of its cap in the adding direction, ramping linearly to `scale_bps` at 100% of the cap. This slows one-sided accumulation without hard-stopping the strategy.

## Configuration

Strategy lives in `config.yaml` (validated), credentials in `.env`, markets on CLI. See [config.example.yaml](config.example.yaml) for full reference. Key sections:

- **`thresholds`**: `midline_bps`, `upper_bps`, `lower_bps` — the entire signal; derived from recorded data via `tools/analyze.py`.
- **`entropy`** / **`hedge`**: `taker_fee_bps`, `max_position_usd`, `max_orders_per_min`.
- **`sizing`**: `take_fraction` (0–1, fraction of crossable depth), `max_order_notional_usd`, `min_order_notional_usd`.
- **`inventory`**: `scale_bps`, `floor_frac` — the inventory ladder.
- **`execution`**: `premium_persist_sec`, `cooldown_sec`, `settle_timeout_sec`, slippage bounds, reconcile cadence, venue probe interval.
- **`recorder`**: `enabled`, `csv` path.
- **`logging`**: `level`, `dashboard`, `file`.

## Credentials (`.env`)

- **Entropy / trade.xyz (Hyperliquid)**: `HL_PRIVATE_KEY` (agent wallet), `HL_ACCOUNT_ADDRESS` (main account). With `--hedge tradexyz`, set `HL_PRIVATE_KEY_XYZ` / `HL_ACCOUNT_ADDRESS_XYZ` to split them (or omit to share the account).
- **Lighter**: `LIGHTER_ACCOUNT_INDEX`, `LIGHTER_API_KEY_INDEX`, `LIGHTER_API_PRIVATE_KEY` — registered on the same deployment as your `--hedge` flag (mainnet and Robinhood chain are separate).

Never commit `.env`. The bot is **live-only**: it either collects data (`--record-only`) or trades real money. There is no paper mode.

## Important Conventions

- **No paper mode exists.** Validate with `--record-only` + tiny position caps, not simulated fills.
- **Markets are always CLI arguments**, never config defaults. This forces an explicit decision on every start.
- **A wrong midline loses money.** The premium center drifts; re-measure regularly and keep `config.yaml` current.
- **All thresholds are net of fees.** The engine adds `taker_fee_bps` on top before firing, so a round trip always nets ≥ `upper + lower` bps after fees.
- **Recorded edges are pre-fee.** When analyzing data, pass `--fees-bps` (sum of both venues' taker fees) to `tools/analyze.py` so its suggestions translate directly to config values.
- **Credentials never live in `config.yaml`**; they go in `.env` (see `.env.example`).
- **Venue locks prevent races.** An execution holds both; a reconcile holds one. Never hold a lock across an `await` that isn't guaranteed to return quickly.

## Testing Notes

Tests are in `tests/` and run with `pytest`. No test config exists; pytest discovers by naming (`test_*.py`). Tests cover:
- `test_book.py` — order book operations and fee-aware crossing math
- `test_config.py` — YAML validation and credential loading
- `test_engine.py` — strategy signal logic
- `test_recorder.py` — minute-bar aggregation
- `test_dashboard.py` — Rich display formatting

When adding features, verify with tests where possible, then validate end-to-end with `--record-only` (data integrity) or live with tiny position caps (execution paths).
