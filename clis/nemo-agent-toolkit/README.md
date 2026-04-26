# nemo-agent-toolkit

> Snapshot date: 2026-04. Upstream: <https://github.com/NVIDIA/NeMo-Agent-Toolkit>
> Latest release: **v1.6.0** (2026-04-10). License: **Apache-2.0** ([`LICENSE.md`](https://github.com/NVIDIA/NeMo-Agent-Toolkit/blob/main/LICENSE.md)).

NVIDIA's framework-agnostic agent connector + optimiser. The pitch isn't
"here is another agent loop" — it's "you already have agents in LangChain,
LlamaIndex, CrewAI, Semantic Kernel, AutoGen; this lets them talk to each
other and profiles the result". Ships as the `nat` CLI (formerly `aiq`).

## 1. Install footprint

- `uv pip install nvidia-nat` (core ~40 MB) plus per-framework adapters:
  `nvidia-nat[langchain]`, `[llama-index]`, `[crewai]`, `[semantic-kernel]`,
  `[agno]`, `[mem0]`, `[zep]`, `[ragaai]`, `[weave]`.
- Python 3.11–3.13. Console script: `nat`.
- Plays nicely with `uv` workspaces; the project itself is a uv workspace.

## 2. License

Apache-2.0 (file: `LICENSE.md`, SPDX `Apache-2.0`). Includes the standard
patent grant. NVIDIA-CLA on contributions.

## 3. Models supported

Whatever the underlying framework supports — NIM/NeMo endpoints, OpenAI,
Anthropic, Gemini, Bedrock, Ollama, vLLM, etc. NIM is first-class and the
toolkit ships pre-built configs for NVIDIA-hosted models.

## 4. MCP

Yes — both directions. Can consume MCP servers as tools and expose any
registered NAT workflow as an MCP server (`nat mcp serve`).

## 5. Agent shape

Composes multi-framework agents into a single workflow graph. A LangChain
ReAct agent can hand off to a CrewAI crew which calls a LlamaIndex RAG
pipeline — all defined in one YAML and traced as one run.

## 6. Telemetry default

Off by default. Optional OpenTelemetry export to Phoenix, Langfuse, Weave,
W&B, Galileo, RagaAI; configured per workflow, not globally.

## 7. Killer feature

`nat profile` and `nat eval`. Drop a workflow into the profiler and get
per-tool latency, token cost, cache hit-rate, and bottleneck flame graphs
without instrumenting your code. Eval harness reuses the same configs.

## 8. Primary use case

Teams that already adopted two or more agent frameworks and now need a
neutral integration + observability layer. Also a clean way to deploy an
existing LangChain/CrewAI graph as a NIM-hosted service.

## 9. Install command

```sh
uv pip install nvidia-nat
nat --help
nat workflow create my-workflow
nat run --config_file my-workflow/configs/config.yml
```
