# agentops

> Snapshot date: 2026-04. Upstream: <https://github.com/AgentOps-AI/agentops>

"**Python SDK for AI agent monitoring, LLM cost tracking,
benchmarking, and more.**" AgentOps is a one-line drop-in
`agentops.init(api_key=...)` that auto-instruments the major agent
frameworks (CrewAI, Agno, OpenAI Agents SDK, LangChain, AutoGen, AG2,
CAMEL, LiteLLM) and the bare LLM SDKs (OpenAI, Anthropic, Cohere,
Mistral, Ollama) so every `Session` becomes a recorded run with
per-LLM-event tokens + cost + latency, every tool call shows up as an
`ActionEvent`, and the entire trace renders as an interactive timeline
in the (hosted or self-hostable) AgentOps dashboard. The SDK ships an
**OpenTelemetry-native span model** under the hood — you can fan the
same spans out to OTel collectors (Jaeger / Tempo / SigNoz / Datadog /
Honeycomb) instead of, or alongside, the AgentOps backend.

## 1. Install footprint

- `pip install agentops` (~5 MB; pulls `opentelemetry-sdk`,
  `opentelemetry-exporter-otlp`, `requests`, `psutil`).
- Per-framework auto-instrumentation extras:
  `pip install "agentops[crewai]"`, `agentops[langchain]`,
  `agentops[openai-agents]`, etc. — each pulls only the wrapper.
- One line: `import agentops; agentops.init(api_key="...")` before
  your agent runs. From this point every LLM / tool / agent call is
  a span; `agentops.end_session("Success")` finalises the trace.
- Python ≥ 3.9. CLI entry point `agentops` for project init / status.

## 2. Repo + version + license

- Repo: <https://github.com/AgentOps-AI/agentops>
- Latest release: **v0.4.21** (2026-04)
- License: **MIT** —
  <https://github.com/AgentOps-AI/agentops/blob/main/LICENSE>
- Default branch: `main`
- Language: Python (SDK) + TS / React (dashboard, separate repo)

## 3. Frameworks instrumented (auto)

CrewAI, Agno, OpenAI Agents SDK (the official one), LangChain,
LangGraph, AutoGen / AG2, CAMEL-AI, LlamaIndex, LiteLLM, Anthropic
Computer Use, OpenAI Swarm, MultiOn, Browser-Use, Cohere, Mistral,
Ollama. Plus raw OpenAI / Anthropic / Bedrock / Together SDKs are
patched at `init()` time — you do not import an instrumentor per
provider. Custom spans via `@agentops.record_action` decorator or
`agentops.record(ActionEvent(...))` for "I built my own framework,
trace this function".

## 4. Sub-agent model

None — observability layer, not an agent runtime. Sub-agent *tracing*
is fully supported: framework instrumentors emit nested spans
(`AGENT` → `LLM` → `TOOL` → `LLM` → `ACTION`), so a CrewAI hierarchical
crew with a manager + 4 workers renders as one tree per `Session` with
cost + latency rolled up at every node. Concurrent agents (parallel
tool calls, `asyncio.gather` fan-out) get parallel sibling spans with
correct parent linkage.

## 5. Telemetry stance

**On by default toward `api.agentops.ai`** when you call
`agentops.init(api_key=...)` — that is the explicit consent: passing
the key opts in. To stay fully local, either skip `init()` (and lose
auto-instrumentation), or set `endpoint="http://localhost:4318"` to
ship spans to your own OTel collector instead, or run the (separately
licensed) self-hosted backend. The OTel span shape is the same in
both modes, so exporting to Phoenix / Langfuse / Datadog in parallel
is a `set_tracer_provider` away. No content training; spans contain
prompts + completions by default — set `inherited_session_id` +
`tags=["pii-redacted"]` plus a `span_processor` filter for sensitive
workloads.

## 6. Killer feature, weakness, when to choose

**Killer feature.** **One line, every framework, dollar-accurate cost
per session** — `agentops.init(api_key=os.environ["AGENTOPS_API_KEY"])`
above your `crew.kickoff()` / `Runner.run(agent)` / `graph.invoke(...)`
turns the next run into a viewable trace with **per-LLM-event USD
spend computed from a maintained provider price table** (refreshed
continuously upstream — the table covers ~50 providers + ~200 models
including frontier reasoning tiers and provider-specific input /
cached-input / output / reasoning-token pricing). For multi-agent
crews this is the one number stakeholders actually ask for ("how much
does one customer-support session cost?") and it lands without you
writing a token-counter wrapper around every provider's SDK. The
dashboard's session-replay view (full message tree + tool I/O + token
counts + rendered LLM responses + per-step latency waterfall) is the
debugger of choice when a CrewAI / Agno run goes off the rails — you
see the exact prompt the worker agent saw on turn 7, what it returned,
and which tool it called next.

**Weakness.** **Hosted control plane is the default revenue surface,
the self-hosted backend is a separate (commercially-licensed) repo.**
The MIT SDK is genuinely MIT, but if you want the dashboard on-prem
(regulated environments, air-gapped, EU residency) you are buying
the EE edition or doing the OTel-fan-out workaround (which gives you
spans in your existing observability stack but not the AgentOps-shaped
session-replay UI). Auto-instrumentation occasionally lags upstream
framework releases (a CrewAI minor that renames a callback hook can
break the wrapper for a week or two — pin both versions in production).
The `record_action` / `record_tool` decorator surface for custom
frameworks is less polished than the auto-instrumented ones.

**When to choose.** You are running **CrewAI / Agno / OpenAI Agents
SDK / LangGraph / AutoGen** in production (or about to ship one) and
you want **per-session cost + the trace tree + the replay UI without
writing instrumentation code**. The cost-per-session surface is the
specific reason to pick AgentOps over [`arize-phoenix`](../arize-phoenix/)
(OTel-native viewer, weaker out-of-the-box pricing) /
[`langfuse`](../langfuse/) (great prompt-management + dataset story,
hosted-grade UI but the framework auto-instrumentation matrix is
narrower) / [`opik`](../opik/) (Apache-2.0 self-hostable bundle,
weaker on the "I run a multi-framework agent zoo" surface). Skip if
you are observing a single-framework Python codebase with a working
OTel pipeline already — wire [`openllmetry`](../openllmetry/) /
Phoenix / Langfuse straight into your collector and call it done; or
if you need *prompt management* + *dataset curation* as the primary
workflow, in which case Langfuse / Opik are better-shaped tools.
