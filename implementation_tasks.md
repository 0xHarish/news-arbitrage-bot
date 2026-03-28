# AlphaClaw — Implementation Tasks
**Spec Version:** 0.3.0  
**Last Updated:** 2026-02-24

> Each task maps directly to a section of `specs.md`. Tasks are ordered for execution dependency — complete them roughly top-to-bottom within each phase.

---

## PHASE 0 — Foundation & Tooling
*One-time setup. Must be complete before any Phase 1 code is written.*

### 0.1 Repository & Version Control
- [ ] Create `alphaclaw` monorepo on GitHub (private)
- [ ] Define branch strategy: `main` (prod), `dev` (integration), `feature/*` (work branches)
- [ ] Add root `.gitignore` covering `.env`, `*.tmp.json`, `__pycache__`, `node_modules`, `.next`, `dist`
- [ ] Add `.gitattributes` to enforce LF line endings across all platforms
- [ ] Create root `README.md` with project overview and service startup instructions
- [ ] Set up GitHub branch protection on `main`: require PR review, no force-push, status checks must pass

### 0.2 Secrets & Credential Management
- [ ] Generate `ADMIN_JWT_SECRET` via `openssl rand -hex 32` — store in password manager, never in repo
- [ ] Create a dedicated `alphaclaw.eth` hot wallet (Metamask or hardware-generated) — record address and private key in vault only
- [ ] Register for Anthropic API access — obtain `ANTHROPIC_API_KEY`
- [ ] Register for Perplexity API access — obtain `PERPLEXITY_API_KEY`
- [ ] Register for SocialData.tools API access — obtain `SOCIALDATA_API_KEY`
- [ ] Register Polymarket CLOB API access — confirm EIP-712 signing requirements for the hot wallet
- [ ] Register Twitter Developer App (Basic tier minimum) — obtain all four Twitter API credentials
- [ ] Create `.env.example` files for all three services (`brain`, `broker`, `terminal`) with placeholder values and comments — commit these; never commit real `.env` files

### 0.3 Local Infrastructure
- [ ] Install Docker Desktop (or Docker Engine + Compose plugin on Linux VPS)
- [ ] Write `infra/docker-compose.yml` with services: `timescaledb`, `redis`
- [ ] Write `infra/timescaledb/init.sql` — placeholder that will run all migrations in Phase 1
- [ ] Write `infra/redis/redis.conf` — disable persistence (`save ""`), set `maxmemory-policy allkeys-lru`, bind to localhost only
- [ ] Verify `docker compose up` starts both services cleanly with health checks
- [ ] Install `psql` CLI locally and confirm connection to TimescaleDB container
- [ ] Install `redis-cli` locally and confirm connection to Redis container

### 0.4 VPS Provisioning
- [ ] Provision VPS (minimum: 4 vCPU, 8GB RAM, 80GB SSD) — Ubuntu 24 LTS
- [ ] Harden VPS: disable root SSH login, create `alphaclaw` user, set up SSH key auth only
- [ ] Configure UFW: allow ports 22 (SSH), 80, 443 only — block all else including Redis and Postgres ports from external access
- [ ] Install Docker Engine + Compose plugin on VPS
- [ ] Set up DNS: point `alphaclaw.ai` A record to VPS IP
- [ ] Install Nginx as reverse proxy and configure SSL via Let's Encrypt / Certbot
- [ ] Configure Nginx: proxy `alphaclaw.ai` → Next.js on port 3000, `wss://` upgrade support
- [ ] Set `ADMIN_ALLOWED_IPS` to the static IP of your primary development machine

### 0.5 CI/CD
- [ ] Create GitHub Actions workflow: on PR to `dev` → run `pytest` (brain) + `bun test` (broker) + `next build` (terminal)
- [ ] Create GitHub Actions workflow: on merge to `main` → SSH deploy to VPS, run `docker compose pull && docker compose up -d`
- [ ] Add Slack or email alert for failed CI runs

---

## PHASE 1 — The Scraper & Schema
*Goal: Narrative Dictionary, Fast-Match engine, all four ingestion tiers running, data landing in TimescaleDB. No trades, no money.*

### 1.1 TimescaleDB Schema
- [ ] Enable TimescaleDB extension in `init.sql`: `CREATE EXTENSION IF NOT EXISTS timescaledb`
- [ ] Write and run migration: create `signals` table with all columns and indexes per spec §4.1
- [ ] Write and run migration: create `opus_verdicts` table per spec §4.2
- [ ] Write and run migration: create `trade_signals` table per spec §4.3
- [ ] Write and run migration: create `trades` table per spec §4.4
- [ ] Write and run migration: create `pnl_snapshots` table per spec §4.5
- [ ] Write and run migration: create `oracle_disputes` table with unique partial index per spec §4.6
- [ ] Write and run migration: create `circuit_breaker_events` table per spec §4.7
- [ ] Write and run migration: create `whitelist_accounts` table with `updated_at` trigger per spec §5.2
- [ ] Write and run migration: create `shadow_trades` table with all `GENERATED ALWAYS AS` computed columns per spec §18.3
- [ ] Convert all tables to hypertables with `create_hypertable()` using 7-day chunk intervals (1-day for `pnl_snapshots`)
- [ ] Create all secondary indexes specified in the schema
- [ ] Write a `db/seed_whitelist.sql` with 20+ starter accounts across all 6 seed categories
- [ ] Validate full schema: run `\d+ <table>` for every table and confirm columns, types, and constraints match spec

### 1.2 Brain Service — Project Scaffold
- [ ] Create `services/brain/` directory with Python 3.12+ virtual environment
- [ ] Create `services/brain/requirements.txt` with: `fastapi`, `uvicorn`, `asyncpg`, `redis[asyncio]`, `httpx`, `websockets`, `spacy`, `nltk`, `pydantic`, `python-dotenv`, `pytest`, `pytest-asyncio`
- [ ] Create `services/brain/app/main.py` — FastAPI app with lifespan manager: starts all background tasks on startup, graceful shutdown on SIGTERM
- [ ] Create `services/brain/app/db.py` — asyncpg connection pool, exposed as a module-level singleton
- [ ] Create `services/brain/app/redis_client.py` — async Redis client singleton
- [ ] Create `services/brain/.env` from `.env.example` — fill all values
- [ ] Add brain service to `docker-compose.yml` with volume mounts for `data/` directory
- [ ] Confirm `uvicorn app.main:app --reload` starts without errors

