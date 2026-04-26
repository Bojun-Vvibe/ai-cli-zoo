# graphiti

> Snapshot date: 2026-04. Upstream: <https://github.com/getzep/graphiti>

A **temporal knowledge-graph framework for AI agents** that
ingests episodic data (conversations, documents, structured
JSON) into a bi-temporal graph in Neo4j / FalkorDB / Kuzu /
Neptune, where every node and edge carries `valid_at` and
`invalid_at` timestamps so the agent can ask "what did the user
believe / want / own at time T", not just "what is currently
true". Built by the Zep team as the open-source core of their
agent-memory product, graphiti also ships a first-party MCP
server so any MCP-aware coding CLI in this catalog can read /
write the graph as long-term memory.

It is the catalog's reference for **temporal, graph-shaped
agent memory** — the structured-memory counterpart to the
embedding-vector approaches in [`mem0`](../mem0/) and
[`zep`](../zep/).

## 1. Install footprint

- `pip install graphiti-core` (Python 3.10+).
- ~30 transitive deps: `pydantic`, `neo4j`, `openai`,
  `anthropic`, `tenacity`, `numpy`. ~80 MB venv.
- Optional graph backends (pick one):
  - **Neo4j** ≥ 5.x via the official driver (default).
  - **FalkorDB** for an in-memory Redis-module graph.
  - **Kuzu** for an embedded single-file graph (no daemon).
  - **AWS Neptune** for managed cloud.
- Models: OpenAI / Anthropic / Gemini / Groq / Azure OpenAI /
  Bedrock / Ollama / any OpenAI-compatible for the
  entity-extraction + edge-summarization LLM hop; OpenAI /
  Voyage / Cohere / local sentence-transformers for embeddings;
  optional cross-encoder reranker for hybrid retrieval.
- MCP server: `uvx graphiti-mcp` (or run the
  `mcp_server/` Python entrypoint from the repo).

## 2. Repo, version, license

- Repo: <https://github.com/getzep/graphiti>
- Version checked: **v0.28.2** (latest tagged release of
  `graphiti-core`; an `mcp-v1.0.2` tag tracks the MCP server
  surface independently).
- HEAD pinned at this snapshot:
  `9cdcc93ad8f72c7235e10265761acb0410e5c8bf`.
