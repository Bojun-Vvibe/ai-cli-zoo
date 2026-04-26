# agency-swarm

> Snapshot date: 2026-04. Upstream: <https://github.com/VRSEN/agency-swarm>

A **Python multi-agent framework** built directly on top of the
OpenAI Assistants / Responses APIs (and, since v1, the
openai-agents SDK underneath). Models a team of agents as an
*agency*: a directed graph of who-can-talk-to-whom, where each
agent has a role, instructions, tools, and a defined set of peers
it is allowed to message. Pitched at "give your agents real jobs",
not at chat-style prototypes.

The mental model is unusual in this catalog: instead of a single
orchestrator + workers, every agent is a peer with a mailbox, and
the *agency chart* declares the communication edges. Practical
effect: you can build a "CEO talks to CTO and CMO; CTO talks to
two engineers; CMO talks to a copywriter" topology in ~20 lines,
and the framework enforces the edges.

## 1. Install footprint

- `pip install agency-swarm` (v1.x line, on top of openai-agents)
  or pin `agency-swarm<1` for the legacy Assistants-API line.
- Pure Python, ~10 MB plus the `openai` SDK and (in v1)
  `openai-agents`.
- State for the legacy line lived in OpenAI's Assistants API
  (threads, files, vector stores). The v1 line uses
  openai-agents `Session` backends instead, so state is local
  unless you opt into a hosted backend.
- Config is code; `OPENAI_API_KEY` is the only required env var
  for the default OpenAI path.

## 2. License

MIT. `LICENSE` at the repo root.

## 3. Models supported

- **OpenAI** — first-class (Responses API by default in v1,
  Assistants API in legacy).
- **Anything OpenAI-compatible** — swap the underlying client's
  `base_url` (Groq, Together, Fireworks, vLLM, llama.cpp server,
  OpenRouter, local Ollama via OpenAI-compat).
- **Anthropic / Gemini / Bedrock / 100+** — through the
  openai-agents `litellm` extra in v1, or community adapters
  on the legacy line.

The framework's strongest defaults assume an OpenAI-compatible
backend with function calling and structured outputs.

## 4. MCP support

**Yes, via the openai-agents foundation in v1.** You can attach
`MCPServerStdio` / `MCPServerSse` / `MCPServerStreamableHttp`
instances to any agent in the agency, and the agent loop will
expose those tools to the model. The legacy (pre-v1) line did
not have native MCP — if you need MCP, stay on v1.

## 5. Sub-agent model

The headline feature. Three primitives:

- `Agent` — role + instructions + tools (+ optional file/vector
  store + optional MCP servers).
- `Agency` — a list of agents and an *agency chart* declaring
  communication edges, e.g.
  `agency_chart=[ceo, [ceo, cto], [ceo, cmo], [cto, eng_a]]`.
- `SendMessage` — the built-in tool the framework injects so an
  agent can deliver a typed message to a peer it is allowed to
  reach. Variants exist for sync, async, and streaming handoffs.

The runner enforces the chart: agent A literally cannot message
agent C unless an edge exists, which is a useful guardrail when
you have many agents and want to keep the topology legible.

## 6. Telemetry stance

**Pass-through to the underlying SDK.** The legacy line inherited
whatever the OpenAI Assistants API logged on the OpenAI side.
The v1 line inherits openai-agents' default-on tracing to the
OpenAI dashboard — which you can disable with
`set_tracing_disabled(True)` or redirect to Logfire / Langfuse /
Phoenix / OTel.

The framework itself adds no separate telemetry channel.

## 7. Prompt-cache strategy

Pass-through to the model adapter. Agency-Swarm reuses each
agent's system prompt across turns, which is the precondition
for OpenAI Responses API prefix caching and Anthropic
`cache_control` blocks (when routed via litellm) to actually
deliver savings. No cache layer at the framework level.

## 8. Hot keybinds

No TUI, but the package ships a Gradio demo UI and several
entry points:

- `agency.run_demo()` — Gradio chat over the whole agency, picks
  the entry-point agent automatically.
- `agency.terminal_demo()` — interactive REPL in the terminal.
- `agency.get_completion("input")` — programmatic one-shot.
- `agency.get_completion_stream("input")` — streamed events.
- `agency-swarm create-agent-template <name>` — scaffold a new
  agent folder with `instructions.md`, `tools/`, and a stub
  agent class.
- `agency-swarm genesis` — interactive agency-builder agent that
  helps you define the chart by chatting with it.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **The agency chart as a first-class object.**
You declare communication edges once and the framework enforces
them, generates the per-agent SendMessage tool, and gives you a
visualisable topology. For agencies with more than three agents
this is dramatically easier to reason about than handing every
agent a list of every other agent and hoping the prompts hold.
The `genesis` agent — an agent whose job is to help you author
new agents — is a nice meta-touch.

**Weakness.** Strongly OpenAI-flavoured. The v1 migration onto
openai-agents made non-OpenAI providers more practical but the
defaults, examples, and docs still assume OpenAI. The legacy
Assistants-API line ties state to OpenAI's hosted threads, which
is convenient until you want to move providers or self-host.
Smaller community than crewai / autogen / langgraph; fewer
ready-made example agencies. The agency-chart abstraction is
powerful but also one more concept to learn on top of "what is
an agent".

**When to choose.**

- You are building a **named, role-based team of agents** (CEO,
  CTO, ops, copywriter) rather than a single agent with many
  tools, and you want the *who-talks-to-whom* topology to be
  declarative and enforced.
- You are already on **OpenAI Responses / Assistants** and want
  a multi-agent layer that does not fight the underlying SDK.
- You want **MCP plus multi-agent plus a Gradio demo UI** in one
  package without wiring three frameworks together.
- You want a **scaffolding CLI** (`create-agent-template`,
  `genesis`) to bootstrap new agents and tools quickly.

**When not to choose.** You want a graph/DAG runtime with
checkpointing (use LangGraph), you want a single-binary terminal
agent (use `opencode`, `claude-code`, `goose`), you want
provider-agnostic defaults (use `pydantic-ai` or
`beeai-framework`), or you cannot tolerate openai-agents'
default-on tracing without remembering to turn it off.
