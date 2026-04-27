# crawlee-python

> Snapshot date: 2026-04. Upstream: <https://github.com/apify/crawlee-python>
> Pinned release: `v1.6.2`. License file: `LICENSE` (sha `bc3bad0dfb2a0c218eda2dab8cc26a143e9d1689`).

A web-scraping and browser-automation framework with a `crawlee` CLI for
project scaffolding. Not an LLM client itself — it sits one layer below,
producing the clean HTML / JSON / Markdown that downstream LLM CLIs and
RAG pipelines actually want to ingest. The Python sibling of the
much-older Node `crawlee`.

## 1. Install footprint

- `pipx install 'crawlee[all]'` for the full feature set (Playwright,
  BeautifulSoup, Parsel, HTTPX backends).
- `crawlee create my-crawler` scaffolds a project with templates for
  Playwright, BeautifulSoup, or HTTP-only flavors.
- Playwright extra pulls in ~400 MB of browser binaries on first
  `playwright install`.
- Python 3.10+. Linux/macOS first-class; Windows works for HTTP backends
  but Playwright is finicky.

## 2. License

Apache-2.0.

## 3. Models supported

None directly. Crawlee is the input stage of an LLM pipeline, not an
LLM client. Common patterns: pipe `crawlee` output into `llm`,
`fabric`, or `files-to-prompt`; or use it as the fetcher inside a
custom LangChain / LlamaIndex loader.

## 4. MCP support

**No.** It exposes Python APIs, not an MCP server. You can wrap a
crawl in an MCP tool yourself (Apify publishes an MCP-shaped Actor
template separately) but the core library is a normal SDK.

## 5. Sub-agent model

N/A — there is no agent loop. The unit of concurrency is the
**request handler**: each URL is dispatched to an async worker, with
configurable concurrency, retries, proxy rotation, and session
pooling. Crawlee handles thousands of concurrent fetches without
spawning subprocesses.

## 6. Telemetry stance

Off by default. The Apify SDK telemetry only activates when you
explicitly run on the Apify platform via `apify push`. Local runs
make zero outbound calls beyond your target sites and configured
proxies.

## 7. Prompt-cache strategy

N/A — not a model client. The relevant cache here is the
**request queue**: deduplicates URLs, persists across runs to a
`storage/` directory, lets you resume a half-finished crawl without
re-fetching pages.

## 8. Hot keybinds (CLI)

The CLI is project-scaffolding and run orchestration:

| Command | Action |
|---------|--------|
| `crawlee create <name>` | Scaffold a new crawler project |
| `crawlee run` | Run the crawler defined in `src/main.py` |
| `crawlee --help` | Show flags |
| `Ctrl+C` (during run) | Graceful shutdown; in-flight requests finish, queue is persisted |

The bulk of authoring happens in Python source, not at the CLI.

## 9. Killer feature, weakness, when to choose

- **Killer:** the **unified abstraction across HTTP, BeautifulSoup,
  Parsel, and Playwright backends**. Switch from a 1000-req/sec HTTP
  crawler to a JS-rendering Playwright crawler by changing one
  import — same request queue, same dataset writer, same session
  pool. Lets you start cheap and escalate only the pages that need
  it.
- **Weakness:** Python-only and code-first. There's no "give me a
  YAML config and a URL" mode like the Node `crawlee` has via
  templates. If you don't want to write Python, use `firecrawl` or
  `crawl4ai` instead.
- **Choose it when:** you're feeding a RAG index or fine-tuning
  dataset and you need a production-grade fetcher with retries,
  proxies, deduplication, and the option to drop into a real browser
  for the JS-heavy pages — without rewriting your pipeline.
