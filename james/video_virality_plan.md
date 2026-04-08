# Video Virality Implementation Plan

Based on [VIDEO_VIRALITY_RESEARCH.md](./VIDEO_VIRALITY_RESEARCH.md) research. Updated 2026-04-04 based on James + Daniel discussion.

Phase 1 is the current priority. Phases 2-4 are independent. Phase 5 needs Phase 3. Phase 6 needs Phase 5.

---

## Phase 1: Trend & Viral Video Discovery System

**Goal**: Daily-updated DB of viral/popular topics and viral/popular short-form videos, tracked over time (days viral, velocity, decay). This feeds content strategy — what to make videos about, what formats are working.

**Effort**: Medium. **Impact**: High (compounds over time — data-driven content decisions from day one).

**Owner**: James (scraping + data). Daniel uses the output to clone winning formats.

**Decisions**: All 4 sources built at once. SQLite-only (no Neon dual-write). DB-only (no JSON file manifests). CLI entry point first (scheduler loop later). `asyncpraw` for Reddit (native async).

### Update 2026-04-04 (James + Daniel discussion)
- [James] Tinker with ScrapeCreators to find most viral / popular videos
- [Dan] Figure out how to take 1 of the videos and clone it in new style
- [James] Nail down the manual operations (TikTok, Facebook, Instagram, YouTube)
- [TBD] Write something to get analytics on published videos (e.g. TikTok Creator API)

### 1A. Data Sources

| Source | What it provides | Cost | Library |
|---|---|---|---|
| **ScrapeCreators** (existing client) | TikTok trending feed, hashtag popularity, keyword video search with engagement metrics | $10/5000 credits | `publishing/scrapecreators.py` — already built |
| **Google Trends** | Real-time trending search topics, interest over time, related queries | Free | `pytrends` (new dep) |
| **YouTube Data API v3** | Viral Shorts by keyword search, trending videos, engagement metrics | Free (10k quota/day) | `google-api-python-client` (already in pipeline extra), uses existing `GOOGLE_API_KEY` |
| **Reddit API** | Trending posts in prediction market / finance subreddits | Free (100 req/min) | `asyncpraw` (new dep, native async) |
| **Kalshi/Polymarket DB** | Existing scraped markets — titles, volumes, categories, price movements | Free (already in `rekko.db`) | `rekko_server.data` |

No cross-referencing trending topics ↔ existing markets for now — each source stores its own data with source attribution and metadata. Market cross-referencing is a future enhancement (keyword overlap → LLM matching).

### 1B. DB Schema (4 tables)

**Modify**: `src/rekko_server/db/schema.py` — add to `init_schema()`. All use `CREATE TABLE IF NOT EXISTS`.

**Note**: `days_trending`/`days_viral` are `GENERATED ALWAYS AS ... STORED` columns — must be EXCLUDED from all INSERT/UPDATE SQL.

