# mcp-agent

> Snapshot date: 2026-04. Upstream: <https://github.com/lastmile-ai/mcp-agent>

"**Build effective agents using Model Context Protocol and simple
workflow patterns.**" mcp-agent is a Python framework whose thesis
is that **MCP is enough**: every tool, retriever, and side-effect
arrives via an MCP server, and agent orchestration is a small set of
composable patterns lifted from Anthropic's *Building Effective
Agents* essay (augmented LLM, prompt chaining, routing, parallel,
orchestrator-workers, evaluator-optimizer, swarm). Adds first-class
**durable execution** via [Temporal](https://temporal.io/) so the
same agent code runs as a quick script or as a pause/resume/retry-safe
workflow without API changes. The CLI surface is `pip install
mcp-agent` plus library-mode use; Cloud deploy via the optional
managed `mcp-agent` Cloud.

## 1. Install footprint

- `pip install mcp-agent` (core; pulls the official `mcp` Python SDK,
  Pydantic v2, `httpx`, `anyio`).
- Provider extras: `mcp-agent[openai]`, `mcp-agent[anthropic]`,
  `mcp-agent[google]`, `mcp-agent[bedrock]`, `mcp-agent[azure]`.
- Durable runtime extra: `mcp-agent[temporal]` (adds the Temporal
  Python SDK; requires a Temporal cluster — `temporal server
  start-dev` for local).
- MCP server lifecycle is managed for you: declare server names in
  `mcp_agent.config.yaml` (or `MCPApp(servers=[...])`) and the
  framework handles spawn / connect / heartbeat / reconnect / shutdown.
- Python ≥ 3.10. Configuration is split: `mcp_agent.config.yaml`
  (committed) + `mcp_agent.secrets.yaml` (gitignored, env-var-merged).
- Optional `mcp-agent[ui]` for the included Streamlit / Gradio chat
  shells; `mcp-agent[eval]` for the eval harness.

## 2. Repo + version + license

- Repo: <https://github.com/lastmile-ai/mcp-agent>
- Latest release: **v0.0.21** (2026-01)
- License: **Apache-2.0** —
  <https://github.com/lastmile-ai/mcp-agent/blob/main/LICENSE>
- Default branch: `main`
- Language: Python

## 3. Models supported

Anthropic (Claude — first-class via `AnthropicAugmentedLLM`), OpenAI
(Chat + Responses via `OpenAIAugmentedLLM`), Google Gemini (AI Studio
+ Vertex via `GoogleAugmentedLLM`), AWS Bedrock
(`BedrockAugmentedLLM`), Azure OpenAI (`AzureOpenAIAugmentedLLM`),
Ollama / LM Studio / vLLM / llama.cpp / OpenRouter / Groq / Together
/ Fireworks / DeepSeek via the `OpenAIAugmentedLLM` with
`base_url` override (any OpenAI-compatible endpoint). Per-agent model
selection is the default — `agent.attach_llm(AnthropicAugmentedLLM)`
on the planner and `agent.attach_llm(OpenAIAugmentedLLM, model="gpt-5-mini")`
on a cheap critic in the same workflow.

## 4. MCP support

Yes — this is the project's whole point. Full client implementation:
**Tools, Resources, Prompts, Notifications, OAuth, Sampling,
Elicitation, Roots** all supported. Server lifecycle (stdio, SSE,
streamable-HTTP) is managed by the runtime; multiple servers per
agent (`server_names=["filesystem", "fetch", "github"]`) are stitched
together transparently. The inverse is also first-class: any
`mcp-agent` `Agent` or `MCPApp` can be **exposed as an MCP server**
via the `mcp_agent_server` helper, so agents become callable from
Claude Desktop, Cursor, [`opencode`](../opencode/),
[`continue`](../continue/), and any other MCP client.
FastMCP-compatible API for hand-rolling new servers.

## 5. Sub-agent model

Pattern-based, not class-hierarchy-based. Eight composable workflow
classes mirror Anthropic's effective-agents patterns:

- **Augmented LLM** — base unit; an LLM bound to a set of MCP servers
  and an optional `Memory`.
- **Prompt chaining** — explicit sequence of LLM calls, each consuming
  the previous output (`Chain([step1, step2, step3])`).
- **Router** — LLM-based dispatcher that picks one of N sub-agents per
  request.
- **Parallel** — fan-out to N agents, fan-in via an aggregator LLM.
- **Orchestrator-workers** — a planner LLM decomposes the task and
  dispatches sub-agents (the canonical "manager + workers" pattern).
- **Evaluator-optimizer** — generator + critic loop with stop
  condition.
- **Swarm** — handoff-based multi-agent (OpenAI Swarm port, MCP-native).
- **Intent classifier + IntentRouter** — for chatbot-style flows.

Patterns are **composable**: an `Orchestrator` worker can itself be a
`Chain`, a `Router`, or another `Orchestrator`. Memory is per-agent,
shared via the workflow's context dict.

## 6. Telemetry stance

Off by default in OSS. The framework runs in your Python / Temporal
process; egress is your model providers + your MCP servers. Built-in
OpenTelemetry instrumentation (every LLM call, tool call, workflow
step is a span) — enable by setting standard OTel env vars to ship
spans to Jaeger / Tempo / Honeycomb / Langfuse / your own collector.
Optional managed `mcp-agent` Cloud (last-mile) deploys workflows as
hosted MCP servers and provides a console for trace replay; opt-in
only.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **The pattern catalogue from Anthropic's essay,
implemented composably, on top of MCP, with optional Temporal
durability for free.** Most MCP-aware frameworks treat MCP as
"another tool format"; mcp-agent treats MCP as the *only* tool
format and earns simplicity from it — server lifecycle, OAuth,
sampling, elicitation, and reconnect logic disappear into the
runtime. The eight workflow patterns compose without ceremony
(`Orchestrator(planner=..., workers=[Router(...), Chain(...),
Evaluator(...)])`), and switching from "asyncio script" to "Temporal
workflow that survives a process crash and resumes on the next
worker" is a config change, not a rewrite. Exposing your agent as
an MCP server with one decorator means the same code is reachable
from Claude Desktop, Cursor, [`opencode`](../opencode/), and CI.

**Weakness.** **Pre-1.0 API churn and a Python-only surface.** v0.0.x
versioning means breaking changes between minors are expected — pin
exact versions and read CHANGELOG. No JS/TS port (use
[`voltagent`](../voltagent/) /
[`mastra`](../mastra/) on the TS side). Temporal is optional but is
the most polished durable-execution path; without it you get
in-process asyncio with no built-in checkpoint/replay (the OTel
spans help, but not the same thing). MCP server bring-up is your
problem — the framework manages lifecycles but you still configure
each server's command/env/secrets in YAML. Pushed less frequently
in 2026 than in 2025; the project is still actively shipping but
release cadence has slowed.

**When to choose.** You are **building** a Python agent product where
**every tool, retriever, and connector should be MCP** (because you
want the same tools usable from your agent and from Claude Desktop),
where you want Anthropic's effective-agents patterns out of the box
without writing a graph compiler, and where you want a clear path
from "asyncio script" to "Temporal workflow" without rewriting. Pair
with [`langfuse`](../langfuse/) / [`agentops`](../agentops/) for
tracing, with [`composio`](../composio/) when you need pre-built MCP
servers for SaaS apps, and with [`assistant-ui`](../assistant-ui/)
for the React chat layer. Skip if you want a graph DAG (use
[`langgraph`](../langgraph/)), a conversation-shaped multi-agent
(use [`ag2`](../ag2/) / [`crewai`](../crewai/)), a terminal coding
agent (use [`opencode`](../opencode/) / [`claude-code`](../claude-code/)
/ [`crush`](../crush/)), or a single-prompt Unix filter (use
[`mods`](../mods/) / [`smartcat`](../smartcat/) / [`llm`](../llm/)).
