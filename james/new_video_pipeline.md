# New Video Pipeline — AI Video Engine

Status: Planning (2026-04-03)

---

## Goal

Build an engine that produces high-quality AI videos optimized for engagement. Start by learning what works: find viral short-form videos, then recreate them with AI-generated visuals using Wan 2.2.

Quality over volume. Traction = views, shares, followers.

---

## Workstream 1: Viral Video Scraper

Find trending short-form videos across platforms. Simple heuristic: high views relative to recency.

### Data sources

| Platform | Method | Cost | Notes |
|----------|--------|------|-------|
| YouTube Shorts | Official YouTube Data API (`search.list`, `videoDuration=short`, `order=viewCount`) | Free (10k quota units/day) | Stable, `GOOGLE_API_KEY` already in codebase |
| TikTok | ScrapeCreators API | Paid (already integrated in codebase) | Used by post-package agent today |
| Instagram Reels | Skip for now | — | No official API, aggressive anti-bot |

### What to store per video

- Platform, video ID, URL
- Title, description, hashtags
- View count, like count, comment count, share count
- Posted timestamp, scraped timestamp
- Creator handle, follower count
- Duration (seconds)
- Category/niche (if available)
- Thumbnail URL

### Virality heuristic (starting point)

Flag a video as "viral" if it has outsized views relative to posting age. Simple formula:
```
views_per_hour = view_count / hours_since_posted
```
Rank by views_per_hour. Threshold TBD after seeing real data.

### Output

A ranked feed of viral short-form videos, refreshed on a schedule. Stored in SQLite or JSON manifests (reuse existing patterns from Kalshi/Polymarket scrapers).

---

## Workstream 2: Video Recreation Pipeline

Take a viral video → download it → extract its structure → recreate the visuals with Wan 2.2, keeping the same script and pacing.

### Step-by-step flow

```
Input: viral video URL (manually pasted for now, scraper-fed later)
  │
  ▼
1. DOWNLOAD
   yt-dlp to grab the video file + metadata
   → video.mp4, metadata.json
  │
  ▼
2. EXTRACT STRUCTURE
   - Transcript: Whisper (local) or platform captions
   - Audio: separate audio track (ffmpeg)
   - Timing: scene boundaries / cuts (ffmpeg scene detection or manual)
   - Visual inventory: what's on screen per segment (LLM vision on keyframes)
   → transcript.json, audio.wav, scenes.json, visual_analysis.json
  │
  ▼
3. GENERATE NEW VISUALS
   For each scene segment, generate an AI video clip with Wan 2.2:
   - Input: scene description from visual_analysis + transcript context
   - Model: Wan 2.2 TI2V-5B (image-to-video, fits in 16GB VRAM)
     - Generate a still image first (fast, OpenAI/Imagen)
     - Animate it with Wan 2.2 image-to-video
   - Output: one video clip per scene segment
   → clips/scene_001.mp4, clips/scene_002.mp4, ...
  │
  ▼
4. COMPOSE
   Assemble new clips with original audio/transcript timing:
   - Match clip durations to original scene durations
   - Layer original audio (or re-synthesize TTS from transcript)
   - Add text overlays / subtitles from transcript
   - Can reuse Remotion for composition (already built)
   → output/recreated.mp4
  │
  ▼
5. REVIEW
   Side-by-side comparison: original vs recreation
   Manual review before publishing
```

### Key decisions

| Decision | Current lean | Revisit when |
|----------|-------------|--------------|
| Keep original audio or re-TTS? | Keep original audio for first test, re-TTS later for original content | After first successful recreation |
| Wan 2.2 vs newer (2.5, 2.6)? | Start with 2.2 TI2V-5B (fits 16GB), upgrade if quality insufficient | After seeing first outputs |
| Remotion for composition? | Yes, reuse existing renderer | If it can't handle arbitrary clip durations |
| Scene detection method? | FFmpeg scene detect + LLM vision on keyframes | If automated detection is too noisy |

### Hardware (James's PC)

- GPU: RTX 5070 Ti 16GB GDDR7 (FP4 Tensor Cores)
- CPU: Ryzen 7 9800X3D
- RAM: 32GB DDR5-6000
- Wan 2.2 TI2V-5B: fits comfortably in 16GB VRAM
- Expected: ~4-7 min per 5s clip at 480p, ~8-12 min at 720p

---

## Immediate next steps

### James

1. **Find a viral video** — pick one short-form video (TikTok or YouTube Short) that got big views and has a style you'd want to recreate. Paste the URL.
2. **Set up Wan 2.2** — install Wan2GP or Wan 2.2 TI2V-5B locally on the RTX 5070 Ti. Verify it runs and generates a test clip.
3. **Align with Daniel** — share this doc. Key question: does this approach (recreate viral videos with AI visuals) fit his recipe, or is he thinking about something different?

### Leo (next session)

1. **Build the download + extraction pipeline** — yt-dlp download, Whisper transcription, FFmpeg scene detection, LLM vision analysis on keyframes
2. **Build the viral scraper** — YouTube Data API + ScrapeCreators TikTok integration
3. **Wire Wan 2.2 into the generation step** — image-to-video pipeline that takes scene descriptions and produces clips

---

## What exists today (reusable)

From the current Rekko pipeline:

| Component | Reusable? | Notes |
|-----------|-----------|-------|
| Remotion renderer | Yes | Can compose arbitrary clips + audio + subtitles |
| TTS (ElevenLabs / Kokoro) | Yes | For when we re-TTS instead of using original audio |
| ScrapeCreators integration | Yes | Already in `publishing/` for TikTok data |
| Image generation (OpenAI, Imagen) | Yes | First frame for image-to-video Wan input |
| FFmpeg post-processing | Yes | Loudnorm, AAC, faststart |
| Discord notifications | Yes | For posting kits and review |
| TikTok Content Posting API | Yes | For publishing finished videos |
| Subtitle rendering | Yes | Karaoke-style word-level sync in Remotion |
| Safe zones / watermark | Yes | TikTok-compliant composition |

Components that are **prediction-market-specific** (router, analysis pipeline, category pipelines, strategy/script/direction agents) are not needed for this workstream but don't need to be removed — they just sit idle.

---

## Open questions

- What niche/topic should the recreated videos target? Same niche as original, or pivot to a Rekko-adjacent topic?
- Should we brand these as @rekko.ai or create a new channel for experimentation?
- What's the day-30 tripwire? Views per video? Number of videos that cross X views? Follower growth?
- Does Daniel's recipe align with this approach, or is he thinking about something fundamentally different?
