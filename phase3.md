# AlphaClaw Phase 3 Handoff — Resolution Schema Buildout

## Date: Feb 27, 2026 (updated Feb 26, 2026)

---

## PROJECT OVERVIEW

AlphaClaw is a Polymarket shadow trading bot running on a VPS. It ingests real-time news signals, matches them to active Polymarket markets, runs them through an LLM (Opus) for verdict, applies Kelly sizing, and executes shadow trades against the CLOB API.

**Architecture**: Ingestion Pollers → Brain (FastMatch → Opus → Kelly) → Broker (Shadow Executor → CLOB API)

**Current State**: Pipeline is fully validated end-to-end. Shadow trades are being placed successfully. But **zero organic FastMatch hits** — all 1,016+ signals drop with `no_resolution_signal_match` because resolution schemas lack the matching criteria needed by FastMatch.

---

## WHAT'S BEEN COMPLETED

### Phase 2 — Shadow Pipeline Validation ✅

1. **Real Polymarket IDs obtained** for 5 active markets via Gamma API
2. **Broker fixed** — now resolves `condition_id → CLOB token_id` from schema files before calling `/midpoint` API
3. **DB password fixed** — Brain was using `pass`, docker/Broker use `password`
4. **Shadow trades confirmed** — 2 test shadow trades successfully written to DB
5. **6 dead schemas removed** (BTC ETF, Stripe IPO, Super Bowl, CA-45, TX Senate, ETH Staking)

### Polymarket WebSocket Fix ✅ (Feb 26, 2026)

6. **WS subscribe message fixed** — was sending per-market `{"type":"subscribe","channel":"price_change","market":"<condition_id>"}` (made-up schema), now sends single `{"assets_ids": [...], "type": "market"}` matching actual Polymarket API
7. **Wrong IDs fixed** — was subscribing with `condition_id` hex strings, now uses CLOB token IDs (`clob_tokens.YES`/`clob_tokens.NO`) from resolution schemas
8. **Message handler updated for Sep 2025 schema change** — `price_change` events now contain `price_changes[]` array with `best_bid`/`best_ask` per asset (midpoint computed); `last_trade_price` events use `price` field directly
9. **Asset-to-market reverse lookup added** — `_asset_to_market` dict maps incoming `asset_id` → `condition_id` so Z-score tracking keys off the correct market ID
10. **`INVALID OPERATION` spam eliminated** — root cause was the invalid subscribe format

---

## 5 ACTIVE MARKETS WITH REAL IDS

### 1. FOMC March 2026 — "No change"
- **Schema file**: `/root/alphaclaw/data/resolution_schemas/fed_march2026.json`
- **category_id**: `macro_fed`
- **condition_id**: `0x257b18205f908aef01ef2d1d50e6fea7d29cf5486fe04031d8b101394476faed`
- **CLOB tokens**: YES=`102559817034631022221500208641784929295731053857601013029449249654006364919935`, NO=`114350083305515876605975857295292847572608542141507135338338719165921048222327`
- **Polymarket slug**: `fed-decision-in-march-885`
- **Current odds**: ~97% YES
- **Status**: `threshold_conditions: [], resolution_signals: []` ← NEEDS POPULATING

### 2. CPI February 2026 Monthly — "0.3%"
- **Schema file**: `/root/alphaclaw/data/resolution_schemas/cpi_feb2026.json`
- **category_id**: `macro_fed`
- **condition_id**: `0xc1691f094b3dfb3bd39ab6b9d1c41792a2421aa3438f23b1aae64b7cbbb601d7`
- **CLOB tokens**: YES=`16244226514787692625509418115934807606115354849618134222354736093244725545119`, NO=`106783299744166537103315878820299597893069284711704516333073781309561641610970`
- **Polymarket slug**: `february-inflation-us-monthly-231`
- **Resolves**: March 11, 2026
- **Current odds**: ~46.5% YES

### 3. NBA Finals 2026 — Celtics
- **Schema file**: `/root/alphaclaw/data/resolution_schemas/nba_finals_2026.json`
- **category_id**: `sports_championship`
- **condition_id**: `0x13e1cc703a120883a709d3bb29707097952845d61ca6793ea2b4328bae2a7451`
- **CLOB tokens**: YES=`98951343420969493497594761179562691809954416596888138302255086663562042936451`, NO=`68988781087071608596579516085819336301201525599500826069950065091990393143710`
- **Polymarket slug**: `2026-nba-champion`
- **Current odds**: ~6.3% YES

