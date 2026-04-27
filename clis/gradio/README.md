# gradio

> Snapshot date: 2026-04. Upstream: <https://github.com/gradio-app/gradio>

"**Build and share delightful machine learning apps, all in
Python.**" Gradio is the *default* way the open-weight model
ecosystem ships demo UIs: a Python file declares
`gr.Interface(fn=predict, inputs=..., outputs=...).launch()` and you
get a typed web UI, a public temporary share URL via the
`gradio.live` tunnel, an OpenAPI-described HTTP API, and a Python /
JavaScript client that calls the same endpoint. The same primitive
underpins Hugging Face Spaces, the `gradio` CLI for live-reload
development, the bundled `gradio deploy` to push to Spaces, and the
high-level `gr.ChatInterface` / `gr.Chatbot` shapes that most
open-source LLM playgrounds (text-generation-webui, llama.cpp's
`server`, vLLM's demo notebooks, sglang's example servers, lmdeploy
`serve gradio`) wrap.

## 1. Install footprint

- `pip install -U gradio` (core); `pip install "gradio[mcp]"` adds
  the MCP server bridge; `pip install "gradio[oauth]"` adds the
  OAuth login helpers.
- Python ≥ 3.10. Pulls FastAPI, Uvicorn, Pydantic v2, `httpx`,
  `pillow`, `numpy`, `huggingface_hub`, `safehttpx`,
  `python-multipart`, `aiofiles`, `orjson`, `markupsafe`, `jinja2`,
  `typer`. Front-end is a pre-built Svelte bundle shipped inside the
  wheel — no `npm` step required.
- `gradio` CLI installed alongside: `gradio app.py` (live-reload dev
  server with hot module replacement), `gradio deploy` (push to
  Hugging Face Spaces), `gradio cc create` (scaffold a custom
  component), `gradio cc dev`, `gradio cc build`, `gradio cc publish`,
  `gradio environment`, `gradio sketch` (no-code UI builder).
- Sister package `gradio_client` (Python) and `@gradio/client` (JS)
  — `pip install gradio_client` is a much smaller install that lets
  any script call any deployed Gradio app's API.

## 2. Repo + version + license

- Repo: <https://github.com/gradio-app/gradio>
- Latest release: **gradio@6.13.0** (monorepo tag; PyPI `gradio==6.13.0`)
- License: **Apache-2.0** —
  <https://github.com/gradio-app/gradio/blob/main/LICENSE>
- Default branch: `main` (HEAD `fffd4c66` at snapshot)
- Language: Python (server) + Svelte / TypeScript (front-end)

## 3. Models supported

Not a model itself — Gradio is the UI / HTTP wrapper. The
`gr.load("models/owner/name")` shortcut fetches a Hub model and
auto-builds an Interface for it (Inference API under the hood); the
`gr.ChatInterface` shape composes with any Python function that
yields strings, so the model wiring is whatever you import inside
that function: `transformers.pipeline(...)`, `openai.OpenAI()`,
`anthropic.Anthropic()`, `huggingface_hub.InferenceClient(...)`,
`ollama.Client()`, `litellm.completion(...)`, a local `vllm`
endpoint, etc. Streaming is first-class (`yield` from the function);
multimodal inputs (image / audio / video / file / dataframe) are
declarative components.

## 4. MCP support

**Yes (server, first-class).** Add `mcp_server=True` to
`demo.launch(...)` and Gradio mounts an MCP server on the same port
that exposes every Interface / Block as an MCP tool — argument
schemas come from the component types, file inputs / outputs are
handled as MCP resources, and the `/gradio_api/mcp/sse` endpoint
plugs straight into Claude Desktop, OpenCode, Cursor, or any
`MCPClient`. The `gradio[mcp]` extra ships the dependency. There is
no built-in MCP client (Gradio is the *server* side of the
relationship); pair with [`mcp-cli`](../mcp-cli/) /
[`huggingface_hub`](../huggingface_hub/) `tiny-agents` /
[`agno`](../agno/) for the client side.

## 5. Sub-agent model

