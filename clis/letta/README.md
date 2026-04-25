# letta

> Snapshot date: 2026-04. Upstream: <https://github.com/letta-ai/letta>
> License file: <https://github.com/letta-ai/letta/blob/main/LICENSE>
> Pinned: **v0.16.7** (2026-03-31, PyPI: `letta`).
> Companion CLI for end-user terminal agents lives at
> <https://github.com/letta-ai/letta-code> (npm: `@letta-ai/letta-code`);
> this entry is about the upstream `letta` server CLI that boots the
> stateful-agent runtime everything else talks to.

Letta (formerly **MemGPT**) is the long-running stateful-agent server.
The headline idea hasn't changed since the MemGPT paper: an agent has a
small **in-context memory window** (`core memory`, edited by the LLM
itself with `core_memory_replace` / `core_memory_append` tool calls) plus
unlimited **archival memory** (vector store) and **recall memory**
(message history) that the agent pages in and out via tool calls. The
LLM is trained-to-know it can rewrite its own system prompt; that single
primitive is what gives Letta agents the "they remember me across
sessions" behavior that flat chat REPLs can't fake.

The `letta` CLI in this repo is the typer-based server entry point
(`letta = "letta.main:app"` in `pyproject.toml`); it boots the FastAPI
agent runtime, runs migrations, and exposes the REST + gRPC + MCP
surface that the SDKs and the separate `letta-code` terminal client
consume.

## 1. Install footprint

- `pip install -U letta` (or `uv pip install letta`). Python 3.11+, <3.14.
- Heavy dependency tree: FastAPI + Uvicorn + SQLAlchemy + Alembic +
  pydantic 2 + llama-index + Anthropic + OpenAI + google-genai +
  Mistral + temporalio + opentelemetry + sentry-sdk + grpcio +
  fastmcp + datadog + clickhouse-connect. First install pulls ~600 MB.
- Storage: SQLite by default (`~/.letta/`); Postgres via the `postgres`
  extra (`pip install 'letta[postgres]'`) with pgvector for archival
  memory. Redis optional.
- Daemon shape: `letta server` starts the FastAPI app on `:8283`. The
  server is the source of truth — every CLI / SDK call hits it.
- Companion terminal CLI `letta-code` (separate npm package) is the
  end-user "code with my stateful agent" surface; it connects to either
  a local `letta server` or the hosted `app.letta.com`.

## 2. License

Apache-2.0 (verified: repo root `LICENSE`, 10759 bytes, full Apache-2.0
text). The `pyproject.toml` declares `license = {text = "Apache License"}`.

## 3. Models supported

LLM-agnostic via direct provider SDKs (not litellm): OpenAI (Chat +
Responses + Realtime), Anthropic, Google Gemini (AI Studio + Vertex),
Mistral, Bedrock (with the `bedrock` extra adding `boto3` + `aioboto3`),
Azure OpenAI, DeepSeek, Groq, Together, OpenRouter, Ollama,
llama.cpp / vLLM via OpenAI-compatible endpoints. Embedding models
configured separately (OpenAI by default; HF / Ollama / Bedrock
selectable for archival-memory vector store). Per-agent model and
embedding model are first-class — different agents on the same server
can run on different providers.

## 4. MCP support

**Yes — both client and server.** Server: every Letta agent's tool
surface is exposed over MCP via `fastmcp` (declared dependency
`fastmcp>=2.12.5`), so an external MCP client can list the agent's
tools and invoke them. Client: a Letta agent can mount external MCP
servers as tools (`mcp[cli]>=1.9.4`), so an agent can call into
filesystem / git / browser MCP servers the same way it calls its
built-in `core_memory_replace`. Stdio + HTTP/SSE transports both
supported.

## 5. Sub-agent model