### 4. Ukraine Ceasefire Q1 2026
- **Schema file**: `/root/alphaclaw/data/resolution_schemas/ukraine_ceasefire.json`
- **category_id**: `geopolitical_ceasefire`
- **condition_id**: `0x1d54eb5eac2cee8f595f3097c65da7d07f8ab5dee63d7c0c6883eb70e1e9af30`
- **CLOB tokens**: YES=`24394670903706558879845790079760859552309100903651562795058188175118941818512`, NO=`82991837887127223585838300135823820957946688171642633953055760547186260528157`
- **Polymarket slug**: `russia-x-ukraine-ceasefire-by-march-31-2026`
- **Current odds**: ~2.75% YES

### 5. Ethereum Flipped 2026
- **Schema file**: `/root/alphaclaw/data/resolution_schemas/eth_flipped_2026.json`
- **category_id**: `crypto_regulatory` (needs new category or reassignment)
- **condition_id**: `0xcbca4926398df4622033099a661c591c5d5b62f3ac11e1dc871c2a7592c74e32`
- **CLOB tokens**: YES=`5696256002830425095773350415970497521807751621606677935878961079946544685641`, NO=`16908800275040414334648923932553225820855394547396165657067073625824818623157`
- **Polymarket slug**: `eth-flipped-in-2026`
- **Current odds**: ~55.5% YES

---

## THE CORE PROBLEM TO SOLVE (Phase 3)

**FastMatch is dropping ALL signals** because schema files have empty `threshold_conditions` and `resolution_signals` arrays, AND the narrative dict categories have resolution_signals that require ALL canonical_tokens to be present — which is too strict for real-world news.

### How FastMatch Works (from `/root/alphaclaw/services/brain/app/fastmatch/engine.py`):

1. **Normalize** raw text → canonical tokens (spaCy lemmatization, stopword removal)
2. **Match category** — tokens overlap with `market_tags` in narrative_dict.json
3. **Match market** — tokens overlap with `key_entities` in resolution schema
4. **Match resolution_signals** — For each signal in the category:
   - ALL `canonical_tokens` must be present in signal tokens
   - At least ONE `regex_pattern` must match raw text
   - Accumulates weighted score
5. **Escalation** — if score ≥ 0.60 (`FAST_MATCH_ESCALATION_THRESHOLD`), escalate to Opus

### The Problem Illustrated:

Reuters signal: `"BREAKING: Federal Reserve holds rates unchanged at FOMC meeting, maint..."`
Normalized tokens: `["federal", "reserve", "hold", "rate", "unchanged", "fomc", "meeti..."]`

The `macro_fed` category's `rs_fed_official` resolution signal requires ALL of these canonical_tokens:
```
["fomc", "rate", "raise", "cut", "hold", "unchanged", "basis", "point", "federal", "reserve"]
```

The signal has most but NOT "basis", "point", "raise", "cut" — so it fails the ALL-tokens check and gets score 0.0.

### What Needs to Happen:

