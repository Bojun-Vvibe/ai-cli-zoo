# chainlit

> Snapshot date: 2026-04. Upstream: <https://github.com/Chainlit/chainlit>
> License file: <https://github.com/Chainlit/chainlit/blob/main/LICENSE>
> Pinned: `2.11.1` (2026-04-22, Python package). Default branch is
> `main`. HEAD verified at `488b74579ace74ad917556a1560e9db68e011219`.
> Install as `pip install chainlit` and run `chainlit run app.py -w`.

A **conversational-UI framework for LLM apps** — Python decorators
turn `@cl.on_chat_start` / `@cl.on_message` / `@cl.step` into a
hosted React chat surface with streaming tokens, intermediate
"thinking" steps, copy-to-clipboard tool calls, file upload, audio
input, image rendering, table widgets, and a session-level data
store backed by SQLAlchemy. The contrast against
[`langflow`](../langflow/) is "code-first chat surface for an
existing Python agent" rather than "drag-and-drop graph IDE", and
against [`langfuse`](../langfuse/) is "live UI for the user" not
"observability dashboard for the engineer".

## 1. Install footprint

- **Library + CLI**: `pip install chainlit` (~100 MB with deps —
  `fastapi`, `uvicorn`, `socket.io`, `literalai`, `watchfiles`,
  `tomli`, `dataclasses-json`, `lazify`, `python-multipart`,
  `nest-asyncio`, `aiofiles`, `httpx`).
- **CLI**: `chainlit run app.py [-w] [--port 8000] [--host 0.0.0.0]
  [--headless] [--debug]`; `chainlit init` scaffolds `chainlit.md`
  + `.chainlit/config.toml`; `chainlit hello` runs a demo;
  `chainlit create-secret` mints `CHAINLIT_AUTH_SECRET` for cookie
  signing.
- **Frontend bundle**: pre-built React app shipped inside the wheel
  (no Node toolchain required at install time); custom theming via
  `public/theme.json` and `public/favicon.svg`.
- **Optional integrations**: `chainlit[litellm]`, OpenAI, LangChain,
  LlamaIndex, Haystack, AutoGen, Semantic Kernel, OpenAI-Agents — each
  has a `cl.LangchainCallbackHandler` / `cl.LlamaIndexCallbackHandler`
  style wrapper that auto-promotes provider events into Chainlit
  steps without manual `cl.Step` boilerplate.

## 2. License

