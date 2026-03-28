# AlphaClaw — Full Technical Specification
**Version:** 0.1.0-draft  
**Status:** Phase 1 Scope  
**Last Updated:** 2026-02-24

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Repository Structure](#2-repository-structure)
3. [Service Architecture](#3-service-architecture)
4. [Data Layer — TimescaleDB Schemas](#4-data-layer--timescaledb-schemas)
5. [Narrative Dictionary — Seed Schema](#5-narrative-dictionary--seed-schema)
6. [Resolution Schema](#6-resolution-schema)
7. [Data Ingestion Pipeline](#7-data-ingestion-pipeline)
8. [Fast-Match Engine (Layer 1)](#8-fast-match-engine-layer-1)
9. [Claude Opus 4.6 Appeals Layer (Layer 2)](#9-claude-opus-46-appeals-layer-layer-2)
10. [Composite Confidence & Kelly Criterion](#10-composite-confidence--kelly-criterion)
11. [Redis Pub/Sub — Inter-Service Contract](#11-redis-pubsub--inter-service-contract)
12. [Node.js Broker — Trade Execution](#12-nodejs-broker--trade-execution)
13. [Circuit Breaker State Machine](#13-circuit-breaker-state-machine)
14. [Neural Trace — WebSocket Protocol](#14-neural-trace--websocket-protocol)
15. [Next.js Frontend — Terminal UI](#15-nextjs-frontend--terminal-ui)
16. [Environment Variables & Secrets](#16-environment-variables--secrets)
17. [Phase Roadmap](#17-phase-roadmap)
18. [Risk Register](#18-risk-register)

---

## 1. Project Overview

AlphaClaw is an autonomous AI trading agent that exploits **Narrative Velocity (NV)** — the delta between real-world information discovery and market-implied probability — in decentralized prediction markets, primarily Polymarket.

**Core Thesis:** Prediction markets are reactive, not anticipatory. High-signal events (court filings, SEC documents, niche technical breakthroughs) take 5–15 minutes to fully price in. AlphaClaw operates in the "Golden Window" — the seconds after a primary source confirms an event but before market liquidity has adjusted.

**Guiding Principles:**
- Safety-first; treasury preservation over opportunity maximization.
- No trade fires without multi-source corroboration.
- Every decision is logged, auditable, and publicly streamed (sanitized).

---

## 2. Repository Structure

```
alphaclaw/
├── services/
│   ├── brain/                   # Python / FastAPI — AI reasoning core
│   │   ├── app/
│   │   │   ├── main.py
│   │   │   ├── ingestion/
│   │   │   │   ├── polymarket_ws.py     # Tier 1: market pulse
│   │   │   │   ├── social_poller.py     # Tier 2: SocialData.tools
│   │   │   │   ├── hn_poller.py         # Tier 3: Hacker News API
│   │   │   │   └── perplexity_client.py # Tier 4: fact verification
│   │   │   ├── fastmatch/
│   │   │   │   ├── engine.py            # Layer 1 deterministic engine
│   │   │   │   ├── normalizer.py        # Text → canonical tokens
│   │   │   │   └── zscore.py            # Z-score anomaly detection
│   │   │   ├── appeals/
│   │   │   │   ├── opus_client.py       # Claude Opus 4.6 wrapper
│   │   │   │   └── prompt_builder.py    # Legalistic prompt assembly
│   │   │   ├── sizing/
│   │   │   │   └── kelly.py             # Composite p + Kelly formula
│   │   │   ├── publisher/
│   │   │   │   └── redis_pub.py         # Redis pub/sub publisher
│   │   │   └── schemas/
│   │   │       ├── narrative_dict.json  # Seed Narrative Dictionary
│   │   │       └── resolution_schema/   # Per-market resolution JSONs
│   │   ├── tests/
│   │   └── requirements.txt
│   │
│   ├── broker/                  # Node.js / Bun — trade execution
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── redis_sub.ts             # Redis subscriber
│   │   │   ├── polymarket/
│   │   │   │   ├── clob_client.ts       # Polymarket CLOB API wrapper
│   │   │   │   ├── order_builder.ts     # TWAP / nibble execution
│   │   │   │   └── wallet.ts            # Hot wallet management
│   │   │   ├── circuit_breaker/
│   │   │   │   └── state_machine.ts     # Daily PnL circuit breaker
│   │   │   └── publisher/
│   │   │       └── neural_trace_pub.ts  # WS event emitter for terminal
│   │   ├── tests/
│   │   └── package.json
│   │
│   └── terminal/                # Next.js — public-facing dashboard
│       ├── app/
│       │   ├── page.tsx                 # Root — Neural Trace stream
│       │   ├── components/
│       │   │   ├── NeuralTrace.tsx      # Live feed component
│       │   │   ├── PnLDisplay.tsx       # Wallet PnL
│       │   │   └── TradeCard.tsx        # Individual trade event
│       │   └── api/
│       │       └── ws/route.ts          # WebSocket proxy
│       └── package.json
│
├── infra/
│   ├── docker-compose.yml
│   ├── timescaledb/
│   │   └── init.sql                     # Schema bootstrap
│   └── redis/
│       └── redis.conf
│
├── data/
│   ├── narrative_dict.json              # Canonical seed file
│   └── resolution_schemas/             # Per-market JSON files
│
└── docs/
    └── specs.md                         # This file
```

---

## 3. Service Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        DATA SOURCES                             │
│  Polymarket WS  │  SocialData.tools  │  HN API  │  Perplexity  │
└────────┬────────┴─────────┬──────────┴────┬─────┴──────┬───────┘
         │                  │               │            │
         ▼                  ▼               ▼            ▼
┌─────────────────────────────────────────────────────────────────┐
│              BRAIN SERVICE  (Python / FastAPI)                  │
│                                                                 │
│  ┌──────────────┐    ┌───────────────┐    ┌──────────────────┐ │
│  │  Fast-Match  │───▶│  Opus Appeals │───▶│  Kelly Sizer     │ │
│  │  Engine L1   │    │  Layer L2     │    │  (composite p)   │ │
│  └──────────────┘    └───────────────┘    └────────┬─────────┘ │
│                                                    │           │
│  ┌──────────────────────────────────────────────── │──────┐    │
│  │              TimescaleDB                        │      │    │
│  │  signals / trades / pnl / oracle_disputes       ▼      │    │
│  └──────────────────────────────────────────────────────┘ │    │
└───────────────────────────────────────────┬───────────────┘    
                                            │ Redis pub/sub
                                            │ channel: trade_signals
                                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              BROKER SERVICE  (Node.js / Bun)                    │
│                                                                 │
│  ┌──────────────┐    ┌───────────────┐    ┌──────────────────┐ │
│  │  Redis Sub   │───▶│  Circuit      │───▶│  CLOB Executor   │ │
│  │              │    │  Breaker FSM  │    │  (TWAP/nibble)   │ │
│  └──────────────┘    └───────────────┘    └────────┬─────────┘ │
│                                                    │           │
│                         Redis pub/sub              │           │
│                         channel: neural_trace_raw  ▼           │
└─────────────────────────────────────────────────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              TERMINAL  (Next.js)                                │
│                                                                 │
│  Public Feed (sanitized)  │  Private Feed (raw, auth-gated)    │
│  alphaclaw.ai             │  alphaclaw.ai/admin                 │
└─────────────────────────────────────────────────────────────────┘
```

**Service boundaries:**
- Brain → Broker communication is **exclusively via Redis pub/sub**. No direct HTTP calls between services.
- Both services write audit records to **TimescaleDB** independently.
- The terminal reads from Redis pub/sub `neural_trace_public` and `neural_trace_raw` channels and exposes them over WebSocket to browser clients.

---

## 4. Data Layer — TimescaleDB Schemas

All tables use `TIMESTAMPTZ` for time columns. Hypertables are partitioned by `created_at` with a 7-day chunk interval unless otherwise noted.

### 4.1 `signals`

Stores every inbound signal before any trade decision.

```sql
CREATE TABLE signals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    source_tier     SMALLINT NOT NULL,         -- 1=polymarket, 2=social, 3=hn, 4=perplexity
    source_handle   TEXT,                      -- e.g. twitter handle or HN item id
    raw_text        TEXT NOT NULL,
    canonical_tokens JSONB,                    -- output of normalizer.py
    market_id       TEXT,                      -- polymarket condition_id if matched
    zscore          DOUBLE PRECISION,
    fast_match_hit  BOOLEAN DEFAULT FALSE,
    fast_match_score DOUBLE PRECISION,
    escalated_to_opus BOOLEAN DEFAULT FALSE
);

SELECT create_hypertable('signals', 'created_at');
CREATE INDEX ON signals (market_id, created_at DESC);
CREATE INDEX ON signals (source_tier, created_at DESC);
```

### 4.2 `opus_verdicts`

Stores every Claude Opus 4.6 invocation and its structured output.

```sql
CREATE TABLE opus_verdicts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    signal_id       UUID REFERENCES signals(id),
    market_id       TEXT NOT NULL,
    prompt_tokens   INT,
    completion_tokens INT,
    raw_response    TEXT,                      -- full JSON response from Opus
    resolution_match BOOLEAN,
    opus_confidence DOUBLE PRECISION,          -- 0.0–1.0 returned by Opus
    reasoning_summary TEXT,                    -- sanitized for public feed
    latency_ms      INT
);

SELECT create_hypertable('opus_verdicts', 'created_at');
CREATE INDEX ON opus_verdicts (market_id, created_at DESC);
```

### 4.3 `trade_signals`

Outbound signals from Brain → Broker via Redis. Also persisted for audit.

```sql
CREATE TABLE trade_signals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    signal_id       UUID REFERENCES signals(id),
    verdict_id      UUID REFERENCES opus_verdicts(id),
    market_id       TEXT NOT NULL,
    side            TEXT NOT NULL CHECK (side IN ('YES', 'NO')),
    composite_p     DOUBLE PRECISION NOT NULL,  -- final Kelly input
    kelly_fraction  DOUBLE PRECISION NOT NULL,
    position_usdc   DOUBLE PRECISION NOT NULL,  -- dollar amount after cap
    wallet_exposure DOUBLE PRECISION NOT NULL,  -- fraction of total wallet
    status          TEXT NOT NULL DEFAULT 'PENDING'
                    CHECK (status IN ('PENDING','EXECUTED','REJECTED','EXPIRED'))
);

SELECT create_hypertable('trade_signals', 'created_at');
```

### 4.4 `trades`

Executed trades as confirmed by the CLOB API.

```sql
CREATE TABLE trades (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    trade_signal_id UUID REFERENCES trade_signals(id),
    market_id       TEXT NOT NULL,
    order_id        TEXT NOT NULL,             -- Polymarket order ID
    side            TEXT NOT NULL,
    size_usdc       DOUBLE PRECISION NOT NULL,
    avg_fill_price  DOUBLE PRECISION,
    slippage_bps    INT,
    fee_usdc        DOUBLE PRECISION,
    status          TEXT NOT NULL
                    CHECK (status IN ('FILLED','PARTIAL','FAILED','CANCELLED')),
    raw_clob_response JSONB
);

SELECT create_hypertable('trades', 'created_at');
CREATE INDEX ON trades (market_id, created_at DESC);
```

### 4.5 `pnl_snapshots`

Wallet state snapshots for circuit breaker evaluation and terminal display.

```sql
CREATE TABLE pnl_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    wallet_address  TEXT NOT NULL,
    total_usdc      DOUBLE PRECISION NOT NULL,
    unrealized_pnl  DOUBLE PRECISION,
    realized_pnl_today DOUBLE PRECISION,
    open_positions  JSONB,
    snapshot_source TEXT                       -- 'scheduled' | 'post_trade'
);

SELECT create_hypertable('pnl_snapshots', 'created_at', chunk_time_interval => INTERVAL '1 day');
```

### 4.6 `oracle_disputes`

Tracks active UMA oracle disputes. AlphaClaw will not trade markets with open disputes.

```sql
CREATE TABLE oracle_disputes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    market_id       TEXT NOT NULL,
    dispute_tx      TEXT,
    dispute_status  TEXT NOT NULL CHECK (dispute_status IN ('ACTIVE','RESOLVED','INVALID')),
    resolved_at     TIMESTAMPTZ,
    notes           TEXT
);

CREATE UNIQUE INDEX ON oracle_disputes (market_id) WHERE dispute_status = 'ACTIVE';
```

### 4.7 `circuit_breaker_events`

Audit log for all circuit breaker state transitions.

```sql
CREATE TABLE circuit_breaker_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    from_state      TEXT NOT NULL,
    to_state        TEXT NOT NULL,
    trigger_reason  TEXT NOT NULL,
    daily_pnl_pct   DOUBLE PRECISION,
    initiated_by    TEXT                       -- 'system' | 'manual'
);
```

---

## 5. Narrative Dictionary — Seed Schema

**File:** `data/narrative_dict.json`  
**Purpose:** Maps normalized canonical tokens to market categories and expected resolution signals. This is the lookup table that powers the Fast-Match engine.

### 5.1 Top-Level Structure

```json
{
  "version": "1.0.0",
  "last_updated": "2026-02-24",
  "categories": {
    "<category_id>": { /* CategoryEntry */ }
  }
}
```

### 5.2 CategoryEntry Schema

```json
{
  "id": "us_election_2026",
  "label": "US Midterm Elections 2026",
  "description": "Congressional and Senate race outcomes",
  "market_tags": ["election", "congress", "senate", "midterm"],
  "resolution_signals": [
    {
      "signal_id": "rs_ap_call",
      "label": "AP Race Call",
      "weight": 0.95,
      "source_domains": ["apnews.com", "ap.org"],
      "canonical_tokens": ["ap", "calls", "race", "winner"],
      "regex_patterns": ["AP\\s+calls\\s+[A-Z][a-z]+\\s+(race|district|seat)"],
      "minimum_sources": 1
    },
    {
      "signal_id": "rs_nyt_call",
      "label": "NYT Race Call",
      "weight": 0.90,
      "source_domains": ["nytimes.com"],
      "canonical_tokens": ["nyt", "projects", "winner"],
      "regex_patterns": ["The New York Times projects"],
      "minimum_sources": 1
    }
  ],
  "adversarial_patterns": [
    "leading", "ahead", "projected by", "sources say"
  ],
  "minimum_independent_sources": 2,
  "golden_window_seconds": 900
}
```

### 5.3 Field Definitions

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique category identifier |
| `label` | string | Human-readable name |
| `market_tags` | string[] | Keywords used to match Polymarket market titles |
| `resolution_signals[].weight` | float 0–1 | How much this source contributes to `fast_match_score` |
| `resolution_signals[].canonical_tokens` | string[] | Normalized tokens that must ALL be present |
| `resolution_signals[].regex_patterns` | string[] | Any ONE must match (OR logic) |
| `adversarial_patterns` | string[] | Token patterns that DISQUALIFY a signal hit (e.g. "leading" ≠ "won") |
| `minimum_independent_sources` | int | Number of distinct `source_domains` needed to pass to Layer 2 |
| `golden_window_seconds` | int | Max age of event for it to be in-window |

### 5.4 Seed Categories (Phase 1)

The following categories must be hand-curated before Phase 1 goes live:

- `us_election_*` — Federal and major state races
- `crypto_regulatory` — SEC/CFTC enforcement actions, ETF approvals
- `macro_fed` — FOMC rate decisions, CPI releases
- `geopolitical_ceasefire` — War/conflict resolution events
- `tech_ipo` — Major IPO filings and pricing dates
- `sports_championship` — Major finals outcomes (NBA, NFL, World Cup)

---

## 6. Resolution Schema

**Directory:** `data/resolution_schemas/<market_id>.json`  
**Purpose:** Per-market legalistic rules file. Consumed by both Fast-Match (for condition matching) and Opus (as system context).

### 6.1 Schema

```json
{
  "market_id": "0xabc123...",
  "condition_id": "0xabc123...",
  "title": "Will the Fed raise rates at the March 2026 FOMC meeting?",
  "category_id": "macro_fed",
  "resolution_source": "Federal Reserve official press release at federalreserve.gov",
  "resolution_criteria": "This market resolves YES if the Federal Open Market Committee announces an increase to the federal funds rate target range at the conclusion of the March 18-19, 2026 meeting. It resolves NO if rates are held or cut.",
  "key_entities": [
    { "name": "Federal Reserve", "aliases": ["Fed", "FOMC", "Federal Open Market Committee"] },
    { "name": "Federal Funds Rate", "aliases": ["fed funds rate", "target rate", "interest rate"] }
  ],
  "threshold_conditions": [
    {
      "condition_id": "tc_yes",
      "outcome": "YES",
      "required_tokens": ["FOMC", "raises", "increases", "rate"],
      "forbidden_tokens": ["holds", "cuts", "reduces", "pauses", "unchanged"],
      "authoritative_sources": ["federalreserve.gov", "wsj.com", "reuters.com", "bloomberg.com"],
      "minimum_sources": 2
    },
    {
      "condition_id": "tc_no",
      "outcome": "NO",
      "required_tokens": ["FOMC", "holds", "pauses", "unchanged"],
      "forbidden_tokens": ["raises", "increases", "hikes"],
      "authoritative_sources": ["federalreserve.gov", "wsj.com", "reuters.com", "bloomberg.com"],
      "minimum_sources": 2
    }
  ],
  "oracle_contract": "0xUMA_contract_address",
  "expiry_timestamp": 1742500000,
  "created_at": "2026-02-20T00:00:00Z"
}
```

---

## 7. Data Ingestion Pipeline

### 7.1 Tier 1 — Polymarket WebSocket (Market Pulse)

**File:** `services/brain/app/ingestion/polymarket_ws.py`

- Connects to Polymarket's public WebSocket feed.
- Subscribes to `book`, `price_change`, and `trade` channels for all tracked markets (market list sourced from `resolution_schemas/` directory).
- On each price tick, computes a rolling **Z-score** over a configurable window (default: 50 ticks, configurable via env).
- Emits a `Tier1Signal` if `|z_score| >= Z_SCORE_THRESHOLD` (default: 2.5).

```python
# Tier1Signal dataclass
@dataclass
class Tier1Signal:
    market_id: str
    current_price: float      # 0.0–1.0 (probability)
    price_delta: float        # change from rolling mean
    zscore: float
    volume_24h: float
    timestamp: datetime
```

### 7.2 Tier 2 — SocialData.tools Poller (Social Intel)

**File:** `services/brain/app/ingestion/social_poller.py`

- Polls SocialData.tools REST API every `SOCIAL_POLL_INTERVAL_SECONDS` (default: 15s).
- Filters against a **High-Signal Whitelist** (`data/whitelist.json`) of 200+ accounts.
- Whitelist entry format: `{ "handle": "@federalreserve", "category_ids": ["macro_fed"], "trust_weight": 0.95 }`.
- Applies normalizer to tweet text before emitting `Tier2Signal`.

```python
@dataclass
class Tier2Signal:
    source_handle: str
    source_domain: str
    raw_text: str
    canonical_tokens: list[str]
    trust_weight: float
    url: str
    timestamp: datetime
```

### 7.3 Tier 3 — Hacker News API (Structural Intel)

**File:** `services/brain/app/ingestion/hn_poller.py`

- Polls HN `/v0/topstories` every `HN_POLL_INTERVAL_SECONDS` (default: 60s).
- Fetches metadata for new stories only (tracks seen IDs in Redis SET `hn:seen`).
- Filters by keyword intersection with active `market_tags` from the Narrative Dictionary.
- Emits `Tier3Signal` only if a story matches at least one active category.

### 7.4 Tier 4 — Perplexity (Fact Verification)

**File:** `services/brain/app/ingestion/perplexity_client.py`

- Invoked **on-demand** (not on a timer). Called by the Fast-Match engine when a Tier 2/3 signal scores above `FAST_MATCH_ESCALATION_THRESHOLD`.
- Uses Perplexity's `sonar-pro` model with `search_recency_filter: "hour"`.
- Returns a `PerplexityResult` containing: `summary`, `citations: list[{url, domain, snippet}]`, `confidence`.
- Adds corroborating source domains to the signal's `source_domains` set.

---

## 8. Fast-Match Engine (Layer 1)

**File:** `services/brain/app/fastmatch/engine.py`  
**Target latency:** <100ms  
**Cost:** Near-zero (no LLM calls)

### 8.1 Processing Pipeline

```
Inbound Signal (Tier 1/2/3)
         │
         ▼
[1] Normalizer → canonical_tokens[]
         │
         ▼
[2] Oracle Dispute Check → query TimescaleDB oracle_disputes
    If ACTIVE dispute for market_id → DROP signal
         │
         ▼
[3] Adversarial Filter → check against category.adversarial_patterns
    If match → DROP signal (flag as adversarial bait)
         │
         ▼
[4] Token Matcher → for each resolution_signal in matched category:
    - Verify ALL canonical_tokens present
    - Verify at least ONE regex_pattern matches
    - Accumulate weighted score
         │
         ▼
[5] Source Independence Check
    - Count distinct source_domains in signal cluster
    - Must meet category.minimum_independent_sources
         │
         ▼
[6] Z-Score Gate (Tier 1 signals only)
    - Must have |zscore| >= Z_SCORE_THRESHOLD
         │
         ▼
[7] Golden Window Check
    - Signal timestamp within category.golden_window_seconds of event time
         │
         ▼
[8] Score Aggregation → fast_match_score (0.0–1.0)
         │
    ┌────┴─────┐
    │          │
score < 0.6  score >= 0.6
    │          │
   DROP     Trigger Perplexity (Tier 4)
              + Escalate to Opus (Layer 2)
```

### 8.2 Text Normalizer

**File:** `services/brain/app/fastmatch/normalizer.py`

```python
def normalize(text: str) -> list[str]:
    """
    1. Lowercase
    2. Strip URLs, mentions, hashtags
    3. Remove punctuation
    4. Tokenize on whitespace
    5. Remove stopwords (NLTK English + financial stopwords)
    6. Apply lemmatization (spaCy en_core_web_sm)
    7. Return deduplicated token list
    """
```

### 8.3 Z-Score Calculator

**File:** `services/brain/app/fastmatch/zscore.py`

```python
# Rolling Z-score with configurable window
# Uses Welford's online algorithm for numerical stability

class RollingZScore:
    def __init__(self, window: int = 50): ...
    def update(self, value: float) -> float: ...  # returns z_score
    def reset(self): ...
```

---

## 9. Claude Opus 4.6 Appeals Layer (Layer 2)

**File:** `services/brain/app/appeals/opus_client.py`  
**Model:** `claude-opus-4-6`  
**Role:** Legalistic nuance check against market resolution criteria.

### 9.1 Invocation Conditions

Opus is called ONLY when Fast-Match score >= 0.6. It is NOT called for:
- Signals that failed the adversarial filter
- Markets with active oracle disputes
- Signals outside the golden window

### 9.2 Prompt Structure

```python
SYSTEM_PROMPT = """
You are a legal analyst specializing in prediction market contract interpretation.
Your task is to determine whether a given news event satisfies the exact resolution 
criteria of a specific prediction market contract.

You must respond with a JSON object ONLY. No prose. No explanation outside the JSON.

Response schema:
{
  "resolution_match": boolean,
  "outcome": "YES" | "NO" | "AMBIGUOUS",
  "confidence": float (0.0–1.0),
  "reasoning": string (1–3 sentences, suitable for public display),
  "disqualifying_factors": string[] | null,
  "requires_perplexity_followup": boolean
}

Rules:
- confidence > 0.80 required to recommend a trade
- If the event is a PRELIMINARY report, not a FINAL ruling, confidence must be <= 0.65
- If ANY forbidden_token from the resolution schema is present in the news, outcome = AMBIGUOUS
- Cite the specific resolution_criteria language in your reasoning
"""

def build_user_prompt(signal: Signal, resolution_schema: dict, perplexity_result: dict) -> str:
    return f"""
MARKET RESOLUTION CRITERIA:
{json.dumps(resolution_schema['resolution_criteria'])}

KEY ENTITIES:
{json.dumps(resolution_schema['key_entities'])}

THRESHOLD CONDITIONS:
{json.dumps(resolution_schema['threshold_conditions'])}

NEWS SIGNAL:
Source: {signal.source_handle} ({signal.source_domain})
Text: {signal.raw_text}

CORROBORATING SOURCES (Perplexity):
{json.dumps(perplexity_result['citations'])}

Does this news event satisfy the resolution criteria? Respond with the JSON schema only.
"""
```

### 9.3 Response Handling

```python
@dataclass
class OpusVerdict:
    resolution_match: bool
    outcome: str              # "YES" | "NO" | "AMBIGUOUS"
    confidence: float         # 0.0–1.0
    reasoning: str
    disqualifying_factors: list[str] | None
    requires_perplexity_followup: bool
    latency_ms: int
    prompt_tokens: int
    completion_tokens: int
```

- If `requires_perplexity_followup` is True, a second Perplexity call is made and Opus is re-invoked with the additional context (max 1 re-invocation).
- If `outcome == "AMBIGUOUS"` → signal is dropped, no trade.
- If `confidence < 0.80` → signal is dropped.

---

## 10. Composite Confidence & Kelly Criterion

**File:** `services/brain/app/sizing/kelly.py`

### 10.1 Composite Probability `p`

```
composite_p = (opus_confidence × W_OPUS) + (normalized_zscore × W_ZSCORE)
```

Where:
- `W_OPUS = 0.75` (primary weight — legalistic reasoning)
- `W_ZSCORE = 0.25` (secondary weight — market anomaly signal)
- `normalized_zscore = min(abs(zscore) / Z_SCORE_MAX, 1.0)` where `Z_SCORE_MAX = 5.0`

These weights are configurable via environment variables and should be re-calibrated during Phase 2 Shadow Mode.

### 10.2 Kelly Criterion

```
kelly_fraction = (composite_p × b - (1 - composite_p)) / b
```

Where `b` is the net odds of the bet:
```
b = (1 / market_price) - 1
```

For example, if market price is 0.30 (30% implied probability):
```
b = (1 / 0.30) - 1 = 2.333
```

### 10.3 Position Sizing Caps

```python
def calculate_position(
    composite_p: float,
    market_price: float,
    wallet_total_usdc: float
) -> float:
    b = (1 / market_price) - 1
    kelly_f = (composite_p * b - (1 - composite_p)) / b
    kelly_f = max(0.0, kelly_f)               # Never negative
    fractional_kelly = kelly_f * KELLY_FRACTION  # KELLY_FRACTION = 0.25 (quarter Kelly)
    raw_position = fractional_kelly * wallet_total_usdc
    capped_position = min(raw_position, wallet_total_usdc * MAX_WALLET_EXPOSURE)
    return round(capped_position, 2)           # USDC, 2 decimal places
```

**Constants:**
| Constant | Value | Description |
|---|---|---|
| `KELLY_FRACTION` | 0.25 | Quarter Kelly to reduce variance |
| `MAX_WALLET_EXPOSURE` | 0.05 | 5% hard cap per trade |
| `MIN_POSITION_USDC` | 5.00 | Minimum position to avoid dust trades |
| `MIN_COMPOSITE_P` | 0.80 | Minimum confidence to size any position |

---

## 11. Redis Pub/Sub — Inter-Service Contract

### 11.1 Channels

| Channel | Publisher | Subscriber | Description |
|---|---|---|---|
| `trade_signals` | Brain | Broker | Trade execution instructions |
| `neural_trace_raw` | Brain + Broker | Terminal | Full internal event stream |
| `neural_trace_public` | Brain + Broker | Terminal | Sanitized public event stream |
| `circuit_breaker_state` | Broker | Brain | Circuit breaker state changes |
| `pnl_updates` | Broker | Brain + Terminal | Wallet PnL snapshots |

### 11.2 `trade_signals` Message Schema

Published by Brain when a signal passes all gates. Consumed by Broker for execution.

```json
{
  "schema_version": "1.0",
  "event": "TRADE_SIGNAL",
  "signal_id": "uuid-v4",
  "verdict_id": "uuid-v4",
  "market_id": "0xabc123...",
  "side": "YES",
  "composite_p": 0.87,
  "kelly_fraction": 0.043,
  "position_usdc": 43.00,
  "wallet_exposure": 0.043,
  "market_price_at_signal": 0.31,
  "reasoning_summary": "AP called the race at 23:14 UTC. Contract resolves on AP call.",
  "primary_source_url": "https://apnews.com/...",
  "expires_at": "2026-02-24T12:30:00Z",
  "timestamp": "2026-02-24T12:15:00Z"
}
```

### 11.3 `neural_trace_raw` Message Schema

Full internal reasoning trace. Includes everything.

```json
{
  "schema_version": "1.0",
  "event_type": "SIGNAL_EVALUATED" | "TRADE_EXECUTED" | "SIGNAL_DROPPED" | "CIRCUIT_BREAKER",
  "event_id": "uuid-v4",
  "timestamp": "2026-02-24T12:15:00Z",
  "payload": {
    "signal_id": "uuid-v4",
    "source_tier": 2,
    "source_handle": "@federalreserve",
    "raw_text": "...",
    "canonical_tokens": ["..."],
    "fast_match_score": 0.82,
    "zscore": 3.1,
    "opus_confidence": 0.91,
    "opus_reasoning": "...",
    "composite_p": 0.89,
    "position_usdc": 43.00,
    "drop_reason": null
  }
}
```

### 11.4 `neural_trace_public` Message Schema

Sanitized version. Omits raw source text, raw Opus reasoning, and internal scores.

```json
{
  "schema_version": "1.0",
  "event_type": "HUNT" | "STRIKE" | "PASS" | "SHIELD",
  "event_id": "uuid-v4",
  "timestamp": "2026-02-24T12:15:00Z",
  "payload": {
    "market_title": "Will the Fed raise rates in March 2026?",
    "action": "BOUGHT YES",
    "edge_pct": 18.2,
    "position_usdc": 43.00,
    "reasoning_summary": "Fed statement confirmed rate hold per federalreserve.gov.",
    "source_url": "https://federalreserve.gov/...",
    "confidence_label": "HIGH"
  }
}
```

**Event type mapping:**
- `HUNT` — Signal detected and being evaluated
- `STRIKE` — Trade executed
- `PASS` — Signal evaluated, no trade (ambiguous or insufficient confidence)
- `SHIELD` — Signal dropped by circuit breaker or oracle dispute

---

## 12. Node.js Broker — Trade Execution

**File:** `services/broker/src/`  
**Runtime:** Bun

### 12.1 Redis Subscriber

`redis_sub.ts` subscribes to `trade_signals` and `circuit_breaker_state`. On each `trade_signals` message:
1. Validate JSON against schema
2. Check circuit breaker state — if `HIBERNATION`, reject and publish `SHIELD` to `neural_trace_raw`
3. Check signal expiry (`expires_at` field)
4. Route to `ClobExecutor`

### 12.2 Polymarket CLOB Client

**File:** `services/broker/src/polymarket/clob_client.ts`

Wraps the [Polymarket CLOB REST API](https://docs.polymarket.com/clob).

```typescript
interface ClobClient {
  getMarket(conditionId: string): Promise<Market>
  getOrderBook(tokenId: string): Promise<OrderBook>
  createOrder(params: OrderParams): Promise<OrderResult>
  cancelOrder(orderId: string): Promise<void>
  getOrder(orderId: string): Promise<Order>
}
```

**Authentication:** EIP-712 signed orders using the `alphaclaw.eth` hot wallet private key (loaded from env var `HOT_WALLET_PRIVATE_KEY`).

### 12.3 TWAP / Nibble Execution

**File:** `services/broker/src/polymarket/order_builder.ts`

To mitigate slippage in thin markets, positions are split into `TWAP_SLICES` (default: 3) executed over `TWAP_WINDOW_SECONDS` (default: 90s).

```typescript
interface NibbleExecution {
  totalUsdc: number
  slices: number           // from TWAP_SLICES env
  intervalMs: number       // TWAP_WINDOW_SECONDS / slices * 1000
  maxSlippage: number      // MAX_SLIPPAGE_BPS (default: 150 bps = 1.5%)
}
```

If any nibble exceeds `MAX_SLIPPAGE_BPS`, the remaining slices are cancelled and a partial fill is recorded.

### 12.4 Post-Trade Flow

After each fill:
1. Write to TimescaleDB `trades` table
2. Fetch updated wallet balance from CLOB API
3. Write to TimescaleDB `pnl_snapshots`
4. Publish to Redis `pnl_updates`
5. Publish execution event to `neural_trace_raw` and `neural_trace_public`
6. Evaluate circuit breaker conditions

---

## 13. Circuit Breaker State Machine

**File:** `services/broker/src/circuit_breaker/state_machine.ts`

### 13.1 States

```
ACTIVE ──────────────────────────────────────────────────────┐
  │                                                           │
  │ daily_pnl <= -15%                                         │
  ▼                                                           │
HIBERNATION ── manual_reset (admin only) ──────────────────▶ │
                                                             │
ACTIVE ◀── auto_reset at 00:00 UTC (new trading day) ────────┘
```

**HIBERNATION actions triggered immediately:**
1. Revoke/disable `HOT_WALLET_PRIVATE_KEY` in-memory (does not modify env; prevents new orders)
2. Cancel all open orders via CLOB API
3. Publish `SHIELD` event with reason `DAILY_LOSS_LIMIT` to both neural trace channels
4. Write to `circuit_breaker_events` table
5. Publish to `circuit_breaker_state` Redis channel

### 13.2 Additional Guards

| Guard | Condition | Action |
|---|---|---|
| Oracle Dispute | Active dispute in `oracle_disputes` table | Drop signal, publish SHIELD |
| Signal Expiry | `expires_at` in past | Drop signal silently |
| Thin Market | Order book depth < `MIN_BOOK_DEPTH_USDC` (default: $500) | Drop signal, log |
| Slippage Exceeded | Fill slippage > `MAX_SLIPPAGE_BPS` | Cancel remaining nibbles |
| Single Source | Source independence check fails | Drop at Fast-Match Layer 1 |

### 13.3 Daily Reset

At `00:00 UTC`, a scheduled job (cron in Brain service):
1. Resets `realized_pnl_today` counter
2. If in `HIBERNATION` due to daily loss: transitions back to `ACTIVE`, re-enables wallet
3. Writes reset event to `circuit_breaker_events`

---

## 14. Neural Trace — WebSocket Protocol

### 14.1 Architecture

```
Brain/Broker → Redis (neural_trace_raw, neural_trace_public)
                    │
                    ▼
         Next.js API Route (/api/ws)
         └── WebSocket upgrade
                    │
          ┌─────────┴──────────┐
          ▼                    ▼
   Public clients        Admin clients
   (sanitized feed)     (raw feed, JWT-gated)
```

### 14.2 Client Connection

Clients connect to `wss://alphaclaw.ai/api/ws`.

**Query params:**
- `feed=public` — Default. Receives `neural_trace_public` events.
- `feed=raw&token=<JWT>` — Admin only. Receives `neural_trace_raw` events. JWT validated against `ADMIN_JWT_SECRET`.

### 14.3 Server → Client Messages

All messages are JSON over WebSocket text frames.

```json
{
  "type": "EVENT" | "PING" | "SNAPSHOT",
  "data": { /* neural_trace_public or neural_trace_raw payload */ }
}
```

- `PING` sent every 30s; client must respond with `PONG` or connection is closed.
- `SNAPSHOT` sent on connect: last 20 events from Redis `LRANGE neural_trace_public:history 0 19`.

### 14.4 History Buffer

Brain and Broker append to Redis lists:
- `neural_trace_public:history` — LPUSH, LTRIM to 100 entries
- `neural_trace_raw:history` — LPUSH, LTRIM to 500 entries

---

## 15. Next.js Frontend — Terminal UI

**Framework:** Next.js 15 (App Router)  
**Styling:** Tailwind CSS  
**Theme:** Dark terminal aesthetic (`#0a0a0a` background, green/amber monospace text)

### 15.1 Pages

| Route | Description |
|---|---|
| `/` | Public Neural Trace terminal + live PnL |
| `/admin` | Raw feed + full trade log (JWT-gated) |
| `/wallet` | Live `alphaclaw.eth` wallet view via Polygonscan embed |

### 15.2 Key Components

**`NeuralTrace.tsx`**
- Opens WebSocket connection to `/api/ws?feed=public`
- Auto-scrolls feed with "pause on hover" behavior
- Color codes by event type: HUNT (yellow), STRIKE (green), PASS (gray), SHIELD (red)
- Renders `edge_pct`, `confidence_label`, and `source_url` per event

**`PnLDisplay.tsx`**
- Subscribes to `pnl_updates` Redis channel via SSE endpoint `/api/pnl`
- Displays: total wallet USDC, today's realized PnL %, open positions count
- Color: green if PnL > 0, red if < 0, amber within 5% of -15% limit

**`TradeCard.tsx`**
- Expandable card for each STRIKE event
- Shows: market title, side (YES/NO), position size, entry price, edge %, source link, reasoning summary

### 15.3 X (Twitter) Auto-Post

Post template triggered on every `STRIKE` event, published by the Broker:
```
🦅 ALPHACLAW STRIKE

Market: {market_title}
Side: {side} @ {entry_price}
Edge: +{edge_pct}%
Position: ${position_usdc}

Signal: {primary_source_url}

Live P&L: alphaclaw.ai | alphaclaw.eth
```
Integration: Twitter API v2 via `POST /2/tweets`. Credentials in env.

---

## 16. Environment Variables & Secrets

### Brain Service (`services/brain/.env`)

```bash
# Redis
REDIS_URL=redis://localhost:6379

# TimescaleDB
TIMESCALE_URL=postgresql://alphaclaw:password@localhost:5432/alphaclaw

# Claude
ANTHROPIC_API_KEY=sk-ant-...
CLAUDE_MODEL=claude-opus-4-6

# Perplexity
PERPLEXITY_API_KEY=pplx-...
PERPLEXITY_MODEL=sonar-pro

# SocialData.tools
SOCIALDATA_API_KEY=...

# Tuning
Z_SCORE_THRESHOLD=2.5
Z_SCORE_WINDOW=50
Z_SCORE_MAX=5.0
FAST_MATCH_ESCALATION_THRESHOLD=0.60
KELLY_FRACTION=0.25
MAX_WALLET_EXPOSURE=0.05
MIN_POSITION_USDC=5.00
MIN_COMPOSITE_P=0.80
W_OPUS=0.75
W_ZSCORE=0.25
SOCIAL_POLL_INTERVAL_SECONDS=15
HN_POLL_INTERVAL_SECONDS=60
```

### Broker Service (`services/broker/.env`)

```bash
# Redis
REDIS_URL=redis://localhost:6379

# TimescaleDB
TIMESCALE_URL=postgresql://alphaclaw:password@localhost:5432/alphaclaw

# Polymarket
POLYMARKET_CLOB_BASE_URL=https://clob.polymarket.com
HOT_WALLET_PRIVATE_KEY=0x...
HOT_WALLET_ADDRESS=0x...

# Execution
TWAP_SLICES=3
TWAP_WINDOW_SECONDS=90
MAX_SLIPPAGE_BPS=150
MIN_BOOK_DEPTH_USDC=500

# Circuit Breaker
DAILY_LOSS_LIMIT_PCT=15.0

# Twitter
TWITTER_API_KEY=...
TWITTER_API_SECRET=...
TWITTER_ACCESS_TOKEN=...
TWITTER_ACCESS_SECRET=...
```

### Terminal (`services/terminal/.env`)

```bash
REDIS_URL=redis://localhost:6379
ADMIN_JWT_SECRET=...
NEXT_PUBLIC_WS_URL=wss://alphaclaw.ai/api/ws
```

---

## 17. Phase Roadmap

### Phase 1 — The Scraper & Schema
**Goal:** Build and validate the Narrative Dictionary and Fast-Match engine. No trades, no money.

Deliverables:
- `data/narrative_dict.json` seeded with 6+ categories
- `data/resolution_schemas/` seeded with 10+ active Polymarket markets
- Fast-Match engine with unit tests (coverage >= 80%)
- Normalizer with test suite covering adversarial edge cases
- TimescaleDB schema deployed and migrated
- All four ingestion tiers running and logging to `signals` table
- Docker Compose for local dev

Exit criteria: Fast-Match correctly classifies 90%+ of a hand-labeled test set of 50 historical signals.

### Phase 2 — The Paper Predator (Shadow Mode)
**Goal:** Calibrate Z-score thresholds and Kelly weights against real markets without real capital.

Deliverables:
- Opus Appeals layer fully wired
- Kelly sizer implemented
- Full Brain → Broker pipeline running with `DRY_RUN=true`
- `trade_signals` published to Redis and logged, but Broker does not call CLOB API
- Neural Trace terminal live at alphaclaw.ai showing shadow trades
- Shadow Mode PnL tracking in TimescaleDB

Exit criteria: 30 days of shadow trading with Sharpe ratio > 1.5 and win rate > 60%.

### Phase 3 — The First Strike (Live Trading)
**Goal:** $1,000 hot wallet; live trading enabled.

Deliverables:
- `DRY_RUN=false`
- TWAP/nibble execution live
- Circuit breaker fully tested
- X auto-posting live
- `alphaclaw.eth` wallet funded and monitored

Exit criteria: 60 days of live trading without triggering circuit breaker more than twice.

### Phase 4 — Sovereign Expansion
**Goal:** Autonomous treasury management and Market Memory learning.

Deliverables:
- Market Memory: TimescaleDB historical win/loss by category → adjusts `W_OPUS`/`W_ZSCORE` weights
- Auto-generate Resolution Schemas from new Polymarket markets via Opus
- Multi-market simultaneous exposure management (portfolio-level Kelly)
- Automated whitelist expansion: identifies new high-signal accounts from winning signal chains

---

## 18. Risk Register

| ID | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R-01 | Slippage in thin markets | High | Medium | TWAP nibble execution; `MIN_BOOK_DEPTH_USDC` gate |
| R-02 | Oracle (UMA) malfunction / wrong resolution | Low | High | Strict resolution schema adherence; oracle dispute monitor |
| R-03 | Adversarial baiting by whales | Medium | High | Multi-source independence; adversarial token filter |
| R-04 | Perplexity rate limit during golden window | Medium | High | Request queue with priority; fallback to cached Perplexity result if <5min old |
| R-05 | Opus latency > golden window | Medium | High | Async processing; signal expiry gate (`expires_at`) |
| R-06 | Redis pub/sub message loss | Low | High | Persist all `trade_signals` to DB before publishing; broker re-queries on reconnect |
| R-07 | Hot wallet key compromise | Very Low | Critical | Key in env only; never logged; Hibernation revokes in-memory on loss limit |
| R-08 | Narrative Dictionary stale / missing category | Medium | Medium | Weekly manual review cycle; Market Memory flags missed signals in Phase 4 |
| R-09 | Polymarket API downtime | Low | Medium | Graceful degradation; re-queue signals on reconnect; WebSocket auto-reconnect with backoff |
| R-10 | False positive fast-match score | Medium | Medium | Quarter Kelly sizing; Opus gate requires confidence >= 0.80 |

---

*End of AlphaClaw Technical Specification v0.1.0*
