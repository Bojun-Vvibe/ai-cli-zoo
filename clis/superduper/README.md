# superduper

> Snapshot date: 2026-04. Upstream: <https://github.com/superduper-io/superduper>

"**End-to-end framework for building custom AI applications and
agents.**" Superduper takes the unusual position that your existing
database (MongoDB, PostgreSQL, Snowflake, MySQL, SQLite, DuckDB) is
already the right place to host vectors, model outputs, and
agent state — so instead of bolting a vector DB on the side, you
*apply* models and agents directly to a database connection. A
`Listener` watches a collection, computes embeddings or LLM outputs
on every insert, and writes them back as a new column or document
field. RAG, classification, semantic search, agent loops, and
fine-tuning runs all become declarative `db.apply(...)` calls
against the same connection — no glue ETL, no parallel vector store
to sync.

## 1. Install footprint

- `pip install superduper-framework` (core).
- DB plugins: `pip install superduper-mongodb`,
  `superduper-sql` (Postgres / MySQL / SQLite / DuckDB / Snowflake
  via SQLAlchemy + Ibis).
- Model plugins: `superduper-openai`, `superduper-anthropic`,
  `superduper-cohere`, `superduper-vllm`, `superduper-llamacpp`,
  `superduper-transformers`, `superduper-sentence-transformers`,
  `superduper-jina`.
- CLI: `superduper` (subcommands `apply`, `drop`, `info`, `start`
  for the optional REST + dashboard server).
- Python ≥ 3.10. Apple Silicon, Linux x86_64, Linux ARM all
  supported. No daemon required for the library mode; the
  dashboard is opt-in.

## 2. Repo + version + license

- Repo: <https://github.com/superduper-io/superduper>
- Latest release: **0.10.0** (2025-08-26)
- License: **Apache-2.0** —
  <https://github.com/superduper-io/superduper/blob/main/LICENSE>
- HEAD SHA: `6d192e7bf255f913b445a44dc9486e7013ddc6bc`
- Default branch: `main`
- Language: Python

## 3. Models supported

LLMs: OpenAI, Anthropic, Cohere, vLLM, llama.cpp, any HF
`transformers` causal-LM, any OpenAI-compatible endpoint via the
OpenAI plugin's `base_url`. Embeddings: OpenAI, Cohere, Jina,
sentence-transformers (any HF model), custom via the `Model`
ABC. Rerankers: any `transformers` cross-encoder. Custom models
register as a subclass of `Model` with `predict` / `predict_batch`
methods — no provider lock-in.

## 4. Simple usage

```python
# point superduper at a database — that connection is now your AI runtime
from superduper import superduper, Listener
from superduper_mongodb import MongoQuery
from superduper_sentence_transformers import SentenceTransformer

db = superduper("mongodb://localhost:27017/docs")

# declare an embedding model and bind it to a collection
embedder = SentenceTransformer(
    identifier="st-mini",
    model="all-MiniLM-L6-v2",
)
db.apply(Listener(
    model=embedder,
    select=MongoQuery(table="documents").find(),
    key="text",                # which field to embed
    identifier="doc_embeddings"
))
# every insert into documents.text now auto-embeds; backfill is automatic
```

```python
# vector search against the same collection — no separate vector DB
hits = db["documents"].like(
    {"text": "what does the SEC say about late filings?"},
    vector_index="doc_embeddings",
    n=5,
).execute()
```

```python
# RAG: chain a retriever + an LLM as another Listener, write answers back
from superduper_openai import OpenAIChatCompletion
llm = OpenAIChatCompletion(identifier="gpt-4o-mini", model="gpt-4o-mini")
db.apply(Listener(
    model=llm.with_template("Answer using context:\n{context}\nQ: {question}"),
    select=MongoQuery(table="questions").find(),
    key="question",
    identifier="rag_answers",
))
```

## 5. Why it's interesting

- **The database is the runtime.** No ETL job to sync your
  Postgres rows into Pinecone and back; embeddings, classifier
  outputs, and LLM completions live alongside the source row in
  the same transaction.
- **`Listener` is the primitive that replaces a pipeline DAG.**
  It declares "when this query's result set changes, apply this
  model and write the output here" — incremental backfills,
  streaming inserts, and re-applies after model swap all flow
  from that one abstraction.
- **One API across MongoDB and SQL.** The same `db.apply()` /
  `db["table"].like()` / `db.execute()` surface works whether
  the underlying store is MongoDB, Postgres, SQLite, or
  Snowflake — useful when prototyping on SQLite and shipping on
  Postgres.
- **Model plugins are thin Apache-2.0 packages.** Adding a new
  provider is one subclass of `Model`, not a fork.

## 6. Caveats

- The "DB is your vector store" pitch only pays off if your DB
  has a usable ANN index — pgvector, Mongo Atlas Search, or a
  bolt-on like SQLite-VSS. Plain MySQL / DuckDB falls back to
  brute-force search.
- API surface has churned across the 0.x line; pin a minor
  version in production.
- Multi-tenant / RBAC is the application's job — Superduper does
  not impose a permission model on top of your database's.
