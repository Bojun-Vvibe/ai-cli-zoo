# paper-qa

> Snapshot date: 2026-04. Upstream: <https://github.com/Future-House/paper-qa>

"**High accuracy RAG for answering questions from scientific
documents with citations.**" `paper-qa` (now `paper-qa2` /
`paperqa` on PyPI) is a Python library + CLI from FutureHouse
purpose-built for one task: answer a question against a corpus of
scientific PDFs, and return the answer with verbatim citations to
the supporting passages. The pipeline is *not* a generic RAG loop —
it is an agentic search-gather-rerank-summarise flow: an LLM agent
decides what to search for, retrieves chunks (locally indexed or
from web sources like Semantic Scholar / Google Scholar / Crossref /
arXiv via separate paper-search tools), generates per-chunk
"contexts" that summarise relevance to the question, reranks, and
synthesises a final answer where every claim is tied back to a
specific paper-and-page citation. The defaults — chunk size, prompt
templates, citation format, evidence count, reranker — are tuned for
the scientific-paper case (long, dense, jargon-heavy, citation-
expectant) rather than chat or generic web RAG.

## 1. Install footprint

- `pip install paper-qa` (Python ≥ 3.11). Pulls `litellm`, `pydantic`,
  `tantivy` (local full-text index), `pymupdf` (PDF parsing),
  `numpy`, `tiktoken`, `tenacity`, `aiohttp`. Optional extras for
  alternate parsers / search backends (`paper-qa[ldp]` for the
  Language Decision Process agent backend, plus the separate
  `paper-search-mcp` tool).
- Ships a CLI: `pqa`. Common subcommands: `pqa ask "<question>"`
  (answer using the default index over the current directory's
  PDFs), `pqa -i <name> add <path>` (add files to a named index),
  `pqa -i <name> search "<query>"` (search a built index without
  generating an answer), `pqa view <answer-id>`, `pqa
  set <key> <value>` (write to the persistent settings file), `pqa
  index` (build / refresh an index). The CLI is a thin wrapper on
  the `Settings` + `ask` / `agent_query` Python API; both surfaces
  are first-class.
- Local index lives at `~/.pqa/<index-name>/` by default — tantivy
  full-text + a JSON manifest of parsed papers + an embedding cache.

## 2. Repo + version + license

- Repo: <https://github.com/Future-House/paper-qa>
- Latest release: **v2026.03.18** (2026-03, calver scheme on
  `main`)
- License: **Apache-2.0** —
  <https://github.com/Future-House/paper-qa/blob/main/LICENSE>
- Default branch: `main`
- Language: Python

## 3. Models supported

Anything `litellm` can route — OpenAI (Chat + Responses + Azure),
Anthropic (Claude 3.5 / 3.7 / 4 with extended thinking), Gemini
(AI Studio + Vertex), Bedrock, Mistral, Cohere, Groq, DeepSeek
(V3, R1), Together, Fireworks, OpenRouter, Cerebras, xAI,
Perplexity, plus local Ollama, vLLM, llama.cpp, LM Studio, mlx-lm
via OpenAI-compatible endpoints. Configured per-role:
`Settings.llm` (synthesis), `Settings.summary_llm` (per-chunk
context summarisation — typically a cheaper model), `Settings.
agent.agent_llm` (the search-and-gather agent — typically a
stronger model), and `Settings.embedding` (default `text-
embedding-3-small`; swap to a local sentence-transformers model
for fully-local). The "small model for many calls, big model for
the final answer" split is the load-bearing cost optimisation for
real corpora.

## 4. MCP support

Yes (via the sister `paper-search-mcp` server, separate repo). The
core `paper-qa` CLI does not embed an MCP server, but the project
ships an MCP-compatible search tool (`paper-search-mcp`) that
exposes `arxiv_search`, `pubmed_search`, `semantic_scholar_search`,
`crossref_search`, etc. as MCP tools — a Claude Desktop or Cursor
host can call `paper-qa`'s search side directly. Going the other
way, exposing `pqa ask` as an MCP server is ~30 lines of FastMCP
wrapper around the `agent_query` async function.

## 5. Sub-agent model

