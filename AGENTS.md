# Rekko AI Documentation

## About this project

- Documentation site for [Rekko AI](https://rekko.ai) built on [Mintlify](https://mintlify.com)
- Pages are MDX files with YAML frontmatter
- Configuration lives in `docs.json`
- OpenAPI spec auto-generated from FastAPI app in `openapi.json`
- Run `mint dev` to preview locally
- Run `mint broken-links` to check links

## Terminology

- Use "market" not "bet" or "wager"
- Use "prediction" not "gamble" or "gambling"
- Use "forecast" not "bet prediction"
- Use "traders" not "bettors" or "gamblers"
- Use "position" not "stake"
- Use "Kalshi" and "Polymarket" — always capitalized, never abbreviated
- Never mention specific AI model names (no "Claude", "Gemini", "GPT") — say "multi-stage research pipeline" or "AI analysis" instead

## Style preferences

- Use active voice and second person ("you")
- Keep sentences concise — one idea per sentence
- Use sentence case for headings
- Bold for UI elements: Click **Settings**
- Code formatting for file names, commands, paths, API endpoints, and parameter names
- Always show code examples in three languages: cURL, Python (httpx), JavaScript (fetch)
- Use `CodeGroup` for multi-language examples

## Content boundaries

- Document only customer-facing API endpoints and integrations
- Do not document internal admin endpoints (`/v1/customers/upsert`, `/v1/customers/reset`, `/v1/customers/profile`)
- Do not document internal infrastructure (Render deployment, Neon database, Stripe webhook internals)
- Do not document the pipeline implementation (prompts, agents, graph nodes)
- Do not reveal competitive methodology (how research/analysis works internally)
