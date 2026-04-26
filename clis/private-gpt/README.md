# private-gpt

> Snapshot date: 2026-04. Upstream: <https://github.com/zylon-ai/private-gpt>
> License file: <https://github.com/zylon-ai/private-gpt/blob/main/LICENSE>
> Pinned: `v0.6.2` (2024-08-08). The repo continues to receive commits on
> `main` after the last tagged release; the v0.6.2 tag is the last stable
> reference point. Pin a commit SHA in production rather than tracking
> `main`.

A Python application (CLI launcher + FastAPI + Gradio UI) that ships a
**fully self-hostable RAG-over-your-documents stack** behind one
`make run` invocation. Same goal as
[`anything-llm`](../anything-llm/) and [`r2r`](../r2r/) and the
ingestion side of [`llmware`](../llmware/), but with a deliberately
narrower scope: **everything runs on your machine, no third-party
service is required at any point**, and the OpenAI-compatible HTTP
surface lets the rest of this catalog point at it as a model + RAG
backend.

The "private" in the name is the design constraint, not just
marketing: every component (LLM, embedder, vector store, document
parser, web UI) has a local-only configuration documented in the
`settings-*.yaml` profiles, and the default profile boots with
zero outbound network calls after weights are downloaded.

## 1. Install footprint

- Python 3.11 + Poetry. `git clone && poetry install --extras
  "ui llms-llama-cpp embeddings-huggingface vector-stores-qdrant"`
  is the canonical local-only install. Extras are namespaced
  (`llms-*`, `embeddings-*`, `vector-stores-*`, `ui`, `rerank-*`)
  so you opt into the dependency cone of the components you actually
  use.
- Default local profile pulls a quantised GGUF model (default
  ~4 GB), runs it via `llama-cpp-python`, embeds with a HuggingFace
  sentence-transformer (~100 MB), and stores vectors in Qdrant
  (in-process or as a sidecar Docker container).
- Config is **YAML profiles**: `settings.yaml` is the base, then
  `PGPT_PROFILES=local make run` overlays `settings-local.yaml`,
  `PGPT_PROFILES=ollama make run` overlays `settings-ollama.yaml`,
  etc. Profiles compose, so you can keep credentials in
  `settings-prod.yaml` and stack `local,prod` for dev.
- Disk: weights + Qdrant index + ingested document copies. A
  10k-PDF corpus on default chunking lands around 2-4 GB of vector
  + payload storage on top of the model.
- Workspace footprint per project: `local_data/` holds the Qdrant
  collection and the original-document store; nothing leaves that
  directory unless you explicitly enable a cloud profile.

## 2. License

Apache-2.0.

Zylon (the company behind private-gpt) sells a commercial managed
build on top, but the OSS CLI / server is unencumbered Apache-2.0
including for production use.

## 3. Models supported

LLM backends are pluggable via the `llm.mode` setting:

- **`llamacpp`** — local GGUF via `llama-cpp-python`. Default. Any
  GGUF Llama / Qwen / Mistral / Mixtral / Phi / Gemma / DeepSeek
  works; configurable `llm_hf_repo_id` + `llm_hf_model_file` pulls
  from HuggingFace on first run.
- **`ollama`** — point at a local [`ollama`](../ollama/) daemon
  (default `http://localhost:11434`); inherits Ollama's full model
  catalog.
- **`openai`** — any OpenAI-compatible endpoint
  (`api_base` overridable), so [`vllm`](../vllm/) /
  [`openllm`](../openllm/) / [`localai`](../localai/) /
  [`litellm`](../litellm/) proxies all work, as does the real
  OpenAI API if you opt in.
- **`sagemaker`** — AWS SageMaker endpoints.
- **`mock`** — deterministic stub for tests / CI.

Embedding backends (`embedding.mode`):

- **`huggingface`** — local sentence-transformers (default
  `BAAI/bge-small-en-v1.5`).
- **`ollama`** — Ollama embedding models.
- **`openai`** / **`sagemaker`** / **`mock`** — same shape as LLM.

Vector stores (`vectorstore.database`): `qdrant` (default,
in-process or sidecar), `chroma`, `postgres` (pgvector),
`clickhouse`, `milvus`. Reranker is optional (`rerank.enabled:
true`) and supports HuggingFace cross-encoders or remote APIs.

The full local profile (llama.cpp + HF embeddings + Qdrant +
optional HF reranker) makes **zero network calls** after weight
download.

## 4. MCP support

**No** as a server. private-gpt exposes an OpenAI-compatible HTTP
API at `/v1/chat/completions` and a richer ingestion / chat / search
API at `/v1/ingest`, `/v1/chunks`, `/v1/completions`,
`/v1/recipes/summarize`, but it does not speak MCP.

To use it from an MCP-aware agent
([`claude-code`](../claude-code/), [`opencode`](../opencode/),
[`goose`](../goose/), [`fast-agent`](../fast-agent/)), point the
agent at the OpenAI-compatible URL as a model backend; the
ingestion API is best wrapped as your own MCP server if the agent
needs to add documents on the fly.

## 5. Sub-agent model

None. The "agent" surface is a single RAG loop: query →
embedding → top-K retrieval (optionally reranked) → prompt
templating → LLM call → response. There is no planner, no tool
loop, no multi-agent composition. The `recipes/` API exposes a
small set of pre-built flows (`summarize` is the only canonical
one as of v0.6.2) but they are templated single-shot calls, not
agent loops.

