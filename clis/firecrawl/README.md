# firecrawl

> Snapshot date: 2026-04-26. Upstream: <https://github.com/firecrawl/firecrawl>

"**The API to search, scrape, and interact with the web for AI.**"
Firecrawl turns any URL or domain into clean, LLM-ready Markdown /
HTML / structured JSON in one HTTP call. The CLI / SDK surface is
`firecrawl-py` (and `@mendable/firecrawl-js`) plus the hosted REST
API and self-hostable Docker stack; the four primitive verbs are
`scrape` (one URL → markdown), `crawl` (whole site, follow links),
`map` (URL inventory only, no body fetch), and `extract` (LLM-graded
structured extraction against a Pydantic / Zod schema).

## 1. Install footprint

- SDK: `pip install firecrawl-py` or `npm i @mendable/firecrawl-js`.
- CLI: bundled `firecrawl` binary on PyPI install for one-off `scrape`
  / `crawl` runs against the hosted or self-hosted endpoint.
- Self-host: `git clone https://github.com/firecrawl/firecrawl &&
  docker compose up` (API + worker + Playwright + Redis).
- Hosted: `api.firecrawl.dev` with free tier; `FIRECRAWL_API_KEY`
  env var picked up automatically.

## 2. Repo + version + license

- Repo: <https://github.com/firecrawl/firecrawl>
- Latest release: **v2.9.0** (2026-04-10)
- License: **AGPL-3.0** —
  <https://github.com/firecrawl/firecrawl/blob/main/LICENSE>
- Default branch: `main`
- Language: TypeScript (API + workers) + Python / TS SDKs
- Stars: ~112k

## 3. Models supported

None directly for `scrape` / `crawl` / `map` — those are pure HTML →
Markdown transforms with Playwright + readability + sitemap parsing.
`extract` and `scrape(formats=[{"type":"json", "schema": ...}])`
ride OpenAI / Anthropic / Gemini / any OpenAI-compatible endpoint
configured server-side; the LLM is graded against the user-supplied
schema with re-ask on validation failure.

## 4. Notable angle

**Markdown-first scraping with first-class JS rendering and a
structured-extraction primitive on the same call.** Where
`crawl4ai` is a Python library you embed and `trafilatura` is a
text-only extractor, Firecrawl is an HTTP service that bundles
Playwright headless rendering + readability + anti-bot evasion +
sitemap-driven recursion + LLM extraction behind one schema —
agent frameworks ([`crewai`](../crewai/), [`langchain`](../langflow/),
[`mcp-agent`](../mcp-agent/)) ship first-party `FirecrawlTool`
wrappers so a "scrape this page and return a typed object" tool call
is one line. The self-hostable AGPL stack means the same surface
works inside an air-gapped network with on-prem Playwright workers
plus a local OpenAI-compatible endpoint for the extract step.

## 5. Last verified

2026-04-26 via `gh api repos/firecrawl/firecrawl` (note: the upstream
recently renamed from `mendableai/firecrawl` — the old URL still
redirects).