```sql
CREATE TABLE IF NOT EXISTS viral_topics (
    id TEXT PRIMARY KEY,                -- source:topic_key (e.g., "google_trends:tariffs")
    source TEXT NOT NULL,               -- google_trends | reddit | scrapecreators
    topic TEXT NOT NULL,                -- human-readable topic name
    category TEXT,                      -- finance, politics, crypto, sports, entertainment, etc.
    metadata JSON,                      -- source-specific: subreddit, search_volume, related_queries, etc.
    first_seen_at TIMESTAMP NOT NULL,
    last_seen_at TIMESTAMP NOT NULL,
    peak_score REAL,                    -- highest trending score observed
    current_score REAL,                 -- latest trending score
    days_trending INTEGER GENERATED ALWAYS AS (
        CAST((julianday(last_seen_at) - julianday(first_seen_at)) AS INTEGER) + 1
    ) STORED
);

CREATE TABLE IF NOT EXISTS viral_topic_snapshots (
    topic_id TEXT NOT NULL REFERENCES viral_topics(id),
    source TEXT NOT NULL,               -- denormalized for query convenience
    measured_at TIMESTAMP NOT NULL,
    score REAL NOT NULL,                -- normalized 0-100 trending score
    metadata JSON,                      -- source-specific raw metrics
    PRIMARY KEY (topic_id, measured_at)
);

CREATE TABLE IF NOT EXISTS viral_videos (
    id TEXT PRIMARY KEY,                -- source:video_id (e.g., "tiktok:7349281234")
    source TEXT NOT NULL,               -- tiktok | youtube_shorts
    platform_video_id TEXT NOT NULL,
    creator_handle TEXT,
    creator_follower_count INTEGER,
    title TEXT,
    description TEXT,
    duration_seconds REAL,
    hashtags JSON,                      -- ["prediction", "kalshi", "stocks"]
    category TEXT,
    url TEXT,
    metadata JSON,                      -- source-specific: sound_id, effect_ids, etc.
    first_seen_at TIMESTAMP NOT NULL,
    last_seen_at TIMESTAMP NOT NULL,
    peak_views INTEGER,
    days_viral INTEGER GENERATED ALWAYS AS (
        CAST((julianday(last_seen_at) - julianday(first_seen_at)) AS INTEGER) + 1
    ) STORED
);

CREATE TABLE IF NOT EXISTS viral_video_snapshots (
    video_id TEXT NOT NULL REFERENCES viral_videos(id),
    source TEXT NOT NULL,               -- denormalized for query convenience
    measured_at TIMESTAMP NOT NULL,
    views INTEGER,
    likes INTEGER,
    comments INTEGER,
    shares INTEGER,
    view_velocity REAL,                 -- views gained since last snapshot / hours elapsed
    metadata JSON,                      -- source-specific raw metrics
    PRIMARY KEY (video_id, measured_at)
);
```

Indexes: `source` + `last_seen_at` on parent tables, FK columns on snapshot tables.

### 1C. Implementation Tasks

#### T-1: Pydantic Models
**New file**: `src/rekko_server/models/viral.py`

`ViralTopic`, `ViralTopicSnapshot`, `ViralVideo`, `ViralVideoSnapshot`. Every `Field()` gets `description`. `metadata` uses `default_factory=dict`. `hashtags` uses `default_factory=list`.

#### T-2: DB Schema
**Modify**: `src/rekko_server/db/schema.py`

Add the 4 tables + indexes to `init_schema()`. Pattern: `CREATE TABLE IF NOT EXISTS` (same as existing tables).

#### T-3: DB Write Functions
**New file**: `src/rekko_server/db/viral.py`

Follow pattern from `db/markets.py`. Functions:
- `upsert_viral_topic(conn, topic)` — `INSERT ON CONFLICT` updates `last_seen_at`, `current_score`, `peak_score = MAX(peak_score, excluded.peak_score)`
- `record_topic_snapshot(conn, snapshot)` — `INSERT ON CONFLICT` updates score + metadata
- `upsert_viral_video(conn, video)` — `INSERT ON CONFLICT` updates `last_seen_at`, `peak_views = MAX(...)`
- `record_video_snapshot(conn, snapshot)` — `INSERT ON CONFLICT` updates all metrics

#### T-4: Config
**Modify**: `src/rekko_server/config.py`

Add to `Settings`: `reddit_client_id: str = ""`, `reddit_client_secret: str = ""`, `reddit_user_agent: str = "rekko-server:v1 (by /u/rekko_ai)"`. YouTube reuses existing `google_api_key`.

#### T-5: Dependencies
**Modify**: `pyproject.toml`

New extra: `trends = ["pytrends>=4.9", "asyncpraw>=7.7"]`. `google-api-python-client` already in `pipeline` extra.

#### T-6: Google Trends Scraper
**New file**: `src/rekko_server/scrapers/google_trends.py`

- `pytrends.trending_searches()` for real-time trending topics
- Wrap synchronous pytrends calls in `asyncio.to_thread()`
- 5s delay between calls (Google rate-limits aggressively)
- Score: rank-based (1st = 100, 20th = 5). Google Trends interest_over_time scores are 0-100 natively.
- ID format: `google_trends:{topic_slug}`
- **Risk**: pytrends is flaky (unofficial wrapper). Fully isolated behind try/except — failure must never crash orchestrator. Fallback: SerpAPI Google Trends endpoint (existing `serp_api_key`).

