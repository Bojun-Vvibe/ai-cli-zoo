# khoj

> Snapshot date: 2026-04. Upstream: <https://github.com/khoj-ai/khoj>
> Binary name: `khoj` (server) + `khoj-cli` (thin client)

`khoj` is the **personal-knowledge-base CLI** of the catalog. Where
[`aichat`](../aichat/) gives you ad-hoc RAG over a folder you point
it at right now, `khoj` runs a long-lived local server that
**continuously indexes** a configured set of sources — a Markdown
notes vault, a PDF library, an org-mode tree, a directory of code,
optionally a Notion or GitHub workspace — and exposes them through a
chat interface that knows which document each answer came from.

The architecture distinction matters for the catalog: every other
RAG-capable entry here treats the index as ephemeral (built per
session, thrown away). `khoj` treats the index as a **persistent,
incrementally-updated personal corpus** with file-watching, embedding
storage, and citation back to the source line. It is a different
shape of tool, even though the surface looks chat-like.

Three things to know about the surface:

- It is **client/server** by design. `khoj` (the server) runs as a
  background daemon (`khoj --anonymous-mode` for local-only, no
  account); `khoj-cli` (or the web UI at `http://127.0.0.1:42110`,
  or an Emacs / Obsidian plugin) is a thin client that talks to it.
  You can use the CLI exclusively if you want and never open the
  web UI.
- It is **local-first by default**. With `--offline-chat` and a
  local embedding model, the entire pipeline (indexing, retrieval,
  chat) runs on your machine via `llama.cpp` / `sentence-transformers`.
  Cloud providers are opt-in, not required.
- It separates **search** (`khoj-cli search "query"`) from **chat**
  (`khoj-cli chat "question"`). Search returns ranked passages with
  source paths; chat synthesizes an answer with inline citations. Use
  search when you want raw recall, chat when you want a synthesized
  answer.

## 1. Install footprint

- Python package: `pipx install khoj` (server + CLI in one wheel).
  Optional GPU extras: `pipx install 'khoj[gpu]'`.
- First-run cost is real: the server downloads embedding models
  (`all-MiniLM-L6-v2` by default, ~80 MB) and, if you opted into
  offline chat, a quantized chat model (~4-8 GB depending on size).
  Plan for the disk and the initial download.
- Daemon ports: HTTP API on `42110` by default. Configurable in
  `~/.khoj/khoj.yml`.
- Optional Docker image (`ghcr.io/khoj-ai/khoj`) if you would rather
  not put the Python deps on your host.
- Persistent storage: `~/.khoj/` holds the SQLite metadata DB, the
  vector index (`chromadb` or `qdrant` depending on config), and the
  configured content roots' last-indexed timestamps for incremental
  re-index.
- Python 3.10+. macOS / Linux / Windows; the GPU extras are CUDA-only.

## 2. License

AGPL-3.0 for the open-source distribution. There is also a hosted
"Khoj Cloud" SaaS run by the upstream company; this catalog entry
covers the self-hosted OSS path only. AGPL means: fine for personal
use and internal company use; redistribution as part of a closed
service requires careful reading of the license.

## 3. Models supported

Two pipelines, configured independently in `khoj.yml`:

**Embedding / retrieval pipeline:**

- Local: `sentence-transformers` models (default `all-MiniLM-L6-v2`,
  swappable for `bge-large-en`, `e5-large-v2`, etc.).
- OpenAI embeddings (`text-embedding-3-small` / `-large`) if you
  prefer cloud and have a key.

**Chat pipeline:**

- Local: any GGUF model loadable by `llama.cpp` (`--offline-chat`
  mode), or an Ollama endpoint if you prefer to manage models there.
- OpenAI: `gpt-4o`, `gpt-4o-mini`, etc.
- Anthropic: Claude family.
- Gemini: Gemini 1.5/2.x.
- Any OpenAI-compatible endpoint via `openai_api_base` in config
  (LiteLLM proxy, vLLM, LM Studio, Groq, OpenRouter).

You can mix: cloud embeddings + local chat, or local embeddings +
cloud chat, depending on your privacy / latency / cost tradeoffs.

## 4. MCP support

None as of the snapshot date. `khoj` exposes its own HTTP API
(documented OpenAPI at `http://127.0.0.1:42110/api/`) for clients,
but does not speak MCP either as client or server. If you want
`khoj`'s personal-corpus retrieval available to an MCP-aware agent
like `claude-code` or `opencode`, you would need to write a thin
MCP-server wrapper around the HTTP API yourself.

## 5. Sub-agent model

Limited. `khoj` has an internal "agent" abstraction (configurable in
the web UI / config: name, persona, default model, tool list) but
these are *user-facing chat personas*, not autonomous sub-agents in
the `Task`-tool sense. There is no planner/builder split, no
agent-spawns-agent loop. Each chat turn is one retrieval + one
generation, with optional web-search and image-generation tools
attached if enabled in config.

