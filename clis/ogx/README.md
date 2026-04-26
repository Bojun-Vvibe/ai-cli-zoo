# ogx

> Snapshot date: 2026-04-26. Upstream: <https://github.com/ogx-ai/ogx>
> (formerly `meta-llama/llama-stack` — the old path still redirects).

"**Open GenAI Stack**" — a vendor-neutral, batteries-included
runtime spec for building agentic apps against any model provider.
ogx ships a server (FastAPI), a client SDK, a CLI (`llama`), and a
plugin system that adapts inference, vector stores, safety, eval,
telemetry, and agents behind one stable REST/OpenAPI surface so the
same agent code targets a local Ollama box, a hosted Together /
Fireworks endpoint, or an on-prem GPU cluster without rewrites.

## 1. Install footprint

- CLI: `pip install llama-stack` exposes the `llama` command (`llama
  stack build`, `llama stack run`, `llama model list`).
- Distros: prebuilt Docker images per provider (`ollama`,
  `together`, `fireworks`, `vllm`, `tgi`).
- Self-host: `git clone https://github.com/ogx-ai/ogx && llama stack
  build --template ollama && llama stack run`.

## 2. Repo + version + license

- Repo: <https://github.com/ogx-ai/ogx>
- Latest release: **v0.7.1**
- License: **MIT** —
  <https://github.com/ogx-ai/ogx/blob/main/LICENSE>
- Default branch: `main`
- Language: Python
- Stars: ~8.3k

## 3. Models supported

Provider-agnostic by design: Ollama, vLLM, TGI, Together,
Fireworks, Groq, Cerebras, Bedrock, plus any OpenAI-compatible
endpoint via the inference adapter. Embedding, safety
(Llama-Guard), and tool-runtime adapters are pluggable through the
same registry.

## 4. Notable angle

**A REST contract for the entire agent loop, not just chat
completion.** Where most stacks normalize only `/v1/chat/completions`,
ogx normalizes `agents/`, `tool_runtime/`, `memory/`, `safety/`,
`eval/`, `post_training/`, and `telemetry/` so an agent harness
written against the ogx client works identically across providers
and across local-vs-cloud deployments — the same `llama stack
run` config swap moves you from laptop Ollama to production vLLM
without touching agent code.

## 5. Last verified

2026-04-26 via `gh api repos/ogx-ai/ogx`.