One agent loop, several specialised LLM roles. The default flow is
the `paper_search` agent: an LLM that, given a question, repeatedly
calls `paper_search(query)` (against the local tantivy index, with
optional web fallbacks), `gather_evidence(question)` (which fans
out per-chunk summary calls to `summary_llm`), and `gen_answer()`
(the final synthesis call to `llm`). The agent decides when it has
enough evidence and stops. No fan-out across multiple agents in
parallel; the parallelism is inside `gather_evidence` (concurrent
per-chunk summarisation, configurable via `Settings.answer.evidence_k`
and `concurrency`). The optional `ldp` (Language Decision Process)
backend is a more sophisticated agent backbone that can be trained
with RL on agent-task data — for almost everyone the default
`ToolSelector` agent is the right pick.

## 6. Telemetry stance

Off. The library makes no analytics calls of its own; egress is
exactly your configured `litellm` LLM calls plus optional web
search calls (Semantic Scholar / Crossref / Google Scholar) which
you opt into by configuring API keys. Tracing hooks via
`Settings.callbacks` (any litellm-compatible callback handler —
Langfuse, Phoenix, Logfire, MLflow). The local index is a flat
directory under `~/.pqa/`; nothing is uploaded.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **Verbatim, page-anchored citations are a
first-class output, not a bolt-on.** Every claim in the
synthesised answer carries a `(Author Year, page X)` style
citation back to the exact chunk it came from, and the
`PQASession` object exposes the full evidence chain — which
papers were searched, which chunks were retrieved, what each
chunk's `summary_llm`-generated relevance summary said, and
which subset survived to the final synthesis prompt. For any
domain where "the model said it" is not enough (lit reviews,
regulatory filings, due diligence, scientific writing, expert
review), this audit trail is the actual product. The two-tier
LLM split (cheap `summary_llm` for the per-chunk fan-out,
expensive `llm` for the final synthesis) plus the local tantivy
index make running queries against a 1000-paper corpus genuinely
affordable — typical end-to-end cost is dollars rather than tens
of dollars, while a naïve "stuff every chunk into the context" RAG
would be 10–50× more. The `pqa` CLI gets you from "folder of
PDFs" to "first answer with citations" in two commands.

**Weakness.** **The defaults are tuned for English-language
scientific PDFs, and the abstraction is opinionated.** If your
corpus is HTML, code, scanned forms, tables-only, or a non-Latin
script, you will fight the parser (`pymupdf` is competent but not
miraculous; consider running [`marker`](../marker/) or
[`docling`](../docling/) upstream and feeding text in). Index
building is single-process and not blazing fast on large corpora
(10k+ papers warrant patience and the embedding-cache path). The
agent loop costs a non-trivial number of LM calls per question
(typically 5–20 search / summary / synthesis calls); a single
direct RAG retrieval + answer would be cheaper if you do not need
the search-iteration behaviour. The CLI surface is small —
genuinely useful for the "ask a folder a question" loop, less so
as a building block for a larger pipeline (use the Python API).
The web-search backends require their own API keys (Semantic
Scholar is free but rate-limited; Google Scholar via SerpAPI is
paid).

**When to choose.** You have a corpus of PDFs (papers,
filings, contracts, reports, dissertations, manuals) and you
need an answer with citations you can verify by clicking through
to page X of paper Y. Pair with [`marker`](../marker/) or
[`docling`](../docling/) upstream if your PDFs are scanned or
table-heavy and `pymupdf` mis-parses; pair with
[`langfuse`](../langfuse/) for tracing the multi-call agent
loop; pair with [`ragas`](../ragas/) for citation-faithfulness
eval against a labelled question set. Skip if your data is
chat-style or web-style — a generic RAG framework
([`llama-index`](../llama-index/), [`haystack`](../haystack/),
[`txtai`](../txtai/)) will be more flexible. Skip if you need
multi-tenant production serving with SLAs — `paper-qa` is a
local-first research-and-analysis tool, not a serving framework
(wrap it in your own FastAPI if you need that). Skip if
citations are nice-to-have rather than must-have — the agent
loop's cost is the price of the citation guarantee, and a
simpler retriever-plus-prompt is cheaper when you can live
without it.