### 1.3 Narrative Dictionary — Seed File
- [ ] Create `data/narrative_dict.json` with top-level structure: `version`, `last_updated`, `categories`
- [ ] Author `us_election_*` category: add AP call, NYT call, and Fox call as resolution signals with correct `canonical_tokens`, `regex_patterns`, `weights`, and `adversarial_patterns` (e.g. "leading", "projected by sources")
- [ ] Author `crypto_regulatory` category: SEC enforcement, CFTC ruling, ETF approval/denial signals — include `sec.gov`, `cftc.gov` as authoritative `source_domains`
- [ ] Author `macro_fed` category: FOMC rate decision signals — `federalreserve.gov` as weight-1.0 source, wire services (Reuters, Bloomberg, WSJ) as secondary sources
- [ ] Author `geopolitical_ceasefire` category: define what constitutes a confirmed ceasefire vs unverified reports — be explicit about `adversarial_patterns` ("sources say", "reportedly", "may agree")
- [ ] Author `tech_ipo` category: SEC S-1 filing, IPO pricing confirmation, first-trade signals
- [ ] Author `sports_championship` category: cover NBA Finals, NFL Super Bowl, FIFA World Cup Final — include official league sources and major wire services
- [ ] For each category, write a minimum of 3 `adversarial_patterns` that disqualify false positives
- [ ] Set `golden_window_seconds` for each category based on realistic market reaction speed (elections: 900s, Fed: 600s, sports: 300s)
- [ ] Set `minimum_independent_sources` for each category (minimum 2 for all, 3 for high-stakes markets)
- [ ] Validate JSON schema against a hand-written Pydantic model — confirm no missing required fields

### 1.4 Resolution Schema — Seed Files
- [ ] Browse Polymarket and identify 10+ active markets across all 6 seed categories
- [ ] For each market, create `data/resolution_schemas/<condition_id>.json` per spec §7 schema
- [ ] Ensure each resolution schema includes: `market_id`, `condition_id`, `title`, `category_id`, `resolution_source`, `resolution_criteria` (verbatim from Polymarket), `key_entities[]`, `threshold_conditions[]` for both YES and NO outcomes, `oracle_contract`, `expiry_timestamp`
- [ ] For each `threshold_condition`, specify both `required_tokens` AND `forbidden_tokens` — cross-check against the raw `resolution_criteria` text
- [ ] Validate all 10+ resolution schemas against a Pydantic model
- [ ] Write a helper script `scripts/list_schemas.py` that prints a summary table of all loaded schemas (market title, category, expiry)

### 1.5 Text Normalizer
- [ ] Create `services/brain/app/fastmatch/normalizer.py`
- [ ] Implement `normalize(text: str) -> list[str]` per spec §9.2: lowercase → strip URLs/mentions/hashtags → remove punctuation → tokenize → remove stopwords → lemmatize → deduplicate
- [ ] Download spaCy model: `python -m spacy download en_core_web_sm`
- [ ] Download NLTK stopwords corpus: add download call to app startup
- [ ] Extend NLTK stopwords with a custom financial stopword list: "said", "according", "report", "told", "sources", "noted", "added" (words that appear in every news headline but carry no signal)
- [ ] Write unit tests for `normalizer.py`:
  - [ ] Test: URL stripping (URLs must be fully removed, not tokenized)
  - [ ] Test: Hashtag and @mention stripping
  - [ ] Test: Adversarial phrase survival — "AP calls race" → tokens include "ap", "call", "race"
  - [ ] Test: False positive suppression — "AP sources say leading" → adversarial tokens present
  - [ ] Test: Lemmatization — "raises" → "raise", "called" → "call", "increasing" → "increase"
  - [ ] Test: Empty string and None input handling

### 1.6 Z-Score Calculator
- [ ] Create `services/brain/app/fastmatch/zscore.py`
- [ ] Implement `RollingZScore` class using Welford's online algorithm per spec §9.3
- [ ] Implement `update(value: float) -> float` — returns z-score; returns 0.0 if window not yet filled
- [ ] Implement `reset()` — clears state (called on WebSocket reconnect)
- [ ] Write unit tests:
  - [ ] Test: Z-score of 0 on all-identical values
  - [ ] Test: Known z-score on a manually computed sequence
  - [ ] Test: Returns 0.0 before window is filled
  - [ ] Test: Numerical stability — no NaN or Inf on extreme inputs
  - [ ] Test: `reset()` correctly clears rolling state

