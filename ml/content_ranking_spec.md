# Content Ranking: Engagement Data Model Spec

**Owner:** Dan (publishing/scheduler)
**Reviewer:** James (ML/data)
**Status:** Draft

---

## Goal

Replace the hand-tuned `CATEGORY_BOOSTS` and weighted formula in `scheduler/scoring.py` with an ML model trained on actual TikTok engagement data. This spec covers the data infrastructure needed to collect training labels.

---

## 1. Data Model

### 1.1 `video_publishes` table

Captures the fact that a video was published to TikTok. One row per publish. Links back to the pipeline run and market.

```sql
CREATE TABLE IF NOT EXISTS video_publishes (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    run_id          TEXT NOT NULL REFERENCES runs(run_id),
    publish_id      TEXT NOT NULL,           -- TikTok publish_id from init_upload response
    published_at    TEXT NOT NULL,           -- ISO timestamp when publish completed
    title           TEXT NOT NULL DEFAULT '',-- Video title/description sent to TikTok API
    category        TEXT NOT NULL DEFAULT '',-- Scoring category from _guess_category()
    market_ticker   TEXT NOT NULL DEFAULT '',-- Kalshi ticker / poly:<id> / rh:<id>
    platform        TEXT NOT NULL DEFAULT '',-- 'kalshi', 'polymarket', 'robinhood'
    hashtags        TEXT NOT NULL DEFAULT '[]', -- JSON array of hashtag strings
    trigger         TEXT NOT NULL DEFAULT '',-- 'scrape', 'shift', 'schedule', 'manual'
    UNIQUE(publish_id)
);

CREATE INDEX IF NOT EXISTS idx_publishes_run ON video_publishes(run_id);
CREATE INDEX IF NOT EXISTS idx_publishes_published ON video_publishes(published_at);
```

### 1.2 `video_analytics` table

Time-series engagement metrics. Polled at fixed intervals after publish to track growth curves.

```sql
CREATE TABLE IF NOT EXISTS video_analytics (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    publish_id      TEXT NOT NULL REFERENCES video_publishes(publish_id),
    views           INTEGER NOT NULL DEFAULT 0,
    likes           INTEGER NOT NULL DEFAULT 0,
    shares          INTEGER NOT NULL DEFAULT 0,
    comments        INTEGER NOT NULL DEFAULT 0,
    watch_time_sec  REAL,                   -- total watch time if available from API
    avg_watch_sec   REAL,                   -- average watch time per view
    polled_at       TEXT NOT NULL,           -- ISO timestamp of this poll
    hours_since_pub REAL NOT NULL DEFAULT 0, -- convenience: hours between published_at and polled_at
    UNIQUE(publish_id, polled_at)
);

CREATE INDEX IF NOT EXISTS idx_analytics_publish ON video_analytics(publish_id);
CREATE INDEX IF NOT EXISTS idx_analytics_polled ON video_analytics(polled_at);
```

### 1.3 Pydantic Models

```python
from pydantic import BaseModel, Field


class VideoPublish(BaseModel):
    """Record of a video published to TikTok."""

    id: int | None = Field(default=None, description="Auto-incremented row ID")
    run_id: str = Field(description="Pipeline run ID (e.g. 'run-abc123')")
    publish_id: str = Field(description="TikTok publish_id from Content Posting API")
    published_at: str = Field(description="ISO timestamp when publish completed")
    title: str = Field(default="", description="Video title/description sent to API")
    category: str = Field(default="", description="Scoring category (crypto, politics, etc.)")
    market_ticker: str = Field(default="", description="Market identifier (Kalshi ticker, poly:<id>, rh:<id>)")
    platform: str = Field(default="", description="Source platform: kalshi, polymarket, robinhood")
    hashtags: list[str] = Field(default_factory=list, description="Hashtag strings without # prefix")
    trigger: str = Field(default="", description="What triggered production: scrape, shift, schedule, manual")


class VideoAnalytics(BaseModel):
    """Single engagement snapshot for a published video."""

    id: int | None = Field(default=None, description="Auto-incremented row ID")
    publish_id: str = Field(description="TikTok publish_id, FK to video_publishes")
    views: int = Field(default=0, ge=0, description="Total view count at poll time")
    likes: int = Field(default=0, ge=0, description="Total like count at poll time")
    shares: int = Field(default=0, ge=0, description="Total share count at poll time")
    comments: int = Field(default=0, ge=0, description="Total comment count at poll time")
    watch_time_sec: float | None = Field(default=None, description="Total watch time in seconds (if available)")
    avg_watch_sec: float | None = Field(default=None, description="Average watch time per view in seconds")
    polled_at: str = Field(description="ISO timestamp of this metrics poll")
    hours_since_pub: float = Field(default=0.0, ge=0, description="Hours elapsed since publish")
```

