# llama-index

> Snapshot date: 2026-04. Upstream: <https://github.com/run-llama/llama_index>

"**The leading document agent and OCR platform.**" `llama-index`
(formerly `gpt_index`) is a Python framework for building LLM
applications over your own data: ingest documents (PDF, DOCX,
Markdown, HTML, Notion, Slack, Google Drive, S3, dozens more via
the LlamaHub `llama-index-readers-*` packages), chunk and embed
them, store in any of ~40 supported vector stores, then query with
a `QueryEngine` that handles retrieval, reranking, response
synthesis, and citation. Beyond plain RAG, the project has spent the
2024-26 cycle leaning hard into *agentic* document workflows — the
`Workflow` event-loop API, `FunctionAgent` / `ReActAgent` /
`CodeActAgent`, multi-step query engines (`SubQuestionQueryEngine`,
`QueryPipeline`), and `LlamaParse` for OCR / table extraction over
complex PDFs. Sister `LlamaCloud` product is hosted; the OSS core
runs entirely local.

## 1. Install footprint

- `pip install llama-index` (Python ≥ 3.9). The meta-package pulls
  `llama-index-core`, the OpenAI LLM + embedding integrations, and
  a default in-memory vector store. *Everything else* is a separate
  `llama-index-<kind>-<name>` package — `llama-index-llms-anthropic`,
  `llama-index-llms-gemini`, `llama-index-llms-ollama`,
  `llama-index-embeddings-huggingface`,
  `llama-index-vector-stores-qdrant`,
  `llama-index-readers-file`, `llama-index-tools-mcp`, etc. The
  modular layout means a real install is typically 5–20 packages;
  the `llama-index` umbrella exists for "give me the OpenAI happy
  path."
- Ships a CLI: `llamaindex-cli` (alongside the legacy `llamaindex`
  alias). Subcommands: `llamaindex-cli rag --files <dir>` (one-shot
  index + chat over a folder), `llamaindex-cli download-llamapack`
  (fetch a LlamaPack template), `llamaindex-cli download-llamadataset`
  (fetch an eval dataset), `llamaindex-cli new-package` (scaffold a
  new integration package). The `rag` subcommand is the most-used
  surface — it is the fastest way to point an LLM at a folder and
  ask questions.
- `LlamaParse` is shipped as a separate `llama-parse` package and
  uses the hosted LlamaCloud API by default (free tier exists);
  there is no fully-local OCR equivalent inside the core repo.

## 2. Repo + version + license

- Repo: <https://github.com/run-llama/llama_index>
- Latest release: **v0.14.21** (2026-04, weekly cadence on `main`)
- License: **MIT** —
  <https://github.com/run-llama/llama_index/blob/main/LICENSE>
- Default branch: `main`
- Language: Python (TypeScript port lives at `run-llama/LlamaIndexTS`,
  separate repo)

## 3. Models supported

Anything with an `llama-index-llms-*` integration package — and the
catalog is exhaustive. First-party packages for OpenAI (Chat +
Responses + Azure), Anthropic, Gemini (AI Studio + Vertex), Bedrock,
Mistral, Cohere, Groq, DeepSeek, Together, Fireworks, OpenRouter,
Cerebras, xAI, Perplexity, Replicate, Predibase, Anyscale, Watsonx,
plus local Ollama, vLLM, llama.cpp, LM Studio, mlx-lm,
HuggingFace transformers / TGI / TEI, NVIDIA NIM, Dashscope
(Qwen). Embeddings: OpenAI, Cohere, Voyage, Jina, Mistral, Gemini,
Bedrock, plus local sentence-transformers / fastembed / mlx-embed /
Ollama. Multi-modal `MultiModalLLM` interface for vision models.
Provider switching is one constructor swap (`Settings.llm =
OpenAI(...)` → `Settings.llm = Anthropic(...)`); the `QueryEngine`
above does not change.

## 4. MCP support

Yes (client + helpers for server). `llama-index-tools-mcp` exposes
`McpToolSpec` — point it at an MCP server (stdio or SSE) and its
tools become `FunctionTool` objects usable by `FunctionAgent` /
`ReActAgent` / any `Workflow`. The companion `BasicMCPClient`
handles the connection lifecycle. Going the other way, the
`create-llama` scaffold and several LlamaPacks ship templates that
expose a `QueryEngine` *as* an MCP server (FastAPI + an MCP server
adapter), so you can plug a llama-index RAG pipeline into Claude
Desktop, Cursor, or any MCP host without writing the server by hand.

## 5. Sub-agent model

