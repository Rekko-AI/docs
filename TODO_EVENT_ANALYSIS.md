# Event Analysis — Remaining Work

Shipped in PRs #347, #355, #356 (April 2026). This document tracks follow-up work for the event-based analysis system.

## Shipped

- [x] Events schema enrichment (mutually_exclusive, description, end_date, status)
- [x] Event aggregate refresh in feed worker
- [x] Scraper capture of new event fields (Kalshi + Polymarket)
- [x] Event analysis pipeline (Research → Synthesize → Analyze → Summarize)
- [x] Feed API: analyze, analysis, probability-map, correlation endpoints
- [x] 4 MCP tools (analyze_event, get_event_analysis, get_event_probability_map, get_event_correlation)
- [x] Event TikTok content generation (3-stage: strategy → scripts → direction)
- [x] Event X threads (EVENT_THREAD content type, 5-7 tweet format)
- [x] Scheduler auto-discovery + scoring + production of event videos
- [x] CLI: analyze-event, event-content subcommands
- [x] Neon backfill (3,875 events with new fields)

## Content & Publishing

- [ ] **TruthSocial event posts** — Add `post_truth_thread_for_event()` to `publishing/truth_social_thread.py` with new prompt `prompts/truth_post_event.md`. Follows existing TruthSocial pattern (own agent, 500-char limit, not reusing X pipeline).

- [ ] **Remotion EventOutcomeCards component** — `remotion-video/src/EventOutcomeCards.tsx` rendering animated probability comparison bars per outcome from `outcome_cards` data in `event_tiktok.json`. Spring-in animation, brand green for positive edge, red for negative. Currently the outcomes section renders as flat voiceover + B-roll.

## Feed API Endpoints

- [ ] **`GET /events/{slug}/signals`** (STRATEGY tier) — Event-level trading signals. Per-market signals within the event plus an event-level composite signal. Parallel to the existing `/markets/{platform}/{market_id}/signals` endpoint.

- [ ] **`POST /events/screen`** (STRATEGY tier) — Batch screen events for analysis worthiness. Input: `{"criteria": "high_volume", "min_markets": 3, "limit": 10}`. Output: scored event list with `score`, `reason`, `recommended_action`. Reuse `scoring.py` pattern.

- [ ] **`GET /events/{slug}/arbitrage`** (DEEP tier) — Cross-platform event arbitrage. For events that exist on both Kalshi and Polymarket (via `event_links`), return per-market price spreads across platforms.

- [ ] **Enrich `GET /events` listing** — Add query params: `min_volume`, `min_markets`, `status` filter. Add expansions to `GET /events/{slug}`: `?expand=markets,analysis` (include latest analysis), `?expand=markets,correlation`.

## TUI / Bridge Integration

- [ ] **`/analyze-event <slug>` TUI command** — Register in `rekko-tui/src/commands.ts`, add `analyze_event` command to `BridgeServer._dispatch()` in `_bridge.py`, event pipeline nodes report with `event_*` prefixed stage names via existing reporter protocol.

- [ ] **Event detail view in TUI** — Show event analysis results with per-market breakdown table, correlations, and probability sum validation in the Ink TUI.

## Scheduler Enhancements

- [ ] **Event X thread auto-generation** — In `_post_production()`, call `generate_and_notify_event_thread()` for event runs (currently only single-market runs get X threads).

- [ ] **Event shift detection** — Extend the shift detector to monitor aggregate event metrics (total volume spikes, top_yes_price changes) and re-enqueue events for updated analysis.

## MCP Tools

- [ ] Mirror the remaining Feed API endpoints as MCP tools: `screen_events`, `get_event_signals`, `get_event_arbitrage`.
