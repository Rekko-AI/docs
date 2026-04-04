# Rekko — James's Reference

Current focus: AI video engine — building a system that produces high-quality trending AI videos to farm engagement and build an audience. ML work is paused. Dan owns: TUI, video pipeline, scrapers, publishing, NDJSON bridge. Dan has an "exact recipe" for what makes videos succeed — that recipe drives technical decisions.

---

## Setup

```bash
uv sync --extra ml                # pandas, scikit-learn, jupyter, plotly, seaborn
uv run jupyter notebook           # launch notebooks
```

No API keys needed for data work — Kalshi's candlestick API is public.

---

## Populating the DB

DB is `data/rekko.db` (gitignored). Build your own local copy:

```bash
# Primary training dataset: resolved markets with full lifetime hourly ticks + known outcomes
uv run python scripts/backfill_resolved.py --days 30                              # ~500 markets, ~200k ticks, ~30 MB, ~5 min
uv run python scripts/backfill_resolved.py --days 90 --max-markets 2000           # ~2k markets, ~800k ticks, ~115 MB, ~20 min
uv run python scripts/backfill_resolved.py --days 90 --min-volume 500 --max-markets 5000  # ~5k markets, ~2.5M ticks, ~360 MB, ~1 hr

# Useful flags:
#   --dry-run          List markets without fetching
#   --force            Re-fetch markets already in DB
#   --interval 60      Candle size: 1, 60, or 1440 min (default: 60)

# Quick test data: 10 curated active markets
uv run python scripts/seed_ticks.py --days 30

# Verify
uv run python -c "from rekko_server.data import load_ticks; df = load_ticks(); print(f'{len(df)} ticks across {df.market_id.nunique()} markets')"
```

Script is **resumable** — re-run with higher limits to add more without re-fetching.

---

## Data Access API

```python
from rekko_server.data import (
    load_markets,        # All scraped markets (id, platform, title, yes_price, volume_24h, total_score, decision)
    load_ticks,          # Hourly ticks (market_id, platform, title, captured_at, yes_price, yes_bid, yes_ask, volume_24h, open_interest)
    load_scores,         # 6-dimension rubric scores (audience_pull, data_availability, time_to_resolution, resolution_clarity, mispricing_likelihood, tradability — each 0-2)
    load_runs,           # Pipeline results (recommendation, confidence, probability, category)
    load_shadow_trades,  # Paper trades (recommendation, pnl_usd, resolved, won)
    load_price_history,  # Daily YES price snapshots
    connect,             # Raw SQLite connection for custom queries
)

# Filter ticks
ticks = load_ticks(market_id=42, since="2026-02-26T14:00:00+00:00")
```

### Safe bet CLI

```bash
# Labeling & inspection
uv run python -m rekko_server.ml.safe_bet label                # label all resolved markets
uv run python -m rekko_server.ml.safe_bet show 505 --plot      # inspect one market + 4-panel PNG
uv run python -m rekko_server.ml.safe_bet list --safe           # list safe bets
uv run python -m rekko_server.ml.safe_bet data 505              # raw tick data + metadata
uv run python -m rekko_server.ml.safe_bet validate --plot       # sanity checks + calibration PNG

# Training
uv run python -m rekko_server.ml.safe_bet train                 # train P(favorite_won) classifier
uv run python -m rekko_server.ml.safe_bet train --test-months 2 # hold out last 2 months
uv run python scripts/train_safe_bet.py                         # standalone training script

# Data collection (Step 5)
uv run python scripts/link_runs_to_markets.py                   # retroactive fuzzy-match linking
uv run python scripts/link_runs_to_markets.py --dry-run         # preview matches
uv run python scripts/batch_analyze_resolved.py --dry-run       # see what would be analyzed
uv run python scripts/batch_analyze_resolved.py --categories tennis,ncaa_basketball --max-markets 50 --cheap  # ~$1
uv run python scripts/batch_analyze_resolved.py --categories tennis,ncaa_basketball --max-markets 200         # ~$70
```

Key imports:

```python
from rekko_server.ml.safe_bet import (
    load_resolved_markets,  # DataFrame: id, title, result, open_time, settlement_ts, category
    label_safe_bet,         # Returns dict: is_safe_bet, strength, eval_price, failure_reason, ...
    build_training_data,    # Feature matrix + favorite_won labels for heavy favorites
    train_model,            # Train LightGBM classifier, print metrics, save model
    extract_features,       # 14 ML features at eval point (no leakage)
)
```

---

## Notebooks

| Notebook | Purpose |
|---|---|
| `00_explore_data.ipynb` | Overview of all tables: markets, runs, shadow trades, scores, price history |
| `01_market_scoring.ipynb` | Rubric calibration: score distributions, dimension correlations, volume vs score |
| `02_shadow_trade_analysis.ipynb` | Shadow trade P&L, confidence calibration, baseline ranker evaluation |
| `03_market_ticks.ipynb` | Tick data exploration: coverage per market, resolved market lifetime trajectories, calibration curves, ML feature extraction |
| `04_safe_bet_labeling.ipynb` | Safe bet labeling heuristic (Steps 2–5 of `safe_bet_classifier.md`): `label_safe_bet()` + `label_safe_bet_strength()`, applied to 495 resolved markets with validation plots |