---

## 2. Data Collection Pipeline

### 2.1 Capture publish_id on publish

**Where:** `publishing/__init__.py` -> `publish_video()`, right after the status polling loop confirms `PUBLISH_COMPLETE`.

**What to do:** After line ~158 (`return result.publish_id`), insert a DB write before returning:

```python
# After PUBLISH_COMPLETE confirmed, before return
_record_publish(
    run_id=run_id,
    publish_id=result.publish_id,
    title=title,
    hashtags=hashtags or [],
    trigger="manual",  # caller can pass trigger kwarg
)
```

Add a `trigger` kwarg to `publish_video()` (default `"manual"`). The scheduler's `_post_production()` passes `trigger=candidate.trigger` (already tracked in `ProducedEntry`).

The `_record_publish()` helper:
1. Reads `category` and `market_ticker` from the run's `analysis.json` or `pipeline.json`
2. Inserts into `video_publishes`
3. Enqueues the first analytics poll (see 2.2)

Also add the same hook in the scheduler's `_post_production()` so both manual CLI publishes and automated publishes are captured.

### 2.2 Analytics Polling Schedule

Poll engagement metrics at these intervals after publish:

| Delay | Why |
|---|---|
| 1 hour | Early signal, catch algorithmic boost |
| 6 hours | Mid-day performance |
| 24 hours | Standard "day 1" metric (primary training label) |
| 48 hours | Secondary viral window |
| 7 days | Steady-state / long-tail |

**Implementation options (pick one):**

**Option A: Scheduler loop (recommended).** Add a 5th async loop to `scheduler/loops.py` -- `engagement_poller()`. Every 15 minutes, query `video_publishes` for videos that are missing expected polls:

```sql
-- Find publishes needing a poll at the 1h mark (between 55min and 2h old, no poll yet)
SELECT vp.publish_id, vp.published_at
FROM video_publishes vp
WHERE NOT EXISTS (
    SELECT 1 FROM video_analytics va
    WHERE va.publish_id = vp.publish_id
    AND va.hours_since_pub BETWEEN 0.5 AND 2.0
)
AND julianday('now') - julianday(vp.published_at) > 0.04  -- > ~1 hour
AND julianday('now') - julianday(vp.published_at) < 0.08  -- < ~2 hours
```

Repeat for each interval bucket. Each poll calls the TikTok API, inserts a `video_analytics` row.

**Option B: Standalone polling script.** `scripts/poll_engagement.py` -- run via cron every 15 minutes. Simpler but requires external scheduling.

### 2.3 TikTok API for Metrics

The Content Posting API does **not** expose view/engagement metrics. Use the **TikTok Business API** video query endpoints instead:

**Endpoint:** `GET /v2/video/query/` with `fields=["id","create_time","like_count","comment_count","share_count","view_count"]`

```python
async def get_video_metrics(
    settings: PublishSettings,
    http_client: httpx.AsyncClient,
    video_ids: list[str],
) -> list[dict]:
    """Fetch engagement metrics for published videos.

    Uses TikTok Content Posting API status endpoint + Video Query API.
    Requires video.list scope in the OAuth token.
    """
    resp = await http_client.post(
        f"{settings.tiktok_api_base}/video/query/",
        headers={
            "Authorization": f"Bearer {settings.tiktok_access_token}",
            "Content-Type": "application/json",
        },
        json={
            "filters": {"video_ids": video_ids},
            "fields": [
                "id", "create_time", "like_count", "comment_count",
                "share_count", "view_count",
            ],
        },
    )
    resp.raise_for_status()
    return resp.json().get("data", {}).get("videos", [])
```