#### T-7: YouTube Shorts Scraper
**New file**: `src/rekko_server/scrapers/youtube_shorts.py`

- `googleapiclient.discovery.build("youtube", "v3", developerKey=...)` (lib already in deps)
- `search.list(type="video", videoDuration="short", order="viewCount", publishedAfter=7d_ago)`
- Then `videos.list(id=..., part="statistics,snippet,contentDetails")` for full stats
- Wrap synchronous API calls in `asyncio.to_thread()`
- Search keywords: `["prediction market", "kalshi", "polymarket", "sports betting odds", "election odds", "crypto prediction"]`
- Quota budget: ~50 searches (5000 units) + 500 video detail lookups (500 units) per day within free 10k tier
- ID format: `youtube_shorts:{video_id}`

#### T-8: Reddit Trends Scraper
**New file**: `src/rekko_server/scrapers/reddit_trends.py`

- `asyncpraw` — native async, no thread wrapping needed
- Subreddits: `wallstreetbets, polymarket, kalshi, sportsbook, sportsbetting, stocks, cryptocurrency`
- Fetch 25 hot posts per subreddit, filter by `MIN_SCORE=50`
- ID format: `reddit:{subreddit}:{post_id}`
- Score = raw upvotes (no normalization). metadata: `{subreddit, upvote_ratio, num_comments, url}`
- Skip gracefully if `reddit_client_id` is empty (log warning, return `[]`)

#### T-9: ScrapeCreators Extension
**Modify**: `src/rekko_server/publishing/scrapecreators.py`

Add `discover_viral_videos(client, conn=None)`:
- Calls existing `get_trending_feed()` (currently unused)
- Parses `aweme_list` items → `ViralVideo` + `ViralVideoSnapshot`
- Extracts: `aweme_id`, `author.unique_id`, `author.follower_count`, `desc`, `statistics.*`
- Uses existing `_extract_hashtags_from_desc()`
- Does NOT modify existing `research_trends()` — additive only

#### T-10: Orchestrator + CLI
**New file**: `src/rekko_server/scrapers/viral_discovery.py`

```bash
uv run python -m rekko_server.scrapers.viral_discovery              # All sources
uv run python -m rekko_server.scrapers.viral_discovery --source tiktok
uv run python -m rekko_server.scrapers.viral_discovery --source youtube
uv run python -m rekko_server.scrapers.viral_discovery --source google_trends
uv run python -m rekko_server.scrapers.viral_discovery --source reddit
```

- `run_all_sources(sources=None, conn=None)` dispatches to each scraper via `asyncio.gather(..., return_exceptions=True)` — one source failing doesn't kill others
- Single `sqlite3.Connection` shared across all sources (same WAL session)
- Prints summary table of topics/videos discovered per source

#### T-11: Data Access
**Modify**: `src/rekko_server/data.py`

Add `load_viral_topics(source=None, limit=200, conn=None)` and `load_viral_videos(source=None, limit=200, conn=None)`. Follow `load_markets()` pattern — return `pd.DataFrame`.

#### T-12: Tests
**New file**: `tests/test_viral_discovery.py`

- In-memory SQLite fixture with `init_schema()`
- Model round-trip tests (all 4 Pydantic models)
- DB upsert tests (insert, re-insert with higher score, verify peak updated)
- Generated column tests (`days_trending`/`days_viral` computed correctly)
- Parsing tests per source (mock API responses → Pydantic models)
- Orchestrator test (mock all 4 sources, verify DB row counts)
- Failure isolation test (one source raises, others still write)

#### T-13: Docs
**Modify**: `CLAUDE.md` commands section + environment variables table + optional deps.

### 1D. Task Dependencies

```
T-1 (models)  T-2 (schema)  T-4 (config)  T-5 (deps)
    └──────┬──────┘               │             │
       T-3 (db writes)           │             │
    ┌──────┼──────┬──────┐       │             │
   T-6    T-7    T-8    T-9     │             │
    └──────┼──────┴──────┘       │             │
       T-10 (orchestrator)       │             │
    ┌──────┴──────┐              │             │
  T-11          T-12 ───────────┘─────────────┘
 (data.py)     (tests)
    │
  T-13 (docs)
```