### 1.7 Fast-Match Engine
- [ ] Create `services/brain/app/fastmatch/engine.py`
- [ ] Implement `FastMatchEngine.__init__`: loads `narrative_dict.json` and all resolution schemas into memory on startup
- [ ] Implement Step 1 — normalization: call `normalizer.normalize()` on inbound signal text
- [ ] Implement Step 2 — oracle dispute check: query `oracle_disputes` table; drop signal if ACTIVE dispute exists for the matched `market_id`
- [ ] Implement Step 3 — adversarial filter: check normalized tokens against `category.adversarial_patterns`; if match → drop and log `drop_reason="adversarial_pattern"`
- [ ] Implement Step 4 — token matcher: for each `resolution_signal` in matched category, verify ALL `canonical_tokens` present AND at least one `regex_pattern` matches; accumulate weighted score
- [ ] Implement Step 5 — source independence check: count distinct `source_domains` in signal cluster; reject if below `minimum_independent_sources`
- [ ] Implement Step 6 — Z-score gate (Tier 1 only): verify `|zscore| >= Z_SCORE_THRESHOLD`
- [ ] Implement Step 7 — golden window check: reject if signal timestamp is older than `golden_window_seconds`
- [ ] Implement Step 8 — score aggregation: produce `fast_match_score` float 0–1
- [ ] Implement escalation trigger: if `fast_match_score >= FAST_MATCH_ESCALATION_THRESHOLD` → call Perplexity + escalate to Opus
- [ ] Implement drop logging: all dropped signals must be written to `signals` table with `fast_match_hit=FALSE` and a `drop_reason` field (add this column to schema if missing)
- [ ] Write unit tests (target ≥80% coverage):
  - [ ] Test: Signal with all tokens present + 2 independent sources → score ≥ 0.6, escalate
  - [ ] Test: Signal with adversarial token ("leading") → dropped
  - [ ] Test: Signal with only 1 source domain → dropped at source independence check
  - [ ] Test: Signal older than `golden_window_seconds` → dropped
  - [ ] Test: Signal for market with active oracle dispute → dropped
  - [ ] Test: Tier 1 signal with |zscore| < threshold → dropped
  - [ ] Test: Score below escalation threshold → not escalated to Opus
- [ ] Build hand-labeled test set of 50 historical signals (25 true positives, 25 negatives) for exit criterion validation

### 1.8 Whitelist Account System
- [ ] Create `services/brain/app/whitelist/exporter.py`
- [ ] Implement `export_snapshot()` per spec §5.3: query DB → validate size → write to `.tmp.json` → atomic `os.replace()` to final path
- [ ] Implement the 50%-shrink abort guard with `alert_admin()` function (log to stdout + write to a `admin_alerts` table)
- [ ] Create `admin_alerts` table migration: `id`, `created_at`, `severity`, `message`
- [ ] Implement scheduled export loop: run every `WHITELIST_EXPORT_INTERVAL_SECONDS`
- [ ] Implement Redis `whitelist:refresh` subscriber that calls `export_snapshot()` immediately on message
- [ ] Implement `WhitelistCache` class per spec §5.5: `load()`, `get(handle)`, `reload_if_stale(max_age_seconds)`
- [ ] Write unit tests:
  - [ ] Test: Atomic write — simulate crash mid-write (truncated tmp file), confirm final file is untouched
  - [ ] Test: 50% shrink guard — mock DB returning 10 rows when previous snapshot had 100 rows → abort
  - [ ] Test: Empty export → abort
  - [ ] Test: `WhitelistCache.reload_if_stale()` correctly reloads when age > threshold
  - [ ] Test: `WhitelistCache.get()` returns correct account by handle; returns None for unknown handle

### 1.9 Tier 1 — Polymarket WebSocket Ingestion
- [ ] Create `services/brain/app/ingestion/polymarket_ws.py`
- [ ] Implement WebSocket connection to Polymarket public feed using `websockets` library
- [ ] Implement subscription to `book`, `price_change`, and `trade` channels for all markets listed in `data/resolution_schemas/`
- [ ] Implement auto-reconnect with exponential backoff (start 1s, max 60s, jitter)
- [ ] On reconnect, call `RollingZScore.reset()` for all affected markets to prevent stale rolling state
- [ ] Implement Z-score computation per tick: call `RollingZScore.update(price)` and emit `Tier1Signal` if `|zscore| >= Z_SCORE_THRESHOLD`
- [ ] Write all inbound ticks to `signals` table regardless of Z-score (for Phase 2 calibration data)
- [ ] Write unit/integration tests: mock WebSocket server → confirm correct channel subscription, Z-score emission, and reconnect behavior

### 1.10 Tier 2 — SocialData.tools Poller
- [ ] Create `services/brain/app/ingestion/social_poller.py`
- [ ] Implement REST polling loop every `SOCIAL_POLL_INTERVAL_SECONDS` against SocialData.tools API
- [ ] Implement whitelist filter: skip any tweet not from a handle in `WhitelistCache`
- [ ] Implement deduplication: track seen tweet IDs in Redis SET `social:seen_ids` with 1-hour TTL
- [ ] Emit `Tier2Signal` dataclass with all fields per spec §8.2
- [ ] Pass `Tier2Signal` to `FastMatchEngine` for processing
- [ ] Write all signals (hit or miss) to `signals` table
- [ ] Write integration test: mock SocialData API response → confirm whitelist filtering, deduplication, and signal emission

### 1.11 Tier 3 — Hacker News Poller
- [ ] Create `services/brain/app/ingestion/hn_poller.py`
- [ ] Implement `/v0/topstories` polling loop every `HN_POLL_INTERVAL_SECONDS`
- [ ] Implement seen-ID tracking in Redis SET `hn:seen` with 24-hour TTL
- [ ] Implement keyword filter: only process stories where title token intersection with any active `market_tags` is non-empty
- [ ] Fetch full story metadata (title, URL, score, time) for matched stories
- [ ] Emit `Tier3Signal` and pass to `FastMatchEngine`
- [ ] Write integration test: mock HN API → confirm seen-ID deduplication, keyword filtering, and correct signal structure

### 1.12 Tier 4 — Perplexity Client
- [ ] Create `services/brain/app/ingestion/perplexity_client.py`
- [ ] Implement `fetch_corroboration(query: str) -> PerplexityResult` using `sonar-pro` model with `search_recency_filter: "hour"`
- [ ] Implement `PerplexityResult` dataclass: `summary`, `citations: list[{url, domain, snippet}]`, `confidence`
- [ ] Extract and normalize `source_domains` from citation URLs (strip subdomains to root domain)
- [ ] Implement request cache: cache results by query hash in Redis with 5-minute TTL (addresses risk R-04)
- [ ] Implement retry with exponential backoff on rate-limit (HTTP 429) responses
- [ ] Write unit tests: mock Perplexity API → confirm citation extraction, domain normalization, and cache hit/miss behavior

