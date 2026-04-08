# James's Backlog

Prioritized list of what's on James's plate for rekko.ai. Updated by `/rekko-end-session`.

## Active

- **Validate Phase 1.5 B-roll quality** — eyeball 5–10 of the 59 downloaded clips in `data/viral_broll/general_viral/`. Decision gate: if usable → clip-segment selection + Remotion integration next; if not → LLM entity extraction to fix query quality
- **Test targeted `fetch` flow end-to-end** — run `viral_broll fetch --market-key kalshi:<id>` against a real Kalshi market, verify per-market subdir layout and real ScrapeCreators + yt-dlp integration
- **Phase 1.5 next session** — clip-segment selection (scene detection + LLM picker) and Remotion integration for stitched-clip B-roll, OR LLM entity extraction — decided by eyeball outcome above

## Parked (P2)

- **Viral discovery: scheduler loop already shipped by Daniel** (`trend_discovery_loop` wired in `05e1c91`)
- **Viral discovery: strategy agent feed already shipped by Daniel** (viral signals injected into daily research ranking in `4c1c180`)
- Safe bet labeling strategy — define labeling approach for safe/risky bet classification
- Whale watching as content trigger
- Automated prediction market index fund
- ML models (category trading rules, price prediction, anomaly detection) — paused per video engine pivot

## Done

- **Phase 1.5: Viral B-roll discovery + download** — new `viral_broll` CLI with `fetch` and `backfill` subcommands, yt-dlp integration, English filter (Unicode script ranges, title-only), storage layout (`general_viral/` vs `{market_key_slug}/`), idempotent ALTER TABLE adding download fields to `viral_videos`, thin `viral_broll_candidates` association table, 55 new tests. 59 English viral clips (166 MB) backfilled from Daniel's trending feed. Branch: `james/viral-discovery`
- **Viral discovery Phase 1 (code + live validation + scheduler wiring)** — 4-source scraper system (ScrapeCreators, Google Trends, YouTube Shorts, Reddit), DB tables, scheduler loop wired every 4h. Merged to main
- **Session skills + agents** — rekko-start-session, rekko-end-session skills; rekko-karen, rekko-code-planner agents; backlog + session log infrastructure