---

## Key Interfaces

### MarketRanker (`src/rekko_server/ml/ranker.py`)

```python
class MarketRanker(Protocol):
    def fit(self, markets: pd.DataFrame, outcomes: pd.DataFrame) -> None: ...
    def score(self, markets: pd.DataFrame) -> pd.Series: ...
    def rank(self, markets: pd.DataFrame, top_n: int = 20) -> pd.DataFrame: ...
```

Baseline: `BaselineRanker` (just uses `total_score` column). Beat it.

### Safe Bet Classifier (`src/rekko_server/ml/safe_bet.py`)

**Labeling heuristic** (4-criterion: directional price >= 0.85, outcome match, stability, volume >= 100):
```python
def label_safe_bet(ticks, outcome, open_time, settle_time) -> dict:  # is_safe_bet, strength, eval_price, ...
def extract_features(ticks, eval_time) -> dict:  # 14 features, no post-eval data
```

**P(favorite_won) classifier** — predicts whether heavy favorites (>= 80%) actually win:
```python
def build_training_data(conn) -> pd.DataFrame:  # feature matrix + favorite_won labels
def train_model(conn, test_months=1, output_path=None) -> dict:  # LightGBM + eval metrics
```

### Existing Scoring System (`src/rekko_server/agents/scoring.py`)

6-dimension rubric scored 0-2 each by Claude Sonnet: audience_pull, data_availability, time_to_resolution, resolution_clarity, mispricing_likelihood, tradability. Total ≥ threshold → COVER, else SKIP.

---

## Key Files

- `src/rekko_server/data.py` — data access API
- `src/rekko_server/ml/safe_bet.py` — safe bet labeling, feature extraction, CLI tool
- `src/rekko_server/ml/ranker.py` — MarketRanker protocol + BaselineRanker
- `src/rekko_server/db/schema.py` — SQLite schema (markets, market_ticks, scrapes, market_scores, trades)
- `src/rekko_server/models/scoring.py` — rubric schema
- `src/rekko_server/models/shadow.py` — shadow trade model
- `scripts/backfill_resolved.py` — resolved market backfill (now stores category)
- `scripts/train_safe_bet.py` — standalone classifier training script
- `scripts/seed_ticks.py` — quick test data seeder
- `scripts/link_runs_to_markets.py` — retroactive fuzzy-match linking of runs to markets
- `scripts/batch_analyze_resolved.py` — batch analysis of resolved markets for ML training data
- `src/rekko_server/watchlist/` — prospective market watchlist (daily research on open markets for clean training data)
- `docs/ml/safe_bet_classifier.md` — labeling strategy spec + training plan + results
- `docs/ml/modeling_journal.md` — experiment log with results and next steps

## Rules from Dan

- All data models: Pydantic v2 with `description=` on every field
- ML code must not import from `tui/`, `rekko-tui/`, or `_bridge.py`
- ML code may import from `rekko_server.data`, `rekko_server.models`, `rekko_server.db`, `rekko_server.config`

## Current Status (2026-04-03)

### Pivot: AI Video Engine (decided 2026-04-03)

James and Daniel agreed to pivot Rekko toward an **AI video engine** that produces high-quality trending AI videos to build an audience. **Traction requires quality, not volume.** One viral video >> fifty mediocre ones. Monetization comes later — audience first.

**P1:** AI video engine. Identify trending topics/formats, generate scripts, produce AI videos, publish — optimized for virality and engagement quality.

**P2 (parked):** Whale watching as content, automated prediction market index fund, peptides e-commerce (legality TBD).

**All ML work paused:** Category trading rules, price prediction, anomaly detection, content ranking — all on hold. Too far from traction.

### What exists (video pipeline)
- Full 4-phase pipeline: Analysis → Content (3 LLM agents) → Video (Remotion) → Publishing
- 551 completed analyses, 0 finished videos (content generation was never auto-chained)
- Remotion renderer: React/TypeScript, 1080x1920 @ 30fps, karaoke subtitles, data viz, safe zones
- TTS: ElevenLabs (cloud, $0.30/min) or Kokoro (local, free)
- B-roll: still images via OpenAI GPT Image ($0.07/img), Google Imagen ($0.02/img), or local mflux (free, slow)
- Scheduler: auto-scrapes Kalshi, scores candidates, produces videos hands-free
- TikTok publishing: Content Posting API (private upload, creator manually flips to public + adds sound)
- Discord notifications: dual-post to #latest-videos + category channel with discussion threads

### Hardware
- GPU: RTX 5070 Ti 16GB GDDR7 (Blackwell, FP4 Tensor Cores)
- CPU: Ryzen 7 9800X3D (8-core/16-thread, 5.2GHz boost)
- RAM: 32GB DDR5-6000

Wan 2.2+ video generation is feasible on this hardware (480p comfortably, 720p with offloading). TI2V-5B (image-to-video, 5B params) fits easily in 16GB VRAM.