**OAuth scope required:** `video.list` (add to the TikTok Developer Portal app scopes). Current scopes likely only include `video.publish` and `video.upload` -- Dan needs to add `video.list`.

**Fallback:** If the Business API isn't available (requires approved app), an alternative is scraping the TikTok Creator Center analytics page via Playwright, similar to `scrapecreators.py`. Less reliable but doesn't need API approval.

**Precedent:** This mirrors the `x_client.py` pattern -- `get_tweet_metrics()` polls per-tweet engagement (likes, retweets, replies, impressions) and returns an `XThreadMetrics` dataclass. `truth_social_client.py` has the same with `get_status_metrics()`.

---

## 3. ML Training Interface

### 3.1 `load_video_engagement()` for `data.py`

```python
def load_video_engagement(
    min_hours: float = 24.0,
    category: str | None = None,
    *,
    conn: sqlite3.Connection | None = None,
) -> "pd.DataFrame":
    """Load published videos with engagement metrics for ML training.

    Returns one row per video, using the analytics snapshot closest to
    min_hours after publish. Joins with market features from runs/markets.

    Columns:
        run_id, publish_id, published_at, title, category, market_ticker,
        platform, trigger, hashtags,
        views, likes, shares, comments, avg_watch_sec, hours_since_pub,
        recommendation, confidence, probability,
        market_yes_price, market_volume_24h,
        like_rate (likes/views), share_rate (shares/views)

    Args:
        min_hours: Use the analytics snapshot closest to this many hours
            after publish. Default 24.0 (day-1 metrics).
        category: Filter to a scoring category. None = all.
        conn: Optional connection -- defaults to rekko.db via Settings.
    """
```

SQL core:

```sql
SELECT
    vp.run_id, vp.publish_id, vp.published_at, vp.title, vp.category,
    vp.market_ticker, vp.platform, vp.trigger, vp.hashtags,
    va.views, va.likes, va.shares, va.comments,
    va.avg_watch_sec, va.hours_since_pub,
    r.recommendation, r.confidence, r.probability,
    m.yes_price AS market_yes_price,
    m.volume_24h AS market_volume_24h
FROM video_publishes vp
JOIN video_analytics va ON vp.publish_id = va.publish_id
LEFT JOIN runs r ON vp.run_id = r.run_id
LEFT JOIN run_markets rm ON r.run_id = rm.run_id
LEFT JOIN markets m ON rm.market_id = m.id
WHERE va.id = (
    SELECT va2.id FROM video_analytics va2
    WHERE va2.publish_id = vp.publish_id
    ORDER BY ABS(va2.hours_since_pub - ?) ASC
    LIMIT 1
)
```

Post-processing: add `like_rate = likes / max(views, 1)` and `share_rate = shares / max(views, 1)`.

### 3.2 Training DataFrame Shape

Target: **views at 24h** (or `like_rate` for quality-adjusted ranking).

| Column | Source | Role |
|---|---|---|
| `category` | video_publishes | Feature (one-hot) |
| `platform` | video_publishes | Feature |
| `trigger` | video_publishes | Feature (shift vs scrape vs schedule) |
| `market_volume_24h` | markets | Feature |
| `market_yes_price` | markets | Feature |
| `confidence` | runs | Feature (pipeline analysis confidence) |
| `probability` | runs | Feature |
| `recommendation` | runs | Feature (BUY/SELL/NO_BET) |
| `num_hashtags` | len(hashtags) | Feature |
| `hour_of_day` | published_at | Feature (posting time) |
| `day_of_week` | published_at | Feature |
| `views` | video_analytics | **Label** |
| `like_rate` | computed | **Alt label** |
| `share_rate` | computed | **Alt label** |

### 3.3 Minimum Data Requirements

- **50 videos** with 24h engagement data to start training a baseline (random forest / gradient boosting).
- **100+ videos** for meaningful category-level signal.
- **200+ videos** to learn posting-time effects.

At current production rate (~8-12 videos/day when scheduler is active), expect:
- 50 labeled examples in ~1 week
- 200 labeled examples in ~3 weeks

---

## 4. Integration Points

