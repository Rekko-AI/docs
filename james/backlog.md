# James's Backlog

Prioritized list of what's on James's plate for rekko.ai. Updated by `/rekko-end-session`.

## Active

- **Test viral discovery scrapers against real APIs** — `uv sync --extra trends`, configure SCRAPECREATORS_API_KEY + REDDIT_CLIENT_ID/SECRET, run `viral_discovery` CLI, validate DB output
- **Viral discovery: scheduler loop** — wire `trend_discovery_loop` into scheduler once scrapers validated
- **Viral discovery: strategy agent feed** — inject trending signals into `tiktok_strategy.md` at runtime

## Parked (P2)

- Safe bet labeling strategy — define labeling approach for safe/risky bet classification
- Whale watching as content trigger
- Automated prediction market index fund
- ML models (category trading rules, price prediction, anomaly detection) — paused per video engine pivot

## Done

- **Viral video discovery Phase 1 (code)** — 4-source scraper system (ScrapeCreators, Google Trends, YouTube Shorts, Reddit), DB tables, CRUD, orchestrator CLI, 22 tests passing. Branch: `james/viral-discovery`
- **Session skills + agents** — rekko-start-session, rekko-end-session skills; rekko-karen, rekko-code-planner agents; backlog + session log infrastructure