# gpt-researcher

> Snapshot date: 2026-04. Upstream: <https://github.com/assafelovic/gpt-researcher>.
> Pinned version: **v3.4.4** (2026-04-16).
> License: **Apache-2.0** — `LICENSE` at the repo root
> (<https://github.com/assafelovic/gpt-researcher/blob/main/LICENSE>).

An autonomous research agent that takes a natural-language question, plans a
multi-step search strategy, fans out parallel sub-queries against the open
web (or your own document set), and emits a cited long-form report. The CLI
ships as a Python entrypoint (`python -m gpt_researcher.cli`) plus a
launchable FastAPI server; the agent loop is the same in both surfaces.

## 1. Install footprint

- `pip install gpt-researcher` (Python 3.11+) or `pipx install gpt-researcher`
  for an isolated CLI install.
- ~250 MB of Python deps once `langchain`, `playwright`, `tiktoken`, and the
  PDF / DOCX parsers land. `playwright install chromium` adds another
  ~150 MB if you want the JS-rendering scraper.
- Docker image and `docker-compose.yml` are first-class for the server mode.
- macOS / Linux first-class; Windows works.

## 2. License

Apache-2.0 — `LICENSE` at the repo root
(<https://github.com/assafelovic/gpt-researcher/blob/main/LICENSE>).

## 3. Models supported

- **LLMs**: OpenAI (default), Anthropic, Google Gemini, Groq, Mistral,
  Together, HuggingFace, Ollama, any OpenAI-compatible endpoint via
  `OPENAI_BASE_URL`. Per-role model assignment — you can put a cheap
  model on the "fast" role (query rewriter) and a frontier model on the
  "smart" role (final report writer).
- **Embeddings**: OpenAI, Cohere, HuggingFace, Ollama, Google.
- **Search backends**: Tavily (default), DuckDuckGo, SerpAPI, Searx,
  Bing, Google CSE, plus a `--report-source local` mode that skips the
  web entirely and researches a local document folder.

## 4. MCP support

Yes (client). The `MCPRetriever` lets the research agent pull tool
results from any MCP server alongside its built-in web retrievers, so
you can mount a private knowledge base, a Jira / Linear tool, or a
SQL-over-MCP server as a first-class research source.

## 5. Sub-agent model

Hierarchical multi-agent pipeline. The default flow is
`Planner → Researchers (parallel) → Reviewer → Writer → Publisher`.
Researchers fan out one per sub-question and run concurrently; the
Reviewer can bounce a draft back for another research pass. The
`langgraph`-based "deep research" mode adds a recursive
question-decomposition loop on top.

## 6. Telemetry stance

Off in the OSS codebase. No analytics calls. Egress is exactly: your
configured LLM provider, your configured search backend, and the URLs
the agent visits during scraping.

## 7. Prompt-cache strategy

Per-research-task in-memory deduplication of fetched URLs and
already-summarised chunks; no cross-run prompt cache. Anthropic
prompt-caching headers are forwarded when you point it at Claude.

## 8. Hot keybinds

CLI is non-interactive — you pass the query and report flags, and it
streams progress logs until the report file is written. The optional
Next.js front-end (`docker compose up`) exposes a chat UI with
streaming citations.

## 9. Killer feature, weakness, when to choose

**Killer feature.** A research-shaped agent loop that *cites everything*:
every paragraph in the final report carries inline source links back to
the URLs the researchers fetched, and the run produces both a Markdown
report and a structured JSON of `{question, sub_questions, sources,
findings}` you can pipe into downstream tooling. Per-role model
assignment + parallel researchers means a 20-source report finishes in
two to three minutes on a frontier model, not twenty.

**Weakness.** Quality is bounded by your search backend — Tavily is the
default for a reason; DuckDuckGo / Searx fallbacks lose a lot of
recall. The default scraper does not execute JavaScript unless you
opt into Playwright, so SPA-only sources read as empty pages. No
built-in eval harness; you grade reports by reading them.

**When to choose.** "I need a written, sourced answer to a research
question, not a chat thread." Pick this over a generic coding agent
when the deliverable is a report with citations rather than a code
diff, and over a single-shot LLM call when you need breadth (10+
sources) and structured outputs.

## Quick example

```bash
# install
pipx install gpt-researcher

# minimal env
export OPENAI_API_KEY=sk-...
export TAVILY_API_KEY=tvly-...

# run a research task → writes report to ./outputs/
python -m gpt_researcher.cli \
  --query "What are the trade-offs between RAG and long-context LLMs in 2026?" \
  --report_type research_report \
  --tone objective

# local-only mode: research a folder of PDFs / md without web egress
python -m gpt_researcher.cli \
  --query "What does our 2025 architecture doc say about the inference path?" \
  --report_source local \
  --doc_path ./design-docs/
```