### ML infrastructure (paused, preserved for future use)
- `market_ticks` table: 1.6M hourly ticks across 4,497 resolved Kalshi markets
- Safe bet CLI: labeling, inspection, validation, training
- Category derivation, run-market linkage (551 runs), analysis features
- Classifier concluded: can't beat market price with current data (see `docs/ml/modeling_journal.md`)

---

## Roadmap

### AI Video Engine (P1 — active)

**Goal:** Produce high-quality AI videos that get traction (views, shares, followers). Quality over quantity.

**Open questions (resolve with Daniel):**
1. What is Daniel's "exact recipe" for what makes videos succeed? Hook writing? Topic selection? Audio pacing? Visual quality? Format?
2. Does the recipe require actual video motion in B-roll, or are well-composed still images + Remotion animations sufficient?
3. What's the target output? 1-3 quality videos/day? Or fewer, higher-effort pieces?
4. What's the day-30 tripwire metric? Views per video? Follower growth rate? Completion rate?

**Video generation upgrade path:**
- Current: still images (OpenAI/Imagen/mflux) with Ken Burns + crossfade in Remotion
- Option: Wan 2.2+ text-to-video or image-to-video for actual motion B-roll
  - TI2V-5B (image-to-video, 5B params) fits in 16GB VRAM, good quality/speed tradeoff
  - T2V-A14B (text-to-video, 14B active MoE) needs offloading at 16GB, ~4-7 min/5s clip at 480p
  - Wan 2.5+ adds audio generation, 2.6+ has better character consistency
  - Hybrid approach: Wan for hook section only (most important for retention), stills for rest
  - Cloud inference (Replicate, Vast.ai) for parallelized generation if local is too slow

**Pipeline end-to-end (for reference):**
1. **Analysis** (~2-5 min): Router → Research (Gemini) → Synthesize → Analyze → Summarize
2. **Content** (~1-3 min scripts, 3-15 min assets): Strategy agent → Script agent → TTS + Visuals (parallel) → Direction agent
3. **Video** (~1-3 min): Python builds props → Remotion renders → FFmpeg loudnorm post-process
4. **Publishing**: TikTok API (private upload) → Discord notification → Manual: add sound + flip to public

### ML Tracks (all paused)

Preserved for future use. Don't work on these without explicit decision to unpause.

| Track | Status | Resume condition |
|-------|--------|-----------------|
| A: Category trading rules | Ready | Unpause decision |
| B: Content ranking | Blocked (no engagement data) | Published videos + analytics pipeline |
| C: Price movement prediction | Ready | Unpause decision |
| D: Watchlist | Built, not running | ≥50 resolved pairs for clean data |
| E: Anomaly detection | Deferred | After A and C evaluated |

Details in `docs/ml/modeling_journal.md`.

---

## Commands Reference

### Safe bet CLI (still useful for data inspection)

```bash
uv run python -m rekko_server.ml.safe_bet label                  # label all resolved markets
uv run python -m rekko_server.ml.safe_bet show 505 --plot         # inspect one market + 4-panel PNG
uv run python -m rekko_server.ml.safe_bet list --safe             # list safe bets
uv run python -m rekko_server.ml.safe_bet data 505                # raw tick data + metadata
uv run python -m rekko_server.ml.safe_bet validate --plot         # sanity checks + calibration PNG
uv run python -m rekko_server.ml.safe_bet train                   # train classifier (parked — see above)
uv run python -m rekko_server.ml.safe_bet train --signal-features # with Haiku binary signals
```

### Data collection

```bash
# Backfill resolved markets from Kalshi
uv run python scripts/backfill_resolved.py --days 90 --max-markets 2000

# Batch analysis (550 already done, resumable)
uv run python scripts/batch_analyze_resolved.py --categories tennis,ncaa_basketball --max-markets 50 --cheap

# Extract text features / binary signals from analysis.json
uv run python scripts/extract_analysis_features.py
uv run python scripts/extract_market_signals.py

# Link runs to markets
uv run python scripts/link_runs_to_markets.py
```

### Watchlist

```bash
uv run python -m rekko_server.watchlist add --market-id 42
uv run python -m rekko_server.watchlist list
uv run python -m rekko_server.watchlist status
uv run python -m rekko_server.watchlist run-once [--dry-run]
uv run python -m rekko_server.watchlist daemon
```

### Data inspection

```bash
# Category win rates vs market prices
uv run python -c "
from rekko_server.ml.safe_bet import build_training_data
from rekko_server.data import connect
df = build_training_data(connect())
print(df.groupby('category')[['favorite_won', 'favorite_price']].mean().sort_values('favorite_won'))
"

# Check research-resolved pair count
uv run python -c "from rekko_server.data import load_runs_with_markets; df = load_runs_with_markets(); print(f'{len(df)} pairs'); print(df.groupby('category').size())"
```

---

## Later: Personalization (paused)

Three-phase plan (see `notes/personalization-thoughts.md`): anomaly detection → relevance ranking → content personalization. Needs a user base and engagement data first.

