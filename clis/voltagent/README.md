# voltagent

> Snapshot date: 2026-04. Upstream: <https://github.com/voltagent/voltagent>

"**Open Source TypeScript AI Agent Framework.**" VoltAgent is a
batteries-included TypeScript framework for building, observing, and
shipping AI agents — `Agent`, `Tool`, `Memory`, `Workflow`, and
`SubAgent` primitives, an HTTP server (`@voltagent/server-hono`) that
exposes them, and a hosted **VoltOps** console that talks back to your
local process for live tracing, prompt diffing, and replay. The CLI
surface is `npm create voltagent-app@latest` (scaffold) plus
`@voltagent/cli` (`volt deploy`, `volt logs`, `volt secrets`) for
managing the optional VoltOps integration.

## 1. Install footprint

- `npm create voltagent-app@latest my-agent` scaffolds a fresh
  TypeScript project with `@voltagent/core`, `@voltagent/server-hono`,
  and a provider package preselected.
- `pnpm add @voltagent/core @voltagent/server-hono` for an existing
  repo; provider packages are separate (`@voltagent/anthropic-ai`,
  `@voltagent/google-ai`, `@voltagent/groq-ai`, etc.) and the framework
  also accepts any [Vercel AI SDK](https://sdk.vercel.ai/) provider via
  `@voltagent/vercel-ai`.
- Node ≥ 20. Runtime is Hono on Node / Bun / Deno / Cloudflare Workers
  (the server package picks the adapter).
- `npm i -g @voltagent/cli` for the optional `volt` binary used to
  authenticate and push to VoltOps.

## 2. Repo + version + license

- Repo: <https://github.com/voltagent/voltagent>
- Latest release: **`@voltagent/core` v2.7.3** (2026-04-25; monorepo
  publishes per-package, latest tagged release `@voltagent/server-hono@2.0.13`)
- License: **MIT** —
  <https://github.com/voltagent/voltagent/blob/main/LICENCE>
- Default branch: `main`
- Language: TypeScript

## 3. Models supported

Anthropic (Claude), OpenAI (Chat + Responses), Google (Gemini AI Studio
+ Vertex), Groq, xAI Grok, Mistral, Cohere, DeepSeek, Cerebras,
Together, Fireworks, OpenRouter, Perplexity, AWS Bedrock, Ollama (any
local GGUF), LM Studio, vLLM, llama.cpp, any OpenAI-compatible endpoint
via `OpenAICompatible`. Because the framework also accepts Vercel AI
SDK providers transitively, anything supported by `ai` works (≈40
providers). Per-agent and per-`SubAgent` model assignment is
first-class — `model: anthropic("claude-sonnet")` here,
`model: groq("llama-3.3-70b")` there.

## 4. MCP support

Yes (client). `@voltagent/mcp` mounts an MCP server's tools onto any
`Agent` via `mcpClient({ url })` or stdio transport; the framework
handles discovery, schema translation to its `Tool` shape, and
session-scoped lifecycles. Multiple MCP servers per agent are
supported, and tool routing is automatic. No first-party
`voltagent --mcp-server` mode (the runtime is HTTP / Hono, not stdio
MCP), but the Hono server can be wrapped externally.

## 5. Sub-agent model

Three composition primitives:

- `Agent` — the unit. Has `model`, `tools`, `memory`, `instructions`,
  optional `hooks` (lifecycle callbacks), `retriever` (RAG), and a
  `subAgents` array.
- `SubAgent` — a child `Agent` invoked by the parent through an
  auto-generated `delegate` tool. Each subagent keeps its own model,
  tool set, and memory namespace; the parent decides when to delegate
  and how to synthesise. Recursion is supported.
- `Workflow` — explicit DAG of steps; each step can be an `Agent`, a
  function, or a nested `Workflow`. Supports branching, parallel
  execution, conditional edges, and human-in-the-loop pauses.

Sessions and memory are scoped per-conversation by the `Memory`
adapter (LibSQL / Postgres / in-memory), so a parent + N subagents
share a thread without trampling state.

## 6. Telemetry stance

Off by default in OSS. The framework runs entirely in your Node
process; egress is your model providers + your DB. Optional opt-in
tracing via **VoltOps** (hosted at `console.voltagent.dev`) which
streams agent / tool / subagent spans from your local process for
live observability and prompt versioning — turn off by not setting
`VOLTAGENT_PUBLIC_KEY` / `VOLTAGENT_SECRET_KEY`. OpenTelemetry export
is opt-in via `@voltagent/observability` and the standard OTel env vars,
so you can ship spans to your own backend (Jaeger, Tempo, Honeycomb,
Datadog) instead.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **TypeScript-native sub-agent delegation that you
can actually trace live in a browser.** Where most multi-agent
frameworks force you to log spans to a file and grep, VoltAgent ships
the framework + a Hono HTTP runtime + a console UI as one coherent
product: declare `subAgents: [researcher, writer, critic]` on a parent
`Agent`, the framework auto-generates a `delegate` tool with typed
schemas, and every delegation, tool call, retry, and token stream
shows up in the VoltOps timeline keyed by trace ID. Per-agent and
per-subagent model assignment is the default (`model: anthropic(...)`
on the planner, `model: groq(...)` on the cheap reranker). The same
TypeScript file runs in `tsx watch`, `bun run`, and as a deployed
Cloudflare Worker — the surface does not change. For TS shops that
already standardised on the [Vercel AI SDK](https://sdk.vercel.ai/),
VoltAgent is the agent-loop layer on top.

**Weakness.** **Framework-shaped, not coding-CLI-shaped, and the
hosted console is the best UX.** The `volt` binary is a project
scaffolder + a VoltOps deploy CLI; there is no "edit my repo with
natural language" mode out of the box. You write TypeScript that
*uses* the framework, not commands you type at a terminal. VoltOps
itself is the most polished trace viewer on offer but it is a SaaS
endpoint — air-gapped users get OpenTelemetry export and their own
backend instead of the click-through replay UI. Monorepo per-package
versioning (`@voltagent/core@2.7.3`,
`@voltagent/server-hono@2.0.13`, `@voltagent/anthropic-ai@1.x`) means
you have to pin each package individually; mismatched minors between
`core` and a provider package have caused breaking-change issues in
2.x. Rapid release cadence (≥1 release/day across the monorepo).

**When to choose.** You are **building** a TypeScript agentic product
(internal RAG assistant, customer support bot, research crew, browser
automation) and want the framework + the HTTP runtime + sub-agent
delegation + live trace UI from one project, and you already standardised
on Node / Bun / Hono / Vercel AI SDK. Pair with
[`pydantic-ai`](../pydantic-ai/) on the Python side of a polyglot stack,
or with [`agno`](../agno/) if you prefer the Python framework that ships
the same "framework + runtime + console" triangle. Skip if you want a
terminal coding agent (use [`opencode`](../opencode/) /
[`claude-code`](../claude-code/) / [`crush`](../crush/) instead) or a
single-prompt Unix filter (use [`mods`](../mods/) /
[`smartcat`](../smartcat/) / [`llm`](../llm/)).
