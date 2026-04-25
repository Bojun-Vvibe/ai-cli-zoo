# crawl4ai

> Snapshot date: 2026-04. Upstream: <https://github.com/unclecode/crawl4ai>
> License file: <https://github.com/unclecode/crawl4ai/blob/main/LICENSE>
> Pinned: `v0.8.5` (2026-03-18).

A web crawler/scraper that produces **LLM-ready Markdown** instead of raw
HTML. Ships a Python library, a Docker server with an HTTP API, and — the
reason it is in this catalog — a CLI named `crwl` that turns "fetch this
URL and give me something I can paste into a prompt" into one command.

It is in the same niche as [`gitingest`](../gitingest/) and
[`marker`](../marker/), but for the open web rather than for git repos or
PDFs: same end-state (clean Markdown for an LLM), different source.

## 1. Install footprint

- `pip install -U crawl4ai` then `crawl4ai-setup` to install the
  Playwright browser binaries (Chromium, ~300 MB) and verify drivers.
  `crawl4ai-doctor` diagnoses missing pieces.
- Alternatively `docker run -p 11235:11235 unclecode/crawl4ai:latest`
  for the server mode; the CLI can hit either embedded or remote
  Playwright.
- Python 3.10+. Linux, macOS, Windows. The binary you actually invoke
  is `crwl` (note: no `crawl4ai` CLI — only the setup helpers use the
  long name).
- Per-project state is none; runtime cache and screenshots default to
  `~/.cache/crawl4ai/`.

## 2. License

Apache-2.0.

## 3. Models supported

Crawl4AI itself is **not** an LLM client — its job is to deliver clean
Markdown to *your* LLM. However, the optional **extraction strategies**
(`LLMExtractionStrategy`) call out to a model to pull structured data
from the crawled page, and these route through `litellm`. So in
extraction mode you get the same provider matrix as
[`aider`](../aider/) / [`llm`](../llm/): OpenAI, Anthropic, Gemini,
Bedrock, Vertex, Cohere, DeepSeek, Mistral, Groq, OpenRouter, Ollama,
llama.cpp, vLLM. Pure crawl mode needs no model at all.

## 4. MCP support

**Yes, server-side.** The Docker server exposes an MCP endpoint
alongside its REST API, so MCP-aware clients (opencode, codex, cline)
can hand it a URL and get back Markdown without any glue code. The
`crwl` CLI itself is not an MCP client.

## 5. Sub-agent model

None in the CLI. The library has a `dispatcher` concept that runs
multiple crawls in parallel (`MemoryAdaptiveDispatcher`,
`SemaphoreDispatcher`) and an `AsyncWebCrawler.arun_many()` for
batched crawls, but that is concurrency, not agency.

## 6. Telemetry stance

**Off, with no opt-in.** No analytics SDK in the package or CLI.
`crawl4ai-doctor` is a local diagnostic; it does not phone home. The
Docker server logs to its own container only.

## 7. Prompt-cache strategy

N/A — the tool produces context, it does not consume tokens
(except in extraction mode, where caching is litellm/provider's
responsibility). The crawler does maintain its own **page cache**:
`CacheMode.ENABLED` will skip re-fetching URLs you have already pulled
in this session, which is the equivalent saving for crawl-bound
workloads.

## 8. Hot keybinds

No TUI. The CLI verbs:

- `crwl <url>` — fetch a URL, dump Markdown to stdout
- `crwl <url> -o markdown-fit` — apply the "fit" content filter
  (heuristic main-article extraction; drops nav, footers, ads)
- `crwl <url> -q "<question>"` — Q&A mode; uses
  `LLMExtractionStrategy` to answer from the page (requires an LLM
  provider configured via env)
- `crwl <url> -s <schema.json>` — structured extraction against a
  JSON schema
- `crwl <url> --css 'article.main'` — CSS-selector pre-filter
- `crwl <url> --screenshot out.png` — full-page PNG via Playwright
- `crwl <url> --js "window.scrollTo(0,document.body.scrollHeight)"`
  — execute JS before extraction (handles infinite-scroll pages)
- `crwl <url> --depth 2 --max-pages 50` — multi-page crawl with the
  built-in BFS strategy

## 9. Killer feature, weakness, when to choose

**Killer feature.** The **`fit_markdown` content filter**. Most
"web-to-Markdown" tools dump the entire DOM and let the model figure
out what is content vs. chrome. Crawl4AI runs a heuristic pass
(headings, link density, text-to-tag ratio, repeated structural
patterns across sibling pages) and emits *only* the article body, with
clean inline links and image alt-text preserved. On a typical
content-heavy page this cuts the token count by 5–10× compared to a
raw `html2text` dump while keeping everything the model actually needs
to reason. Combined with full Playwright support (so SPAs and
JS-rendered pages Just Work), it is the highest-leverage piece in the
"give my agent a browser" toolchain.

**Weakness.** Heavy install (~300 MB browser binaries) — not the right
choice if you only ever scrape static HTML and could use `curl |
html2text`. The CLI surface is thinner than the Python API; advanced
features (custom extraction strategies, hooks, dispatchers) push you
back into Python. Server mode is the better deployment target for
serious use; the CLI is more "interactive helper" than "production
crawler."

**When to choose.**

- You are building an **agent that needs to read the web** and you
  want a single, well-maintained component to handle fetch +
  rendering + cleaning.
- You want a **drop-in MCP server** that gives your existing agent
  CLI the ability to read URLs without building it yourself.
- You need to **scrape SPA / JS-heavy sites** for downstream LLM work
  and `curl` is no longer enough.
- You want **structured extraction (JSON schema)** from web pages
  with a one-line CLI invocation.

**When not to choose.** You scrape only static HTML at small volume —
pick a lighter tool. You need a production crawl infrastructure with
distributed scheduling, deduplication, robots.txt arbitration at
scale — use a real crawler (Scrapy, Apache Nutch) and post-process.
You want to keep zero browser binaries on your build machine — there
is no reduced-install path that drops Playwright.
