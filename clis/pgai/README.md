# pgai

- **Repo:** https://github.com/timescale/pgai
- **Version:** extension-0.11.2 (released 2025-10-14)
- **License:** PostgreSQL (`LICENSE` @ SHA `6facbf253d086d59ed027f62f6a59b97e370cdcc`)

## What it does

`pgai` is a suite of PostgreSQL extensions plus a Python library that lets
you build RAG, semantic search, and other LLM-backed applications with
SQL as the primary interface. Two pieces matter most:

- **Vectorizer** — declare a vectorizer on a table once, and the extension
  watches the source table, chunks rows, calls an embedding provider
  (OpenAI, Ollama, Voyage, Cohere, HuggingFace), writes results into a
  managed view, and keeps embeddings in sync as rows change. No app-side
  embedding pipeline required.
- **Model-calling SQL functions** — `ai.openai_chat_complete(...)`,
  `ai.ollama_generate(...)`, `ai.cohere_rerank(...)`, etc. invoke models
  directly from a query, so RAG pipelines collapse into joins.

## Install / Run

```sh
# Easiest path: the prebuilt image bundles Postgres + pgvector + pgai
docker run -d --name pgai -p 5432:5432 \
  -e POSTGRES_PASSWORD=postgres timescale/timescaledb-ha:pg17

psql "postgres://postgres:postgres@localhost/postgres" -c \
  "CREATE EXTENSION IF NOT EXISTS ai CASCADE;"
```

```sql
SELECT ai.create_vectorizer(
  'public.docs'::regclass,
  embedding => ai.embedding_openai('text-embedding-3-small', 1536),
  chunking  => ai.chunking_recursive_character_text_splitter('body')
);

SELECT chunk, embedding <=> ai.openai_embed('text-embedding-3-small', 'query')
  AS distance
FROM   docs_embedding
ORDER  BY distance
LIMIT  5;
```

A separate `pgai` Python package wraps the same primitives for app code that
prefers ORM-style access.

## AI-native angle

Treats the database as the agent's memory and embedding pipeline rather
than bolting a vector store on the side: schema migrations carry your RAG
config, embeddings stay consistent with source rows by construction, and
permissions / backups / replication apply uniformly because it is all just
Postgres. Strong fit when an existing service is already on Postgres and
the team would rather add an extension than stand up a parallel vector
database.
