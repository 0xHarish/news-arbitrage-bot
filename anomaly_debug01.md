# AlphaClaw Anomaly Engine — Debug Session Summary
**Date:** 2026-02-28
**Status:** BLOCKED — PM2 ignoring ANOMALY_Z_THRESHOLD setting

---

## What We're Trying To Do
Enable the Phase 3 Price Anomaly Engine to detect and log anomalies on Polymarket markets. We lowered `ANOMALY_Z_THRESHOLD` from 2.0 → 0.5 to see detections fire in real time. The engine works — we proved it — but PM2 keeps overriding the threshold back to 1.5.

## What's Working
- Anomaly engine is alive: 5 markets tracked, ~150-200 ticks/min flowing
- Z-score math is now FIXED (CC fixed a bug where new value was included in its own mean/std calculation)
- DB writes work (schema exists, connection works)
- When running **directly with uvicorn** (bypassing PM2), detections fire perfectly:
  ```
  ANOMALY_Z_THRESHOLD=0.5 python -m uvicorn app.main:app --host 0.0.0.0 --port 8000
  ```
  Result: `detections=4` → `detections=173` → `detections=999` ✅

## The Problem: PM2 Always Sets Z_THRESHOLD=1.5
No matter what we do, PM2 injects `ANOMALY_Z_THRESHOLD=1.5` into the process environment:

```
cat /proc/$(pm2 pid Brain)/environ | tr '\0' '\n' | grep Z_THRESHOLD
# Always returns: ANOMALY_Z_THRESHOLD=1.5
```

### What We've Tried (ALL FAILED to change PM2's 1.5):
1. ✅ Set `.env` to `ANOMALY_Z_THRESHOLD=0.5` — Python reads 0.5 standalone, PM2 overrides to 1.5
2. ✅ Set `ecosystem.config.js` env to `"0.5"` — PM2 still injects 1.5
3. ✅ `pm2 delete Brain` + `pm2 start ecosystem.config.js` — still 1.5
4. ✅ `pm2 kill` (stops daemon) + restart — still 1.5
5. ✅ `rm ~/.pm2/dump.pm2` — still 1.5
6. ✅ `rm -rf ~/.pm2` (full nuke of PM2 state) — still 1.5
7. ✅ Deleted all `__pycache__` and `.pyc` files — still 1.5
8. ✅ Verified `config.py` default is `0.5` on disk — still 1.5
9. ✅ Verified `ecosystem.config.js` shows `"0.5"` on disk — still 1.5
10. ✅ `python -c "from app.config import settings; print(settings.ANOMALY_Z_THRESHOLD)"` prints `0.5` — still 1.5 in PM2

### Root Cause Theory
PM2 environment variable injection overrides pydantic-settings. The `env` block in `ecosystem.config.js` is set to the process environment, but something about PM2's caching or the way it serializes/deserializes the config is reverting to 1.5. The git-tracked version of `ecosystem.config.js` has `"1.5"` — every `git restore` or `git pull` resets our local edits.

## Key Discovery: git restore resets ecosystem.config.js
When we ran `git restore services/` to recover from accidental deletion, it restored the **git-committed** version of `ecosystem.config.js` which has `"1.5"`. Our `sed` edits to change it to `"0.5"` are local-only and get overwritten.

Even after re-applying the sed edit, PM2 STILL shows 1.5 in the process env. This suggests PM2 may be reading from an internal cache that persists even after `pm2 delete`.

## Bugs Found and Fixed (by Claude Code)

### Bug 1: Z-Score Formula (FIXED)
Old code included the new value in the rolling window before computing mean/std. This dampened every anomaly signal to ~1.0. Fix: compute mean/std from previous window, then add new value.
- Commit: `27779ac fixing ZSCORE bug`

### Bug 2: Config Threshold Mismatch (PARTIALLY FIXED)
CC changed `config.py` default from 1.5 → 0.5. But PM2 overrides env vars regardless.

### Bug 3: Hidden Drop Counter (FIXED)
Added `stats_failed` to STATS log line — ticks dropped by zero-variance were previously silent.

## Other Observations
- `vol_zero=5` on all markets — the CLOB `/markets` endpoint doesn't return volume data. `ANOMALY_REQUIRE_VOLUME=False` prevents this from blocking.
- SocialData API returns 402 (Payment Required) — unrelated to anomaly engine
- The EPIPE error from `head -5` is harmless

## Accidental Deletion Incident
While copy-pasting commands, two lines merged:
```
rm -rf ~/.pm2cd /root/alphaclaw/services/brain
```
This deleted the entire brain directory. Fixed with `git restore services/`. The `.env` file (gitignored) was lost and had to be recreated with API keys.

## Files on VPS
- **Config:** `/root/alphaclaw/services/brain/app/config.py` — default 0.5
- **Ecosystem:** `/root/alphaclaw/services/brain/ecosystem.config.js` — needs to be 0.5 (resets on git pull)
- **Env:** `/root/alphaclaw/services/brain/.env` — has `ANOMALY_Z_THRESHOLD=0.5`
- **Engine:** `/root/alphaclaw/services/brain/app/anomaly/engine.py` — z-score logic fixed
- **Venv:** `/root/alphaclaw/services/brain/venv/` — freshly rebuilt

## Next Steps to Try
1. **Commit the ecosystem.config.js change to git** so it survives pulls/restores
2. **Try a completely different approach:** Instead of fighting PM2 env vars, hardcode the threshold temporarily in engine.py as `threshold = 0.5` to bypass config entirely — just to get detections working in PM2
3. **Or bypass PM2:** Run Brain directly with systemd or a simple bash script + nohup:
   ```
   cd /root/alphaclaw/services/brain
   source venv/bin/activate
   nohup python -m uvicorn app.main:app --host 0.0.0.0 --port 8000 > brain.log 2>&1 &
   ```
4. **Investigate PM2 deeper:** Check if PM2 has a global config or another ecosystem file being loaded. Try `pm2 conf` and check for stored env vars.

## Quick Test Command (proven to work)
```
cd /root/alphaclaw/services/brain
source venv/bin/activate
ANOMALY_Z_THRESHOLD=0.5 python -m uvicorn app.main:app --host 0.0.0.0 --port 8000
```
This WORKS and produces detections. The issue is exclusively PM2.