### 4.1 Schema Registration

Add the two new tables and indexes to `db/schema.py`:
- Append `CREATE TABLE` statements to the `TABLES` list
- Append `CREATE INDEX` statements to the `INDEXES` list
- `init_schema()` handles the rest -- no migration needed for new tables

### 4.2 Publish Hook

**File:** `publishing/__init__.py` -> `publish_video()`

After the status polling loop confirms `PUBLISH_COMPLETE` (line ~132), before returning `result.publish_id`:

```python
from rekko_server.db import get_db
conn = get_db()
conn.execute(
    """INSERT OR IGNORE INTO video_publishes
       (run_id, publish_id, published_at, title, category, market_ticker, platform, hashtags, trigger)
       VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)""",
    (run_id, result.publish_id, datetime.now(timezone.utc).isoformat(),
     title, category, ticker, platform, json.dumps(hashtags or []), trigger),
)
conn.commit()
```

Add `trigger: str = "manual"` parameter to `publish_video()`. Update the scheduler's `_post_production()` call to pass `trigger=candidate.trigger`.

Category and ticker extraction: read from `analysis.json` in run_dir, or accept as kwargs from the scheduler which already has this info in `BetInput`.

### 4.3 Engagement Poller

**File:** `scheduler/loops.py` -- new `engagement_poller()` async function.

Pattern: same as `shift_detector()` -- infinite loop with `asyncio.sleep(POLL_INTERVAL)`.

```python
ENGAGEMENT_POLL_SCHEDULE_HOURS = [1, 6, 24, 48, 168]  # 7 days = 168h

async def engagement_poller(settings: Settings) -> None:
    """Poll TikTok for engagement metrics on recent publishes."""
    poll_interval = 900  # 15 minutes
    while True:
        try:
            conn = get_db()
            # Find publishes needing polls at each scheduled hour
            for target_hours in ENGAGEMENT_POLL_SCHEDULE_HOURS:
                pending = _find_pending_polls(conn, target_hours)
                for publish_id in pending:
                    metrics = await _fetch_tiktok_metrics(publish_id)
                    _insert_analytics(conn, publish_id, metrics, target_hours)
        except Exception:
            logfire.warn("Engagement poll failed (non-fatal)")
        await asyncio.sleep(poll_interval)
```

Wire into `scheduler/__main__.py` alongside the existing `scrape_loop`, `shift_detector`, and `production_worker` tasks.

**Config:** `SCHEDULER_ENGAGEMENT_POLL_INTERVAL` (default 900s). Set to 0 to disable.

### 4.4 ML Model in Scoring

Once we have enough data, the trained model plugs into `scheduler/scoring.py`:

```python
# In score_candidate(), replace the hand-tuned formula:

def score_candidate(bet_input: BetInput, ...) -> float:
    # If ML model is available, use it
    model = _load_engagement_model()
    if model is not None:
        features = _extract_features(bet_input, volume_24h, open_interest, price_delta, ...)
        return model.predict(features)

    # Fall back to current hand-tuned formula
    ...
```

The model lives in `ml/engagement_ranker.py`, implements the existing `MarketRanker` Protocol from `ml/__init__.py`, and is loaded lazily on first call. Training happens offline via `scripts/train_engagement_model.py` or `notebooks/`. The trained artifact (joblib/pickle) goes in `data/models/`.

### 4.5 `data.py` Registration

Add `load_video_engagement` to `data.py` alongside the existing loaders. Update the docstring at the top of `data.py` and the import list in `JAMES.md`.

---

## 5. Implementation Order

1. **Schema** -- add tables to `db/schema.py` + Pydantic models in `models/engagement.py`
2. **Publish hook** -- wire into `publish_video()` to record publishes
3. **Metrics client** -- add `get_video_metrics()` to `publishing/tiktok.py` (or new `tiktok_analytics.py`)
4. **Poller loop** -- add `engagement_poller()` to `scheduler/loops.py`
5. **Data loader** -- add `load_video_engagement()` to `data.py`
6. **Let it cook** -- run for 1-2 weeks to accumulate data
7. **Train** -- baseline model in `notebooks/`, then productionize in `ml/engagement_ranker.py`