### 1.13 Brain Service — Wiring & Integration
- [ ] Wire all four ingestion tiers as background `asyncio` tasks in `main.py` lifespan
- [ ] Wire `WhitelistCache` startup and scheduled refresh into lifespan
- [ ] Wire `FastMatchEngine` initialization (load dicts and schemas) into lifespan
- [ ] Implement a `/health` FastAPI endpoint returning: service status, last signal timestamp, whitelist account count, active markets count, Redis ping, DB ping
- [ ] Implement a `/metrics` FastAPI endpoint (internal only): signals_per_minute, escalations_per_hour, drops_by_reason
- [ ] Run full integration test: start all tiers, manually inject a test signal, confirm it flows through Fast-Match and lands in the `signals` table

### 1.14 Phase 1 Exit Validation
- [ ] Run hand-labeled test set of 50 signals through Fast-Match engine
- [ ] Measure classification accuracy: must hit ≥90% correct classification
- [ ] Confirm all 7 TimescaleDB tables exist, have correct schemas, and hypertables are active
- [ ] Confirm `whitelist_accounts` table is seeded with ≥20 active accounts across all 6 categories
- [ ] Confirm all 10+ resolution schemas are present and Pydantic-validated
- [ ] Confirm `narrative_dict.json` has all 6 seed categories with adversarial patterns
- [ ] Confirm Docker Compose starts the full local stack cleanly from a fresh clone
- [ ] Confirm `pytest` passes with ≥80% coverage on all Brain service modules

---

## PHASE 2 — The Paper Predator (Shadow Mode)
*Goal: Full pipeline running end-to-end with `DRY_RUN=true`. Shadow trades tracked in TimescaleDB. Terminal live. 30-day calibration run.*

### 2.1 Claude Opus 4.6 Appeals Layer
- [ ] Create `services/brain/app/appeals/opus_client.py`
- [ ] Implement `OpusClient.invoke(signal, resolution_schema, perplexity_result) -> OpusVerdict`
- [ ] Implement exact system prompt per spec §10.2 — do not paraphrase; the legalistic framing is load-bearing
- [ ] Implement `build_user_prompt()` per spec §10.2: inject `resolution_criteria`, `key_entities`, `threshold_conditions`, signal text, and Perplexity citations
- [ ] Implement JSON response parsing with strict validation against `OpusVerdict` dataclass
- [ ] Implement `requires_perplexity_followup` re-invocation loop (max 1 re-invocation)
- [ ] Implement `outcome == "AMBIGUOUS"` drop path — log to `opus_verdicts` with drop reason
- [ ] Implement `confidence < 0.80` drop path — log to `opus_verdicts` with drop reason
- [ ] Implement latency tracking: record `latency_ms` on every invocation
- [ ] Write `opus_verdicts` row to TimescaleDB on every Opus call (pass or drop)
- [ ] Create `services/brain/app/appeals/prompt_builder.py` — all prompt construction logic lives here, not in `opus_client.py`
- [ ] Write unit tests with mocked Anthropic API:
  - [ ] Test: Valid confident YES verdict → `OpusVerdict` correctly populated
  - [ ] Test: AMBIGUOUS verdict → signal dropped, verdict written to DB
  - [ ] Test: confidence = 0.79 → signal dropped
  - [ ] Test: `requires_perplexity_followup=true` → re-invocation triggered, not triggered a second time
  - [ ] Test: Malformed JSON from Opus → handled gracefully, not a 500 crash

### 2.2 Kelly Criterion Sizer
- [ ] Create `services/brain/app/sizing/kelly.py`
- [ ] Implement `calculate_composite_p(opus_confidence, zscore) -> float` per spec §11.1: `(opus_confidence × W_OPUS) + (normalized_zscore × W_ZSCORE)`, load weights from env
- [ ] Implement `normalize_zscore(zscore) -> float`: `min(abs(zscore) / Z_SCORE_MAX, 1.0)`
- [ ] Implement `calculate_kelly_fraction(composite_p, market_price) -> float` per spec §11.2
- [ ] Implement `calculate_position(composite_p, market_price, wallet_total_usdc) -> float` per spec §11.3: apply quarter-Kelly, 5% hard cap, `MIN_POSITION_USDC` floor, round to 2 decimal places
- [ ] Validate edge cases: `market_price` of 0.01 or 0.99, composite_p of exactly 0.80, wallet of $1,000
- [ ] Write unit tests:
  - [ ] Test: composite_p = 0.87, market_price = 0.31, wallet = $1000 → position matches manual calculation
  - [ ] Test: position never exceeds 5% of wallet regardless of inputs
  - [ ] Test: position = 0.0 if composite_p < MIN_COMPOSITE_P
  - [ ] Test: position = MIN_POSITION_USDC floor if Kelly produces a very small number
  - [ ] Test: negative Kelly fraction (bet is unprofitable) → returns 0.0, never negative position

### 2.3 Redis Publisher (Brain)
- [ ] Create `services/brain/app/publisher/redis_pub.py`
- [ ] Implement `publish_trade_signal(signal, verdict, position) -> None`: serialize to `trade_signals` JSON per spec §12.2, write to TimescaleDB `trade_signals` table FIRST, then publish to Redis channel
- [ ] Implement `publish_neural_trace_raw(event_type, payload) -> None`: publish to `neural_trace_raw` and append to Redis list `neural_trace_raw:history` (LPUSH + LTRIM 500)
- [ ] Implement `publish_neural_trace_public(event_type, payload) -> None`: sanitize payload (remove `raw_text`, internal scores), publish to `neural_trace_public` and append to `neural_trace_public:history` (LPUSH + LTRIM 100)
- [ ] Implement `publish_drop_event(signal, drop_reason) -> None`: publishes `PASS` or `SHIELD` event to public channel with sanitized reason
- [ ] Write unit tests: confirm DB write happens before Redis publish (ordering matters for R-06 mitigation)

