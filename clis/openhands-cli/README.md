# openhands-cli

> Snapshot date: 2026-04. Upstream: <https://github.com/OpenHands/OpenHands-CLI>

A standalone, single-binary CLI front-end for the OpenHands agent runtime. Where
the upstream `openhands` repo ships the full Docker-bound multi-agent platform
(browser, sandbox, runtime images, web UI), `openhands-cli` strips that down to
a self-contained executable you can drop on a laptop and run against any
LiteLLM-routed model. Same core agent loop, none of the platform surface area.

## 1. Install footprint

- `pipx install openhands-cli` or grab the prebuilt binary from the GitHub
  release page.
- Single executable; no Docker, no `runtime` image, no web server.
- Python 3.12+ for the pipx route. macOS, Linux, Windows.
- State in `~/.openhands/` (config, conversation history, MCP server registry).

## 2. License

MIT. License file: [`LICENSE`](https://github.com/OpenHands/OpenHands-CLI/blob/main/LICENSE).

## 3. Latest version

`1.15.0` (released 2026-04-24). Tag list:
<https://github.com/OpenHands/OpenHands-CLI/tags>.

## 4. Models supported

Anything LiteLLM speaks — Anthropic, OpenAI, Gemini, Bedrock, Vertex, Groq,
DeepSeek, Mistral, Cohere, OpenRouter, Ollama, vLLM, llama.cpp, any
OpenAI-compatible endpoint. Provider config is a TOML block in
`~/.openhands/config.toml`.

## 5. MCP support

Yes (client). Adds external MCP servers via `~/.openhands/mcp.json`; the CLI
auto-discovers tools at session start and exposes them to the agent loop.

## 6. Sub-agent model

Inherits the OpenHands CodeAct loop: a single primary agent, with the option to
delegate to specialised sub-agents (browsing, search, repo-mapping) declared in
the agent registry. No long-running concurrent agents — each delegation is a
synchronous call that returns to the parent.

## 7. Telemetry stance

Off by default. Crash reports and anonymous usage stats are opt-in via a
config flag; the CLI never phones home unless you flip it.

## 8. When to choose it

- You like the OpenHands agent shape (CodeAct, tool registry, MCP-friendly) but
  do not want to manage Docker images, a runtime container, or the full web UI.
- You want a single binary you can `scp` to a remote box and immediately run
  against a hosted LLM endpoint with no platform dependencies.
- You are building automation around OpenHands and need a stable command-line
  contract instead of driving the web app.

## 9. When to skip it

- You actually want the sandboxed Linux runtime, the browser agent, or the
  web UI — those live in the parent `openhands` repo, not here.
- You need something with built-in repo-map context like `aider`, or a
  trust-prompt-per-step UX like `cline`. The CLI is closer to a thin agent
  shell than to either of those.