- License: Apache-2.0. License file at
  [`LICENSE`](https://github.com/getzep/graphiti/blob/main/LICENSE).

## 3. What it actually does

The core API is `add_episode` + `search`:

```python
from graphiti_core import Graphiti

g = Graphiti(uri="bolt://localhost:7687", user="neo4j", password="…")
await g.build_indices_and_constraints()

await g.add_episode(
    name="standup-2026-04-22",
    episode_body="Alice said the migration ships Friday; Bob owns the rollback.",
    source_description="standup notes",
    reference_time=datetime(2026, 4, 22, 9, 0, tzinfo=UTC),
)

hits = await g.search("who owns the rollback")
```

What happens inside `add_episode`:

1. **Entity + relation extraction** — an LLM call extracts
   typed nodes (`Person:Alice`, `Project:migration`,
   `Task:rollback`) and edges (`OWNS(Bob, rollback)`,
   `SHIPS_ON(migration, 2026-04-25)`).
2. **Temporal reconciliation** — new edges that contradict
   existing ones don't overwrite; the old edge gets an
   `invalid_at` timestamp and the new edge gets a `valid_at`,
   so history is preserved. "Bob owns the rollback as of
   2026-04-22" doesn't erase "Carol owned it as of 2026-04-15".
3. **Embedding + persistence** — node and edge summaries get
   embedded; everything is written to the graph backend in one
   transaction.

`search` is **hybrid by default**: BM25 over node / edge
summaries + vector similarity + graph-distance reranking,
filtered by an optional `valid_at` time window so the agent
sees the world as it was at any given point.

The MCP server (`mcp_server/`) exposes `add_memory`,
`search_nodes`, `search_facts`, and `delete_episode` as MCP
tools, plus a `set_group_id` mechanism so multiple agents
(or multiple users) can share one graph backend without
collisions.

## 4. MCP support

First-party MCP server in `mcp_server/` (separate version line,
currently `mcp-v1.0.2`). It runs over stdio (for Claude
Desktop, opencode, cline) or SSE / Streamable HTTP (for
remote agents), wraps the same `Graphiti` core, and uses
`group_id` namespacing to scope memories per-agent. This is
one of the few catalog memory tools that ships as an MCP
server out of the box rather than requiring a custom adapter.

## 5. Sub-agent model

Not an agent framework — graphiti is the *memory substrate*
agents read from / write to. There is no LLM loop or planner.
The single piece of internal concurrency is per-episode
ingestion: entity extraction + edge resolution + embedding
happen in async-parallel batches, with `tenacity` retries on
LLM rate-limits.

## 6. Telemetry stance

Off by default in `graphiti-core`. The library makes no
outbound calls beyond the LLM / embedding providers you
configure and the graph backend you point it at. The Zep
hosted product (`getzep.com`) is a separate offering that
runs graphiti for you — using OSS graphiti directly keeps all
data in your own Neo4j / FalkorDB / Kuzu / Neptune.
OpenTelemetry instrumentation is opt-in via standard Python
OTel auto-instrumentation against the LLM client + neo4j
driver.

## 7. Token / context strategy

Per-episode token usage is bounded by the size of the episode
body you pass in (typically a single conversation turn or
document chunk; the project recommends keeping episodes under
~10 KB so the entity-extraction LLM call stays cheap).
Retrieval is the place where graphiti shines on context
economy: instead of stuffing N raw conversation chunks into
the LLM prompt (vector-RAG style), `search_facts` returns a
small set of *summarized edges* like "Bob OWNS rollback (valid
since 2026-04-22)" — a lossy but query-shaped representation
that fits dozens of facts into the same token budget that
held three RAG chunks.

## 8. Hot keybinds

None — graphiti is a Python library plus an MCP server.
Inspection happens in the graph backend's own UI (Neo4j
Browser, FalkorDB Insight) where you can run Cypher queries
against the live graph; the Zep hosted product adds a memory
browser UI.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Bi-temporal graph memory** — every fact
the agent learns is stamped with both *event time* (when it
happened in the world) and *ingestion time* (when the agent
learned it), so "what did the user believe last Tuesday" and
"what was true last Tuesday" are different, answerable queries.
Combined with the **first-party MCP server** and the
**hybrid BM25 + vector + graph-distance retrieval**, graphiti
gives a coding agent a memory layer that survives
contradictions across sessions without overwriting history,
which embedding-only memories (the dominant pattern) cannot.

**Weakness.** The graph backend is a real piece of operational
weight — even the embedded Kuzu option is one more thing to
back up, and the Neo4j / FalkorDB / Neptune options are full
databases. Every `add_episode` costs at least one LLM call for
entity extraction, so high-volume ingestion (chat logs,
streaming events) gets expensive fast unless you batch /
sample. The `0.x` API is still moving (entity-type schemas,
retrieval reranking, `group_id` semantics shifted in
mid-2025) — production users pin a minor version.

**Choose graphiti when** the agent needs **structured,
queryable, temporal memory** that can answer "who owned X at
time T" or "what did the user believe before they changed
their mind", when MCP-shaped memory is the integration
contract you want, or when the existing memory layer is a
vector-RAG store that keeps overwriting itself on
contradictory facts. **Choose something else when** vector
similarity over flat documents is sufficient (use any vector
DB — [`chroma`](../chroma/), [`qdrant`](../qdrant/),
[`weaviate`](../weaviate/), [`marqo`](../marqo/)), when a
key-value short-term memory is enough (use
[`mem0`](../mem0/)), when you want the managed product
without running the graph yourself (use Zep cloud), or when
the operational cost of a graph database is not justified by
the queries you need.
