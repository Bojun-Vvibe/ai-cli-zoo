# fast-graphrag

> Snapshot date: 2026-04. Upstream: <https://github.com/circlemind-ai/fast-graphrag>
> Pinned HEAD: `23b3a1bef64338faf9f35c521a20465dbd09419f` (default branch `main`,
> last upstream activity 2025-11-01; project ships from `main`, no tagged
> releases yet at this snapshot).
> License file: [`LICENSE`](https://github.com/circlemind-ai/fast-graphrag/blob/main/LICENSE)
> (sha `ba5d2f67d090630853965982e45090eac811a119`, MIT).

`fast-graphrag` is a small, opinionated **GraphRAG** library and
companion script harness from circlemind that takes the
GraphRAG idea — extract a typed entity / relation graph from a
corpus, then answer queries by walking the graph instead of
just doing top-k vector lookup — and ships it as a *single
Python package* with a deterministic prompt pipeline, a
PageRank-shaped traversal, and an explicit "domain + example
queries + entity types" config knob. It is the catalog's
reference for **a GraphRAG implementation small enough to read
in one sitting** — sibling to heavier RAG-frameworks like
[`r2r`](../r2r/), [`ragflow`](../ragflow/),
[`docsgpt`](../docsgpt/), and a peer to graph-aware memory
stores like [`graphiti`](../graphiti/) and
[`cognee`](../cognee/).

## 1. Install footprint

- `pip install fast-graphrag` (Python 3.10+) installs the
  library plus a small CLI surface exposed through the example
  scripts in `examples/` (`python examples/query.py …`).
- Runtime deps are intentionally narrow: `instructor`,
  `pydantic`, `tenacity`, `numpy`, `igraph`, `xxhash`,
  `tiktoken`, plus an OpenAI-shaped client (defaults to
  OpenAI; any OpenAI-compatible base URL works, including
  Ollama, vLLM, [`litellm`](../litellm/)).
- Storage is local-first: pickled namespaces in a working
  directory hold the entity store, relation store, chunk
  store, and the `igraph` graph; no Postgres / Neo4j / vector
  cluster required.
- The whole working set for a 50-document corpus is typically
  under 100 MB on disk and fits comfortably on a laptop.

## 2. Repo, version, license

- Repo: <https://github.com/circlemind-ai/fast-graphrag>
- No GitHub releases tagged; project distributes from `main`
  and PyPI.
- HEAD pinned at this snapshot:
  `23b3a1bef64338faf9f35c521a20465dbd09419f`.
- License: MIT. Pinned LICENSE blob:
  [`LICENSE`](https://github.com/circlemind-ai/fast-graphrag/blob/main/LICENSE)
  (sha `ba5d2f67d090630853965982e45090eac811a119`).

## 3. What it actually does

The end-to-end shape is `insert → query`:

```python
from fast_graphrag import GraphRAG

grag = GraphRAG(
    working_dir="./book_kg",
    domain="A book about a wizarding school",
    example_queries="Who is Harry Potter?\nWho killed Voldemort?",
    entity_types=["character", "place", "event", "object"],
)
grag.insert(open("book.txt").read())
print(grag.query("Who is Harry's best friend?").response)
```

Under the hood `insert` chunks the input, runs an
`instructor`-typed LLM extraction pass that emits
`(entity, type, description)` and `(source, target,
description)` tuples, deduplicates against the existing
namespaces, and updates the `igraph` graph + per-node /
per-edge embeddings. `query` does a hybrid step: embed the
question, retrieve seed entities by vector similarity, then
run a personalised-PageRank walk from those seeds across the
graph to surface a wider neighbourhood of relevant entities,
chunks, and relations — and finally answers from that
context bundle.

There is also an async variant (`grag.async_insert`,
`grag.async_query`) and a streaming-response variant for
chat-shaped UIs.

## 4. MCP support

None first-party — `fast-graphrag` is a library, not a daemon,
and there is no built-in MCP server flavour. The expected
integration is to wrap `GraphRAG.query` from a host's tool /
MCP server (e.g. an [`fastmcp`](../fastmcp/) shim or inside a
[`langgraph`](../langgraph/) / [`crewai`](../crewai/) tool
node) and expose it as `kg.query` to the agent.

## 5. Sub-agent model

None — the library does not spawn sub-agents. Internally it is
a pipeline of typed LLM calls (extract → dedupe → answer);
parallelism is pure asyncio fan-out over chunks. Agentic
orchestration is the host's job.

## 6. Telemetry stance

Off — no telemetry, no analytics, no phone-home. The only
network egress is the configured LLM / embedding endpoint;
storage is local pickles + an `igraph` graph in
`working_dir`. Pair with Ollama or a self-hosted
OpenAI-compatible endpoint for a fully air-gapped GraphRAG
loop.

## 7. Token / context strategy

The library encodes the GraphRAG promise that **a typed graph
walk reaches further per token than top-k vector lookup**. At
query time it does *not* stuff a fixed-K chunk window into the
prompt; instead it runs personalised PageRank on the entity
graph from the seed entities the question embeds against,
then materialises a context bundle of entities, relations, and
the original chunks they came from, capped by a configurable
`entity_range_func` / `relation_range_func` / `chunk_range_func`
trio that picks the top slice by score. The trade is paid in
*ingest-time* tokens — the extraction pass is more expensive
than chunk-and-embed — but every later query reads from the
graph instead of paying that cost again.

## 8. Hot keybinds

None — `fast-graphrag` is a non-interactive library + script
surface. The clickable layer belongs to whatever host UI
imports it.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **A complete GraphRAG implementation in
one small Python package, with the typed-extraction prompt,
the PageRank walk, the local stores, and the
domain / example-queries / entity-types config knobs all
visible and editable.** Most of the GraphRAG ecosystem ships
either as a research artefact (the original GraphRAG paper's
reference repo, out of catalog scope) or as a heavyweight
platform ([`r2r`](../r2r/), [`ragflow`](../ragflow/));
`fast-graphrag` is the readable reference: ~3 kLOC of Python
that you can fork, swap embedding model on, swap LLM on, and
deploy on a laptop without any infrastructure beyond the
filesystem.

**Weakness.** It is **the small reference, not the production
platform** — there is no UI, no multi-tenant story, no
incremental delete, no Neo4j / property-graph backend, and no
release cadence (the project ships from `main`). For a
corpus past a few thousand documents, or for a team that
needs ACL'd multi-corpus serving, the right move is to keep
the typed-extraction prompt and port the storage tier to a
real graph database — at which point `fast-graphrag` has
served its job as the spec.

**Choose `fast-graphrag` when** the goal is "GraphRAG, on this
laptop, against this 50-document corpus, by Friday" and the
constraint is staying readable enough that the team can fork
the prompts and the traversal scoring. **Choose something
else when** the corpus is large enough to need a real graph
database and a real serving tier ([`graphiti`](../graphiti/),
[`r2r`](../r2r/), [`ragflow`](../ragflow/),
[`docsgpt`](../docsgpt/)), the workload is conventional
top-k vector RAG without graph structure
([`langchain`-style stacks via [`langgraph`](../langgraph/),
or any vector store in this catalog]), or graph-of-memories
for a long-running agent is the actual need
([`graphiti`](../graphiti/), [`cognee`](../cognee/),
[`mem0`](../mem0/), [`zep`](../zep/)).
