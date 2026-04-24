# pydantic-ai

> Snapshot date: 2026-04. Upstream: <https://github.com/pydantic/pydantic-ai>
> Latest release: **v1.86.1** (2026-04-24). License: **MIT** ([`LICENSE`](https://github.com/pydantic/pydantic-ai/blob/main/LICENSE)).

The Pydantic team's take on agent frameworks: every agent input, tool
parameter, and final output is a **validated Pydantic model**, and validation
errors feed back to the model as retry signal. The catalog-relevant surface
is `clai`, the framework's standalone CLI for chatting with any configured
model from the terminal with code-block extraction and streaming.

## 1. Install footprint

- `pip install pydantic-ai` (full bundle ~150 MB resolved, includes adapters
  for OpenAI / Anthropic / Gemini / Bedrock / Cohere / Mistral / Groq / HF).
- `pip install pydantic-ai-slim[openai,anthropic]` to pick only the providers
  you need (~30 MB).
- `pip install clai` ships the standalone CLI as a separate package so you
  can `uv tool install clai` without dragging in your project's deps.
- Python 3.10+. macOS, Linux, Windows.

## 2. License

MIT (file: `LICENSE`, SPDX `MIT`). Same license across the umbrella, the
slim package, `pydantic-evals`, `pydantic-graph`, and `clai`.

## 3. Models supported

First-party adapters: OpenAI (Responses API + Chat Completions), Anthropic,
Google (Gemini AI Studio + Vertex), Groq, Mistral, Cohere, Bedrock,
HuggingFace Inference, Ollama (via OpenAI-compatible), Heroku Inference.
Anything else via the `OpenAIChatModel(base_url=...)` shim — same trick the
rest of the catalog uses for vLLM / LiteLLM / OpenRouter / DeepSeek /
Together / Fireworks.

`clai` accepts `--model openai:gpt-4o`, `--model anthropic:claude-sonnet-4`,
`--model google-gla:gemini-2.0-flash`, etc., picking the adapter from the
prefix.

## 4. MCP support

Yes (client + server). `pydantic_ai.mcp.MCPServerStdio` and `MCPServerSSE`
mount external MCP servers as tool sets the agent can call. The framework
also exposes `Agent.to_mcp_server()` to **publish** an agent as an MCP
server — one of the few catalog entries that goes both directions.

## 5. Sub-agent model

Two flavours:

- `pydantic_graph` — declare a state-machine of agent nodes; transitions are
  typed Python, not LLM-decided.
- Tool-style delegation: any `Agent` instance is a callable, so a parent
  agent's tool can be `child_agent.run(...)` and the framework propagates
  message history + tracing.

No automatic role-based crew abstraction (use `crewai` for that); pydantic-ai
is happier when sub-agents are a *typed graph* you control.

## 6. Telemetry stance

**Off by default.** Pydantic Logfire integration (`logfire.configure()`) is
opt-in and ships a `--logfire` flag on `clai`; without it nothing leaves
your machine beyond the LLM provider call. OTel export is also wired and
disabled until you point it at a collector.

## 7. Prompt-cache strategy

Whatever the underlying provider does. The framework does *not* mutate
system prompts to insert cache breakpoints — you get exactly the cache
behaviour of the upstream API. `clai` keeps history in `~/.cache/clai/` so
re-runs of `clai --resume` reuse the conversation but not the response
cache.

## 8. CLI surface (`clai`)

```
clai                              # interactive chat, default model
clai --model anthropic:claude-sonnet-4 "prompt..."
clai --agent module:agent_var     # use a Python-defined Agent (with tools)
clai --code-theme monokai         # render fenced code with pygments
clai --no-stream                  # buffer the whole reply (cheaper for pipes)
clai --logfire                    # ship traces to Pydantic Logfire
/exit /multiline /markdown        # in-REPL slash commands
```

`clai --agent path.to:my_agent` is the killer move: you define an
`Agent[Deps, Output]` in your project (with typed tools, retries, and
`output_type`) and the CLI lets you talk to *that* agent — same code that
runs in your service runs at your prompt.

## 9. Strengths

- **Type-safety is the load-bearing wall.** Tools are typed `def
  search(q: str, limit: int = 5) -> list[Result]`, output is a Pydantic
  model, validation failures are *fed back to the model* as a retry
  message — the loop is "model produces → Pydantic validates → on failure,
  ValidationError text becomes the next user turn". You stop writing
  defensive parsing code.
- **`clai` is the rare framework CLI that respects your codebase.**
  `--agent module:var` means the same `Agent` your FastAPI service uses
  is what you chat with — no separate prompt file, no drift between
  "what I tested in the terminal" and "what's in prod".
- **First-class evals via `pydantic-evals`** (sibling package, same repo):
  declare a dataset of inputs + expected outputs + scoring functions,
  run agents over it, get a leaderboard. Closest catalog peer is
  `promptfoo`, but `pydantic-evals` is Python-native and shares the
  Agent / Output types with your runtime.

## 10. Weaknesses / when not to use

- **Type-first agents are awkward for free-form chat.** If the use case is
  "let me just talk to Sonnet from the terminal", `aichat` / `mods` /
  `tgpt` / `shell-gpt` are zero-config; pydantic-ai wants you to define
  an `Agent` and an output type to get its value. `clai` works without
  any of that, but then you're using a fraction of the framework.
- **Multi-agent orchestration is hand-rolled.** No `Process="hierarchical"`
  knob like crewai — you compose agents through `pydantic_graph` or by
  calling them as tools, which is more code for a "team of five roles"
  shape. If your real need is role-based collaboration, prefer
  [`crewai`](../crewai/).

## 11. Comparison vs nearest neighbor in zoo

The closest catalog entry is **[`magentic`](../magentic/)** — both treat
"the LLM returns a typed Pydantic object" as the primary API. The split:
`magentic` is **a thin decorator library** (`@prompt def f(...) ->
list[Recipe]`) optimised for "I want a typed function call that happens
to be an LLM" — single round-trip, no agent loop, no CLI. **`pydantic-ai`**
is a **full agent framework with tools, retries, message history, MCP,
graph-shaped multi-step composition, and a real CLI (`clai`)** — picks up
where `magentic` stops. Pick `magentic` for typed one-shot extractors in
a Python service. Pick `pydantic-ai` when the same shape needs to grow
into "agent that calls tools, retries on validation failure, can be
chatted with from the terminal, and ships as an MCP server."
