# pyspur

> Snapshot date: 2026-04-26. Upstream: <https://github.com/PySpur-Dev/pyspur>

"**A visual playground for agentic workflows: iterate over your
agents 10× faster.**" PySpur is a self-hostable
visual-DAG editor + Python SDK for AI agents: drag-and-drop
nodes (LLM calls, tool calls, branches, loops, RAG, human-in-the-
loop) wired into a workflow that you can also write directly in
Python, with a per-node test panel, file/URL upload, dataset-driven
batch runs, and one-click deploy to a REST endpoint — the
LangGraph / Flowise / Langflow lineage with a heavier focus on
*test cases as a first-class object* attached to every node.

## 1. Install footprint

- Library: `pip install pyspur` exposes the `pyspur` Python SDK
  (workflow construction, node primitives) plus the `pyspur`
  CLI; FastAPI backend + React/Next.js front-end ship as the
  same Python package.
- CLI: `pyspur init my-project && pyspur serve --sqlite` boots
  the visual playground at `http://localhost:6080` against a
  local SQLite store; `pyspur serve` (no `--sqlite`) expects a
  Postgres URL via `DATABASE_URL` for production.
- Docker: `docker compose up -d` from the cloned repo brings up
  Postgres + backend + front-end together; the project ships a
  reference `docker-compose.yml` and a Helm-friendly image.
- Workflows: defined as JSON the editor reads/writes, or as
  Python via `from pyspur import …` — same execution engine
  either way; deploy any workflow as a versioned REST endpoint
  with `pyspur deploy`.

## 2. Repo + version + license

- Repo: <https://github.com/PySpur-Dev/pyspur>
- Latest release: **v0.1.18** (published 2025-03-25)
- License: **Apache-2.0** —
  <https://github.com/PySpur-Dev/pyspur/blob/main/LICENSE>
- Default branch: `main`
- Language: Python (FastAPI) + TypeScript (Next.js front-end)
- Stars: ~5.6k

## 3. Models supported

- LLM nodes: OpenAI (GPT-4o / 4.1 / o-series), Anthropic (Claude
  3.5 / 3.7 Sonnet, Haiku, Opus), Gemini (1.5 / 2.x), Groq,
  Mistral, Together, OpenRouter, AWS Bedrock, plus any
  OpenAI-compatible endpoint via `base_url` (Ollama, vLLM,
  LiteLLM, llama.cpp server) — routing handled by the LiteLLM
  layer underneath.
- Embedding + RAG nodes: OpenAI / Cohere / VoyageAI / Jina /
  sentence-transformers embeddings into pgvector, Chroma,
  Pinecone, or Qdrant; built-in document-chunking + retrieval
  nodes for "drop a PDF, ask questions" workflows without
  hand-wired ingestion code.
- Tool nodes: shell, HTTP, Python-eval (sandboxed), browser
  (via `browser-use` integration), MCP client, plus a generic
  "function-as-node" wrapper that lifts any Python callable
  into the editor.

## 4. Notable angle

**Test cases live next to the workflow, not in a separate test
suite.** Where [`langflow`](../langflow/) and
[`flowise`](../flowise/) give you a beautiful node editor and
then leave you to validate the graph by clicking "Run" and
eyeballing the output, PySpur attaches a **test-cases panel** to
the workflow itself: you define `(input → expected output)`
pairs once, and every iteration of the graph runs against the
whole set with diff-style output comparisons in the same UI —
the "regression suite for prompts" pattern that
[`promptfoo`](../promptfoo/) does for single prompts, lifted up
to the multi-step agent layer. Combined with the per-node
**evaluation column** (each node remembers its last N runs with
inputs / outputs / latency / cost) and the human-in-the-loop
node that pauses execution and waits for an approval webhook,
PySpur is one of the few visual builders in the catalog that is
genuinely usable as the *production* surface and not just the
prototype surface. Trade-offs: smaller community than Langflow /
Flowise (releases are infrequent — v0.1.18 from 2025-03 was
still current at snapshot), the Python SDK is younger than the
visual editor, and the deploy story assumes you are happy
self-hosting Postgres. Use it when "I want to A/B-test prompt
graph variants against a fixed test set in a UI" is the actual
job-to-be-done; pick [`langgraph`](../langgraph/) when you want
to live entirely in Python with no editor in the loop.

## 5. Last verified

2026-04-26 via `gh api repos/PySpur-Dev/pyspur` and
`gh api repos/PySpur-Dev/pyspur/releases/latest` → v0.1.18
(2025-03-25), license Apache-2.0 at
<https://github.com/PySpur-Dev/pyspur/blob/main/LICENSE>.
