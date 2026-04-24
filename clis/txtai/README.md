# txtai

> Snapshot date: 2026-04. Upstream: <https://github.com/neuml/txtai>
> Binary name: `txtai` (console / CLI), plus `python -m txtai.api`,
> `python -m txtai.console`, `python -m txtai.workflow`

`txtai` is the **local vector-search CLI** of the catalog. Where
[`khoj`](../khoj/) is a long-lived personal-corpus daemon with a
chat UI and citations, `txtai` is one level lower: a portable
embeddings index you can build, query, save, and ship as a single
SQLite + Faiss file, driven entirely from the shell.

The mental model: an embeddings index is a *file* (`index.tar.gz`),
not a server. You build it once, copy it anywhere — laptop, edge
device, CI runner — and query it with a one-liner.

```
# Build an index from a text file (one document per line)
python -m txtai.console
> .index notes.txt
> .save notes.index

# Query it
> .search "what did I write about kafka?"

# Or run a workflow declared in YAML
python -m txtai.workflow workflow.yml "input string"
```

The console is REPL-style; `txtai.workflow` runs a declared YAML
pipeline non-interactively (good for cron / CI); `txtai.api` boots
a FastAPI server if you want HTTP. All three surfaces operate on
the same on-disk index format.

## 1. Install footprint

- Python 3.9+. `pip install txtai` pulls a CPU-friendly default
  (sentence-transformers + Faiss-CPU + SQLite) at ~600 MB.
- `pip install txtai[pipeline]` adds transformers / torch for
  summarization, transcription, captioning, translation pipelines
  (~3 GB).
- `pip install txtai[graph]` adds NetworkX for the graph-RAG mode.
- No API key required for the default local-embedding path.
  Optional OpenAI / Anthropic / Ollama backends if you want hosted
  embeddings or LLM-rerank.

## 2. License

Apache-2.0.

## 3. Models supported

- **Embeddings:** any sentence-transformers model (default
  `all-MiniLM-L6-v2`), OpenAI `text-embedding-3-*`, Cohere, any
  Hugging Face model.
- **LLM (rerank / RAG):** local llama.cpp / Ollama / Hugging Face,
  OpenAI, Anthropic, any LiteLLM-routed provider.

## 4. MCP support

None natively. The FastAPI surface (`txtai.api`) is a clean way to
wrap an index for any HTTP-aware agent.

## 5. Sub-agent model

None. Workflows are linear pipelines, not agent loops.

## 6. Telemetry stance

Off. No analytics. Fully offline by default; egress only if you
configure a hosted embedding or LLM backend.

## 7. Prompt-cache strategy

The index *is* the cache — embeddings are computed once at index
time and reused on every query. No per-call prompt caching.

## 8. Hot keybinds

`txtai.console` is a `.command`-style REPL (`.index`, `.save`,
`.load`, `.search`, `.summary`, `.help`). Workflows are declarative
YAML.

## 9. Killer feature, weakness, when to choose

**Killer feature.** Index-as-a-file portability. You can build an
embeddings index on a workstation, scp the resulting tarball to an
air-gapped box, and query it there with the same CLI — no server
to keep alive, no schema migration, no API key. Combined with the
graph-RAG and pipeline subsystems, you get a Faiss-class vector DB
plus a Hugging Face inference toolkit behind one shell surface.

**Weakness.** It is a *toolkit*, not an opinionated product. The
console is the lowest-friction surface, but the YAML workflow
language has a learning curve, and the sheer breadth (vector
search + graph + pipelines + agents + API) can make it unclear
where to start. The sentence-transformers default is fine for
English prose and mediocre for code.

**When to choose.** You want a self-contained vector index you
can ship, version, and query offline; you want the embedding /
search step decoupled from the chat-UI question. If you instead
want a *daemon* that watches a notes folder and answers chat
queries with citations, use [`khoj`](../khoj/). If you want a
session-scoped chat-with-this-folder one-shot, use
[`aichat`](../aichat/)'s built-in RAG.

**Niche vs neighbors.** [`khoj`](../khoj/) is daemon + chat UI
for one personal corpus. [`aichat`](../aichat/) is single-binary
RAG scoped to one chat session. `txtai` is the only entry whose
*output* is a portable on-disk index file — closer in spirit to
SQLite than to a chat tool. It also covers ground none of the
other entries do (graph-RAG, audio transcription pipelines,
declarative YAML workflows).
