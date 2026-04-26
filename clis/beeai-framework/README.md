# beeai-framework

> Snapshot date: 2026-04. Upstream: <https://github.com/i-am-bee/beeai-framework>

A **dual-language (Python + TypeScript) agent framework** from the
BeeAI project (LF AI & Data Foundation sandbox). Aimed squarely at
production deployments: typed tools, structured outputs, configurable
memory backends, OpenTelemetry tracing, and a runtime-agnostic agent
loop that can be embedded in a FastAPI service or a Node worker
without dragging in a graph engine.

The differentiator is **language parity**. Most agent frameworks
either pick Python (LangChain-family, smolagents, agno) or pick
TS (Mastra, AI SDK). BeeAI ships both with the same primitives so a
team with a Python data backend and a TS edge runtime does not have
to maintain two mental models.

## 1. Install footprint

- **Python:** `pip install beeai-framework` or `uv add beeai-framework`.
  ~8 MB plus optional extras for MCP (`beeai-framework[mcp]`),
  watsonx, and tracing exporters.
- **TypeScript:** `npm install beeai-framework` or
  `pnpm add beeai-framework`. Pure JS bundle, ~3 MB plus deps.
- No daemon. State lives in pluggable `Memory` backends
  (in-memory, summarising-memory, sliding-window, or your own).
- No global config; the framework is library-only. Credentials
  flow through the underlying provider SDKs (`OPENAI_API_KEY`,
  `WATSONX_*`, etc.).

## 2. License

Apache-2.0. `LICENSE` at the repo root.

## 3. Models supported

Provider adapters in core:

- **OpenAI** and any OpenAI-compatible endpoint (vLLM, Together,
  Groq, Fireworks, OpenRouter via base-url override).
- **Anthropic Claude.**
- **Google Gemini** (AI Studio + Vertex).
- **IBM watsonx.ai** — first-class, since the project originated
  inside IBM Research.
- **Ollama** for local models.
- **Groq, xAI Grok, AWS Bedrock** via dedicated adapters.

Models are constructed via `ChatModel.from_name("openai:gpt-4.1")`
style URIs in both languages.

## 4. MCP support

**Yes, both as client and server.** The `MCPTool` adapter wraps any
MCP server's tools so a BeeAI agent can call them. There is also an
**MCP server export** that turns a BeeAI agent into an MCP server
other clients can consume — useful for offering an agent as a tool
to `claude-code`, `goose`, `opencode`, or any other MCP client.

## 5. Sub-agent model

Two layers:

- **Workflows** — declarative composition of agents into a graph
  (sequential, parallel, conditional). Lower friction than LangGraph
  for most cases; you do not write node functions, you wire agents.
- **Handoffs** — a `RequirementAgent` can hand off to a specialist
  agent mid-loop using a built-in handoff tool, similar in spirit
  to the openai-agents-python pattern but explicit rather than
  derived from the agents list.

There is also a `MultiAgentWorkflow` helper for the common
"orchestrator + workers" pattern.

## 6. Telemetry stance

**Off by default; OpenTelemetry-ready.** The framework emits
structured spans (LLM calls, tool calls, handoffs, retries) but
will not export them anywhere unless you configure an OTel
exporter or one of the supported observability integrations
(Arize Phoenix, Langfuse, Logfire). No vendor dashboard is wired
in by default.

## 7. Prompt-cache strategy

Provider pass-through. The Anthropic adapter accepts
`cache_control` blocks on system messages; the OpenAI Responses
adapter benefits from automatic prefix caching. The framework
itself does not insert cache markers. Memory backends do the
related job of keeping the conversation prefix stable across
turns (which is the precondition for provider-side caching to
actually help).

## 8. Hot keybinds

No TUI. Relevant entry points:

**Python**

- `agent = RequirementAgent(llm=..., tools=[...], memory=...)`
- `await agent.run("input")` — async one-shot
- `agent.run("input", stream=True)` — streaming events
- `Workflow(...).add_step(agent_a).add_step(agent_b).run(...)`
- `from beeai_framework.adapters.mcp import MCPTool`
- `agent.serve(mcp=True)` — expose as an MCP server

**TypeScript**

- `const agent = new RequirementAgent({ llm, tools, memory })`
- `await agent.run({ prompt: "..." })`
- `for await (const ev of agent.run(...).stream())`

CLI surface is intentionally minimal — there is a `beeai`
helper from the parent BeeAI platform repo, but the framework
itself is consumed as a library.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Same primitives in Python and TypeScript,
plus MCP both ways.** You can prototype an agent in Python, port
it line-for-line to TS for an edge deployment, and expose either
version as an MCP server that other tools in this catalog can
consume. That round-trip is unique among the production-oriented
frameworks here.

**Weakness.** Smaller ecosystem than LangChain / LlamaIndex /
agno; fewer pre-built tools and integrations, so you write more
adapters yourself. Documentation is improving but still trails
the bigger players. The TS and Python APIs are aligned in spirit
but not 1:1 identical — porting between them requires light
translation, not a literal copy.

**When to choose.**

- You have a **mixed Python + TS stack** and want one agent
  framework whose mental model survives the language switch.
- You need to **publish an agent as an MCP server** so it is
  consumable from `claude-code`, `goose`, `opencode`, etc.
- You want **OpenTelemetry-native tracing** from day one, no
  vendor lock-in, and OSS-foundation governance (LF AI & Data).
- You target **IBM watsonx** alongside the usual frontier
  providers and want first-class adapter support, not a community
  plugin.
- You want a **smaller, more readable core** than LangChain /
  LangGraph but more structure than rolling your own loop on top
  of the OpenAI SDK.

**When not to choose.** You need a huge pre-built tool catalog
out of the box, you want a no-code agent builder, you need a
single-binary terminal CLI rather than a library, or your stack
is single-language and you would rather pick the strongest
single-language option (`pydantic-ai` for Python,
`mastra` for TS).
