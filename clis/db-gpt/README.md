# db-gpt

> Snapshot date: 2026-04. Upstream: <https://github.com/eosphoros-ai/DB-GPT>
> License file: <https://github.com/eosphoros-ai/DB-GPT/blob/main/LICENSE>
> Pinned: **v0.8.0** (2026-03-27, PyPI: `dbgpt`).

DB-GPT is an **agent framework whose entire ergonomics revolve around
talking to your data**. Where most agent CLIs treat databases as one
optional tool among many, DB-GPT treats SQL — schema introspection,
text-to-SQL, multi-database join planning, result-set explanation,
chart rendering, RAG over schema documentation — as the primary
surface, and bolts a general-purpose multi-agent loop (AWEL: Agentic
Workflow Expression Language) on top.

The `dbgpt` CLI is the all-in-one entry point: it boots the FastAPI
web server (default `:5670`), runs the agent runtime, manages model
deployments, registers data sources, and ships subcommands for
knowledge-base ingestion, agent invocation, and one-shot SQL
generation. Bundled connectors cover MySQL, PostgreSQL, ClickHouse,
SQLite, Oracle, MS SQL, DuckDB, Spark, Hive, Doris, StarRocks,
Snowflake, BigQuery, plus several vector stores (Milvus, Chroma,
Weaviate, OceanBase Vector, ElasticSearch).

## 1. Install footprint

- `pip install "dbgpt[default]"` (or `uv pip install "dbgpt[default]"`)
  for the standard server + agent runtime. Slim install:
  `pip install dbgpt` — core only, no connectors.
- Install extras line up with the surface you want:
  `dbgpt[openai]`, `dbgpt[anthropic]`, `dbgpt[gemini]`,
  `dbgpt[ollama]`, `dbgpt[vllm]`, `dbgpt[milvus]`, `dbgpt[chroma]`,
  `dbgpt[mysql]`, `dbgpt[postgres]`, `dbgpt[clickhouse]`, ...
- Python 3.10+. Linux + macOS are the primary targets; Windows works
  via WSL.
- Daemon shape: `dbgpt start webserver` runs FastAPI + Uvicorn on
  `:5670`; `dbgpt start worker` runs a model-worker process you can
  scale horizontally; `dbgpt start controller` runs the model
  registry. Single-process mode (`dbgpt start webserver --controller-
  addr=auto`) is the default for local use.
- Storage: SQLite by default at `pilot/meta_data/`; swap to MySQL
  via `dbgpt-config.toml`. Knowledge bases live as files +
  vector-store rows.
- Docker images published per release; `docker compose` recipe in the
  repo wires server + Milvus + MySQL with one command.

## 2. License

MIT (verified: repo root `LICENSE`, 1067 bytes, MIT text). The
project is governed by the eosphoros-ai community; contributions are
under DCO sign-off.

## 3. Models supported

Provider-agnostic, with a long list of first-class adapters: OpenAI
(Chat + Responses), Anthropic, Google Gemini (AI Studio + Vertex),
DeepSeek, Moonshot, Zhipu (`glm-4`), Qwen (DashScope + open weights),
Yi, Baichuan, Spark, MiniMax, Doubao, Bedrock, Azure OpenAI, plus
fully-local hosting via Ollama / vLLM / FastChat / llama.cpp /
TensorRT-LLM. Embedding model registry mirrors the LLM one: OpenAI,
Bedrock, Cohere, plus local sentence-transformers and Qwen
embeddings. Per-data-source you can pin a different model — the
text-to-SQL agent on Qwen, the chart-explainer on Claude, the
embeddings on a local bge model.

## 4. MCP support

**Yes (client).** As of the v0.7.x line DB-GPT exposes an MCP-tool
adapter so any agent in an AWEL workflow can mount external MCP
servers as tools alongside the built-in SQL / chart / RAG tools; the
recently-added MCP gateway lets the web UI discover MCP servers from
a config file and surface their tools in the agent builder. Server
mode (exposing DB-GPT's own data tools over MCP for an external
client to consume) is on the roadmap but not yet shipping in v0.8.0;
the REST API is the canonical external surface today.

## 5. Sub-agent model