None — Gradio is a UI primitive, not an agent runtime. The
`gr.Chatbot` + `gr.ChatInterface` pair gives you the chat shell and
streaming rendering; everything inside the `fn=` callback is yours
to compose (a single LLM call, a multi-agent orchestrator, a tool
loop). The community pattern is to wrap an existing agent framework
([`smolagents`](../smolagents/), [`langgraph`](../langgraph/),
[`agno`](../agno/), [`autogen`](../autogen/)) with a `ChatInterface`
in 20 lines.

## 6. Telemetry stance

- Anonymous usage analytics on by default (counts of `launch()`
  calls, component types used, Python / Gradio version, OS — no
  prompts, no inputs, no outputs). Disable with
  `GRADIO_ANALYTICS_ENABLED=False` or `analytics_enabled=False` in
  `Blocks(...)`.
- `share=True` opens an FRP tunnel through `gradio.live` (Hugging
  Face-operated) — every request to the share URL transits HF
  infrastructure. Off by default; `share=False` keeps traffic on
  `127.0.0.1`.
- `gradio deploy` uploads your code to Hugging Face Spaces (the
  hosted runtime owns logs / runtime metrics from then on).

## 7. Killer feature, weakness, when to choose

**Killer feature.** **One Python function becomes a typed web UI, a
streaming HTTP API with OpenAPI docs, a Python / JS client SDK, an
MCP server, and a public share URL — in one `launch()` call.** The
`gr.Interface` / `gr.Blocks` / `gr.ChatInterface` triplet covers
"quick demo of one function", "multi-tab dashboard with custom
layout", and "ChatGPT-shaped streaming chat with tool-use rendering"
respectively, all with the same component vocabulary
(`gr.Image`, `gr.Audio`, `gr.Video`, `gr.Dataframe`,
`gr.JSON`, `gr.File`, `gr.Plot`, `gr.Markdown`, `gr.Code`,
`gr.Model3D`, `gr.Gallery`, `gr.HighlightedText`). The bundled
`gradio_client` means a notebook in another repo can `from
gradio_client import Client; Client("user/space").predict(...)` and
call your demo as if it were a Python function — Spaces become
remote-callable functions for free, which is why the open-source
inference ecosystem standardised on Gradio as the demo layer.
`mcp_server=True` upgrades that to "any Gradio app is also an MCP
server" with zero schema authoring.

**Weakness.** **Opinionated UI shell that fights bespoke design.**
The Svelte bundle is ~5 MB on first load, the styling system
(`theme=`, CSS overrides, `gr.HTML` escapes) is workable but not
Tailwind-clean, and complex multi-page apps eventually want a real
front-end framework — Gradio is great up to "rich single-page
demo + maybe a tab or two", less great as an SaaS product surface
(use [`chainlit`](../chainlit/) for chat-shaped apps with auth +
data layer + observability, or a real React / Svelte front-end if
the UI is the product). The `share=True` tunnel routes through
`gradio.live` (Hugging Face) — fine for demos, not fine for
production. Component schema changes between major versions
(3.x → 4.x → 5.x → 6.x) have broken Spaces in the past; pin a
version. Custom components (`gradio cc`) are powerful but require a
real Node toolchain.

**When to choose.** You have a Python function — a model inference,
a RAG query, an agent loop, an audio / image / video pipeline — and
you want a **shareable web UI + HTTP API + MCP server** in 20 lines
without standing up FastAPI + a front-end project. Default choice
for Hugging Face Spaces submissions, model-card live demos, internal
tools where "the data scientist owns the UI", and any case where
`mcp_server=True` is the cheapest route from "Python function" to
"tool callable by Claude Desktop / Cursor / OpenCode". Pair with
[`huggingface_hub`](../huggingface_hub/) for hosting on Spaces, with
[`agno`](../agno/) / [`langgraph`](../langgraph/) /
[`smolagents`](../smolagents/) for the agent inside the callback,
with [`bentoml`](../bentoml/) / [`litserve`](../litserve/) when the
HTTP layer needs to be the product not the demo.
