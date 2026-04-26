# agent-kit

> Snapshot date: 2026-04. Upstream: <https://github.com/inngest/agent-kit>

"**Build multi-agent networks in TypeScript with deterministic routing
and rich tooling via MCP.**" AgentKit is Inngest's TypeScript framework
for composing multi-agent workflows where the orchestration is
**code, not LLM-decided**: you write a `Network` of `Agent`s with a
`Router` function (deterministic, LLM-routed, or hybrid), each `Agent`
has typed `tools`, and the whole thing runs either standalone or on top
of Inngest's durable execution engine for retries / replay / human
approval. The CLI surface is `npm install @inngest/agent-kit` for the
library; `npx inngest-cli@latest dev` for the optional local dev server
that gives you a step-debug UI.

## 1. Install footprint

- `pnpm add @inngest/agent-kit` (core); model providers piggyback on
  the [Vercel AI SDK](https://sdk.vercel.ai/) — `pnpm add @ai-sdk/anthropic
  @ai-sdk/openai` etc.
- Optional `pnpm add inngest` to lift the network onto Inngest's
  durable runtime; without it AgentKit runs as a plain in-process
  framework.
- `npx inngest-cli@latest dev` (no install) for the local dev server +
  step-debug UI at `http://localhost:8288`.
- Node ≥ 20. Works on Node, Bun, Deno, Cloudflare Workers, Vercel
  Edge.

## 2. Repo + version + license

- Repo: <https://github.com/inngest/agent-kit>
- Latest release: **`@inngest/agent-kit` v0.13.2** (2025-11-13)
- License: **Apache-2.0** —
  <https://github.com/inngest/agent-kit/blob/main/LICENSE.md>
- Default branch: `main`
- Language: TypeScript

## 3. Models supported

Anthropic (Claude), OpenAI (Chat + Responses), Google (Gemini AI Studio
+ Vertex), Groq, xAI Grok, Mistral, Cohere, DeepSeek, Cerebras,
Together, Fireworks, OpenRouter, Perplexity, AWS Bedrock, Ollama,
LM Studio, vLLM, llama.cpp, any OpenAI-compatible endpoint. Routing
is delegated to the [Vercel AI SDK](https://sdk.vercel.ai/)
provider abstraction, so anything `ai` supports works (≈40 providers
as of v5.x). Per-`Agent` model assignment is first-class —
`model: anthropic("claude-sonnet")` on the planner,
`model: openai("gpt-4o-mini")` on the cheap classifier.

## 4. MCP support

Yes (client), and it is a headline feature. Pass `mcpServers: [{ name,
transport: { type: "sse" | "stdio" | "ws", ... } }]` to an `Agent`
constructor and AgentKit auto-discovers the server's tools, translates
their JSON schemas into typed `Tool` shapes, and routes calls
transparently. Multiple MCP servers per agent are supported with
auto-namespacing on tool collisions. MCP support also works in the
Inngest-durable mode, so MCP tool calls become checkpointed steps that
survive process restarts.

## 5. Sub-agent model

Three composition primitives:

- `Agent` — the unit. Has `model`, `tools`, `lifecycle` hooks
  (`onStart`, `onResponse`, `onFinish`), `system` prompt, optional
  `mcpServers`.
- `Network` — a graph of `Agent`s plus a `defaultModel` and a
  `router`. The `router` is a TypeScript function (sync or async) that
  inspects the network state and returns the next `Agent` to run, or
  `undefined` to terminate. You can write deterministic routing (state
  machine), LLM routing (call a model to decide), or hybrid
  (deterministic with LLM fallback). This is the framework's central
  bet: **routing is your code, not the LLM's whim.**
- `State` — a typed shared store passed between agents in a `Network`,
  with helpers (`state.kv.set`, `state.kv.get`, `state.results`) so
  agents can leave structured artefacts for the router and downstream
  agents to read.

When wrapped in an Inngest `step.run`, every agent invocation, tool
call, and router decision becomes a durable, replayable step — failures
retry exactly once per attempt, human-in-the-loop pauses are
first-class via `step.waitForEvent`.

## 6. Telemetry stance

Off by default in OSS. The framework runs entirely in your Node
process; egress is your model providers + your DB + optional MCP
servers. Optional opt-in tracing via the **Inngest Cloud** dashboard
(when running on Inngest's durable runtime) which shows every step,
retry, sleep, and event — turn off by not setting
`INNGEST_EVENT_KEY` / `INNGEST_SIGNING_KEY` and running in plain
in-process mode. The local `npx inngest-cli dev` server stays on
`localhost:8288` and ships no spans externally.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **Deterministic, code-defined routing on top of
durable execution — multi-agent workflows that don't lose state when
the process dies.** Where most multi-agent frameworks make the LLM
pick the next agent (and silently retry from scratch if the process
crashes), AgentKit makes you write a `router(state) => nextAgent`
function in TypeScript, then optionally lifts the whole `Network` onto
Inngest's durable engine so each agent run, each MCP tool call, and
each router decision is a checkpointed step. A 30-minute multi-agent
research run that crashes at minute 27 resumes from the last completed
step instead of restarting; a workflow with a "wait for human approval"
step pauses for days without keeping a process pinned. Per-agent model
mixing on the [Vercel AI SDK](https://sdk.vercel.ai/) means a
TypeScript codebase can use AgentKit alongside the rest of its `ai`
SDK calls without a second provider abstraction. MCP tools as durable
steps is the cleanest version of the pattern in any TS framework.

**Weakness.** **The full value lands only when you adopt Inngest as
your runtime.** Standalone, AgentKit is "yet another TS agent
framework" with deterministic routing — nice but not unique.
Durability, replay, the step-debug UI, the human-in-the-loop pauses,
and the production observability all assume `inngest` is in the
dependency tree and either an Inngest dev server or Inngest Cloud is
reachable. Self-hosting Inngest is possible but operationally
non-trivial (Postgres + Redis + the Inngest server). Slow release
cadence (v0.13 over ~12 months); the framework is intentionally small
and stable but lacks the batteries-included tool catalogue of
[`agno`](../agno/) or [`upsonic`](../upsonic/) — you wire MCP servers
or write `Tool`s yourself. Documentation skews toward the Inngest
Cloud path.

**When to choose.** You are **building** a TypeScript multi-agent
workflow where **state survival across restarts** matters (long-running
research crews, multi-step ETL with LLM steps, ticket-routing pipelines
with human approval, scheduled retries against flaky tools) and you
already use or are willing to adopt [Inngest](https://www.inngest.com/)
for durable execution. Pair with [`voltagent`](../voltagent/) if you
want a richer in-process tracing UI without Inngest, with
[`mastra`](../mastra/) if you want a TS framework with a built-in
workflow runtime that does not depend on Inngest, or with
[`pydantic-ai`](../pydantic-ai/) /
[`upsonic`](../upsonic/) for the Python equivalent of typed,
deterministic agent control. Skip if you want a terminal coding agent
(use [`opencode`](../opencode/) / [`claude-code`](../claude-code/) /
[`crush`](../crush/) instead), a single-prompt Unix filter (use
[`mods`](../mods/) / [`llm`](../llm/) / [`smartcat`](../smartcat/)),
or a Python framework (use [`agno`](../agno/) /
[`pydantic-ai`](../pydantic-ai/) / [`crewai`](../crewai/)).
