# praisonai

> Snapshot date: 2026-04. Upstream: <https://github.com/MervinPraison/PraisonAI>
> Latest release: **v4.6.31** (2026-04-24). License: **MIT** ([`LICENSE`](https://github.com/MervinPraison/PraisonAI/blob/main/LICENSE)).

A multi-agent framework whose pitch is "five lines of YAML to a working
crew of agents". Wraps `crewai` and `autogen` style orchestration behind
a single `praisonai` CLI that reads a `agents.yaml`, spins up the roles,
runs the workflow to completion, and dumps results.

## 1. Install footprint

- `pip install praisonai` (core, ~30 MB). Optional extras: `praisonai[crewai]`,
  `praisonai[autogen]`, `praisonai[ui]` (Chainlit), `praisonai[code]`
  (interpreter tools), `praisonai[realtime]` (voice).
- Python 3.10+. Console scripts: `praisonai`, `praisonai-ui`, `praisonai-code`.
- Node companion `praisonai` package exists for JS users.

## 2. License

MIT (file: `LICENSE`, SPDX `MIT`). No patent grant, no copyleft. Safe to embed.

## 3. Models supported

100+ providers via LiteLLM under the hood: OpenAI, Anthropic, Gemini,
Bedrock, Vertex, Mistral, Cohere, Groq, DeepSeek, Together, OpenRouter,
Ollama, vLLM, anything OpenAI-compatible. Per-agent model overrides in YAML.

## 4. MCP

Yes — client-side. Agents can be wired to MCP servers via `tools:` block
in `agents.yaml`; tool names are auto-discovered from the server.

## 5. Agent shape

True multi-agent. The YAML defines roles (e.g. `researcher`, `writer`,
`reviewer`), each with its own goal, backstory, model, and tool list. The
runtime sequences them per the declared workflow (sequential, hierarchical,
or auto). No hand-coded glue required.

## 6. Telemetry default

Opt-in. PostHog hook exists but ships disabled; set
`PRAISONAI_TELEMETRY=true` to enable.

## 7. Killer feature

`praisonai --init "build me a market-research crew"` generates a complete
`agents.yaml` from a one-line prompt, then `praisonai` executes it.
Lowest "blank page → working multi-agent system" ceremony in the catalog.

## 8. Primary use case

Quickly stand up small autonomous workflows (research → draft → review,
issue triage, content pipelines) without writing Python boilerplate, then
graduate to the Python SDK only if the YAML stops being expressive enough.

## 9. Install command

```sh
pip install praisonai
praisonai --init "summarise today's hacker news"
praisonai
```