Apache-2.0 (file is `LICENSE` at the repo root,
<https://github.com/Chainlit/chainlit/blob/main/LICENSE>). The
managed companion product is Literal AI (separate repo, commercial
SaaS); the OSS package has no telemetry call-home requirement and
runs fully self-hosted with `chainlit run`.

## 3. Models supported

Provider-agnostic — Chainlit is a UI surface, not an LLM client.
You bring the LLM call (`openai`, `anthropic`, `litellm`, LangChain,
LlamaIndex, Haystack, raw `httpx` against any OpenAI-compatible URL)
inside `@cl.on_message`, and Chainlit renders the streamed tokens
via `cl.Message().send()` / `await msg.stream_token(t)`. Tool calls
appear as collapsible `cl.Step` blocks; multi-modal content
(`cl.Image`, `cl.Audio`, `cl.Pdf`, `cl.Video`, `cl.File`) renders
inline. The framework does not pin any provider, so swapping
`openai` → `litellm` is a one-line change.

## 4. MCP support

No first-party MCP server. Chainlit is an output surface for whatever
agent loop runs inside `@cl.on_message` — if that loop calls MCP
tools (via [`opencode`](../opencode/) / [`claude-code`](../claude-code/)
/ a custom `mcp.client` session), each tool call surfaces as a
`cl.Step` in the chat with collapsible JSON input/output. The
inverse direction ("expose the Chainlit chat as an MCP tool") is
non-idiomatic — the value of Chainlit is the human-in-the-loop
React UI, not headless tool invocation.

## 5. Sub-agent model

Chainlit's `cl.Step` is the sub-agent unit — `async with
cl.Step(name="planner", type="llm") as step: step.output = ...`
nests arbitrarily, and the UI renders the tree as a collapsible
"Took N seconds" block per node. AutoGen / LangGraph / OpenAI-Agents
multi-agent runs auto-promote each agent turn to a step via the
provided callback handler, so the user sees the full agent tree
without per-turn manual wiring. `cl.Action` buttons let the user
approve / reject / branch a step inline ("Plan and Act"-style HITL).

## 6. Telemetry stance

OSS package does not phone home. Setting
`LITERAL_API_KEY` opts in to Literal AI tracing (commercial). Local
session data lands in a SQLite/Postgres-backed `cl_user_session` table
when `chainlit.data_layer` is configured; absent that, sessions are
in-memory and evict on server restart. Auth is opt-in (`@cl.password_auth_callback`,
`@cl.oauth_callback`, header-based, custom callback) — disabled by
default in dev, must be wired before exposing publicly.

## 7. Prompt-cache strategy

None at the framework layer. Caching belongs to the LLM call inside
`@cl.on_message` — pair with [`gptcache`](../gptcache/) on the
provider side, or use Anthropic / Google native prompt caching via
the underlying SDK. Chainlit *does* persist message history in the
configured data layer, so a "resume conversation" UX gets the prior
turns as context for free without re-invoking the model.

## 8. Hot keybinds

No CLI TUI — the surface is the React app at `localhost:8000`.

```python
import chainlit as cl
from openai import AsyncOpenAI

client = AsyncOpenAI()

@cl.on_chat_start
async def start():
    cl.user_session.set("history", [
        {"role": "system", "content": "You are a code review assistant."}
    ])

@cl.on_message
async def main(msg: cl.Message):
    history = cl.user_session.get("history")
    history.append({"role": "user", "content": msg.content})
    reply = cl.Message(content="")
    stream = await client.chat.completions.create(
        model="gpt-4o-mini", messages=history, stream=True
    )
    async for chunk in stream:
        if (tok := chunk.choices[0].delta.content):
            await reply.stream_token(tok)
    await reply.send()
    history.append({"role": "assistant", "content": reply.content})
```

`chainlit run app.py -w` — `-w` enables hot reload on file save.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **A production-quality React chat UI delivered as
a `pip install` + four lines of Python decorators**, with streaming
tokens, nested step trees that visualise multi-agent runs without
manual instrumentation, file/image/audio/PDF rendering inline,
auth + persistence + OAuth wired by config, and zero Node toolchain
at install time. The same `app.py` runs under `chainlit run` for
local dev and behind any FastAPI deployment for prod — so a Python
team prototypes a chat agent in an afternoon and ships it without
spinning up a separate frontend repo.

**Weakness.** The bundled React app is the only frontend — custom
layouts beyond theming + the per-element widgets (`cl.Image`,
`cl.Plotly`, `cl.Pdf`) require forking and rebuilding the bundle.
The Literal AI tracing tier is the monetisation surface, so the
"give me observability without leaving Chainlit" path nudges toward
that SaaS; for self-hosted alternatives wire
[`langfuse`](../langfuse/) / [`arize-phoenix`](../arize-phoenix/) /
[`opik`](../opik/) via a callback handler. WebSocket-heavy session
model means scaling beyond a single replica needs sticky sessions
or a Redis-backed pub/sub adapter.

**When to choose.**

- You have a Python LLM agent and need **a chat UI by Friday** with
  streaming + step trees + file upload, not a hand-rolled React app.
- You want **multi-agent runs visualised live** (LangGraph, AutoGen,
  OpenAI-Agents) without writing a custom dashboard.
- You need a **HITL approval surface** (`cl.Action` buttons, file
  upload, audio input) for a research / internal-tool workflow.

**When not to choose.** You need a graph IDE for non-engineers —
[`langflow`](../langflow/) is the canvas-first choice. You need
agent observability over many production runs, not a per-user chat
session — [`langfuse`](../langfuse/) / [`arize-phoenix`](../arize-phoenix/)
is the right layer. You want a TS-first UI framework — `Vercel AI
SDK` + `shadcn/chat` is the JS-stack analogue. You want a
*headless* agent runtime to embed in another app — Chainlit's value
*is* the React chat shell, so [`langgraph`](../langgraph/) /
[`mastra`](../mastra/) / [`agno`](../agno/) are leaner cores.
