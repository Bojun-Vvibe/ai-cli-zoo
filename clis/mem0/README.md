# mem0

> Snapshot date: 2026-04. Upstream: <https://github.com/mem0ai/mem0>
> License file: <https://github.com/mem0ai/mem0/blob/main/LICENSE>
> Pinned: **v2.0.0** (PyPI: `mem0ai`, 2026-04). Repo also ships a
> first-party terminal CLI under `cli/` (Python + Node variants;
> latest tag `cli-v0.2.4`, 2026-04-22).
> Companion docs: <https://docs.mem0.ai>.

Mem0 is a **memory layer for AI agents**: a small SDK + REST surface
+ terminal CLI that takes the role of "the thing that remembers what
the user told you, across sessions, across agents, across processes."
The pitch is the same problem statement as Letta — flat chat history
isn't memory, you need a typed extract → store → retrieve loop — but
the architectural answer is opposite. Letta gives you a stateful
*server* with its own runtime; Mem0 gives you a *library* (Python +
TS) and a *managed API* you bolt onto whatever agent framework you
already use ([crewai](../crewai/), [pydantic-ai](../pydantic-ai/),
LangGraph, raw OpenAI / Anthropic SDK calls).

The `mem0` CLI in `cli/` is the ergonomic wrapper for the same SDK:
init a project, push and search memories from the terminal, run
imports / migrations, and exercise the hosted API without writing
Python.

## 1. Install footprint

- Python: `pip install -U mem0ai` (or `uv pip install mem0ai`).
  Python 3.9+. Pulls `pydantic`, `openai`, `qdrant-client`,
  `posthog`, `pytz`. Slim install ~30 MB.
- Node: `npm install mem0ai` (TS SDK lives in `mem0-ts/`).
- CLI: `pipx install mem0ai-cli` (Python flavour) or
  `npm install -g mem0ai-cli` (Node flavour); both follow the spec
  in `cli/CLI_SPECIFICATION.md` so commands are interchangeable.
