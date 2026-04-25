# swarms

> Snapshot date: 2026-04. Upstream: <https://github.com/kyegomez/swarms>

"**The enterprise-grade production-ready multi-agent orchestration
framework.**" `swarms` is a Python framework that gives you a zoo of
*structures* — `SequentialWorkflow`, `ConcurrentWorkflow`,
`AgentRearrange`, `MixtureOfAgents`, `GraphWorkflow`,
`HiearchicalSwarm`, `SwarmRouter`, `SpreadSheetSwarm`,
`MajorityVoting`, `ForestSwarm`, `AgentLoadBalancer` — each of which
takes a list of `Agent` objects and dictates the topology by which
they pass messages, vote, fan out, fan in, or hand off. An `Agent`
itself is a fairly thin wrapper around an `LLM` (any `litellm`-routed
model), a system prompt, an optional tool list, optional memory
(`AgentRAG` over a vector store), and a per-step retry / reflection
policy. The pitch: stop hand-rolling for-loops over LLM calls; pick
the topology that matches the task and let the framework wire it.

## 1. Install footprint

- `pip install -U swarms` (Python ≥ 3.10). Pulls `litellm`, `pydantic
  ≥ 2`, `loguru`, `tenacity`, `tiktoken`, `pypdf`, `docstring-parser`,
  `python-dotenv`, `httpx`, plus optional extras for vector backends
  (`swarms[chromadb]`, `swarms[qdrant]`, `swarms[pinecone]`).
- Ships a CLI: `swarms` (Click-based). Subcommands include
  `swarms onboarding` (interactive setup writing `.env` keys),
  `swarms run-agents --yaml agents.yaml`, `swarms agent` (one-shot
  agent invocation), `swarms autoswarm "<task>"` (auto-pick a
  topology), `swarms check-login`, `swarms get-api-key`. The CLI is a
  thin frontend on the same Python objects — most users instantiate
  in Python, not via CLI.
- Also ships `swarms-cloud` (optional separate package) for hosted
  agent execution against the swarms.ai control plane; the OSS core
  runs entirely local.

## 2. Repo + version + license

- Repo: <https://github.com/kyegomez/swarms>
- Latest release: **6.8.1** (2026-04, monthly cadence on `main`)
- License: **Apache-2.0** —
  <https://github.com/kyegomez/swarms/blob/master/LICENSE>
- Default branch: `master`
- Language: Python

## 3. Models supported

Anything `litellm` can route — OpenAI (Chat + Responses + Azure),
Anthropic (Claude 3.5 / 3.7 / 4 with extended thinking), Gemini
(Studio + Vertex), Bedrock, Mistral, Cohere, Groq, DeepSeek (V3, R1),
Together, Fireworks, OpenRouter, Cerebras, xAI, Perplexity, plus
local Ollama, vLLM, llama.cpp, LM Studio, mlx-lm via OpenAI-compatible
endpoints. Per-agent `model_name="anthropic/claude-sonnet-4-5"` style
strings select the backend; mixing providers across agents in the
same swarm is the explicit happy path (a planner Agent on Opus, ten
worker Agents on a cheap local Qwen, a judge Agent on GPT-5).
Embedding / vector retrieval is bring-your-own (`AgentRAG` accepts
any `chromadb` / `qdrant` / `pinecone` client).

## 4. MCP support

Yes (client). Recent releases (6.x) added an `MCPServerSseParams` /
`MCPServerStdioParams` config you can attach to any `Agent` via
`mcp_url=...` or `mcp_config=...`; the agent then sees that server's
tools alongside its locally-declared `tools=[...]`. No first-party
MCP server mode — you wrap a `swarm.run()` call in your own server
if you want to expose a topology as an MCP tool.

## 5. Sub-agent model

The defining feature. *Every* swarm structure is a sub-agent
topology in a different shape:

- `SequentialWorkflow` — pipeline; each agent's output is the next
  agent's input.
- `ConcurrentWorkflow` — fan out the same task to N agents in
  parallel, return all outputs.
- `AgentRearrange` — DSL like `"agent1 -> agent2, agent3 -> agent4"`
  to express a directed graph of handoffs.
- `MixtureOfAgents` — N "expert" agents in parallel + an
  "aggregator" agent that synthesises a final answer from all N
  responses.