Two layers. The classic layer is the `QueryEngine` family —
`SubQuestionQueryEngine` decomposes a complex question into
sub-questions, dispatches each to a child `QueryEngine` (possibly
over a different index), and synthesises; `RouterQueryEngine`
classifies a question and routes to one of N child engines;
`QueryPipeline` is a DAG of operators (retriever → reranker →
synthesiser → citation). The newer agentic layer is the
`Workflow` API: an event-driven state machine where `@step`-decorated
methods consume and emit typed `Event` objects, including parallel
`ctx.send_event(...)` fan-out and `ctx.collect_events(...)` fan-in.
Multi-agent setups use `AgentWorkflow` (a curated `Workflow`
subclass) where each agent is a `FunctionAgent` with its own LLM,
tools, and `can_handoff_to=[other_agent]` policy — the framework
handles the handoff message conventions and shared context. No
manager-LLM-spawning-worker-LLMs autoscaler; the topology is
declared in the workflow definition.

## 6. Telemetry stance

Off in the OSS core. Optional global instrumentation via
`llama_index.core.instrumentation` (custom dispatcher / event
handlers), plus first-class adapters for OpenTelemetry,
Phoenix / Arize, Langfuse, MLflow, Weights & Biases, OpenLLMetry,
Literal AI, Honeycomb, and DeepEval — all opt-in imports. The CLI
makes no analytics calls; the only network egress from the OSS
package is what your configured LLM / embedding / vector store
clients make. `LlamaParse` (separate package) does call LlamaCloud
by default — if you want fully-local PDF OCR, swap in `pymupdf` /
`unstructured` / `docling` via the readers layer.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **The richest catalog of data-loading,
indexing, and query-time primitives in any RAG framework, and a
genuine agentic story on top of it.** ~300 first-party reader
packages cover the long tail of "where is your data" (S3,
SharePoint, Notion, Confluence, Slack, Discord, Gmail, Jira,
GitHub, Wikipedia, arXiv, YouTube transcripts, Google Drive,
Airtable, Asana, Salesforce, Stripe, Zendesk…), ~40 vector store
integrations cover "where do you put it", and the query layer
gives you composable knobs (hybrid search, reranking, recursive
retrieval over node summaries, auto-merging retrievers,
citation-preserving response synthesis) that you would otherwise
write by hand. The `Workflow` API is one of the cleaner agent
runtimes in Python — typed events, explicit state, native
parallelism, native streaming, and it composes with the
`QueryEngine` machinery instead of replacing it. `LlamaParse` is
genuinely best-in-class for ugly PDFs (multi-column scientific
papers, financial filings with embedded tables, scanned forms)
when you can accept a hosted dependency. The CLI's
`llamaindex-cli rag --files ./docs` is the one-line "ask
questions of a folder" demo that actually works.

**Weakness.** **The package sprawl is real, and the framework
is large enough that there are usually three ways to do anything.**
A real project's `pyproject.toml` carries 10–20 `llama-index-*`
sub-packages and figuring out which one provides the integration
you want involves searching the LlamaHub. Documentation is broad
but uneven — the headline tutorials are excellent, but second-tier
features (some `QueryPipeline` operators, less-common readers,
older agent classes that pre-date `Workflow`) are sometimes
documented only by example notebook or by source. The 0.10 → 0.11
→ 0.12 → 0.13 → 0.14 series broke imports repeatedly as packages
moved between `core` and standalone packages; pin tightly and read
the migration notes. The agent layer overlaps in spirit with
[`langgraph`](../langgraph/) / [`pydantic-ai`](../pydantic-ai/) /
[`crewai`](../crewai/) / [`swarms`](../swarms/) / [`dspy`](../dspy/);
choosing llama-index for the agent story specifically (rather
than for the RAG story plus an agent on top) is a less obvious
pick. Performance at scale (10M+ chunks) requires care — the
default in-memory store and naive retriever are demo-grade; real
deployments use a real vector DB and a reranker.

**When to choose.** Your problem is fundamentally about getting
an LLM to answer questions or take actions over *your data* —
documents, knowledge base, support tickets, codebase, research
corpus — and you want a framework that has already solved the
"how do I read 40 file formats and put them in 40 vector stores"
problem so you can focus on retrieval quality and answer
synthesis. Pair with [`docling`](../docling/) or
[`marker`](../marker/) for fully-local PDF parsing if you cannot
use LlamaParse, with [`qdrant`](../qdrant/) /
[`chroma`](../chroma/) / [`weaviate`](../weaviate/) /
[`marqo`](../marqo/) / [`lancedb`](../lancedb/) for the vector
store, with [`ragas`](../ragas/) for retrieval-quality eval, and
with [`langfuse`](../langfuse/) / [`arize-phoenix`](../arize-phoenix/)
for production tracing. Skip if your data fits in the model's
context window — a long-context call to Gemini or Claude with
the whole document inline is simpler than any RAG pipeline. Skip
if you want a single-purpose framework — `haystack`,
[`txtai`](../txtai/), or a hand-rolled retriever over
[`embedchain`](https://github.com/mem0ai/mem0) / [`mem0`](../mem0/)
will be smaller surface area. Skip if your "agentic" needs are
better served by a typed-program optimiser ([`dspy`](../dspy/))
or a topology zoo ([`swarms`](../swarms/)) — llama-index agents
are RAG-shaped first, generic-agent second.
