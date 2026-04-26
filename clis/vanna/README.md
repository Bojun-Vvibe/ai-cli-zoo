# vanna

> Snapshot date: 2026-04. Upstream: <https://github.com/vanna-ai/vanna>

"**Chat with your SQL database.**" Vanna is a Python library + thin
CLI / Flask UI that turns natural-language questions into executable
SQL via **agentic retrieval**: you train it on your schema, your
documentation, and a corpus of known-good `(question, sql)` pairs
stored in a vector store; at query time it retrieves the most relevant
context, asks the LLM to write SQL, runs it against your warehouse,
and returns the dataframe (plus an auto-generated Plotly chart, if you
want one). Read-only by convention; the strong claim is "accurate
text-to-SQL on your real schema, not a generic benchmark" because the
training corpus is yours.

> **Status note.** As of 2026-04 the upstream repo is **archived**
> (last release `v2.0.2`, 2026-02-02). The library still installs and
> runs against current LLM and vector-store SDKs, but no further
> upstream development is happening. Treat as a battle-tested classic
> for the text-to-SQL pattern, not a living roadmap.

## 1. Install footprint

- `pip install vanna` (or `pip install 'vanna[chromadb,openai,postgres]'`
  for a typical local stack; extras pick the LLM + vector store + DB
  driver combo).
- Python ≥ 3.9.
- Library entry points: `vanna.openai.OpenAI_Chat`,
  `vanna.anthropic.Anthropic_Chat`, `vanna.ollama.Ollama`,
  `vanna.chromadb.ChromaDB_VectorStore`, `vanna.pinecone.PineconeDB_VectorStore`,
  plus a `VannaBase` class you mix together.
- Apps: `vn.ask("how many users signed up last week?")` returns
  `(sql, df, fig)`; `VannaFlaskApp(vn).run()` boots a local web UI;
  there is also a hosted SaaS variant at `vanna.ai` (OSS library is
  fully self-hostable and the SaaS is optional).
- Configure via env / kwargs: `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`,
  database connection string passed to `vn.connect_to_postgres(...)` /
  `connect_to_snowflake(...)` / `connect_to_bigquery(...)` /
  `connect_to_mysql(...)` / `connect_to_sqlite(...)`.

## 2. Repo + version + license

- Repo: <https://github.com/vanna-ai/vanna>
- Latest release: **v2.0.2** (2026-02-02)
- License: **MIT** —
  <https://github.com/vanna-ai/vanna/blob/main/LICENSE>
- Default branch: `main`
- Language: Python
- Status: archived upstream (read-only)

## 3. Models supported

LLM: OpenAI, Anthropic, Google Gemini, Mistral, Ollama / any
OpenAI-compatible endpoint, AWS Bedrock, Azure OpenAI, ZhipuAI, plus
hosted `Vanna.AI` defaults. Each backend is a small mixin class
(`OpenAI_Chat`, `Anthropic_Chat`, `Ollama`, …) that you combine with a
vector-store mixin to form your concrete `Vanna` class.

Vector store: ChromaDB (default), Pinecone, Qdrant, Weaviate,
Marqo, Milvus, Faiss, Postgres `pgvector`, Opensearch, plus the hosted
`Vanna.AI` store. Same mixin pattern.

## 4. MCP support

None first-party. Vanna exposes a Python API and a Flask web UI; if
you want to mount it in a coding agent, wrap `vn.ask()` in a custom
MCP server yourself, or call it from a tool inside a framework that
already speaks MCP ([`pydantic-ai`](../pydantic-ai/),
[`crewai`](../crewai/), [`openai-agents-python`](../openai-agents-python/)).

## 5. Sub-agent model

None as crews. The "agentic" in the tagline refers to the **retrieval
loop** (DDL retrieval → documentation retrieval → similar-Q&A
retrieval → SQL generation → optional self-correction on execution
error), not to spawning sub-agents. One Vanna instance, one
conversation, one SQL string back.

## 6. Telemetry stance

OSS library is **opt-out telemetry**: by default `vanna` pings a
hosted endpoint with anonymized usage events. Disable with
`vn = MyVanna(config={"vanna_api_key": "...", "telemetry": False})` or
the `VANNA_TELEMETRY=false` env var. The hosted `vanna.ai` SaaS is a
separate product with its own data plane; the self-hosted library is
fully usable without it.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **Train-on-your-own-corpus text-to-SQL with
retrieval as the load-bearing wall.** `vn.train(ddl=...)`,
`vn.train(documentation=...)`, `vn.train(question=..., sql=...)`
populate a vector store with the three things the model actually needs
to write correct SQL on your schema: the column names, the business
glossary, and a corpus of known-good answers. Subsequent
`vn.ask("...")` calls retrieve the top-k of each, stitch them into the
prompt, and ship — so the same generic LLM produces dramatically
better SQL on your warehouse than on a fresh chat. The mixin design
(`class MyVanna(ChromaDB_VectorStore, OpenAI_Chat): pass`) makes the
model and store choices a two-line decision.

**Weakness.** Archived upstream as of 2026-04 — `v2.0.2` is the end
of the line; bug fixes will not arrive without a community fork. The
out-of-the-box accuracy ceiling depends almost entirely on how much
`(question, sql)` pairs you bother to label, and the library does not
help you bootstrap that corpus. Default telemetry-on posture is unusual
for a self-hosted SQL tool and will surprise some teams. Read-only is
a convention, not an enforced sandbox: nothing prevents an answer
returning `DROP TABLE` if your DB role allows it, so always connect
with a least-privilege read-only credential.

**When to choose.** You have a real warehouse (Snowflake / BigQuery /
Postgres / etc.), a stable schema, and a stack of analyst questions
you want non-analysts to self-serve. Pair with a privileged read-only
DB user and a labeled question/SQL corpus. Skip if you want a coding
agent that edits files ([`opencode`](../opencode/), [`aider`](../aider/),
[`claude-code`](../claude-code/)) or a generic chat-with-data UI on top
of a data platform you do not own ([`db-gpt`](../db-gpt/) is a more
actively-maintained alternative for the same shape of problem).