## 6. Telemetry stance

**Off, no opt-in.** No analytics SDK in the codebase. The default
`local` profile makes zero outbound calls after model weights are
downloaded. Cloud profiles obviously contact the configured
provider (OpenAI, SageMaker, Ollama URL, etc.); the OSS server
itself does not phone home, and the Gradio UI binds to
`127.0.0.1` by default — flip to `0.0.0.0` in `server.host` only
if you intentionally want LAN exposure.

## 7. Prompt-cache strategy

Provider-pass-through. With the `llamacpp` backend you inherit
`llama.cpp`'s KV cache (`n_ctx`-bounded, intra-conversation only).
With `openai` mode against an Anthropic-compatible proxy you
inherit that provider's prompt cache. There is no
private-gpt-level cross-request cache layer; the only persistent
cache is the embeddings + chunks index in the configured vector
store, which is reused across queries by design.

## 8. Hot keybinds

No native TUI. The Gradio web UI (default
`http://127.0.0.1:8001`) provides three tabs: **Query Files**
(RAG over the ingested corpus), **Search Files** (raw retrieval
without LLM), and **LLM Chat** (chat with no retrieval). CLI
surface:

- `make run` / `PGPT_PROFILES=local make run` — boot the FastAPI
  server + Gradio UI under the named profile stack.
- `make ingest -- /path/to/folder` — bulk-ingest a directory of
  documents (PDF / DOCX / PPTX / XLSX / HTML / Markdown / EPUB /
  CSV / image-with-OCR) into the vector store.
- `make wipe` — drop the local vector store and document store;
  starts ingestion from a clean slate.
- `make stats` — print collection sizes, document counts, and
  approximate disk footprint.
- `make api-curl` (developer convenience) — curl the
  OpenAI-compatible endpoint to verify the server is up.
- Any HTTP client against `/v1/ingest/file`,
  `/v1/ingest/folder`, `/v1/chunks`, `/v1/chat/completions`,
  `/v1/completions`, `/v1/embeddings`, `/v1/recipes/summarize`.

UI hotkeys are stock Gradio: `Ctrl-Enter` submits, `Ctrl-K`
focuses the input, file-upload widget accepts drag-and-drop.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **A complete private RAG stack with one `make
run`.** Most of the alternatives in this catalog give you one
component (an embedder, a vector DB, a chunker, a reranker, an LLM
runtime) and assume you will assemble the rest. private-gpt
assembles them, picks defaults that actually work, and exposes the
result as both a Gradio UI for end users and an OpenAI-compatible
HTTP API for everything else. The default `local` profile boots
with **zero third-party network dependencies** — model weights
fetched from HuggingFace on first run, then air-gapped — which is
the unique selling point against [`anything-llm`](../anything-llm/)
(desktop-app shaped, less air-gappable),
[`langflow`](../langflow/) (visual DAG builder, not a finished
RAG), and direct framework assembly with
[`llama-index`](../llama-index/) /
[`haystack`](../haystack/) (you build the assembly yourself). For
"my company will not let me send PDFs to OpenAI but I need
ChatGPT-over-our-docs by Friday", `make run` is the answer.

**Weakness.** **Release cadence has slowed** — v0.6.2 was tagged in
August 2024, with subsequent work landing on `main` but no new
tagged release; pin a SHA. The application surface is
deliberately narrow (RAG over documents and a chat tab) — it is not
a workflow builder, not an agent framework, not a multi-tenant SaaS
backplane. Default chunking and prompt templates are tuned for
English narrative prose; Asian languages, code corpora, and
heavily structured documents (forms, invoices) need profile
customisation. Quality is bounded by the local model you choose:
running on a 7B Q4_K_M GGUF gives noticeably weaker answers than
the same RAG pipeline against GPT-4o, and that is the price of
"private" — the stack does not hide it from you.

**When to choose.**

- You need **chat-with-your-documents** running on a single
  workstation or air-gapped server with **no third-party data
  egress**, and you want to be productive in a day, not a sprint.
- You want a **finished Gradio UI** that non-technical users can
  hit, plus an **OpenAI-compatible HTTP endpoint** that the rest
  of this catalog (any agent CLI) can point at as a model + RAG
  backend.
- You want **swappable backends** for LLM (llama.cpp / Ollama /
  OpenAI-compatible) and vector store (Qdrant / Chroma / pgvector
  / ClickHouse / Milvus) configured by **YAML profile**, not by
  forking Python.
- You are evaluating self-hosted RAG stacks against
  [`anything-llm`](../anything-llm/) /
  [`r2r`](../r2r/) / [`langflow`](../langflow/) and the deciding
  axis is "fully local default profile, no telemetry, no
  required cloud control plane".

**When not to choose.** You want a **multi-agent / tool-using
agent**, not a RAG chat app — pick [`openhands`](../openhands/),
[`crewai`](../crewai/), [`pydantic-ai`](../pydantic-ai/). You want
a **visual workflow builder** to compose retrieval + prompt + tool
chains — pick [`langflow`](../langflow/) or [`flowise`](../flowise/).
You want a **production-grade RAG framework** to embed in your own
app rather than a finished app — pick
[`llama-index`](../llama-index/), [`haystack`](../haystack/), or
[`r2r`](../r2r/). You want **active feature velocity on the OSS
side** — Zylon's commercial product is where the new work lands;
the OSS repo's tagged-release cadence is slow.
