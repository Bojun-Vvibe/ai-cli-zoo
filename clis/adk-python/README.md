# adk-python

> Snapshot date: 2026-04. Upstream: <https://github.com/google/adk-python>

"**Agent Development Kit (ADK) — Google's open framework for
building, evaluating, and deploying production agents.**" ADK is
the Python SDK + `adk` CLI behind Google's first-party agent
stack. It treats agents as composable Python objects (`LlmAgent`,
`SequentialAgent`, `ParallelAgent`, `LoopAgent`, custom `BaseAgent`
subclasses), gives them tools (functions, OpenAPI specs, MCP
servers, other agents), wires in a `SessionService` /
`MemoryService` / `ArtifactService`, and runs them via a `Runner`
that streams events. The `adk` CLI runs agents (`adk run`),
serves them over HTTP / WebSocket (`adk api_server`, `adk web`),
runs eval suites (`adk eval`), and deploys to Cloud Run / Vertex
AI Agent Engine / GKE (`adk deploy …`).

## 1. Install footprint

- `pip install google-adk` (SDK + `adk` CLI). Extras:
  `google-adk[eval]`, `google-adk[a2a]` (Agent-to-Agent protocol),
  `google-adk[gcp]`.
- CLI surface: `adk run`, `adk web`, `adk api_server`, `adk eval`,
  `adk deploy cloud_run`, `adk deploy agent_engine`,
  `adk deploy gke`, `adk create`, `adk completion`.
- Python ≥ 3.10. Also has a separate Java SDK
  (`com.google.adk:google-adk`) for JVM workloads.
- Built on Gemini-first defaults but model-agnostic via LiteLLM
  (`LiteLlm("anthropic/claude-3-5-sonnet")`,
  `LiteLlm("openai/gpt-4o")`, Ollama, etc.) or any
  `BaseLlm` subclass.

## 2. Repo + version + license

- Repo: <https://github.com/google/adk-python>
- Latest release: **v1.31.1** (2026-04-21)
- License: **Apache-2.0** —
  <https://github.com/google/adk-python/blob/main/LICENSE>
- HEAD SHA: `7de5bc54e11986f70a48a9dd83ea39be58ebce40`
- Default branch: `main`
- Language: Python

## 3. Models supported

Gemini 1.5 / 2.x / 2.5 first-class via `google-genai`; any
LiteLLM-routed backend (Anthropic Claude, OpenAI GPT-4 / o-series,
Mistral, Cohere, Groq, Ollama, vLLM, OpenRouter) via the `LiteLlm`
wrapper. Vertex AI Model Garden registrations also surface as
`BaseLlm` instances. Multi-modal in/out where the model supports
it (image, audio, video for Gemini 2.0+).

## 4. Simple usage

```bash
# scaffold an agent project
adk create my_agent
cd my_agent
```

```python
# my_agent/agent.py
from google.adk.agents import LlmAgent
from google.adk.tools import google_search

root_agent = LlmAgent(
    name="research_agent",
    model="gemini-2.5-flash",
    instruction="Answer the user's question. Use google_search when needed.",
    tools=[google_search],
)
```

```bash
# run it three different ways
adk run my_agent                  # one-shot REPL in terminal
adk web                           # local dev UI on :8000 with trace viewer
adk api_server                    # FastAPI: POST /run, SSE /run_sse, WS

# eval suite
adk eval my_agent eval/my_eval.json

# deploy
adk deploy cloud_run --project=my-proj --region=us-central1 my_agent
adk deploy agent_engine --project=my-proj --region=us-central1 my_agent
```

```python
# multi-agent composition
from google.adk.agents import SequentialAgent, ParallelAgent, LlmAgent
plan = LlmAgent(name="planner", model="gemini-2.5-pro", instruction="...")
exec_a = LlmAgent(name="exec_a", model="gemini-2.5-flash", instruction="...")
exec_b = LlmAgent(name="exec_b", model="gemini-2.5-flash", instruction="...")
pipeline = SequentialAgent(
    name="pipeline",
    sub_agents=[plan, ParallelAgent(name="fanout", sub_agents=[exec_a, exec_b])],
)
```

## 5. Why it's interesting

- **Composition primitives are the API, not a DSL** — `LlmAgent`,
  `SequentialAgent`, `ParallelAgent`, `LoopAgent`, plus custom
  `BaseAgent` subclasses that override `_run_async_impl`, give
  you graph-shaped agent topologies in plain Python without
  YAML / JSON configs or a separate runtime spec.
- **`adk web` is a real local trace viewer**, not a marketing
  demo: every event (model call, tool call, sub-agent transfer,
  state delta, artifact write) shows up in a per-session timeline
  with full input/output payloads, runnable from the same
  process you're editing.
- **First-class deploy targets**: `adk deploy cloud_run` packages
  the agent into a container with a FastAPI shim;
  `adk deploy agent_engine` deploys to Vertex AI's managed agent
  runtime; `adk deploy gke` to Kubernetes. Same agent code, three
  serving shapes.
- **A2A protocol built in** — `RemoteA2aAgent` lets one ADK agent
  call another agent (ADK or otherwise) over the open Agent-to-
  Agent JSON-RPC protocol, so multi-vendor agent meshes don't
  require custom RPC glue.
- **Eval as a first-class step**: `adk eval` runs a JSON suite of
  expected tool calls + final-response rubrics against an agent,
  with metrics for `tool_trajectory_avg_score` and
  `response_match_score` — repeatable regression tests for agent
  behavior in CI.

## 6. Caveats

- Optimized for the Google Cloud serving path (Cloud Run / Vertex
  Agent Engine); deploying the same agent to AWS / Azure works
  via the FastAPI server but you wire up the runtime yourself.
- `SessionService` / `MemoryService` defaults are in-memory;
  production needs the Vertex / DB-backed implementations.
- The `adk web` UI assumes localhost — exposing it over a network
  needs a reverse proxy with auth in front, no built-in auth on
  the dev server.
