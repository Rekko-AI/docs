# James's Backlog

Prioritized list of what's on James's plate for rekko.ai. Updated by `/rekko-end-session`.

## Active

- **End-to-end viral B-roll render validation** — run a full pipeline on a fresh analysis and eyeball the rendered video. Verify viral clips composite cleanly into each section and pacing doesn't break. Disable via `VIDEO_VIRAL_BROLL_ENABLED=false` if it looks off.
  ```bash
  uv run python -m rekko_server analyze --bet "<market>" --platform polymarket
  uv run python -m rekko_server content --run-id <id>
  uv run python -m rekko_server.video --run-id <id>
  ```
- **Top up ScrapeCreators credits + re-run discovery** — unblock TikTok path (currently 402). Aim for a second viral pool fill-up with both sources.
- **Celebrity-skew keyword iteration** — `GENERAL_VIRAL_KEYWORDS` currently skews general-hype. Add celebrity/talent terms (e.g., "celebrity reaction", specific athlete/artist names) once the engine is proven to use the existing pool.

## Paused / dormant

- **Market-targeted B-roll fetch** — `fetch_broll_for_market()`, `viral_broll_candidates` table, `fetch --market-key` CLI. Code stays in place; not exercised. May re-enter scope as a ranking signal within the pool, not as a search strategy.
- **LLM entity extraction for B-roll search queries** — shelved 2026-04-13. No market titles to extract from under the general-pool approach. Do not revisit without re-opening the per-market architecture question first.
- **Daniel's `viral_mapper.py` (viral→market semantic matcher)** — untouched, but output is no longer consumed by anything James owns. Daniel's call whether to keep running it.

## Parked (P2)

- **Viral discovery: scheduler loop already shipped by Daniel** (`trend_discovery_loop` wired in `05e1c91`)
- **Viral discovery: strategy agent feed already shipped by Daniel** (viral signals injected into daily research ranking in `4c1c180`)
- Safe bet labeling strategy — define labeling approach for safe/risky bet classification
- Whale watching as content trigger
- Automated prediction market index fund
- ML models (category trading rules, price prediction, anomaly detection) — paused per video engine pivot

## Done

- **Phase 1.6 Step 7: viral pool wired into Remotion B-roll** (2026-04-17) — eyeball gate passed (17 usable clips). New `video/viral_broll_picker.py` with random-sample + dedup, Phase 1.5 in `video/visuals.py` that copies picks as `broll_{section}_viral_{k}.mp4` (existing `_build_render_props` glob picks them up, zero Remotion/TS changes). Config: `viral_broll_enabled=True`, `_per_section=1`, `_sections=[hook,context,evidence,reveal,cta]`. 6 new tests in `test_viral_broll_picker.py` (67 viral tests green). Plan doc at `docs/james/viral_broll_remotion_wiring.md`.
- **YouTube lookback fix** (2026-04-17) — plumbed `settings.viral_broll_lookback_days` through `_search_youtube` → `search_viral_shorts(published_after=...)`. Re-ran discovery at 30-day lookback: 13 new YouTube survivors downloaded (vs 1 at 7-day).
- **Tighter English filter** (2026-04-17) — `_is_likely_english` now rejects on ANY non-Latin script char (was 0.7 Latin-script ratio, which let single-char Devanagari leak through). Retroactively cleaned 1 non-English clip from disk.
- **Phase 1.6 Steps 1–6: Hard-filtered general viral pool** (2026-04-16) — `GENERAL_VIRAL_KEYWORDS` constant (16 terms, 5 buckets), `viral_broll_min_peak_views=5_000_000` config, `purge-below-floor` CLI subcommand with `--dry-run`, 5M floor enforced at 3 ingest paths (TikTok `discover_viral_videos`, YouTube `scrape_youtube_shorts`, `backfill_pending_downloads`), `discover_general_viral_pool()` keyword loop + discovery loop rewired, 5 new Phase 1.6 tests (60 total in `test_viral_broll.py`). Live purge executed: 55 clips purged, 4 survivors.
- **Validate Phase 1.5 B-roll quality** — eyeballed clips in `general_viral/`. Verdict: still non-English, low quality, no viral payoff. Decision: LLM entity extraction path, not clip segmentation.
- **Phase 1.5: Viral B-roll discovery + download** — new `viral_broll` CLI with `fetch` and `backfill` subcommands, yt-dlp integration, English filter (Unicode script ranges, title-only), storage layout (`general_viral/` vs `{market_key_slug}/`), idempotent ALTER TABLE adding download fields to `viral_videos`, thin `viral_broll_candidates` association table, 55 new tests. 59 English viral clips (166 MB) backfilled from Daniel's trending feed. Branch: `james/viral-discovery`
- **Viral discovery Phase 1 (code + live validation + scheduler wiring)** — 4-source scraper system (ScrapeCreators, Google Trends, YouTube Shorts, Reddit), DB tables, scheduler loop wired every 4h. Merged to main
- **Session skills + agents** — rekko-start-session, rekko-end-session skills; rekko-karen, rekko-code-planner agents; backlog + session log infrastructure
