# zep

> Snapshot date: 2026-04. Upstream: <https://github.com/getzep/zep>

A **long-term memory service for AI agents** built around a temporal
knowledge graph. Where most "memory" libraries store chat turns in a
vector store and call retrieval, Zep extracts entities and facts from
conversations into a typed graph (nodes = entities, edges = relations
with `valid_from` / `invalid_at` timestamps), then answers retrieval
queries against the graph plus a vector index over message and summary
text — so a question like "what did Alice say about the Q3 budget last
month?" returns the graph fact, the message it came from, and the
relevant summary, instead of three vague chunks.

## 1. Install footprint

- **Python SDK:** `pip install zep-cloud` (hosted) or
  `pip install zep-python` (self-hosted client against the OSS server).
- **Server (self-hosted):** `docker compose up` against the
  `docker-compose.ce.yaml` in the repo brings up Postgres + the Go-based
  Zep Community Edition server. Optional Neo4j / Graphiti backend for
  the knowledge-graph layer.
- **Integrations:** first-class `langchain`, `llama-index`,
  `crewai`, `autogen`, `openai-agents` adapters.
- ~80 MB Go binary for the server, plus Postgres (and optionally Neo4j
  for the graph backend).

## 2. Repo, version, license

- Repo: <https://github.com/getzep/zep>
- Version checked: latest tag **`zep-crewai-v1.1.1`** (2025-09-11);
  `main` at `faf2ace` (2026-04-09). The repo is an examples /
  integrations monorepo; the server lives in
  `getzep/zep` (see `community-edition/` subtree) with companion
  repos `getzep/graphiti` (the temporal-graph engine) and
  `getzep/zep-python` (the Python SDK).
- License: **Apache-2.0**. License file at the repo root: `LICENSE`.
- PyPI: `zep-cloud` (hosted SDK), `zep-python` (OSS server SDK).

## 3. What it actually does

Three primitives, one consistent surface:

1. **Sessions** — per-user conversational state. `client.memory.add(
   session_id, messages=[...])` writes turns; `client.memory.get(
   session_id)` returns the recent message window plus a rolling LLM
   summary.
2. **Knowledge graph** — extraction runs asynchronously after each
   `add`. An LLM (configurable; OpenAI / Anthropic / local) reads the
   new turns and emits typed `(subject, predicate, object,
   valid_from)` triples, deduplicates against existing nodes, and
   marks superseded edges as `invalid_at` (this is the "temporal" in
   temporal knowledge graph — facts have lifetimes).
3. **Search** — `client.memory.search(session_id, text="Q3 budget",
   search_scope="messages")` returns ranked messages;
   `search_scope="facts"` returns ranked graph facts;
   `search_scope="summary"` returns matching summaries. The graph
   answers "what was true at time T" queries that a flat vector
   store cannot.

The temporal model is the differentiator. If Alice says "I'm based in
Berlin" on day 1 and "I moved to Tokyo" on day 30, a vector store
returns both as similar chunks; Zep's graph marks the Berlin edge
`invalid_at=day_30` and returns Tokyo as the current location while
keeping the Berlin fact queryable for historical questions.

## 4. MCP support

Yes — a community MCP server (`@getzep/mcp-server-zep`, separate
package) exposes `add_memory`, `search_memory`, and
`get_session_facts` as MCP tools that any MCP-aware agent CLI
([`opencode`](../opencode/), [`claude-code`](../claude-code/),
[`crush`](../crush/), [`fast-agent`](../fast-agent/)) can mount.
The server itself talks to either Zep Cloud or a self-hosted CE
instance.

## 5. Sub-agent model

None — Zep is memory infrastructure, not an agent runner. The
extraction pipeline does use an LLM internally (configurable model)
but that is an implementation detail, not a user-facing agent.

## 6. Telemetry stance

**Self-hosted:** off. The Community Edition server emits OTel spans
to the exporter you configure and otherwise calls nothing home.

**Hosted (Zep Cloud):** the SDK ships requests to `api.getzep.com` —
that is the product. Pick CE if cloud is a no-go.

