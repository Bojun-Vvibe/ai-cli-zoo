# simpleaichat

> Snapshot date: 2026-04. Upstream: <https://github.com/minimaxir/simpleaichat>

"**Python package for easily interfacing with chat apps, with robust
features and minimal code complexity.**" Max Woolf's deliberately tiny
chat client: one `AIChat` class wraps OpenAI / Anthropic chat
completions with **session management, structured output via Pydantic
schemas, streaming, async, and parameter control**, with no LangChain,
no agent loop, no telemetry, and no hidden prompts. The CLI surface is
a thin `python -m simpleaichat` REPL on top of the same primitives.

## 1. Install footprint

- `pip install simpleaichat` (or `uv pip install simpleaichat`).
- Python ≥ 3.8. Pulls Pydantic v1, `httpx`, `orjson`, `python-dotenv` —
  no LangChain, no LLM-framework deps.
- Library: `from simpleaichat import AIChat; ai = AIChat(api_key=...)`.
- Quick CLI: `python -m simpleaichat` opens a REPL against the default
  configured key (`OPENAI_API_KEY` env var or constructor arg).
- Configure model + system prompt + parameters per session: `AIChat(
  model="gpt-4o-mini", system="You are a code reviewer.",
  params={"temperature": 0.0, "max_tokens": 1000})`.

## 2. Repo + version + license

- Repo: <https://github.com/minimaxir/simpleaichat>
- Latest release: **v0.2.2**
- License: **MIT** —
  <https://github.com/minimaxir/simpleaichat/blob/main/LICENSE>
- Default branch: `main`
- Language: Python

## 3. Models supported

OpenAI Chat Completions API by default; Anthropic Claude via the
`AIChat(model="claude-3-...")` path. Any **OpenAI-compatible endpoint**
works by passing `api_url=` (Ollama at `http://127.0.0.1:11434/v1/`,
vLLM, LM Studio, LiteLLM, OpenRouter, Groq, DeepSeek). No provider
auto-discovery — you pass the URL and key explicitly.

## 4. MCP support

None. Out of scope by design. `simpleaichat` is a chat-completion
client, not an agent runtime; tool/function calling is exposed as the
raw `tools=` parameter of the underlying API, not as an MCP plumbing
layer.

## 5. Sub-agent model

None. The library ships an `AIChat` (single session) and an
`AsyncAIChat` for concurrency. Multi-session is "instantiate N
`AIChat`s and call them yourself" — a deliberate non-feature meant to
keep the surface area readable.

## 6. Telemetry stance

Off. Zero analytics in the OSS codebase, no phone-home. Egress is
exactly the chat endpoint you configured. Sessions are kept in memory
(or written to disk via `ai.save_session("file.json")`) — never
uploaded.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **A readable, ~500-line chat client you can
actually audit.** Where most "chat with an LLM in Python" stacks pull
in a transitive forest of agent / orchestrator / vector-store
dependencies, `simpleaichat` is one class with a well-typed
constructor: pick a model, set a system prompt, set parameters, get
back a session that supports streaming (`ai.stream("...")`), structured
output (`ai("...", output_schema=MyPydanticModel)`), session save/load,
and message-list inspection. The whole library fits in your head in
twenty minutes, which makes it the right substrate for **glue scripts,
notebooks, and small services** where pulling in a 500-MB framework to
make one chat call is the actual problem.

**Weakness.** **Project is in maintenance mode** — last release v0.2.2
predates GPT-5 / Claude 3.7 launch features (no built-in support for
Anthropic prompt caching, OpenAI Responses API, parallel tool calls
with structured arguments). Pydantic v1 dependency is sticky in
modern stacks. Not an agent: no loop, no tool execution, no file edits,
no MCP. Newer APIs (`o1`, `o3`, extended thinking) work but you may
need to pass parameters via `params=` rather than first-class kwargs.

**When to choose.** You want a **boring, dependency-light chat client**
inside a Python script, notebook, or microservice — single round-trips
or simple multi-turn conversations with structured outputs, no
framework. Pair with [`marvin`](../marvin/) if you want typed
function-shaped wrappers on top, with [`pydantic-ai`](../pydantic-ai/)
if you need a real agent loop, or with [`gptme`](../gptme/) /
[`aider`](../aider/) if you actually need a terminal coding agent.
Skip if you need active maintenance with day-of model launches, or if
you need agent / tool / MCP behaviors.