## 6. Telemetry stance

Off when run with `--anonymous-mode`. By default, the OSS server
asks you to create a local account on first launch, which enables a
small amount of usage telemetry to the upstream project (documented
in the privacy policy); `--anonymous-mode` skips both the account
and the telemetry.

For paranoid setups: run with `--anonymous-mode --offline-chat`,
local embedding model, no `openai_api_key` in config, and the only
network traffic is the initial model download. After that the
daemon is fully offline.

## 7. Prompt-cache strategy

Implicit only. The server constructs a fresh prompt per turn from
(retrieved passages + conversation history + user message), so
prompt-prefix caching at the provider would only fire on the
short, stable system-prompt prefix. There is no `claude-code`-style
explicit cache annotation. The retrieval cache (vector index) is
persistent and incremental, which is the cache that actually
matters for `khoj`'s workload.

## 8. Hot keybinds

`khoj-cli` is a one-shot CLI, not a TUI, so there are no keybinds
in the usual sense. The relevant ergonomics:

- `khoj-cli chat "question"` — one-shot chat, prints the answer
  with inline citations like `[1]`, `[2]` and a source list at the
  end. Exits.
- `khoj-cli chat --stream "question"` — streams tokens to stdout.
- `khoj-cli search "query" -n 10` — returns top-10 ranked passages
  as JSON or human-readable, no LLM call.
- `khoj-cli configure` — interactive config wizard for content
  sources and chat model.
- `khoj --reindex` — force a full re-index of all configured
  content roots; normally the server picks up changes via
  file-watch within a few seconds.

If you want a TUI surface, the web UI at `http://127.0.0.1:42110`
is the supported interactive client; the CLI is intentionally
scriptable.

## 9. Killer feature, weakness, when to choose

**Killer feature.** A persistent, incrementally-indexed
personal-knowledge-base service that you can chat against from the
CLI, with citations back to source files and the option to run
fully offline. No other entry in the catalog gives you "background
daemon that keeps watching my notes vault and lets me ask it
questions tomorrow without re-indexing". `aichat`'s RAG is
session-scoped; `llm`'s embedding plugins are scriptable building
blocks but not a service; `marker` only converts. `khoj` is the
only catalog entry whose unit of work is "a long-lived corpus".

**Weakness.** Five real ones:

1. **Operational weight.** It is a daemon, with a port, a config
   file, a database, and a model cache. For "I just want to ask
   one question about a folder once", this is over-engineered;
   reach for `aichat`'s `.rag` or pipe `files-to-prompt` into
   `llm`.
2. **Disk and RAM.** Local embedding model + index + (optional)
   local chat model can easily reach 10 GB on disk and several
   GB of resident RAM. Not a fit for laptops with constrained
   storage.
3. **No MCP.** A `khoj`-as-MCP-server bridge would be the natural
   integration into the agentic CLIs in this catalog, and it does
   not exist out of the box.
4. **AGPL-3.0.** Fine for personal use; non-trivial if you are
   embedding `khoj` into a hosted product you sell.
5. **Citation quality varies.** Like every RAG system, it depends
   on chunking and embedding choices. The defaults are reasonable
   for prose-shaped notes (Markdown, org-mode); for code corpora,
   results are weaker than a code-aware tool like `aider`'s
   repo-map or [`symbex`](../symbex/) + a chat CLI.

**When to choose.**

- You have a **personal notes vault** (Obsidian, Logseq, org-mode,
  a Markdown wiki, a PDF reading library) and want **chat over it
  with citations**, queryable from the terminal, persistent across
  sessions.
- You want **fully local** chat-with-your-docs and you have the
  disk/RAM budget for a local model.
- You want a **single retrieval service** that several clients (CLI,
  web, Emacs, Obsidian plugin) can all share, instead of each tool
  rebuilding its own index.
- You are willing to spend an afternoon on initial setup in
  exchange for a tool that earns its keep daily.

**When not to choose.**

- You want **one-off RAG over an ad-hoc folder** → [`aichat`](../aichat/)
  (`.rag` command, single binary, no daemon).
- You want a **tiny pipe-style primitive** → [`llm`](../llm/) plus
  an embedding plugin.
- You need the corpus to be **queryable from an MCP-aware agent**
  → as of snapshot date, no first-class path; consider exposing
  your corpus via [`repomix`](../repomix/) `--mcp` if it is a code
  repo, or write a custom MCP wrapper.
- Your corpus is **already a code repository** and you want the
  agent to also edit it → use [`aider`](../aider/) or any agentic
  CLI; they understand code structure better than a generic
  embedding pipeline.
- You require an **OSI-permissive** license for redistribution →
  AGPL-3.0 will not fit; pick an Apache-2.0 / MIT entry.

## Links

- Upstream: <https://github.com/khoj-ai/khoj>
- Docs: <https://docs.khoj.dev/>
- PyPI: <https://pypi.org/project/khoj/>