- `GraphWorkflow` — arbitrary DAG with conditional edges.
- `HiearchicalSwarm` — manager agent that decomposes a task and
  delegates sub-tasks to worker agents, then collects.
- `SwarmRouter` / `MultiAgentRouter` — classifier agent picks which
  downstream swarm or agent to route a task to.
- `MajorityVoting` — N agents answer, vote, return the plurality.
- `ForestSwarm` — tree of agents with leaf specialists.
- `SpreadSheetSwarm` — operate over a CSV / dataframe with one
  agent-call per row, aggregate.

Every structure is a Python class with a `.run(task: str)` method —
they compose (a `HiearchicalSwarm` manager whose workers are
themselves `MixtureOfAgents` swarms is one constructor call away).

## 6. Telemetry stance

Mixed. The OSS core makes no analytics calls of its own — `loguru`
logs go to stdout / file only, LM traffic is whatever `litellm`
emits to its configured callbacks (off by default). `swarms
onboarding` and `swarms get-api-key` *do* talk to the swarms.ai
hosted control plane (they fetch / register an API key against the
hosted service); skip those subcommands and the framework is
fully local. Optional first-party telemetry hooks for
`agentops`, Phoenix / Arize, Langfuse, and Weights & Biases via
`Agent(callback_handlers=[...])` — all opt-in.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **The widest catalog of named multi-agent
topologies in any single OSS framework.** If you can sketch your
problem on a whiteboard as boxes-and-arrows ("five experts vote;
one synthesises", "manager decomposes, four workers run in
parallel, results aggregate by majority", "router classifies into
one of three downstream pipelines"), there is a one-line `swarms`
constructor that already implements that shape with retries,
parallelism, error handling, and per-agent telemetry hooks wired
in. You stop spending the first week of any agentic project
re-inventing `asyncio.gather` + retry + aggregator-prompt
plumbing. The `MixtureOfAgents` and `MajorityVoting` patterns in
particular are battle-tested for "raise quality at the cost of N×
tokens" inference-time-compute scaling. `AgentRearrange`'s
DSL string is genuinely the fastest way to wire a non-trivial
graph without drawing it.

**Weakness.** **The catalog is wide, the abstractions are thin,
and the docs are uneven.** Many structure classes are 80 % the
same code with a different aggregation rule — the choice of
`HiearchicalSwarm` vs `GraphWorkflow` vs `AgentRearrange` for a
given problem is rarely obvious from the docs and often comes
down to "read the source for each one." The release cadence is
fast (multiple minor releases per month) which means breaking
changes between minors are not unheard of; pin
the version. The hosted swarms.ai surface is intermixed with the
OSS code in ways that occasionally surface ("`onboarding` wants
to fetch a key") — clean if you ignore those subcommands but
worth knowing. Memory / RAG is functional but spartan compared
to a dedicated retrieval framework like
[`llama-index`](../llama-index/) or [`haystack`](../haystack/);
most real deployments wire an external vector store and pass
results in as context. Single-agent quality is roughly "an Agent
class around a `litellm` call" — it is the *composition* layer
where `swarms` earns its keep.

**When to choose.** You have a problem that is genuinely
parallelisable across LLM calls — N-expert ensembling, fan-out
classification, manager / worker decomposition, voting for
quality, routing a heterogeneous workload — and you want the
topology to be one constructor call instead of a week of
`asyncio` plumbing. Pair with [`litellm`](../litellm/) (already a
dependency) for provider routing, with
[`langfuse`](../langfuse/) or [`arize-phoenix`](../arize-phoenix/)
for per-agent traces, and with [`promptfoo`](../promptfoo/) or
[`deepeval`](../deepeval/) for swarm-level eval (does the
9-expert ensemble actually beat the 1-expert baseline on your
metric, or are you burning tokens for noise?). Skip if you have
a single-agent task — an `Agent` instance is fine but the value
is in the structures, and a dedicated single-agent framework
([`pydantic-ai`](../pydantic-ai/), [`smolagents`](../smolagents/))
will be tighter. Skip if you need a typed-program / compile-time
optimiser story like [`dspy`](../dspy/) — `swarms` topologies
are runtime, not compiled. Skip if your "agents" are really one
LLM call and a tool — that is a function, not a swarm.