1. **Relax resolution_signals** in narrative_dict.json — reduce required canonical_tokens to a realistic minimum set (2-4 tokens that MUST appear together)
2. **Add more resolution_signal variants** — separate signals for "rate hold", "rate cut", "rate raise" etc.
3. **Populate schema `key_entities`** — so FastMatch can match to specific markets within a category
4. **Populate schema `resolution_signals`** — market-specific resolution criteria
5. **Add new categories** for ETH flipped (currently no matching category — `crypto_regulatory` doesn't fit)
6. **Consider adding CPI-specific resolution signals** to `macro_fed` category

---

## KEY FILE LOCATIONS

| File | Path |
|------|------|
| Narrative Dict | `/root/alphaclaw/data/narrative_dict.json` |
| Resolution Schemas | `/root/alphaclaw/data/resolution_schemas/*.json` |
| FastMatch Engine | `/root/alphaclaw/services/brain/app/fastmatch/engine.py` |
| Normalizer | `/root/alphaclaw/services/brain/app/fastmatch/normalizer.py` |
| Z-Score | `/root/alphaclaw/services/brain/app/fastmatch/zscore.py` |
| Brain Config | `/root/alphaclaw/services/brain/app/config.py` |
| Brain .env | `/root/alphaclaw/services/brain/.env` |
| Broker Shadow Executor | `/root/alphaclaw/services/broker/src/shadow/shadow_executor.ts` |
| Broker .env | `/root/alphaclaw/services/broker/.env` |
| Docker Compose | `/root/alphaclaw/infra/docker-compose.yml` |
| Opus Client | `/root/alphaclaw/services/brain/app/appeals/opus_client.py` |
| Prompt Builder | `/root/alphaclaw/services/brain/app/appeals/prompt_builder.py` |
| Social Poller | `/root/alphaclaw/services/brain/app/ingestion/social_poller.py` |
| HN Poller | `/root/alphaclaw/services/brain/app/ingestion/hn_poller.py` |
| Polymarket WS | `/root/alphaclaw/services/brain/app/ingestion/polymarket_ws.py` |

---

## DB CONFIG

- **Docker postgres password**: `password`
- **Broker .env**: `postgresql://alphaclaw:password@localhost:5432/alphaclaw` ✅
- **Brain .env**: `DATABASE_URL=postgresql://alphaclaw:password@localhost:5432/alphaclaw` ✅ (was `pass`, fixed)

---

## DB SCHEMA (signals table)

```
id                        uuid
created_at                timestamp with time zone
source_tier               smallint (1=polymarket, 2=social, 3=hn, 4=perplexity)
source_handle             text
raw_text                  text
canonical_tokens          jsonb
market_id                 text
zscore                    double precision
fast_match_hit            boolean
fast_match_score          double precision
escalated_to_opus         boolean
drop_reason               text
```

---

## PROCESS MANAGEMENT

```bash
pm2 list                    # Show all services
pm2 restart Brain           # Restart Brain after schema changes
pm2 restart Broker          # Restart Broker
pm2 logs Brain --lines 50   # Watch Brain logs
pm2 logs Broker --lines 50  # Watch Broker logs
```

---

## SIGNAL FLOW STATS (as of Feb 27 04:00 UTC)

- **Signals ingested**: 1,016+ (643 in last 6hrs)
- **Sources**: @Reuters, @WSJ, @AFP, @NBCNews, @FoxNews, @CBSNews, @ABC, @Cointelegraph
- **FastMatch hits**: 0 (all drop at `no_resolution_signal_match`)
- **Opus verdicts**: 11 (all from test signals)
- **Trade signals**: 2 (test only)
- **Shadow trades**: 2 (test only)

---

## SAMPLE REAL SIGNALS (for calibrating resolution_signals)

```
@Reuters  | "BREAKING: Federal Reserve holds rates unchanged at FOMC meeting, maint..."
tokens: ["federal", "reserve", "hold", "rate", "unchanged", "fomc", "meeting"]

@Reuters  | "Nvidia shares fall as investors fret over returns, look past strong re..."
tokens: ["nvidia", "share", "fall", "investor", "fret", "return", "look", "strong"]
```

---

## HEALTH CHECK QUERY

```bash
cd /root/alphaclaw/services/brain && source venv/bin/activate
python3 -c "
import asyncio, asyncpg
async def check():
    pool = await asyncpg.create_pool('postgresql://alphaclaw:password@localhost:5432/alphaclaw')
    total = await pool.fetchval('SELECT COUNT(*) FROM signals')
    recent = await pool.fetchval(\"SELECT COUNT(*) FROM signals WHERE created_at > NOW() - INTERVAL '1 hour'\")
    hits = await pool.fetchval(\"SELECT COUNT(*) FROM signals WHERE fast_match_hit = true AND created_at > NOW() - INTERVAL '1 hour'\")
    verdicts = await pool.fetchval('SELECT COUNT(*) FROM opus_verdicts')
    shadows = await pool.fetchval('SELECT COUNT(*) FROM shadow_trades')
    print(f'Signals: {total} total, {recent} last hr, {hits} FM hits | Verdicts: {verdicts} | Shadows: {shadows}')
    await pool.close()
asyncio.run(check())
"
```

---

## NEXT STEPS (in order)

1. ~~**Fix Polymarket WS subscribe format**~~ ✅ DONE
2. **Deploy WS fix** — `pm2 restart Brain` and verify price ticks flow (`pm2 logs Brain --lines 50`)
3. **Rewrite narrative_dict.json resolution_signals** — relax token requirements, add variants
4. **Add new categories** — `crypto_market_cap` for ETH flipped, `macro_cpi` for CPI data
5. **Populate schema key_entities** — so markets match within categories
6. **Test with real signal replay** — replay the Reuters FOMC signal through updated FastMatch
7. **Monitor organic hits** for 1-2 hours
8. **Fund wallet** ($50-100) and configure CLOB credentials
9. **Flip DRY_RUN=false** with tight guardrails

---

## PREVIOUS TRANSCRIPTS

- `/mnt/transcripts/2026-02-26-23-59-44-alphaclaw-phase2-pipeline-validation.txt`
- `/mnt/transcripts/2026-02-27-04-14-28-polymarket-id-mapping-phase2.txt`
- `/mnt/transcripts/journal.txt` (catalog)