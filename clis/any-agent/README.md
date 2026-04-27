# any-agent

> Snapshot date: 2026-04. Upstream: <https://github.com/mozilla-ai/any-agent>
> Pinned release: `1.18.0` (HEAD `144b46f33bd89db4d7ee74f1b8a320987f532f67`).
> License file: [`LICENSE`](https://github.com/mozilla-ai/any-agent/blob/main/LICENSE)
> (sha `2e0b794cae0ca3640a3f72fcf5660056612b63ce`, Apache-2.0).

`any-agent` is mozilla-ai's framework-agnostic abstraction
layer over the Python agent ecosystem. It exposes one
`AnyAgent` API and dispatches to a pluggable backend —
[`smolagents`](../smolagents/), [`openai-agents-python`](../openai-agents-python/),
[`langchain`](../langflow/) / `langgraph`,
[`google-adk`](../adk-python/), [`agno`](../agno/),
[`tinyagent`](https://github.com/askbudi/tinyagent), or its
own minimal built-in runner. The pitch is the one part of agent
infra that is genuinely hard to swap once chosen: the framework
itself. `any-agent` lets a project pick a framework for
prototyping and switch later (or per-deployment) without
rewriting the agent loop, the tools, or the eval harness.

## 1. Install footprint

- `pip install any-agent` (Python 3.11+, pulls only the abstraction layer; backends are extras).
- Pick a backend with extras: `pip install 'any-agent[smolagents]'`, `'any-agent[openai]'`, `'any-agent[langchain]'`, `'any-agent[google_adk]'`, `'any-agent[agno]'`, `'any-agent[tinyagent]'`, or `'any-agent[all]'`.
- `pip install 'any-agent[mcp]'` adds the MCP client surface so the same `tools=[…]` list can mix Python callables and MCP servers across backends.
- `pip install 'any-agent[a2a]'` adds an Agent-to-Agent protocol server so an `AnyAgent` can be exposed (and consumed) as a peer over HTTP, regardless of the backend it runs on.

## 2. Repo, version, license

- Repo: <https://github.com/mozilla-ai/any-agent>
- Latest release: `1.18.0` (2026-02-18).
- HEAD pinned at this snapshot:
  `144b46f33bd89db4d7ee74f1b8a320987f532f67`.
- License: Apache-2.0. Pinned LICENSE blob:
  [`LICENSE`](https://github.com/mozilla-ai/any-agent/blob/main/LICENSE)
  (sha `2e0b794cae0ca3640a3f72fcf5660056612b63ce`).

## 3. Why it is in the catalog

The catalog already has the major Python agent frameworks as
their own entries. `any-agent` is the *meta* entry that makes
that plurality cheap:

- **One config, many runtimes**: `AgentConfig(model_id="gpt-5", instructions=…, tools=[…])` runs on smolagents, openai-agents, langchain, google-adk, agno, or tinyagent by changing the framework string passed to `AnyAgent.create(...)`. Tools are normalised — Python callables, MCP servers, and A2A endpoints look the same to every backend.
- **Honest tracing across frameworks**: spans are produced via OpenTelemetry with a unified shape regardless of backend, so a project can keep one Phoenix / Langfuse / Logfire dashboard while it migrates from one agent library to another (the painful step that usually blocks the migration).
- **Built-in evaluation hook**: ships an `AgentTrace` object and an LLM-as-judge eval helper so the same prompts can be regressed across frameworks before a switch — answers "is the new backend at least as good?" empirically rather than vibes.
- **A2A as a first-class citizen**: `agent.serve(A2AServingConfig(...))` and `await A2ATool.from_agent_card(url)` bridge any backend onto the Agent-to-Agent protocol — a route into the catalog's [`livekit-agents`](../livekit-agents/), [`agentstack`](../agentstack/), and other multi-agent topologies that does not lock the team into a single framework.

## 4. Example invocation

```python
from any_agent import AgentConfig, AnyAgent

# Same config; pick the runtime at the call site.
config = AgentConfig(
    model_id="gpt-4.1-mini",
    instructions="You are a careful research assistant.",
    tools=[],  # add Python callables or MCP servers here
)

# Prototype on smolagents...
agent = AnyAgent.create("smolagents", config)
trace = agent.run("Summarise the late-interaction retrieval idea in two sentences.")
print(trace.final_output)

# ...later, ship on openai-agents-python without rewriting tools or eval:
prod_agent = AnyAgent.create("openai", config)
prod_trace = prod_agent.run("Summarise the late-interaction retrieval idea in two sentences.")
```

## 5. Where it sits in the catalog

- Sits *above* the framework entries it dispatches to: [`smolagents`](../smolagents/), [`openai-agents-python`](../openai-agents-python/), [`adk-python`](../adk-python/), [`agno`](../agno/), and the langchain / langgraph family ([`langflow`](../langflow/), [`langgraph`](../langgraph/)).
- Complementary to [`litellm`](../litellm/): litellm abstracts the *model* behind one OpenAI-compatible interface; `any-agent` abstracts the *agent loop* behind one config object. Most projects want both.
- Pairs with [`fastmcp`](../fastmcp/) / [`mcp-agent`](../mcp-agent/) for tool servers and with [`langfuse`](../langfuse/) / [`arize-phoenix`](../arize-phoenix/) / [`logfire`](../logfire/) for the OTel-shaped traces it emits.
- Sibling philosophy to [`aisuite`](../aisuite/) (one API across model providers) and [`simplemind`](../simplemind/) (provider-agnostic model wrapper), but for *agent frameworks* rather than chat completions.
