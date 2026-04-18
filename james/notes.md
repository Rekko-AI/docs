  To test with real APIs:
  uv sync --extra trends
  uv run python -m rekko_server.scrapers.viral_discovery --source tiktok  # needs SCRAPECREATORS_API_KEY
  uv run python -m rekko_server.scrapers.viral_discovery --source google_trends  # needs pytrends installed

  claude --resume b3531fc2-4f84-46d3-8f4d-c521951d0743e 

/clear -- clear history
/rewind - Go back to a previous state
/statusline - Customize with branch, context %, todos
/checkpoints - File-level undo points
/compact - Manually trigger context compaction

Keyboard Shortcuts
Ctrl+U - Delete entire line (faster than backspace spam)
! - Quick bash command prefix
@ - Search for files
/ - Initiate slash commands
Shift+Enter - Multi-line input
Tab - Toggle thinking display
Esc Esc - Interrupt Claude / restore code

obsidian &

  Next time:
  - Check overnight trigger results in the morning (python3 scripts/yt_ingest.py --status)
  - Blog post #1 (pretrain-finetune) still the standing P0 synthesis artifact
  - Monday: Pinsight M1 into work-Leo, Alok schema


python3 scripts/yt_ingest.py --retry


- "Turn on the overnight KB trigger"
- "Turn on the daily scout trigger"


claude --resume 235bda3f-9aca-4b5b-8362-9115f15709b2 