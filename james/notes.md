  To test with real APIs:
  uv sync --extra trends
  uv run python -m rekko_server.scrapers.viral_discovery --source tiktok  # needs SCRAPECREATORS_API_KEY
  uv run python -m rekko_server.scrapers.viral_discovery --source google_trends  # needs pytrends installed