### 2.4 Broker Service — Project Scaffold
- [ ] Create `services/broker/` directory with `bun init`
- [ ] Create `package.json` with dependencies: `ioredis`, `@polymarket/clob-client`, `ethers`, `pg`, `zod`, `typescript`, `bun-types`
- [ ] Create `services/broker/src/index.ts` — entry point, starts Redis subscriber and circuit breaker
- [ ] Create `services/broker/src/db.ts` — `pg` connection pool singleton
- [ ] Create `services/broker/.env` from `.env.example` — fill all non-wallet values (wallet filled in Phase 3)
- [ ] Add broker service to `docker-compose.yml`
- [ ] Confirm `bun run src/index.ts` starts without errors

### 2.5 Broker — Redis Subscriber
- [ ] Create `services/broker/src/redis_sub.ts`
- [ ] Subscribe to `trade_signals` channel
- [ ] Subscribe to `circuit_breaker_state` channel — update in-memory circuit breaker state on message
- [ ] Subscribe to `whitelist:refresh` channel — re-publish to Brain via Redis (Brain's exporter handles it)
- [ ] On `trade_signals` message: validate JSON with `zod` schema; reject and log malformed messages
- [ ] On valid message: check circuit breaker state → if `HIBERNATION`, reject and publish `SHIELD` event
- [ ] On valid message: check `expires_at` → reject stale signals
- [ ] Route valid, non-expired signals to `ClobExecutor` (Phase 3) or `ShadowExecutor` (Phase 2)
- [ ] Write unit tests: mock Redis → confirm message validation, hibernation rejection, expiry rejection

### 2.6 Shadow Executor
- [ ] Create `services/broker/src/shadow/shadow_executor.ts`
- [ ] Implement `executeShadowTrade(signal)`: fetch top-of-book mid-price from CLOB API (read-only, no auth required for order book), write `shadow_trades` row with `entry_price`, `composite_p`, `kelly_fraction`, `position_usdc`, `tracking_status='PENDING'`
- [ ] Confirm `DRY_RUN=true` env guard: if `DRY_RUN !== 'true'`, `ShadowExecutor` must throw at construction time
- [ ] Publish shadow `STRIKE` event to both `neural_trace_raw` and `neural_trace_public`
- [ ] Write unit tests: confirm `shadow_trades` row is written, correct `entry_price` is fetched, neural trace events published

### 2.7 Shadow Price Tracker
- [ ] Create `services/brain/app/shadow/price_tracker.py`
- [ ] Implement `fetch_mid_price(market_id) -> float | None` per spec §18.5
- [ ] Implement `fetch_market_resolution(market_id) -> dict` per spec §18.5: returns `resolved`, `settlement_price`, `resolved_at`
- [ ] Implement `process_shadow_row(row)` per spec §18.4 — full resolution-aware logic:
  - [ ] For `price_15m` and `price_1h`: use mid-price once horizon passes
  - [ ] For `price_24h`: use settlement price if market resolved before T+24h; use mid-price otherwise
  - [ ] Update `max_favorable` and `max_adverse` continuously within the 24h window while market is unresolved
  - [ ] Set `tracking_status='COMPLETE'` when all horizons are filled
  - [ ] Set `resolved_at`, `resolved_price` when settlement is detected
- [ ] Implement `track_shadow_trades()` async loop: runs every 60 seconds
- [ ] Write unit tests:
  - [ ] Test: market resolves YES at T+3h → `price_24h=1.0`, `resolved_before_24h=true`, `tracking_status='COMPLETE'`
  - [ ] Test: market still active at T+24h → `price_24h` = mid-price, `price_24h_basis='midprice'`
  - [ ] Test: MFE/MAE correctly track the highest and lowest price seen in the 24h window
  - [ ] Test: API error in `fetch_mid_price` → row not updated, no crash, retried next cycle
  - [ ] Test: already-COMPLETE rows are skipped

### 2.8 Broker — Neural Trace Publisher
- [ ] Create `services/broker/src/publisher/neural_trace_pub.ts`
- [ ] Implement `publishRaw(eventType, payload)`: publish to `neural_trace_raw`, LPUSH to history list, LTRIM to 500
- [ ] Implement `publishPublic(eventType, payload)`: sanitize payload, publish to `neural_trace_public`, LPUSH to history list, LTRIM to 100
- [ ] Map broker event types to public labels: `TRADE_EXECUTED → STRIKE`, `SIGNAL_REJECTED → SHIELD`, `DAILY_RESET → SHIELD`
- [ ] Write unit test: confirm sanitization removes sensitive fields before public publish

### 2.9 Next.js Terminal — Project Scaffold
- [ ] Create `services/terminal/` with `npx create-next-app@latest` (TypeScript, App Router, Tailwind)
- [ ] Install dependencies: `ioredis`, `jsonwebtoken`, `zod`
- [ ] Set up Tailwind dark theme: background `#0a0a0a`, primary green `#00ff41`, amber `#ffb703`, red `#e63946`
- [ ] Choose and install monospace font (JetBrains Mono or IBM Plex Mono via `next/font`)
- [ ] Create base layout: full-viewport dark terminal with fixed header showing `ALPHACLAW` wordmark and circuit breaker status pill
- [ ] Add `services/terminal/.env` from example — fill `REDIS_URL`, `ADMIN_JWT_SECRET`, `ADMIN_ALLOWED_IPS`
- [ ] Add terminal to `docker-compose.yml`
- [ ] Confirm `next dev` runs without errors

### 2.10 Terminal — WebSocket API Route
- [ ] Create `services/terminal/app/api/ws/route.ts`
- [ ] Implement WebSocket upgrade handler using Next.js route handler pattern
- [ ] Implement `feed=public` path: subscribe to `neural_trace_public` Redis channel, forward events to connected clients
- [ ] Implement `feed=raw` path: validate IP (middleware), validate token (`timingSafeEqual`), subscribe to `neural_trace_raw`
- [ ] On client connect: send `SNAPSHOT` message with last 20 events from history list
- [ ] Implement 30-second PING/PONG keepalive — close connection if no PONG received
- [ ] On client disconnect: unsubscribe from Redis channels to prevent memory leak
- [ ] Write integration test: connect mock WebSocket client → confirm snapshot delivery, event forwarding, PING/PONG

### 2.11 Admin Security Middleware
- [ ] Create `services/terminal/middleware.ts` with IP allowlist check per spec §17.3
- [ ] Return `404 Not Found` (not `403`) for requests from non-allowed IPs to `/admin/*` and `/api/ws/*`
- [ ] Create `services/terminal/lib/auth.ts` with `validateAdminToken()` using `crypto.timingSafeEqual` per spec §17.4
- [ ] Create `services/terminal/lib/rate_limit.ts` with sliding window per spec §17.5: 10 requests per 60-second window, `429` response with `Retry-After` header
- [ ] Implement header sanitization in `services/terminal/lib/log_sanitizer.ts` — `[REDACTED]` for `Authorization`, `x-admin-token`, `cookie` per spec §17.6
- [ ] Confirm that `next.config.ts` does not log full URLs
- [ ] Write unit tests for each security utility (rate limiter, token validator, header sanitizer)

### 2.12 Terminal — UI Components
- [ ] Build `NeuralTrace.tsx`:
  - [ ] WebSocket connection with auto-reconnect (exponential backoff, max 30s)
  - [ ] Auto-scrolling feed with "pause on hover" using `useRef` + `IntersectionObserver`
  - [ ] Color coding: HUNT=amber, STRIKE=green, PASS=gray, SHIELD=red
  - [ ] Per-event display: timestamp, event type badge, market title (truncated to 60 chars), action summary
  - [ ] Progressive reveal: Δ15m, Δ1h fields render as `—` until filled, then animate in
  - [ ] `[SHADOW]` badge on all Phase 2 events in amber italic
- [ ] Build `TradeCard.tsx`:
  - [ ] Expandable card triggered by clicking a STRIKE event
  - [ ] Shows: market title (full), side (YES/NO pill), entry price, edge %, position USDC, composite_p
  - [ ] Shows: Δ15m, Δ1h, Δ24h with directional arrow and green/red color
  - [ ] Shows: MFE/MAE bars — visual horizontal bar showing favorable vs adverse excursion magnitude
  - [ ] Shows: `price_24h_basis` label ("Settlement ✓" or "Mark-to-Market")
  - [ ] Shows: source URL as clickable link with external icon
  - [ ] Shows: reasoning summary in monospace italic
- [ ] Build `PnLDisplay.tsx`:
  - [ ] Subscribe to `/api/pnl` SSE endpoint for real-time wallet updates
  - [ ] Display: total wallet USDC (large, prominent), today's realized PnL% (color-coded), open positions count
  - [ ] Warning state: amber color when within 5% of `-15%` daily loss limit
  - [ ] Critical state: red + flash animation when circuit breaker is in HIBERNATION
- [ ] Build `/api/pnl` SSE endpoint: reads `pnl_updates` Redis channel, streams to client as SSE
- [ ] Build `/` root page: full-height layout with `NeuralTrace` on left (60%), `PnLDisplay` + stats sidebar on right (40%)
- [ ] Build `/admin` page: shows raw `neural_trace_raw` feed + full `trade_signals` table with all internal fields

### 2.13 Terminal — Public Deployment
- [ ] Build and deploy terminal to VPS: `docker compose build terminal && docker compose up -d terminal`
- [ ] Confirm `https://alphaclaw.ai` serves the terminal with valid SSL certificate
- [ ] Confirm WebSocket connection works through Nginx proxy (`wss://alphaclaw.ai/api/ws?feed=public`)
- [ ] Confirm `/admin` route returns `404` from a non-allowlisted IP
- [ ] Confirm `/admin` route is accessible from allowlisted IP with correct token

### 2.14 Shadow Mode — 30-Day Calibration Run
- [ ] Deploy full stack to VPS with `DRY_RUN=true`
- [ ] Confirm signals are flowing from all four ingestion tiers into `signals` table
- [ ] Confirm Opus escalations are occurring and `opus_verdicts` table is populating
- [ ] Confirm `shadow_trades` rows are being created on qualifying signals
- [ ] Confirm price tracker is running and filling `price_15m`, `price_1h`, `price_24h` columns
- [ ] Confirm MFE/MAE fields are being updated in real-time
- [ ] Monitor daily: check `signals` table for unexpected drop patterns; tune `Z_SCORE_THRESHOLD` and `W_OPUS`/`W_ZSCORE` weights if needed
- [ ] At 30-day mark: run Phase 2 exit criteria query per spec §18.7
  - [ ] Verify `overall_win_rate >= 0.60`
  - [ ] Verify `overall_edge_ratio >= 3.0`
- [ ] Document tuned parameter values in a `docs/calibration_log.md`
- [ ] If criteria not met: diagnose, adjust parameters, extend calibration run

---

## PHASE 3 — The First Strike (Live Trading)
*Goal: $1,000 hot wallet, live execution enabled, circuit breaker battle-tested, X posting live.*

### 3.1 Polymarket CLOB Client
- [ ] Create `services/broker/src/polymarket/clob_client.ts`
- [ ] Implement `getMarket(conditionId: string) -> Promise<Market>`: fetch full market metadata from CLOB API
- [ ] Implement `getOrderBook(tokenId: string) -> Promise<OrderBook>`: fetch live bids/asks
- [ ] Implement `createOrder(params: OrderParams) -> Promise<OrderResult>`: construct and submit EIP-712 signed limit order
- [ ] Implement `cancelOrder(orderId: string) -> Promise<void>`
- [ ] Implement `getOrder(orderId: string) -> Promise<Order>`: poll for fill status
- [ ] Load private key from `HOT_WALLET_PRIVATE_KEY` env var into an `ethers.Wallet` — never log the key, not even partially
- [ ] Implement request retry (3 attempts, exponential backoff) on 5xx errors
- [ ] Implement graceful handling of 429 rate limits: respect `Retry-After` header
- [ ] Write integration tests against Polymarket testnet/sandbox if available; otherwise mock HTTP responses for all methods

### 3.2 Hot Wallet Management
- [ ] Create `services/broker/src/polymarket/wallet.ts`
- [ ] Implement `getWalletBalance() -> Promise<number>`: query USDC balance via CLOB API or direct Polygon RPC
- [ ] Implement `revokeWalletInMemory()`: sets the in-memory private key reference to null, preventing any further order creation — does NOT modify env file
- [ ] Implement `isWalletActive() -> boolean`: returns false if key has been revoked
- [ ] Add wallet active check as first guard in `ClobExecutor` before any order is placed
- [ ] Write unit tests: confirm revoke prevents order creation, confirm balance query

### 3.3 TWAP / Nibble Executor
- [ ] Create `services/broker/src/polymarket/order_builder.ts`
- [ ] Implement `NibbleExecution` orchestrator per spec §13.3: split `totalUsdc` into `TWAP_SLICES` equal parts
- [ ] Implement per-slice execution: place limit order at best-ask, wait `intervalMs`, check fill
- [ ] Implement slippage check per slice: if `fill_price > entry_price * (1 + MAX_SLIPPAGE_BPS/10000)` → cancel remaining slices, record partial fill
- [ ] Implement order status polling: poll `getOrder()` every 5 seconds until filled, cancelled, or 30-second timeout
- [ ] Write fill record to `trades` table after each slice: `order_id`, `size_usdc`, `avg_fill_price`, `slippage_bps`, `status`
- [ ] Publish execution event to neural trace channels after each fill
- [ ] Write unit tests: mock CLOB API → confirm slice splitting, slippage cancellation, partial fill recording

### 3.4 Live Circuit Breaker
- [ ] Create `services/broker/src/circuit_breaker/state_machine.ts`
- [ ] Implement `CircuitBreakerFSM` with states: `ACTIVE`, `HIBERNATION` per spec §14
- [ ] Implement `checkDailyPnL(realizedPnlTodayPct: number)`: if `<= -DAILY_LOSS_LIMIT_PCT` → trigger `enterHibernation()`
- [ ] Implement `enterHibernation(reason: string)`:
  1. Call `revokeWalletInMemory()`
  2. Cancel all open orders via CLOB API
  3. Publish `SHIELD` event to both neural trace channels with `trigger_reason`
  4. Write to `circuit_breaker_events` table
  5. Publish to `circuit_breaker_state` Redis channel
- [ ] Implement `dailyReset()`: called at 00:00 UTC — reset PnL counter, transition back to ACTIVE if in HIBERNATION (re-load wallet key from env), write reset event
- [ ] Schedule `dailyReset()` with a cron-like mechanism (check on every PnL update if date has changed)
- [ ] Implement additional guards per spec §14.2: Oracle Dispute, Signal Expiry, Thin Market (`MIN_BOOK_DEPTH_USDC`), Slippage Exceeded
- [ ] Write unit tests:
  - [ ] Test: PnL at -14.9% → ACTIVE, no action
  - [ ] Test: PnL at -15.0% → HIBERNATION, wallet revoked, cancel orders called
  - [ ] Test: Daily reset → PnL counter cleared, state back to ACTIVE
  - [ ] Test: Thin market → signal rejected, SHIELD event published
  - [ ] Test: Circuit breaker in HIBERNATION → all subsequent trade signals rejected

### 3.5 PnL Snapshot Job
- [ ] Add PnL snapshot writer to Broker: after every filled trade, fetch wallet balance and write `pnl_snapshots` row
- [ ] Add scheduled PnL snapshot: write snapshot every 15 minutes regardless of trades (`snapshot_source='scheduled'`)
- [ ] Publish to `pnl_updates` Redis channel after every snapshot
- [ ] Compute `realized_pnl_today`: sum of `(fill_price - entry_price) * size` for all filled trades since 00:00 UTC
- [ ] Pass `realized_pnl_today_pct` to circuit breaker check after every snapshot

### 3.6 X (Twitter) Auto-Poster
- [ ] Create `services/broker/src/publisher/twitter_poster.ts`
- [ ] Implement `postTradeAnnouncement(trade: FilledTrade)`: compose tweet from template per spec §16.3
- [ ] Implement tweet template: market title, side, entry price, edge %, position USDC, source URL, terminal link
- [ ] Implement character count guard: if tweet > 280 chars, truncate market title with ellipsis
- [ ] Wire to Broker: call `postTradeAnnouncement()` after every confirmed full or partial fill
- [ ] Add `TWITTER_ENABLED=false` env guard — allows disabling without code changes
- [ ] Write unit test: mock Twitter API → confirm tweet template construction and character limit handling

### 3.7 Hot Wallet Funding & Go-Live
- [ ] Fund `alphaclaw.eth` with $1,000 USDC on Polygon
- [ ] Confirm CLOB API can read wallet balance correctly
- [ ] Set `DRY_RUN=false` in broker `.env`
- [ ] Set `TWITTER_ENABLED=true`
- [ ] Deploy updated broker config to VPS
- [ ] Place one manual test order on a low-stakes market to confirm end-to-end execution: order creation → fill → DB record → neural trace event → X post
- [ ] Confirm circuit breaker daily reset fires correctly at 00:00 UTC (monitor for 2 consecutive nights)
- [ ] Monitor for 48 hours: check every `trades` row for correct `slippage_bps` values and compare to `MAX_SLIPPAGE_BPS`

### 3.8 Operations & Monitoring
- [ ] Set up log aggregation on VPS: configure Docker logging driver to write to `/var/log/alphaclaw/` with log rotation (max 100MB per service, 7-day retention)
- [ ] Set up uptime monitoring: use UptimeRobot or similar to alert on `https://alphaclaw.ai/health` going down
- [ ] Set up a daily cron job on VPS that emails a summary: trades today, PnL today, circuit breaker state, open positions
- [ ] Set up a weekly manual task: review `signals` drop reasons in DB, identify if any signal categories are over-filtering or under-filtering
- [ ] Document runbook: what to do if circuit breaker fires, how to manually reset, how to ban a compromised whitelist account

---

## PHASE 4 — Sovereign Expansion
*Goal: Self-improving system. Market Memory, auto-schema generation, portfolio-level Kelly, whitelist self-expansion.*

### 4.1 Market Memory
- [ ] Create `services/brain/app/memory/market_memory.py`
- [ ] Implement `compute_category_performance(category_id: str) -> dict`: query `shadow_trades` and `trades` joined by `signals.category_id`; compute win rate, avg MFE, avg MAE, avg edge ratio per category
- [ ] Implement `adjust_weights()`: based on category performance over the last 30 days, compute new `W_OPUS` and `W_ZSCORE` per category (higher-performing categories get higher effective weight)
- [ ] Store adjusted weights in a new `category_weights` TimescaleDB table: `category_id`, `w_opus`, `w_zscore`, `computed_at`, `sample_size`
- [ ] Integrate: `kelly.py` reads `category_weights` table instead of env vars when a category-specific weight exists
- [ ] Run `adjust_weights()` on a weekly cron schedule
- [ ] Write unit tests: confirm weight adjustment logic, confirm fallback to env defaults when table is empty

### 4.2 Auto-Schema Generation via Opus
- [ ] Create `services/brain/app/schemas/schema_generator.py`
- [ ] Implement `discover_new_markets()`: poll Polymarket CLOB API for markets matching active `market_tags` but not yet in `data/resolution_schemas/`
- [ ] Implement `generate_resolution_schema(market: dict) -> dict`: invoke Opus with the raw Polymarket market object and a prompt instructing it to produce a valid `resolution_schema` JSON per spec §7 structure
- [ ] Implement validation: run generated schema through the Pydantic model; if invalid → log and skip, do not deploy
- [ ] Implement human-in-the-loop gate: write generated schemas to `data/resolution_schemas/pending/` for operator review before they go live in `data/resolution_schemas/`
- [ ] Add `/admin/pending-schemas` terminal page: operator can approve or reject each pending schema with a button click
- [ ] On approval: move file from `pending/` to active directory and trigger `FastMatchEngine` schema reload

### 4.3 Portfolio-Level Kelly
- [ ] Extend `kelly.py`: implement `portfolio_kelly(open_positions: list, new_signal) -> float` that considers total current wallet exposure across all open positions
- [ ] Rule: total open exposure across all positions must never exceed 20% of wallet (4× the per-trade 5% cap)
- [ ] If adding new position would breach 20% cap → reduce new position size to fill remaining headroom, or reject if headroom < `MIN_POSITION_USDC`
- [ ] Write `open_positions` to `pnl_snapshots.open_positions` JSONB column on every snapshot
- [ ] Write unit tests: confirm 20% portfolio cap is enforced, confirm partial sizing when headroom is limited

### 4.4 Whitelist Self-Expansion
- [ ] Create `services/brain/app/whitelist/expander.py`
- [ ] Implement `identify_high_signal_handles()`: query `signals` table for source handles that produced at least 3 winning trade signals in the last 60 days (joined via `signal_id` through `trade_signals` and `trades`)
- [ ] For each candidate handle not already in `whitelist_accounts`: log to `admin_alerts` table as a `whitelist_candidate` alert with handle, win count, and category
- [ ] Add `/admin/whitelist-candidates` terminal page: operator can one-click add candidates to `whitelist_accounts`
- [ ] Run `identify_high_signal_handles()` on a weekly cron schedule
- [ ] Write unit test: mock `signals` table with known winning handles → confirm correct candidates are identified

### 4.5 Phase 4 Operational Hardening
- [ ] Add TimescaleDB continuous aggregates for performance dashboards (pre-compute hourly PnL, win rate by category)
- [ ] Add data retention policy: `signals` table auto-drops rows older than 90 days; `opus_verdicts` older than 180 days
- [ ] Add Prometheus metrics endpoint to Brain service — expose: signals_per_minute, opus_invocations_per_hour, kelly_position_sizes_histogram
- [ ] Set up Grafana dashboard on VPS: wire to Prometheus, display real-time signal flow, Opus latency, circuit breaker state, daily PnL
- [ ] Review and update Risk Register: add any new risks discovered during live Phase 3 trading

---

## CROSS-CUTTING CONCERNS
*These tasks span all phases. Schedule them deliberately rather than letting them slip.*

### Testing & Quality
- [ ] Set up `pytest-cov` for Brain service — enforce 80% minimum coverage in CI (fails build if below)
- [ ] Set up `bun test` coverage for Broker service
- [ ] Write an end-to-end smoke test script: injects a synthetic signal at the Brain HTTP debug endpoint → verifies it flows to `signals` table → (in Phase 2+) verifies a `shadow_trades` row is created
- [ ] Add a `make test-all` target to root Makefile that runs Brain, Broker, and Terminal test suites in sequence

### Documentation
- [ ] Write `docs/runbook.md`: operating procedures for starting/stopping services, handling circuit breaker trips, banning whitelist accounts, approving pending schemas
- [ ] Write `docs/calibration_log.md`: record every parameter change (Z-score threshold, Kelly weights, etc.) with date, reason, and observed effect
- [ ] Update `docs/specs.md` with any implementation divergences discovered during build — specs are a living document
- [ ] Add API comments (docstrings) to all public functions in Brain service
- [ ] Add JSDoc to all public functions in Broker service

### Security Hygiene
- [ ] Audit all log statements in all three services for accidental credential leakage before first VPS deploy
- [ ] Run `truffleHog` or `gitleaks` on the full repo history before making any repo public
- [ ] Confirm `HOT_WALLET_PRIVATE_KEY` is never written to `circuit_breaker_events`, `trades`, or any DB table
- [ ] Add a pre-commit hook that blocks commits containing strings matching `sk-ant-`, `pplx-`, `0x` (private key prefix), or hex strings > 60 chars

---

*End of AlphaClaw Implementation Tasks*