**Yes — first-class via the `letta-code` companion CLI's subagent
primitive**, plus the underlying server supports multi-agent
orchestration directly: agents can list other agents on the server and
send them messages as tool calls (`send_message_to_agent_async` /
`send_message_to_agents_matching_tags`). The server-side composition is
"agents talk to agents over the same REST surface a human would use" —
no nested process model, all agents share the same Postgres / SQLite
state, and the message log is durable. The `subagents` feature in
`letta-code` is the ergonomic wrapper over that primitive for terminal
use.

## 6. Telemetry stance

**On by default, opt-out.** Sentry-SDK (`sentry-sdk[fastapi]==2.19.1`)
captures server errors; OpenTelemetry instrumentation (`opentelemetry-
instrumentation-requests`, `-sqlalchemy`) emits traces; Datadog
(`datadog>=0.49.1`, `ddtrace>=4.2.1`) wired in. Disable with the
standard environment variables (`SENTRY_DSN=`, `OTEL_SDK_DISABLED=true`,
`DD_TRACE_ENABLED=false`) or build a slim image without the
observability extras. Cloud sync to `app.letta.com` is a separate
explicit opt-in (set `LETTA_API_KEY`); the OSS server runs fully
self-hosted.

## 7. Prompt-cache strategy

Anthropic ephemeral cache (`cache_control: {type: "ephemeral"}`) wired
on the system prompt + core-memory blocks + tool definitions — exactly
the prefix that's stable across turns is exactly the prefix that's
cached, so the per-turn cost on a long-running agent is dominated by
the new user message + new tool results, not the system prompt. OpenAI
and Bedrock rely on automatic provider-side prefix cache. No
Letta-specific cache layer.

## 8. Hot keybinds (TUI / REPL)

The upstream `letta` package ships a server, not a TUI. The terminal
experience is in the separate `@letta-ai/letta-code` npm CLI, which
follows roughly the same shape as other modern coding TUIs:

| Key / command | Action |
|---------------|--------|
| `Enter` | Send turn |
| `Shift+Enter` | Newline in prompt |
| `Ctrl+C` | Cancel current LLM stream |
| `Ctrl+D` | Exit |
| `/agents` | List agents on the connected server |
| `/agent <name-or-id>` | Switch active agent |
| `/memory` | Show current core memory blocks |
| `/skills` | List loaded skills |
| `/clear` | Reset the in-flight turn (does **not** wipe agent memory — that's the point) |

For the `letta` CLI itself, useful one-shots:

- `letta server` — boot the FastAPI runtime
- `letta server --port 8283 --host 0.0.0.0` — bind for remote clients
- `letta migrate` — run Alembic migrations against the configured DB
- `letta version` — print server version

## 9. Killer feature, weakness, when to choose

- **Killer:** **Memory is the architecture, not a plugin.** Every other
  agent framework gives you a chat history and calls it memory. Letta
  gives the LLM tool calls to *edit its own system prompt*
  (`core_memory_replace`) and to *page in and out of unbounded archival
  memory* (`archival_memory_insert` / `archival_memory_search`), then
  durably persists every block to Postgres. The result is the only
  open-source agent that genuinely behaves like it remembers you across
  weeks of separate sessions, because the persistence model is the
  product, not an afterthought layered on top of an in-memory dict.
- **Weakness:** the heaviest dependency surface in this catalog. You
  are running a FastAPI server with Postgres + pgvector + Alembic
  migrations + Sentry + Datadog + OTel + temporalio whether you want
  them or not; the "just let me chat with a stateful agent in a
  terminal" path requires running `letta server`, then *also* installing
  the separate `@letta-ai/letta-code` npm CLI to get a usable client.
  Not the right pick if you want a single-binary terminal tool.
- **Choose it when:** you need agents that *persist* — a customer-
  support agent that should remember past tickets, a research assistant
  that should accumulate findings across weeks, a coding agent that
  should learn the conventions of *your* monorepo over time. Pair the
  server with `letta-code` for the terminal coding surface, or with the
  Python / TypeScript SDK for embedding into your own product. If your
  bar is "remembers things between sessions without me building a
  vector-store-and-prompt-stuffing layer myself," Letta is the answer.
