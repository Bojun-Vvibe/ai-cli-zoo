# strands-agents

> Snapshot date: 2026-04. Upstream: <https://github.com/strands-agents/sdk-python>

"**A model-driven approach to building AI agents in just a few
lines of code.**" Strands Agents is the open-source agent SDK
behind several AWS-internal agent products (Q Developer, Q CLI,
Glue, VPC Reachability Analyzer per the upstream README), released
under Apache-2.0 and *not* AWS-coupled at the SDK level. The model
*itself* drives the loop — you declare tools with the
`@tool` decorator, hand them to an `Agent`, and the agent reasons
about which tools to call, when to stop, and what to return. No
hand-coded state machine, no orchestration graph required for the
common case; the SDK provides graph / swarm / workflow / handoff
primitives only when you actually want multi-agent topologies.

## 1. Install footprint

- `pip install strands-agents` (core SDK).
  - `pip install strands-agents-tools` for the curated tool
    library (file ops, shell, HTTP, Python REPL, web search, etc.).
  - `pip install strands-agents-builder` for the meta-agent that
    builds new Strands agents.
- Companion CLI: `pip install strands-agents-cli` →
  `strands` command for project scaffolding, agent run, eval.
- Python ≥ 3.10. Provider extras: `strands-agents[anthropic]`,
  `strands-agents[openai]`, `strands-agents[litellm]`,
  `strands-agents[ollama]`, `strands-agents[mistral]`,
  `strands-agents[writer]`, `strands-agents[a2a]`,
  `strands-agents[otel]`.
- Defaults to Bedrock Claude on AWS but works against any of the
  above without an AWS account.

## 2. Repo + version + license

- Repo: <https://github.com/strands-agents/sdk-python>
- Latest release: **v1.37.0** (2026-04-22)
- License: **Apache-2.0** —
  <https://github.com/strands-agents/sdk-python/blob/main/LICENSE>
- HEAD SHA: `b340dc4d390f7effbcf75db9311fdd20cf6ae67b`
- Default branch: `main`
- Language: Python (Java / TypeScript SDKs in sibling repos under
  the same `strands-agents` org)

## 3. Models supported

Bedrock (Claude, Nova, Llama, Mistral, Cohere, Titan); Anthropic
direct; OpenAI direct; LiteLLM-routed (any provider); Ollama;
llama.cpp; Mistral La Plateforme; Writer Palmyra; SageMaker
endpoints; OpenAI-compatible endpoints (vLLM, LM Studio,
TGI). Custom models via the `Model` interface.

## 4. Simple usage

```python
# minimal: model picks the tool, model decides when to stop
from strands import Agent, tool

@tool
def add(a: int, b: int) -> int:
    """Add two integers."""
    return a + b

agent = Agent(tools=[add])           # default model = Bedrock Claude on AWS
result = agent("what is 17 + 25?")   # model calls add(17, 25), returns 42
print(result.message)
```

```python
# pick any provider via LiteLLM
from strands import Agent
from strands.models.litellm import LiteLLMModel
agent = Agent(model=LiteLLMModel(model_id="openai/gpt-4o-mini"))
```

```python
# multi-agent: graph + swarm + workflow + handoff are first-class
from strands.multiagent import Graph, Swarm
researcher = Agent(name="researcher", system_prompt="Find facts.")
writer     = Agent(name="writer",     system_prompt="Write report.")
graph = Graph().add_node(researcher).add_node(writer).add_edge(researcher, writer)
graph("Summarize the latest agent SDK landscape.")
```

```bash
# CLI surface (companion package)
pip install strands-agents-cli
strands init my_agent
strands run my_agent --prompt "explain my repo"
strands eval my_agent suites/regression.json
```

## 5. Why it's interesting

- **Model-driven loop, not graph-driven**: you don't author a
  `StateGraph` for trivial cases — you declare tools and hand
  them to an `Agent`, the model decides the trajectory. This
  is much closer to how `claude-code` / `codex` work internally
  than to LangGraph's explicit state machines.
- **Multi-agent primitives when you need them**: `Graph` (DAG),
  `Swarm` (peer-to-peer with handoff), `Workflow` (declarative
  pipeline), and `Agent2Agent` (open A2A protocol over
  JSON-RPC) are all in-tree, so you can start single-agent and
  graduate to multi-agent without switching frameworks.
- **OpenTelemetry built in** — `strands-agents[otel]` exports
  every model call, tool call, and agent step as OTel spans;
  drops cleanly into Logfire / Langfuse / Phoenix / Honeycomb /
  Jaeger without a vendor-specific adapter.
- **Provider abstraction is honest**: a `Model` is an interface
  with `stream` / `format_request` / `format_chunk`, not a
  config dict — adding a new backend is a subclass, not a fork.
- **First-class production examples** — the `samples/` repo and
  AWS's published case-studies show real high-throughput agents
  (Q CLI, Q Developer) running on the same SDK, so the common
  scaling questions (timeouts, retries, streaming, conversation
  managers, sliding-window context) have battle-tested answers
  in-tree.

## 6. Caveats

- Defaults assume Bedrock + AWS credentials; non-AWS users must
  pass `model=` explicitly or set the provider extra.
- Multi-agent `Graph` / `Swarm` / `Workflow` are newer than the
  single-`Agent` core — APIs in this surface have churned across
  recent minor versions, pin a version in production.
- Tool-library extras (`strands-agents-tools`,
  `strands-agents-builder`, `strands-agents-cli`) ship as
  separate PyPI packages on independent release cadences; check
  compatibility ranges in the install command.