**Yes — AWEL (Agentic Workflow Expression Language) is the entire
composition primitive.** AWEL is a Python DSL of operators
(`MapOperator`, `JoinOperator`, `BranchOperator`, `ReduceStreamOperator`,
`HumanInLoopOperator`) that wire agents and tools into a DAG. The
shipped templates include:

- **DataAgent** — text-to-SQL → execute → explain → chart
- **PlannerAgent** — decomposes a user goal into AWEL steps
- **CodeAssistantAgent** — code reasoning with retrieval
- **DashboardAgent** — multi-step BI report generation
- **ToolAssistantAgent** — generic tool-use loop

You compose them by writing an AWEL DAG (`with DAG("my-flow") as dag:
... node_a >> node_b >> node_c`), or by clicking through the
no-code builder in the web UI which serializes to the same AWEL
representation. Sub-agents share the agent registry and a common
memory store (configurable: in-process, Redis, or the metadata DB).

## 6. Telemetry stance

**Off in OSS.** No analytics SDK in the default install path.
Optional OpenTelemetry hooks are opt-in via `dbgpt-config.toml`
(`[trace]` section pointing at your collector). Egress = your
configured LLM provider + your configured data sources + the URLs
your agents fetch. Self-hosted by design.

## 7. Prompt-cache strategy

Anthropic ephemeral cache wired on the system prompt + tool
definitions for the Claude adapter; OpenAI / Bedrock rely on
provider-side automatic prefix cache. DB-GPT's own contribution is
**schema-cache**: the text-to-SQL agent caches per-database schema
introspection (table list, column types, FK graph, sample rows) in
the metadata DB so the first prompt of each session doesn't pay the
schema-fetch round trip. Invalidate with `dbgpt knowledge refresh`.

## 8. Hot keybinds (TUI / REPL)

DB-GPT is primarily a web UI (browser at `:5670`); there is no
Textual / blessed TUI. The CLI surface is one-shot subcommands.
Useful invocations:

| Command | Action |
|---------|--------|
| `dbgpt start webserver` | Boot the all-in-one server |
| `dbgpt start webserver --port 5670` | Bind a different port |
| `dbgpt start worker --model-name qwen2.5-72b --model-type llm` | Start a model worker |
| `dbgpt knowledge load --space my_docs --vector_store_type Chroma --local_doc_path ./docs` | Ingest a doc set into a knowledge base |
| `dbgpt knowledge list` | List knowledge bases |
| `dbgpt trace list` | Show recent agent traces |
| `dbgpt model list` | List registered model deployments |
| `dbgpt --help` | Subcommand catalog |

In the web chat, the standard browser shortcuts apply
(`Enter` to send, `Shift+Enter` for newline, `Esc` to close
side-panels). Every conversation is durably stored in the metadata
DB and replayable from the **Trace** tab — same session resumable
from a different browser.

## 9. Killer feature, weakness, when to choose

- **Killer:** **Production-grade text-to-SQL with a real connector
  catalog**, not a notebook demo. The `DataAgent` template wires
  schema introspection → schema-aware retrieval → SQL synthesis →
  execution against the actual database → result interpretation →
  chart rendering as one AWEL DAG, with safe-mode SQL filtering
  (`SELECT`-only enforcement, row-cap injection, dry-run plan
  preview) and per-database privilege scoping. The connector list is
  long enough that "wire DB-GPT to the warehouse and let analysts
  ask questions in Chinese / English" is a one-day install, not a
  six-month project.
- **Weakness:** **heavy and opinionated.** You are deploying a
  FastAPI server, a model-worker registry, a metadata DB, a vector
  store, and a web UI even if all you wanted was "ask Claude a SQL
  question from the terminal." The slim `pip install dbgpt` is
  genuinely slim, but the moment you add data-source extras the
  dependency tree explodes (oracledb, pyhive, snowflake-connector,
  ibm-db, ...). Documentation is bilingual (Chinese + English) and
  the English side trails the Chinese side by a few releases.
- **Choose it when:** the *job* is "let non-engineers query our
  databases / data lake in natural language and get correct,
  explained, charted answers," and you have ops capacity to run a
  small server. If you only need a CLI that pipes a NL question to a
  hosted LLM and prints SQL, [`aichat`](../aichat/) plus a custom
  function or [`llm`](../llm/) plus a tiny plugin is lighter; if you
  want a multi-database analytics agent with a real UI and a real
  connector catalog, DB-GPT is the most complete OSS option.
