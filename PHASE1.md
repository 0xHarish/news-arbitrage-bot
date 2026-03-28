
  Phase 1: The Scraper & Schema is now fully built. Here's what was delivered:

  Files Created (40+ files)

  Infrastructure (Phase 0)
  - .gitignore, .gitattributes — repo hygiene
  - services/brain/.env.example, services/broker/.env.example, services/terminal/.env.example
  - .github/workflows/ci.yml, .github/workflows/deploy.yml — CI/CD
  - infra/docker-compose.yml — TimescaleDB + Redis
  - infra/timescaledb/init.sql — 10 tables, hypertables, indexes, triggers
  - infra/redis/redis.conf — minimal production config
  - db/seed_whitelist.sql — 23 accounts across 6 categories

  Data Layer
  - data/narrative_dict.json — 6 seed categories with 25 resolution signals
  - data/resolution_schemas/ — 10 active market schemas
  - scripts/list_schemas.py — validation + summary script

  Brain Service (Python/FastAPI)
  - app/main.py — Full wiring: lifespan, 4 ingestion tiers, FastMatch, health + metrics endpoints
  - app/config.py — 20+ env vars via pydantic-settings
  - app/db.py — asyncpg connection pool
  - app/redis_client.py — async Redis singleton
  - app/fastmatch/normalizer.py — 7-step NLP pipeline (14 tests)
  - app/fastmatch/zscore.py — Rolling Z-score with deque (8 tests)
  - app/fastmatch/engine.py — 8-step Fast-Match pipeline (7 tests)
  - app/whitelist/exporter.py — Atomic export + WhitelistCache (12 tests)
  - app/ingestion/polymarket_ws.py — Tier 1 WebSocket with auto-reconnect
  - app/ingestion/social_poller.py — Tier 2 SocialData.tools poller
  - app/ingestion/hn_poller.py — Tier 3 HN API poller
  - app/ingestion/perplexity_client.py — Tier 4 on-demand fact verification
  - tests/ — 41 unit tests across 4 test files + conftest.py