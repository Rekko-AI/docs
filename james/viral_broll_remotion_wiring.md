# Wiring the general-viral pool into Remotion B-roll

**Context**: Phase 1.6 Step 7 eyeball gate passed (2026-04-17) — 17 curated mp4 clips ≥5M peak views sit in `data/viral_broll/general_viral/` and are tracked in `viral_videos` (download_status='downloaded'). Zero of them are being used by the video pipeline. This doc wires them in.

## Goal

Next rendered Rekko video includes real viral clips alongside AI-generated B-roll images. No Remotion/TypeScript changes needed — `BRollSequence.tsx` already handles mp4.

## Design

**Additive, not replacement.** Viral clips are appended as additional assets per section. Existing AI image generation is untouched. The section-asset glob (`broll_{section}_*.mp4` in `remotion.py:_build_render_props`) picks them up automatically.

**Pick strategy.** Random sample from eligible pool (downloaded, passes English filter, ≥5M peak). Dedup across sections within a single render so no clip repeats.

**No semantic matching.** The pool is general viral-hype content, not topic-specific. Per eyeball gate ("not about celebrities — can come next"), we're not trying to match clips to bet subject yet.

## Scope cuts

- No clip-to-section semantic routing.
- No replacement of AI image count — viral clips stack additively.
- No celebrity/topic keywords — that's a Phase 2 keyword-list iteration.
- No per-run tracking of which pool clips got used.

## Implementation

### 1. New module: `src/rekko_server/video/viral_broll_picker.py`

```python
def pick_clips(
    count: int,
    *,
    exclude_video_ids: set[str] = frozenset(),
    settings: Settings | None = None,
    conn: sqlite3.Connection | None = None,
) -> list[tuple[str, Path]]:
    """Return up to `count` random viral clips as (video_id, file_path) tuples.

    Queries viral_videos for rows with download_status='downloaded' whose
    file_path exists on disk, excludes video_ids in exclude_video_ids, and
    returns a random sample of size <= count.
    """
```

### 2. Integration point: `video/visuals.py:run_visuals()`

Add Phase 1.5 between the hook screenshot setup (Phase 1) and AI task collection (Phase 2):

```python
# ── Phase 1.5: Inject viral pool clips per section (additive) ──
if settings.viral_broll_enabled:
    picked: set[str] = set()
    for sec_name in sections_to_process:
        if sec_name not in settings.viral_broll_sections:
            continue
        clips = pick_clips(
            settings.viral_broll_per_section,
            exclude_video_ids=picked,
        )
        for k, (video_id, src_path) in enumerate(clips):
            dst = state.visuals_dir / f"broll_{sec_name}_viral_{k}.mp4"
            shutil.copy2(src_path, dst)
            picked.add(video_id)
            if deps.reporter:
                deps.reporter.file_saved(dst, f"B-roll {sec_name} (viral pool)")
```

### 3. Settings: `video/config.py:VideoSettings`

```python
viral_broll_enabled: bool = True
viral_broll_per_section: int = 1
viral_broll_sections: list[str] = ["hook", "context", "evidence", "reveal", "cta"]
```

### 4. Tests

- `pick_clips` returns empty list when pool empty
- `pick_clips` respects exclude set (no duplicates across sections)
- `pick_clips` filters out rows whose file_path doesn't exist on disk
- Integration: `run_visuals` with flag enabled copies files with the expected naming; disabled = no-op

## Validation

1. Run end-to-end render on one existing analysis:
   ```
   uv run python -m rekko_server.video --run-id <existing-run> --stage visuals --force
   uv run python -m rekko_server.video --run-id <existing-run>
   ```
2. Inspect `visuals_dir` — expect one `broll_<section>_viral_0.mp4` per enabled section.
3. Watch the output. Hook section should open with a viral clip (or keep the scroll/screenshot and cut to viral).
4. If pacing is too fast, cut sections from `viral_broll_sections` or reduce `viral_broll_per_section`.

## Fallback plan if the render feels off

- Disable via `VIDEO_VIRAL_BROLL_ENABLED=false` — everything reverts to pure AI.
- Narrow to hook only: `VIDEO_VIRAL_BROLL_SECTIONS=hook`.
- Pool too small / same clips repeating — iterate `GENERAL_VIRAL_KEYWORDS` (Phase 1.6 follow-up).
