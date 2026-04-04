# Video Virality Implementation Plan

Based on [VIDEO_VIRALITY.md](./VIDEO_VIRALITY.md) research. Phases 1-4 are independent. Phase 5 needs Phase 2. Phase 6 needs Phase 5.

---

## Phase 1: Script Quality Upgrades (prompt-only edits)

**Goal**: Incorporate VIDEO_VIRALITY anti-AI-slop and hook density best practices.

**Effort**: Low (prompt text only). **Impact**: High (directly improves retention metrics).

### 1A. Anti-AI-Slop Rules
**File**: `src/rekko_server/prompts/tiktok_script.md`

- Banned phrases: "Let's dive in", "It's worth noting", "In this video", "Without further ado", "Game changer", "Breaking down"
- Sentence variation mandate: alternate 2-4 word punches with 12-18 word explanations, never 3 same-length sentences in a row
- Self-correction inclusion: "Include one self-correction or incomplete thought per script"

### 1B. Platform-Specific Pacing
**File**: `src/rekko_server/prompts/tiktok_script.md`

Current: flat 2.5 words/sec. Change to section-differentiated:
- Hook: 3.5-4.0 words/sec (punchy, rapid)
- Context: 2.5 words/sec (explanation)
- Evidence: 2.0-2.2 words/sec (data density, pauses)
- Reveal: 2.3 words/sec (clarity for payoff)
- CTA: 2.8 words/sec (energy, urgency)

### 1C. First-Frame Triple Hook
**File**: `src/rekko_server/prompts/tiktok_script.md`

First 0.5 seconds must contain ALL THREE simultaneously: (1) on-screen text with 2+ SEO keywords, (2) spoken hook opening, (3) visual element that creates scroll-stop urgency.

### 1D. Chain-of-Thought for Strategy
**File**: `src/rekko_server/prompts/tiktok_strategy.md`

Add: "Before outputting your strategy, write a 3-sentence reasoning chain explaining WHY this hook archetype and bias type are the best fit for THIS specific market."

---

## Phase 2: Multi-Platform Distribution

**Goal**: 3x distribution reach per video (TikTok + YouTube Shorts + Instagram Reels).

**Effort**: Medium. **Impact**: High.

### 2A. Platform Export Profiles
**New file**: `src/rekko_server/publishing/platform_profiles.py`
**Modify**: `src/rekko_server/video/config.py`

Per VIDEO_VIRALITY optimal settings:

| Platform | Bitrate | Duration sweet spot | Hashtag limit |
|---|---|---|---|
| TikTok | 2-4 Mbps | 21-34s | 3-5 |
| YouTube Shorts | 8-15 Mbps | 30-50s | #Shorts + 2-3 |
| Instagram Reels | 3.5-4.5 Mbps | 11-17s | max 5 |

**Approach**: Render master at highest quality (YouTube's 8-15 Mbps). FFmpeg transcode for platform variants (bitrate + optional duration trim for Instagram). Generate platform-specific metadata via post_package agent.

### 2B. YouTube Shorts Publishing
**New file**: `src/rekko_server/publishing/youtube.py`

- YouTube Data API v3 via `google-api-python-client`
- `GOOGLE_API_KEY` already in codebase (currently used for Imagen)
- Upload as private, manual review, then publish
- Title includes #Shorts hashtag
- YouTube-optimized description + keywords from post_package

### 2C. Instagram Reels Publishing
**New file**: `src/rekko_server/publishing/instagram.py`

- Meta Graph API (Instagram Content Publishing API)
- Upload video container -> publish
- Platform-specific caption (max 5 hashtags, keyword-rich first 55 chars)

### 2D. Staggered Cross-Post Loop
**New file**: `src/rekko_server/scheduler/cross_post.py`
**Modify**: `src/rekko_server/scheduler/__init__.py`

Per VIDEO_VIRALITY: TikTok first -> Instagram 2h later -> YouTube 4h later. Never simultaneous. New scheduler loop watches for newly published TikTok videos, waits appropriate intervals, publishes platform variants.

---

## Phase 3: Motion Video Generation (replace broken Sora)

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

## Phase 4: Trend Research System

**Goal**: Data-driven content strategy informed by what's actually going viral.

**Effort**: Medium. **Impact**: Medium (compounds over time).

### 4A. Viral Video Analyzer
**New file**: `src/rekko_server/scrapers/viral_analyzer.py`
**New model**: `src/rekko_server/models/viral_research.py`

Extend existing ScrapeCreators integration (`publishing/scrapecreators.py`) to:
1. Find top-performing prediction market / finance videos on TikTok
2. Calculate outlier score: views / creator's median views (identifies format hits vs audience hits)
3. Store in SQLite for analysis
4. Feed trending formats into strategy agent prompt

### 4B. Feed Into Strategy Agent
**Modify**: `src/rekko_server/prompts/tiktok_strategy.md`

Add a "trending formats" context section with current high-performing hook types, durations, and hashtags from the viral analyzer.

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
Phase 1 (Script quality)     ─── independent
Phase 2 (Multi-platform)     ──> Phase 5 (Analytics) ──> Phase 6 (A/B)
Phase 3 (Motion video)       ─── independent
Phase 4 (Trend research)     ─── independent
```

## Owner Mapping

| Phase | Owner | Why |
|---|---|---|
| 1 - Script quality | James or Daniel | Prompt edits only |
| 2 - Multi-platform | Daniel | Publishing + video pipeline |
| 3 - Motion video | Daniel | Video pipeline backend |
| 4 - Trend research | James | Scraping + data analysis |
| 5 - Analytics | James | Analytics + ML |
| 6 - A/B testing | Both | James: measurement, Daniel: variant pipeline |

## Budget

| Item | Cost/mo | Phase |
|---|---|---|
| ElevenLabs Creator (existing) | $22 | - |
| Kling AI 3.0 Pro | $37 | Phase 3 |
| YouTube Data API | Free | Phase 2 |
| Instagram Graph API | Free | Phase 2 |
| **Total** | **$59/mo** | |

Budget-constrained: Veo 3.1 ($20/mo) instead of Kling -> **$42/mo total**.
