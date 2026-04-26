# openai-agents-python

> Snapshot date: 2026-04. Upstream: <https://github.com/openai/openai-agents-python>

OpenAI's official **multi-agent orchestration framework** for Python.
A lightweight evolution of the experimental "Swarm" reference
implementation, hardened into a production SDK with handoffs,
guardrails, sessions, tracing, and a real tool-loop. It is not a
standalone CLI in the `aichat` / `mods` sense — it is a Python
library with a CLI surface (`uvx openai-agents`) for running and
inspecting agent definitions, plus a tracing dashboard hook.

The framework's whole pitch is **minimal primitives**: an `Agent`
(LLM + instructions + tools), a `Runner` (the loop), `Handoff`
(transfer control between agents), `Guardrail` (input/output
validators), `Session` (memory). No DAGs, no graph builders, no
node editors. If you can read 200 lines of Python you can read
the whole framework.

## 1. Install footprint

- `pip install openai-agents` or `uv add openai-agents`.
- Pure Python, ~5 MB plus the `openai` SDK. Optional extras for
  voice (`openai-agents[voice]`), litellm-routed providers
  (`openai-agents[litellm]`), and Redis sessions
  (`openai-agents[redis]`).
- No daemon. State lives in whatever `Session` backend you wire up
  (in-memory by default, SQLite / Redis / SQLAlchemy as extras).
- Config is code: there is no `~/.config/openai-agents/`. The
  `OPENAI_API_KEY` env var is the only required setting.

## 2. License

MIT. `LICENSE` at the repo root.

## 3. Models supported

- **OpenAI** — first-class via the standard `openai` SDK
  (Responses API by default, Chat Completions opt-in).
- **Anything OpenAI-compatible** — point `AsyncOpenAI` at a
  custom `base_url` (Groq, Together, Fireworks, vLLM, llama.cpp
  server, OpenRouter, local Ollama via its OpenAI-compat shim).
- **Anthropic / Gemini / Bedrock / 100+ others** — install the
  `litellm` extra and use `LitellmModel(model="claude-3.7-sonnet")`
  as the agent's `model` argument.

## 4. MCP support

**Yes, first-class.** `MCPServerStdio`, `MCPServerSse`, and
`MCPServerStreamableHttp` classes are in core. You attach a server
to an agent via `Agent(mcp_servers=[server])` and its tools become
callable in the agent loop. Tool listings are cached per server
with a configurable TTL. This is one of the cleanest MCP client
surfaces in the catalog.

## 5. Sub-agent model

Multi-agent is the headline feature. Two patterns:

- **Handoffs** — `agent_a.handoffs = [agent_b]` exposes "transfer
  to agent_b" as a tool to agent_a. The runner swaps active agent
  and continues the same conversation.
- **Agents-as-tools** — wrap an agent with `agent.as_tool(...)` to
  expose it as a callable function to a parent agent without
  transferring control. Good for a triage-agent that fans out to
  specialists and aggregates.

The runner handles the loop, max-turns cap, and guardrail
short-circuit. No planner/builder split is imposed — you compose
whatever topology you want from the two primitives.

## 6. Telemetry stance

**Tracing is on by default and goes to OpenAI's dashboard.** Every
agent run produces a trace (spans for each LLM call, tool call,
handoff, guardrail). Disable globally with
`set_tracing_disabled(True)` or per-run via
`RunConfig(tracing_disabled=True)`. You can also redirect to a
custom processor (`add_trace_processor`) to send to Logfire,
Langfuse, Braintrust, Weights & Biases, AgentOps, Scorecard, or
your own OTel collector — integrations for all of those ship in
the README.

If you are sensitive about sending agent inputs/outputs to OpenAI
even when using non-OpenAI models, **explicitly disable tracing
or wire a non-OpenAI processor before your first run**.

## 7. Prompt-cache strategy

Pass-through to the underlying provider. The Responses API caches
system-prompt prefixes automatically when reused. For Anthropic
via the `litellm` extra, set `cache_control` blocks on your
instructions through `model_settings.extra_body`. The framework
itself does not insert cache markers.

`Session` is the relevant memory primitive — it deduplicates
conversation history across turns so you are not paying to re-send
the entire transcript every loop iteration.

## 8. Hot keybinds

No TUI. The relevant surfaces are:

- `Runner.run(agent, "input")` — async one-shot
- `Runner.run_sync(...)` — blocking variant
- `Runner.run_streamed(...)` — streaming events (deltas + tool
  calls + handoffs as they happen)
- `agents.repl.run_demo_loop(agent)` — built-in interactive REPL
  for poking at an agent from a terminal
- `uvx openai-agents-mcp inspect <server>` — list tools an MCP
  server exposes
- Tracing dashboard at <https://platform.openai.com/traces> when
  default tracing is on

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Handoffs as a first-class primitive backed
by tracing.** Most multi-agent frameworks reinvent process-style
coordination on top of LLM tool calls and then make you bring your
own observability. This ships handoffs, guardrails, sessions, and
spans-per-step traces in one box, with a 200-line core you can
actually read. The MCP client is a real one (stdio + SSE +
streamable-HTTP), not an afterthought.

**Weakness.** Opinionated toward the OpenAI Responses API and the
OpenAI tracing backend; using non-OpenAI models is a one-line swap
but the defaults push you toward OpenAI infra. No graphical agent
builder, no DAG editor, no checkpoint/resume of long-running runs
out of the box (you build that on top of `Session`). Python-only —
the JS / TS sibling lives in a different repo with different
ergonomics.

**When to choose.**

- You want a **small, readable multi-agent SDK** with handoffs and
  agents-as-tools, not a 50k-line graph framework.
- You need **MCP client support that actually works today** in a
  Python agent loop.
- You want **per-step tracing for free** and you are happy with
  the OpenAI dashboard or one of the supported OTel exporters.
- You are building **production agent services** and want
  guardrails (input + output validators) as a built-in concept,
  not a bolt-on.
- You are migrating off the experimental Swarm reference and want
  the supported successor.

**When not to choose.** You want a no-code agent builder, you want
a single-binary terminal CLI rather than a Python library, you
need a graph/DAG runtime with checkpointing, or you cannot tolerate
any default-on telemetry and do not want to remember to disable it
at startup.