T-1, T-2, T-4, T-5 parallelizable. T-6 through T-9 parallelizable once T-3 done.

### 1E. Verification

```bash
uv sync --extra trends                                             # Install deps
uv run python -m rekko_server.scrapers.viral_discovery             # Run all sources
uv run python -m rekko_server.scrapers.viral_discovery --source tiktok  # Single source
uv run python -c "
from rekko_server.data import load_viral_topics, load_viral_videos
print(load_viral_topics())
print(load_viral_videos())
"
uv run pytest tests/test_viral_discovery.py -v                     # Tests
```

### 1F. Future (not this deliverable)

- **Scheduler loop**: `trend_discovery_loop` (every 4-6h) in `scheduler/__init__.py`
- **Strategy agent feed**: inject trending signals into `prompts/tiktok_strategy.md` at runtime
- **Market cross-referencing**: match trending topics to Kalshi/Polymarket markets

---

## Phase 1.5: Viral B-roll Source Discovery & Download

**Goal**: Download actual mp4 files for viral short-form videos so they can be used as source material for "found-footage" stitched B-roll inside Rekko-generated TikToks. Two complementary flows:

1. **Targeted** — given a market we're about to produce, search TikTok / YouTube Shorts for videos featuring that market's subject and download them.
2. **Backfill** — drain the existing `viral_videos` table (already populated by Phase 1's trending feed scraper) by downloading every viral video we haven't grabbed yet.

Both flows write into the same download infrastructure on the existing `viral_videos` table — one source of truth for "is this video downloaded yet."

**Effort**: Medium. **Impact**: High (unlocks the found-footage workflow Daniel has been describing — replaces AI stills + Ken Burns with cuts of real viral content).

**Owner**: James (discovery + download). Daniel uses the downloaded clips to build the stitched-clip video pipeline next.

### 1.5.0 Decisions Locked

- **Search query**: market title verbatim. No LLM entity extraction yet — cheapest version that lets us validate result quality before adding intelligence.
- **Sources**: TikTok (existing `ScrapeCreatorsClient.search_keyword`) + YouTube Shorts (existing `search_viral_shorts`). No Instagram, no Reddit.
- **Filter**: duration 5–60s, sort by most-liked, posted within last ~30 days.
- **Storage**: flat layout — `data/viral_broll/{video_id}.mp4`. Same file can serve multiple markets (no per-market subdirectories, no duplication).
- **Download tool**: `yt-dlp` Python library, wrapped in `asyncio.to_thread()`.
- **Download metadata**: lives on `viral_videos` (added via idempotent ALTER TABLE).
- **Market association**: lives in a new thin `viral_broll_candidates` table.
- **No production-worker / scheduler hookup this session.** CLI is the only entry point. Validation = run on three markets, eyeball the downloads.

### 1.5A. Schema (modify `db/schema.py`)

**Migration** (idempotent, follows existing pattern at `schema.py:257-262`):

```python
for col_ddl in [
    "ALTER TABLE viral_videos ADD COLUMN file_path TEXT",
    "ALTER TABLE viral_videos ADD COLUMN file_size_bytes INTEGER",
    "ALTER TABLE viral_videos ADD COLUMN download_status TEXT NOT NULL DEFAULT 'pending'",
    "ALTER TABLE viral_videos ADD COLUMN download_error TEXT",
    "ALTER TABLE viral_videos ADD COLUMN downloaded_at TIMESTAMP",
]:
    try:
        conn.execute(col_ddl)
    except sqlite3.OperationalError:
        pass
conn.commit()
```

**New association table**:

```sql
CREATE TABLE IF NOT EXISTS viral_broll_candidates (
    id              TEXT PRIMARY KEY,    -- "{market_key}:{viral_video_id}"
    market_key      TEXT NOT NULL,       -- "{platform}:{platform_id}"
    market_title    TEXT,
    viral_video_id  TEXT NOT NULL REFERENCES viral_videos(id),
    query_used      TEXT NOT NULL,
    score           REAL NOT NULL DEFAULT 0.0,
    discovered_at   TIMESTAMP NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_viral_broll_market
    ON viral_broll_candidates(market_key, score DESC);
CREATE INDEX IF NOT EXISTS idx_viral_videos_download_status
    ON viral_videos(download_status);
```

### 1.5B. Pydantic Model (modify `models/viral.py`)

Add `ViralBrollCandidate`:

```python
class ViralBrollCandidate(BaseModel):
    id: str = Field(description="Synthetic ID: {market_key}:{viral_video_id}")
    market_key: str = Field(description="Market identifier as {platform}:{platform_id}")
    market_title: str | None = Field(default=None, description="Market title at discovery time")
    viral_video_id: str = Field(description="FK to viral_videos.id")
    query_used: str = Field(description="Search query that surfaced this video")
    score: float = Field(default=0.0, description="Ranking score")
    discovered_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
```

Status field is plain `str`, not `Literal[...]`, to match `viral_videos.source` convention.

### 1.5C. CRUD (modify `db/viral.py`)

All take Pydantic models / kwargs, follow existing `INSERT ... ON CONFLICT(id) DO UPDATE SET` pattern:

- `upsert_broll_candidate(conn, *, candidate: ViralBrollCandidate)` — upsert, refreshes `score` and `query_used` on conflict.
- `mark_video_downloaded(conn, *, video_id: str, file_path: str, file_size_bytes: int)` — sets `download_status='downloaded'`, `downloaded_at=now`.
- `mark_video_failed(conn, *, video_id: str, error: str)` — sets `download_status='failed'`, `download_error`.
- `get_pending_video_downloads(conn, *, source: str | None = None, limit: int = 50) -> list[dict]` — returns `viral_videos` rows where `download_status='pending'`, ordered by `last_seen_at DESC`.
- `get_broll_candidates_for_market(conn, *, market_key: str, limit: int = 20) -> list[dict]` — joins `viral_broll_candidates` with `viral_videos`, ordered by `score DESC`.
- `count_broll_for_market(conn, *, market_key: str) -> dict[str, int]` — `{pending, downloaded, failed}` counts via join.

### 1.5D. yt-dlp Downloader (new `scrapers/viral_broll_downloader.py`)

```python
async def download_video(
    url: str,
    dest_dir: Path,
    *,
    video_id: str,
) -> tuple[Path, int]:
    """Download a single short-form video via yt-dlp.

    Returns (file_path, file_size_bytes). Raises yt_dlp.utils.DownloadError on failure.
    Synchronous yt-dlp call wrapped in asyncio.to_thread().
    """
```

- Output template: `{dest_dir}/{video_id}.%(ext)s`
- Format: `"best[height<=1080][ext=mp4]/best[ext=mp4]/best"`
- `quiet=True`, `no_warnings=True`, `noprogress=True`
- Resolves actual file via `prepare_filename`

### 1.5E. Orchestrator + CLI (new `scrapers/viral_broll.py`)

Two public coroutines:

```python
async def fetch_broll_for_market(
    *,
    market_key: str,
    query: str | None = None,
    market_title: str | None = None,
    max_per_source: int = 10,
    download: bool = True,
    conn: sqlite3.Connection | None = None,
    settings: Settings | None = None,
) -> BrollFetchStats:
    """Targeted flow: search → upsert viral_videos → upsert candidates → download top N."""

async def backfill_pending_downloads(
    *,
    source: str | None = None,
    limit: int = 50,
    conn: sqlite3.Connection | None = None,
    settings: Settings | None = None,
) -> BackfillStats:
    """Backfill flow: drain viral_videos.download_status='pending' via yt-dlp."""
```

Helpers:
- `_resolve_query_and_title(conn, market_key, query, market_title)` — splits `market_key` on `:`, calls `get_market_id` + `get_market` from `db/markets.py:45`.
- `_normalize_tiktok_result(item, *, query) -> tuple[ViralVideo, ViralVideoSnapshot]` — uses existing `ViralVideo` model so we write into the same table the trending feed writes into.
- `_normalize_youtube_result(viral_video, viral_snapshot) -> tuple[ViralVideo, ViralVideoSnapshot]` — passthrough; `search_viral_shorts` already returns the right shape.
- `_score_candidate(views, duration_seconds) -> float` — `log1p(views) + duration_fit_bonus`. Duration fit: 1.0 for 10–30s, 0.5 for 5–60s, 0 outside.

CLI uses argparse subcommands:

```bash
# Targeted: download viral videos featuring this market's subject
uv run python -m rekko_server.scrapers.viral_broll fetch \
    --market-key kalshi:KXMARKET-25 [--query "Trump speech"] [--no-download] [--max-per-source 10]

# Backfill: download viral videos already discovered by Phase 1's trending feed
uv run python -m rekko_server.scrapers.viral_broll backfill \
    [--source tiktok|youtube_shorts] [--limit 50]
```

Both subcommands print a summary at the end (counts + total bytes).

### 1.5F. Config (modify `config.py`)

```python
viral_broll_dir: str = "data/viral_broll"
viral_broll_max_per_source: int = 10
viral_broll_min_duration: float = 5.0
viral_broll_max_duration: float = 60.0
viral_broll_lookback_days: int = 30
```

### 1.5G. Dependency (modify `pyproject.toml`)

Add `yt-dlp>=2025.1` to the existing `trends` extra. Same install command James already runs: `uv sync --extra trends`.

### 1.5H. Tests (new `tests/test_viral_broll.py`)

In-memory SQLite + `init_schema()`. Covers:

- `ViralBrollCandidate` Pydantic round-trip
- `upsert_broll_candidate` insert + re-insert preserves `score` semantics
- `mark_video_downloaded` and `mark_video_failed` transitions on `viral_videos`
- `get_pending_video_downloads` returns only pending rows, respects `source` filter
- `get_broll_candidates_for_market` ordering and join
- `_score_candidate` boundary cases (5s, 10s, 30s, 60s, 61s)
- `_normalize_tiktok_result` parses fixture
- `fetch_broll_for_market` orchestrator with mocked TikTok + YouTube + downloader (`download=False` and `download=True`)
- `backfill_pending_downloads` with mocked downloader, asserts state transitions
- Failure isolation: TikTok client raises → YouTube still writes; one yt-dlp failure → others still succeed

No live API tests in default suite. Optional `@pytest.mark.live` smoke (skipped without keys).

### 1.5I. Docs (modify `CLAUDE.md`)

Add `viral_broll fetch` and `viral_broll backfill` CLI commands under the scrapers block.

### 1.5J. Verification

```bash
uv sync --extra trends                                      # installs yt-dlp
uv run pytest tests/test_viral_broll.py -v                  # unit tests

# Backfill test (drains existing trending feed videos)
uv run python -m rekko_server.scrapers.viral_broll backfill --limit 5

# Targeted test on three markets:
uv run python -m rekko_server.scrapers.viral_broll fetch --market-key kalshi:<entertainment-id>
uv run python -m rekko_server.scrapers.viral_broll fetch --market-key kalshi:<sports-id>
uv run python -m rekko_server.scrapers.viral_broll fetch --market-key kalshi:<politics-id>

ls -lh data/viral_broll/
```

**End-of-session decision gate**: Open 5–10 of the downloaded mp4s by hand. Are they real clips of the right people/subject? Are durations workable for stitching? If yes → next session is clip-segment selection (scene detection + LLM picker) + Remotion integration. If no → next session is LLM entity extraction.

### 1.5K. Risks

- **TikTok yt-dlp reliability**: TikTok rotates anti-scraping defenses; the yt-dlp TikTok extractor breaks frequently. Mitigation: catch `DownloadError`, mark `failed`, continue. Discovery metadata is still useful even when the file isn't.
- **Storage growth**: 10 markets × 20 clips × ~5 MB ≈ 1 GB. No cleanup job in this session. Future Phase 1.5b: delete clips for resolved markets after 30 days.
- **Touching Daniel's table**: ALTER TABLE on `viral_videos`. Migration is additive + idempotent (matches existing pattern at `schema.py:257-262`); existing INSERTs are unaffected.
- **Score function is heuristic**: views + duration_fit only. Validate by inspecting top-5 vs bottom-5 after a real run.
- **Copyright**: clips will be reused inside derivative Rekko videos. James/Daniel decision; out of scope for this plan.

---

## Phase 2: Script Quality Upgrades (prompt-only edits)

**Goal**: Incorporate VIDEO_VIRALITY anti-AI-slop and hook density best practices.

**Effort**: Low (prompt text only). **Impact**: High (directly improves retention metrics).

### 2A. Anti-AI-Slop Rules
**File**: `src/rekko_server/prompts/tiktok_script.md`

- Banned phrases: "Let's dive in", "It's worth noting", "In this video", "Without further ado", "Game changer", "Breaking down"
- Sentence variation mandate: alternate 2-4 word punches with 12-18 word explanations, never 3 same-length sentences in a row
- Self-correction inclusion: "Include one self-correction or incomplete thought per script"

### 2B. Platform-Specific Pacing
**File**: `src/rekko_server/prompts/tiktok_script.md`

Current: flat 2.5 words/sec. Change to section-differentiated:
- Hook: 3.5-4.0 words/sec (punchy, rapid)
- Context: 2.5 words/sec (explanation)
- Evidence: 2.0-2.2 words/sec (data density, pauses)
- Reveal: 2.3 words/sec (clarity for payoff)
- CTA: 2.8 words/sec (energy, urgency)

### 2C. First-Frame Triple Hook
**File**: `src/rekko_server/prompts/tiktok_script.md`

First 0.5 seconds must contain ALL THREE simultaneously: (1) on-screen text with 2+ SEO keywords, (2) spoken hook opening, (3) visual element that creates scroll-stop urgency.

### 2D. Chain-of-Thought for Strategy
**File**: `src/rekko_server/prompts/tiktok_strategy.md`

Add: "Before outputting your strategy, write a 3-sentence reasoning chain explaining WHY this hook archetype and bias type are the best fit for THIS specific market."

---

## Phase 3: Multi-Platform Distribution

**Goal**: 3x distribution reach per video (TikTok + YouTube Shorts + Instagram Reels).

**Effort**: Medium. **Impact**: High.

### 3A. Platform Export Profiles
**New file**: `src/rekko_server/publishing/platform_profiles.py`
**Modify**: `src/rekko_server/video/config.py`

Per VIDEO_VIRALITY optimal settings:

| Platform | Bitrate | Duration sweet spot | Hashtag limit |
|---|---|---|---|
| TikTok | 2-4 Mbps | 21-34s | 3-5 |
| YouTube Shorts | 8-15 Mbps | 30-50s | #Shorts + 2-3 |
| Instagram Reels | 3.5-4.5 Mbps | 11-17s | max 5 |

**Approach**: Render master at highest quality (YouTube's 8-15 Mbps). FFmpeg transcode for platform variants (bitrate + optional duration trim for Instagram). Generate platform-specific metadata via post_package agent.

### 3B. YouTube Shorts Publishing
**New file**: `src/rekko_server/publishing/youtube.py`

- YouTube Data API v3 via `google-api-python-client`
- `GOOGLE_API_KEY` already in codebase (currently used for Imagen)
- Upload as private, manual review, then publish
- Title includes #Shorts hashtag
- YouTube-optimized description + keywords from post_package

### 3C. Instagram Reels Publishing
**New file**: `src/rekko_server/publishing/instagram.py`

- Meta Graph API (Instagram Content Publishing API)
- Upload video container -> publish
- Platform-specific caption (max 5 hashtags, keyword-rich first 55 chars)

### 3D. Staggered Cross-Post Loop
**New file**: `src/rekko_server/scheduler/cross_post.py`
**Modify**: `src/rekko_server/scheduler/__init__.py`

Per VIDEO_VIRALITY: TikTok first -> Instagram 2h later -> YouTube 4h later. Never simultaneous. New scheduler loop watches for newly published TikTok videos, waits appropriate intervals, publishes platform variants.

---

## Phase 4: Motion Video Generation (replace broken Sora)

**Goal**: Replace dead Sora 2 backend with working AI video generation.

**Effort**: Medium. **Impact**: Medium-high (motion B-roll >> Ken Burns on stills).

Sora shut down March 24, 2026. The `visuals_sora.py` backend is dead code.

### Options (pick one)

| Backend | Cost | Quality | Notes |
|---|---|---|---|
| Kling AI 3.0 Pro | $37/mo | 8.4/10 (best value) | REST API, polling pattern |
| Google Veo 3.1 | $20/mo (watermarked) | Best realism | Uses existing GOOGLE_API_KEY |
| Wan 2.2 TI2V-5B | Free (local) | Good, fits 16GB VRAM | Heaviest integration |

**Recommendation**: Kling AI 3.0 Pro for best quality/cost. Veo 3.1 if budget-constrained.

### Implementation
**New file**: `src/rekko_server/video/visuals_kling.py` (or `visuals_veo.py`)
**Modify**: `src/rekko_server/video/visuals.py` (add dispatch for new prefix)
**Modify**: `src/rekko_server/video/config.py` (add API key setting)

Follow existing pattern: `VIDEO_IMAGE_MODEL=kling:kling-3-pro` dispatches to new backend. Same async interface as `generate_sora_video()`. Polling pattern (submit job -> poll status -> download) reusable from `visuals_sora.py`.

**Hybrid approach** (per VIDEO_VIRALITY): Motion video for hook section only (most important for retention), stills for rest (cheaper + faster).

---

## Phase 5: Analytics Feedback Loop

**Goal**: Track published video performance, feed back into content decisions.

**Effort**: High. **Impact**: High long-term.

### 5A. Performance Tracking
**New package**: `src/rekko_server/analytics/`

Poll each platform's API for metrics:
- TikTok: views, likes, comments, shares, hook retention at 4s
- YouTube: views, watch time, engagement rate
- Instagram: reach, saves, shares, engagement

### 5B. Database Schema
**Modify**: `src/rekko_server/db/schema.py`

New `video_performance` table:
```sql
CREATE TABLE video_performance (
    run_id TEXT,
    platform TEXT,
    platform_video_id TEXT,
    published_at TIMESTAMP,
    views INTEGER,
    likes INTEGER,
    comments INTEGER,
    shares INTEGER,
    avg_view_duration_seconds REAL,
    hook_retention_pct REAL,
    measured_at TIMESTAMP,
    PRIMARY KEY (run_id, platform, measured_at)
);
```

### 5C. 7/30/90 Day Decision Framework
**New file**: `src/rekko_server/analytics/decisions.py`

Per VIDEO_VIRALITY:
- **7 days**: Hook retention analysis (4-second mark)
- **30 days**: Format validation (which hook archetypes perform best?)
- **90 days**: Strategic direction (follower conversion, platform comparison)
- Auto-outputs: "Double down on X", "Pivot Y", "Kill Z"

### 5D. Scheduler Integration
Add analytics polling loop (every 6h) to scheduler.

---

## Phase 6: A/B Testing (future)

Generate 2-3 hook variants per video (different archetypes), publish alternating versions, measure via Phase 5 analytics. Also: thumbnail A/B testing via YouTube Studio's native "Test & Compare" feature.

---

## Dependency Graph

```
Phase 1 (Trend discovery)    ─── CURRENT PRIORITY
Phase 2 (Script quality)     ─── independent
Phase 3 (Multi-platform)     ──> Phase 5 (Analytics) ──> Phase 6 (A/B)
Phase 4 (Motion video)       ─── independent
```

## Owner Mapping

| Phase | Owner | Why |
|---|---|---|
| 1 - Trend discovery | James | Scraping + data + DB |
| 2 - Script quality | James or Daniel | Prompt edits only |
| 3 - Multi-platform | Daniel | Publishing + video pipeline |
| 4 - Motion video | Daniel | Video pipeline backend |
| 5 - Analytics | James | Analytics + ML |
| 6 - A/B testing | Both | James: measurement, Daniel: variant pipeline |

## Budget

| Item | Cost/mo | Phase |
|---|---|---|
| ElevenLabs Creator (existing) | $22 | - |
| ScrapeCreators | ~$2 (at current usage) | Phase 1 |
| Google Trends (pytrends) | Free | Phase 1 |
| YouTube Data API | Free | Phase 1, 3 |
| Reddit API (praw) | Free | Phase 1 |
| Instagram Graph API | Free | Phase 3 |
| Kling AI 3.0 Pro | $37 | Phase 4 |
| **Total** | **$61/mo** | |

Budget-constrained: Veo 3.1 ($20/mo) instead of Kling -> **$44/mo total**.
