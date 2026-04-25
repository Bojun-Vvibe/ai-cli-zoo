# scrapegraphai

> Snapshot date: 2026-04. Upstream: <https://github.com/ScrapeGraphAI/Scrapegraph-ai>
> License file: <https://github.com/ScrapeGraphAI/Scrapegraph-ai/blob/main/LICENSE>
> Pinned: **v2.0.0** (2026-04-19, PyPI: `scrapegraphai`).
> Companion docs: <https://scrapegraph-ai.readthedocs.io/>.

ScrapeGraphAI is **LLM-driven web scraping framed as a directed
graph**. You hand it a URL (or a list of URLs, or a local HTML
file, or a search query) and a *natural-language extraction prompt*
("give me a JSON list of every press release, with title, date, and
summary"), and it routes the page through a graph of typed nodes —
fetch → parse → chunk → embed → semantic-filter → LLM-extract →
schema-validate → merge — until you have structured output. The v2
rewrite consolidates everything around the
**`SmartScraperGraph`** family with first-class typed schemas via
Pydantic.

This is the orthogonal answer to [crawl4ai](../crawl4ai/): crawl4ai
gives you clean Markdown for an LLM to *read* downstream;
scrapegraphai *runs the LLM inside the scraping pipeline itself*
and emits the typed object you actually wanted, skipping the
"prompt engineer your way from Markdown to JSON" step.

## 1. Install footprint

- `pip install -U scrapegraphai` (or
  `uv pip install scrapegraphai`). Python 3.10+, <3.14.
- `playwright install chromium` (one-time browser install — ~300 MB)
  unless you stick to plain `requests` fetcher mode.
- Pulls a sizable LangChain dependency tree
  (`langchain`, `langchain-community`, `langchain-openai`,
  `langchain-anthropic`, `langchain-google-genai`,
  `langchain-mistralai`, `langchain-ollama`,
  `langchain-aws`, `langchain-fireworks`), plus
  `playwright`, `beautifulsoup4`, `html2text`, `free-proxy`,
  `tiktoken`, `pydantic`, `pandas`, `minify-html`, `pillow`,
  `googlesearch-python`. Slim install ~600 MB after Chromium.
- Storage: in-memory by default. Optional vector caches via
  Chroma / Qdrant / FAISS for RAG-shaped scrapes
  (`pip install 'scrapegraphai[other-language-models]'`).
- Hosted plane: `scrapegraphai.com` (paid). The OSS library is
  fully self-hosted; `SCRAPEGRAPH_API_KEY` enables the managed
  endpoint via the separate `scrapegraph-py` SDK.

## 2. License

MIT (verified: repo root `LICENSE`, 1066 bytes, MIT text). The
`pyproject.toml` declares `license = "MIT"`. PyPI metadata matches.

## 3. Models supported

LangChain-routed LLM-agnostic: OpenAI, Anthropic, Gemini (AI Studio
+ Vertex), Mistral, Bedrock, Azure OpenAI, Fireworks, Together,
Groq, DeepSeek, Ollama, Hugging Face Inference, vLLM, llama.cpp via
OpenAI-compatible endpoints, plus the hosted ScrapeGraphAI
endpoint. Per-graph LLM and embedding-model configuration is a
plain dict in the `config` argument — `{"llm": {"model": "openai/gpt-4o-mini"}, "embeddings": {"model": "openai/text-embedding-3-small"}}`
is the canonical shape, with `provider/model` strings interpreted
by the LangChain provider registry.

Vision-capable graphs (`SmartScraperGraph` with
`headless=False, screenshot=True`) route page screenshots through
GPT-4o / Claude 3.5 Sonnet / Gemini 1.5 Pro for layouts where DOM
extraction loses too much structure (sites with heavy CSS
positioning, image-text overlays, infinite scroll cards).

## 4. MCP support

**No first-party MCP server in v2.0.0**. Community MCP wrappers
exist around the `scrapegraph-py` hosted client for use inside
[opencode](../opencode/) / [claude-code](../claude-code/) /
[crush](../crush/) / [continue](../continue/), but they are not
canonical. The OSS library is plain Python — call it from your
own MCP server if you want one.

## 5. Sub-agent model

**Implicit through the graph.** Each node in a `SmartScraperGraph`
is a discrete unit (`FetchNode`, `ParseNode`, `RAGNode`,
`GenerateAnswerNode`, `MergeAnswersNode`, `SearchInternetNode`,
`GraphIteratorNode`); composition is by wiring nodes into a
`BaseGraph`. The shipped graphs include `SmartScraperGraph` (single
URL → typed object), `SearchGraph` (Google query → top-N URLs →
fan out → merge), `SmartScraperMultiGraph` (list of URLs in
parallel), `SpeechGraph` (page → summary → TTS), `OmniScraperGraph`
(page → text + screenshots → multimodal extraction). No
manager-LLM-spawning-worker-LLMs crew — composition is data-flow,
not role-play.

## 6. Telemetry stance

**Off by default in the OSS package.** No analytics in
`scrapegraphai/`; egress is exactly the LLM provider you configure
plus the URLs you scrape plus (optionally) `googlesearch.com` for
`SearchGraph`. The hosted `scrapegraphai.com` API is opt-in via
the separate `scrapegraph-py` package and `SCRAPEGRAPH_API_KEY`.
Playwright's bundled Chromium does not phone home; turn on
`headless=True` (the default) to avoid surfacing a visible browser.

## 7. Prompt-cache strategy

None at the ScrapeGraphAI layer. The library passes prompts
through the underlying LangChain provider, so Anthropic ephemeral
cache and OpenAI / Bedrock automatic prefix caches apply if the
upstream client is configured for them. The `RAGNode` does
embed-cache page chunks across calls within a single graph run via
an in-memory FAISS index, which materially cuts cost on
multi-question extractions over the same page.

## 8. Hot keybinds (CLI)

The `scrapegraphai` package does not ship a TUI; it is a Python
library. The terminal surface is one-shot Python invocations or
the optional `scrapegraph` console script that the hosted
`scrapegraph-py` SDK installs.

Useful one-shots once the library is installed:

```python
from scrapegraphai.graphs import SmartScraperGraph

graph = SmartScraperGraph(
    prompt="List every blog post with title, author, and date",
    source="https://example.com/blog",
    config={"llm": {"model": "openai/gpt-4o-mini",
                    "api_key": "sk-..."}},
)
result = graph.run()
print(graph.get_execution_info())   # token + latency breakdown per node
```

For a shell-pipeline shape, wrap the call in a `python -c` or
`uvx --from scrapegraphai python -m ...` invocation; the library
intentionally does not own the CLI surface, so it composes with
[`mods`](../mods/) / [`llm`](../llm/) / [`fabric`](../fabric/) by
emitting JSON to stdout.

## 9. Killer feature, weakness, when to choose

- **Killer:** **the LLM lives *inside* the scraping pipeline, not
  downstream of it.** You declare a Pydantic schema and a
  natural-language prompt; the graph fetches, parses, chunks,
  semantic-filters the irrelevant DOM, runs the LLM extraction on
  the filtered subset, validates against the schema, and re-prompts
  on validation failure. The output is the typed object you
  actually wanted — `list[BlogPost]`, not a Markdown blob you then
  have to parse a second time. For "I have 200 vendor product
  pages and I need a CSV of {sku, price, availability}" workloads,
  this is the catalog's clearest answer.
- **Weakness:** **heavy install + high per-call cost compared to a
  static scraper plus a single LLM cleanup pass.** The LangChain
  dependency tree is ~600 MB after Playwright Chromium; the
  multi-node graphs make 3-5 LLM calls per page (semantic filter +
  extract + validate-retry) before you see output. For static,
  consistently-shaped sites where one CSS selector would do, a
  `requests` + `selectolax` + one LLM rollup is an order of
  magnitude cheaper. Also: every LangChain version bump in the
  dependency tree is a regression risk because the graph nodes
  thread `Runnable` objects directly.
- **Choose it when:** the scraping target is *heterogeneous* (every
  page lays out the same data differently) and the deliverable is
  *typed structured data* (CSV / JSONL / DB rows) rather than
  Markdown for an agent to read. Pick [crawl4ai](../crawl4ai/)
  instead when the deliverable is clean Markdown for downstream LLM
  reasoning ("read this docs site and answer questions"), or
  [`marker`](../marker/) when the input is PDF rather than HTML, or
  a static scraper (`scrapy`, `requests` + `selectolax`) when the
  pages are uniform enough that one selector beats one LLM call per
  page on every axis.
