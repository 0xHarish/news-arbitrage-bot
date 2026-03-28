AlphaClaw — Price Anomaly Engine

Version: 0.1
Status: Phase 3 Extension
Purpose: Enable high-frequency mean-reversion scalping on Polymarket markets alongside the slower narrative-driven strategy.

1. Overview

AlphaClaw currently operates as a Narrative Velocity trading agent, identifying high-confidence resolution events before prediction markets fully price them in. 

specs

This system is powerful but naturally low frequency, because resolution events occur rarely.

The Price Anomaly Engine adds a second trading mode:

Narrative trades → slow, high conviction
Anomaly trades → fast scalps, small size

Goal:

~1 trade/hour average

without interfering with the existing pipeline.

2. Architectural Philosophy

Two independent strategies will coexist:

Strategy A — Narrative Resolution

Existing pipeline:

Signal
↓
FastMatch
↓
Opus reasoning
↓
Kelly sizing
↓
Broker
↓
Trade

This system remains unchanged.

Strategy B — Price Anomaly Scalping

New pipeline:

Polymarket WS
↓
Rolling statistics
↓
Anomaly detection
↓
Trade signal
↓
Broker
↓
Trade

Key differences:

Feature	Narrative	Anomaly
Latency	seconds-minutes	milliseconds
Position size	larger	small
Confidence source	news	market microstructure
Frequency	low	high
3. Design Goals

The anomaly system must:

Detect statistically abnormal price moves

Avoid noise and micro-ticks

Trade mean reversion

Use small position sizing

Integrate with the existing Broker

Avoid breaking the Brain architecture

4. Integration with Existing AlphaClaw Architecture

Current architecture (simplified): 

specs

Polymarket WS
↓
Brain
↓
Redis trade_signals
↓
Broker
↓
CLOB execution

The anomaly engine will integrate inside the Brain ingestion layer.

New flow:

Polymarket WS
↓
Rolling stats
↓
Anomaly detector
↓
Redis trade_signals
↓
Broker

FastMatch and Opus are bypassed.

5. Data Source

The anomaly engine relies entirely on Polymarket WebSocket data.

Source:

Polymarket CLOB WebSocket

Events used:

price_change
trade
book

Fields captured:

market_id
token_id
price
timestamp
volume
best_bid
best_ask

Important: the decimal CLOB token_id must be used, not the hex condition_id.

This avoids the /midpoint API error previously encountered.

6. Market Data State

The engine maintains rolling price buffers per market.

Structure:

MarketState
 ├ market_id
 ├ token_id
 ├ prices[]
 ├ timestamps[]
 ├ mean
 ├ std
 ├ last_price

Example:

prices = [0.45,0.46,0.45,0.47,...]
window_size = 60

Window length:

60 ticks
7. Mid-Price Calculation

Prediction markets often have wide spreads, so naive price reads can cause poor fills.

Mid-price is defined as:

mid = (best_bid + best_ask) / 2

If orderbook data unavailable:

mid = last_trade_price

This value is used for:

z-score calculation
entry logic
Kelly input
8. Z-Score Anomaly Detection

Primary anomaly signal:

z = (price - mean) / std

Rolling statistics update on every tick.

Implementation:

Rolling window: 60
Algorithm: Welford online variance
9. Trigger Conditions

An anomaly is triggered when:

|z| ≥ 2.0
AND
|price_delta| ≥ 0.02

Aggressive configuration chosen to increase trade frequency.

Example:

0.45 → 0.51
z = 2.8
trigger = TRUE
10. Mean Reversion Strategy

The anomaly engine trades against the move.

Example:

price spike
0.45 → 0.52

Trade:

SELL YES
or
BUY NO

Expectation:

price reverts toward mean
11. Trade Direction Logic
if z > threshold:
    trade NO

if z < -threshold:
    trade YES
12. Trade Size

Anomaly trades must be small.

Default configuration:

wallet_exposure = 0.5% per trade

Example:

$1000 wallet
trade size = $5

This prevents cascading risk during volatile periods.

13. Execution Pricing

Orders should aim to fill inside the spread.

Recommended:

limit_price = mid ± 0.01

Examples:

YES trade
limit_price = mid + 0.01

NO trade
limit_price = mid - 0.01

This improves fill probability while controlling slippage.

14. Slippage Protection

The Broker already enforces:

MAX_SLIPPAGE_BPS

For anomaly trades we tighten:

MAX_SLIPPAGE_BPS = 80
15. Trade Cooldown

To avoid overtrading the same market:

cooldown_per_market = 10 minutes

No new anomaly trades allowed during cooldown.

16. Liquidity Filters

Markets must pass:

volume_24h ≥ $5k
orderbook_depth ≥ $500
spread ≤ 5%

These parameters prevent trading illiquid markets.

17. Redis Trade Signal Format

Anomaly trades reuse the existing schema with one change:

signal_type: "ANOMALY"

Example message:

{
  "schema_version": "1.0",
  "event": "TRADE_SIGNAL",
  "signal_type": "ANOMALY",
  "market_id": "0xabc123",
  "clob_token_id": "16244226...",
  "side": "NO",
  "position_usdc": 5.00,
  "reason": "zscore_reversion",
  "zscore": 2.8,
  "entry_price": 0.52,
  "timestamp": "2026-02-27T14:12:00Z"
}
18. Broker Changes Required

Broker must support:

signal_type = ANOMALY

Differences vs narrative trades:

Parameter	Narrative	Anomaly
Kelly sizing	yes	fixed
Opus verdict	yes	no
confidence	yes	optional

Broker execution logic remains unchanged.

19. Expected Trade Frequency

With ~30 markets monitored:

3-10 anomalies per hour

After filters:

1-2 trades per hour

Combined with narrative trades:

~1 trade/hour sustained
20. Failure Modes
Liquidity vacuum

Large moves in thin markets.

Mitigation:

spread filter
depth filter
News-driven moves

A real news event causes the anomaly.

Mitigation:

position sizing

Narrative engine will eventually detect the event.

Whale manipulation

Large orders briefly distort price.

Mean reversion usually profits here.

21. Monitoring Metrics

Track:

anomaly_trades_per_day
win_rate
average_reversion
holding_time

Stored in TimescaleDB.

22. Phase Rollout

Phase A:

detect anomalies
log only

Phase B:

shadow trades

Phase C:

live trades
23. Summary

The Price Anomaly Engine introduces a high-frequency scalping layer to AlphaClaw.

Characteristics:

fast
mean-reversion
small positions
high frequency

Combined with narrative trading:

AlphaClaw becomes both
information-driven
and market-microstructure-driven