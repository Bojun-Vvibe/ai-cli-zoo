# r2r

> Snapshot date: 2026-04. Upstream: <https://github.com/SciPhi-AI/R2R>

"**The most advanced AI retrieval system. Containerized,
SOC2-compliant RAG with a RESTful API.**" R2R (Reason-to-Retrieve)
is a self-hostable production RAG stack: ingestion pipeline +
hybrid (vector + keyword + KG) retrieval + reranking + agentic
RAG + auth/users/permissions + observability, all behind one
FastAPI server with a CLI (`r2r`) and Python / TypeScript SDKs.
The killer move vs "build it yourself with LangChain" is that the
graph extraction, hybrid search fusion, multi-tenant document
permissions, and conversation history are real engineered
components, not glue you have to maintain.

## 1. Install footprint

- `pip install r2r` (CLI + lightweight client). Server is run via
  Docker Compose for the typical full stack:
  `r2r serve --docker --full` brings up Postgres+pgvector,
  Hatchet (workflow engine), Unstructured (parsers), and the R2R
  API on `:7272`.
- Python SDK: `pip install r2r` (`from r2r import R2RClient`).
- TypeScript SDK: `npm install r2r-js`.
- CLI surface: `r2r serve`, `r2r ingest-files`, `r2r ingest-sample-files`,
  `r2r documents-overview`, `r2r search`, `r2r rag`, `r2r agent`,
  `r2r users-overview`, `r2r generate-report`,
  `r2r update-default-prompt`.
- Python ≥ 3.10. Server side needs Docker for the standard
  deployment (Postgres + pgvector + Hatchet).

## 2. Repo + version + license

- Repo: <https://github.com/SciPhi-AI/R2R>
- Latest release: **v3.6.5** (2025-06-06)
- License: **MIT** —
  <https://github.com/SciPhi-AI/R2R/blob/main/LICENSE.md>
- HEAD SHA: `9c5a94d151f90876bd7eb860f300a8fd662dc481`
- Default branch: `main`
- Language: Python (TypeScript SDK in sibling repo)

## 3. Models supported

LLMs via LiteLLM — OpenAI, Anthropic, Gemini, Bedrock, Azure,
Mistral, Cohere, Groq, Together, Ollama, vLLM, any OpenAI-
compatible. Embeddings: OpenAI, Cohere, Voyage, Bedrock,
HuggingFace local. Rerankers: Cohere Rerank, Jina Rerank, local
cross-encoders. KG extraction runs on the configured LLM (Claude
Sonnet / GPT-4o-class recommended).

## 4. Simple usage

```bash
# bring up the full local stack (Postgres+pgvector+Hatchet+R2R API)
pip install r2r
r2r serve --docker --full
# → API on http://localhost:7272, dashboard on :7273

# ingest, search, rag, agent — all from CLI
r2r ingest-sample-files
r2r documents-overview
r2r search --query "what does the SEC say about disclosure?"
r2r rag --query "summarize the company's risk factors"
r2r agent --query "compare risk sections across the ingested 10-Ks"
```

```python
# Python SDK against the same server
from r2r import R2RClient
c = R2RClient("http://localhost:7272")
c.ingest_files(file_paths=["./report.pdf"], metadatas=[{"team": "legal"}])

# hybrid search (vector + full-text fused)
hits = c.search(
    query="late filing risk",
    vector_search_settings={"use_hybrid_search": True},
)

# RAG with citations
ans = c.rag(
    query="Summarize the late-filing risk and cite sources.",
    rag_generation_config={"model": "openai/gpt-4o-mini", "temperature": 0.2},
)

# Agentic RAG: the model plans multi-hop retrieval itself
trace = c.agent(message="compare disclosures across all ingested 10-Ks")
```

```python
# knowledge-graph build over the corpus, then KG-enriched RAG
c.create_graph(collection_id="default")
c.enrich_graph(collection_id="default")          # entity / relation extraction
c.rag(query="who reports to the CFO at Acme?",
      kg_search_settings={"use_kg_search": True})
```

## 5. Why it's interesting

- **Hybrid search is a one-flag config**, not a re-architecture —
  pgvector does the dense side, Postgres `tsvector` does the
  sparse side, and the server fuses them with reciprocal-rank
  fusion behind a single `use_hybrid_search=True` setting.
- **Knowledge-graph layer over the same corpus**:
  `create_graph` + `enrich_graph` runs LLM-based entity / relation
  extraction across ingested docs and stores the graph in the
  same Postgres; subsequent `rag()` calls can fuse vector hits
  with KG-walked neighbors, so multi-hop questions ("who reports
  to whom") get real answers, not vector-only approximations.
- **Auth / users / collections / permissions are real**, not
  examples — multi-tenant document scoping ships in the API
  surface (`users`, `collections`, `groups`), so SaaS-shaped RAG
  apps don't need a custom permission model bolted on.
- **Agentic RAG is in-tree**: the `agent` endpoint runs an LLM
  loop with `search` and `kg_search` as tools, returning the
  full reasoning trace plus citations — closer to a Perplexity-
  shaped UX than the typical single-shot `retrieve → stuff →
  generate` template.
- **Containerized full stack with one command**: `r2r serve
  --docker --full` brings up Postgres + pgvector + Hatchet +
  Unstructured + the R2R API wired together — no day-2
  Helm-chart archaeology to get something running on a single
  machine.

## 6. Caveats

- Latest tagged release `v3.6.5` (2025-06) lags `main` by several
  months; the upstream now publishes most changes via the Docker
  image tag and the `main` branch — pin to a SHA or image
  digest in production rather than the tag.
- "One command" full deploy assumes Docker; bare-metal install
  needs you to provision Postgres+pgvector and the Hatchet
  workflow engine yourself.
- The KG enrichment pass is LLM-token-heavy on large corpora;
  budget for it explicitly and consider gating it per
  collection.
- License footer in the repo is MIT (OSS), but the upstream also
  offers a hosted SaaS / enterprise tier — read `LICENSE.md`
  before assuming everything you see in the dashboard is
  bundled OSS.