- Storage: pluggable. Default vector store is **Qdrant** (in-memory
  for dev, server URL for prod); also supports
  [chroma](https://github.com/chroma-core/chroma),
  [pinecone](https://github.com/pinecone-io/pinecone-python-client),
  [weaviate](https://github.com/weaviate/weaviate-python-client),
  [pgvector](https://github.com/pgvector/pgvector),
  Milvus, Redis, Elasticsearch, Vertex AI Vector Search, OpenSearch,
  Azure AI Search, MongoDB, FAISS, LanceDB, Supabase. Graph store
  (optional) is Neo4j or Memgraph.
- Hosted plane: `app.mem0.ai` (free tier + paid). The OSS library
  works fully self-hosted; `MEM0_API_KEY` enables the managed
  endpoint.

## 2. License

Apache-2.0 (verified: repo root `LICENSE`, 11349 bytes, full
Apache-2.0 text). PyPI metadata declares `Apache 2.0`. The
`mem0-ts/` and `vercel-ai-sdk/` subpackages share the same root
license.

## 3. Models supported

LLM-agnostic via direct provider SDKs and LiteLLM passthrough:
OpenAI, Anthropic, Gemini (AI Studio + Vertex), Bedrock, Azure
OpenAI, Mistral, Groq, Together, Fireworks, DeepSeek, Cohere,
Ollama, vLLM, llama.cpp via OpenAI-compatible endpoints. The LLM
is used at *write* time (extract facts from the conversation turn)
and *search* time (re-rank candidates against the query). Embedding
model is configured separately — OpenAI by default, with
HuggingFace, Ollama, Bedrock, Vertex, Azure as drop-ins.

## 4. MCP support

**No first-party MCP server in the OSS package** (as of v2.0.0 the
`opentelemetry-instrumentation-mcp` is wired through the
[openllmetry](../openllmetry/) instrumentation list, not Mem0 itself).
Community MCP servers wrap the REST API for use inside
[opencode](../opencode/) / [claude-code](../claude-code/) /
[crush](../crush/) / [continue](../continue/), but they are not
canonical. The `mem0` CLI itself does not speak MCP; it is a thin
shell over the SDK.

## 5. Sub-agent model

**None — Mem0 is the substrate, not the agent.** It exposes a
typed `add` / `search` / `update` / `delete` / `get` surface
scoped per `user_id` / `agent_id` / `run_id` / `app_id`. The
multi-agent shape it enables is "two agents on the same `user_id`
share long-term memory" — a coding assistant agent and a
scheduling agent see the same notes about the user without
explicitly handing context. Composition belongs to the framework
that consumes Mem0.

## 6. Telemetry stance

**On by default, opt-out.** PostHog (`posthog>=3.5.0`) is a hard
dependency and emits anonymous usage events on every `Memory` /
`MemoryClient` instantiation. Disable with
`MEM0_TELEMETRY=False` or `mem0.telemetry.disable_telemetry()`
before constructing the client. The hosted `app.mem0.ai` API is a
separate explicit opt-in (set `MEM0_API_KEY`); the OSS library
runs fully self-hosted with whatever vector store you point it at.

## 7. Prompt-cache strategy

None at the Mem0 layer. Mem0's per-call prompt is short by design
(the *facts* it extracts, not the conversation), so caching is
left to the underlying provider — Anthropic ephemeral cache and
OpenAI / Bedrock automatic prefix caches handle the system prompt
of the *consuming* agent, not Mem0's extraction prompt.

## 8. Hot keybinds (CLI)

The `mem0` CLI is a flat command surface (no TUI), spec defined
in `cli/cli-spec.json`. Useful one-shots:

| Command | Action |
|---------|--------|
| `mem0 init` | Scaffold a config + .env in the current dir |
| `mem0 add "<text>" --user-id alice` | Extract facts and store |
| `mem0 search "<query>" --user-id alice` | Vector + LLM-reranked recall |
| `mem0 list --user-id alice` | Dump everything Mem0 remembers about a user |
| `mem0 update <id> --text "<new>"` | Replace a memory by ID |
| `mem0 delete <id>` / `mem0 reset` | Delete one / wipe all |
| `mem0 export --format json > backup.json` | Portable dump |
| `mem0 import backup.json` | Restore (idempotent on IDs) |
| `mem0 config show` / `mem0 config set <k> <v>` | Inspect / change provider + store |
| `mem0 server` | Boot the optional FastAPI server (`server/`) for cross-process clients |

## 9. Killer feature, weakness, when to choose

- **Killer:** **Memory as a thin SDK that snaps onto whatever agent
  you already have.** The whole loop — extract structured facts from
  a turn, embed them, vector-search at recall time, LLM-rerank for
  relevance, dedupe and update on conflict — is one
  `m.add(messages, user_id="alice")` and one
  `m.search(query, user_id="alice")` call. The agent framework
  doesn't change. The prompt template doesn't change. You don't run
  a new server. That makes Mem0 the "I just want my chat assistant
  to remember the user" answer for teams that can't or won't adopt a
  full stateful-agent runtime like [letta](../letta/).
- **Weakness:** **the magic happens inside an LLM extraction prompt
  you do not own and cannot audit cheaply.** Every `add` call costs
  one LLM round-trip to decide what to remember; every `search`
  costs another to rerank. Bills scale with conversation volume,
  not with how often you actually use the memory. The default
  PostHog telemetry is also a hard pip dependency, not a soft
  optional one — you must explicitly disable it before the first
  client construction or your usage events leave the box.
- **Choose it when:** you already have an agent (your own
  [crewai](../crewai/) crew, a [pydantic-ai](../pydantic-ai/)
  service, a raw OpenAI Chat Completions loop) and the missing
  piece is "remember things about the user across sessions." Pick
  [letta](../letta/) instead when memory is so central that you
  want it to *be* the runtime — server, durable Postgres, agents
  editing their own system prompts. Pick a vector store
  ([chroma](https://github.com/chroma-core/chroma) /
  [qdrant](https://github.com/qdrant/qdrant) /
  [pgvector](https://github.com/pgvector/pgvector)) directly when
  your retrieval is over documents you already have, not facts you
  need extracted from chat.