The extraction LLM call (whichever provider you point Zep at) sees
the conversation contents — for sensitive workloads, point the
extractor at a local model via the OpenAI-compatible base URL setting.

## 7. Token / context strategy

The whole point: Zep replaces "stuff the entire chat history into the
context" with "fetch the relevant facts plus a summary". Typical
agent integration calls `memory.get()` to retrieve a small `(facts,
summary, recent_messages)` bundle, prepends it to the system prompt,
and lets the model work with O(1) tokens regardless of session age.
Long-running agents that would otherwise blow the context window
keep working indefinitely.

Summary generation runs on a configurable cadence (every N messages)
in the background, so the surface latency is one short Postgres
query, not an LLM call.

## 8. Hot keybinds

N/A — Zep is a service. The CLI surface is the `zep` admin command
(self-hosted) for migrations and user/session inspection; everything
else is SDK-shaped.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Temporal knowledge graph as memory.** Every
other memory library in the catalog ([`mem0`](../mem0/) being the
closest sibling) is a vector store with extraction on top. Zep adds
explicit `valid_from` / `invalid_at` edges so superseded facts are
correctly retired without losing history — the only entry in the
catalog that handles "what is true now vs. what was true then" as a
first-class query, not a re-ranking heuristic.

**Weakness.** Operationally heavier than a pure vector store —
Postgres is required, Neo4j/Graphiti is recommended for the full
graph experience, and the extraction pipeline is asynchronous so
"I just wrote this fact, why doesn't search return it?" is a real
foot-gun (eventual consistency, typically seconds). The repo layout
is confusing: the headline `getzep/zep` repo is mostly examples and
integrations, the server lives in a `community-edition/` subtree,
and core graph logic is in the sibling `getzep/graphiti` repo —
budget time to learn the topology before deploying.

**When to choose.**
- You build **long-running agents** (customer support, personal
  assistant, ops copilot) where session lifetimes are weeks or
  months and "stuff the whole history into context" stops working.
- You need **temporal correctness** — facts that change over time
  must retire cleanly, not be averaged with their successors by
  semantic similarity.
- You want **self-hostable Apache-2.0 memory** with an MCP surface
  that any agent CLI can mount.

**When to skip.**
- You want **one-file simplicity** — use [`mem0`](../mem0/) (vector
  store + extraction, no graph DB to operate).
- You want a **general-purpose vector DB** for RAG, not
  per-conversation memory — use [`qdrant`](../qdrant/),
  [`chroma`](../chroma/), [`weaviate`](../weaviate/), or
  [`lancedb`](../lancedb/).
- You want **document RAG** with citations across a knowledge base
  — use [`r2r`](../r2r/), [`paper-qa`](../paper-qa/), or
  [`khoj`](../khoj/).
- The agent is a **one-shot** code edit / shell command / commit
  message — memory is overhead.

## 10. Compared to neighbors in the catalog

| Tool | Storage shape | Temporal | Self-hosted | License |
|------|---------------|----------|-------------|---------|
| zep | Temporal knowledge graph + vector + summary | Yes (`valid_from`/`invalid_at`) | Yes (CE) | Apache-2.0 |
| [mem0](../mem0/) | Vector + extracted facts | No (overwrite-on-update) | Yes | Apache-2.0 |
| [letta](../letta/) | Hierarchical memory blocks (MemGPT) | No (block versioning only) | Yes | Apache-2.0 |
| [chroma](../chroma/) | Pure vector store | No | Yes | Apache-2.0 |
| [qdrant](../qdrant/) | Pure vector store | No | Yes | Apache-2.0 |

Decision shortcut:

- "Facts have lifetimes; superseded facts must retire" → `zep`.
- "Per-user fact extraction over a vector store" → `mem0`.
- "MemGPT-style tiered memory blocks for one long agent run" →
  `letta`.
- "Just a vector store" → `chroma` / `qdrant` / `weaviate` /
  `lancedb`.
