# oterm

> Snapshot date: 2026-04. Upstream: <https://github.com/ggozad/oterm>
> Binary name: `oterm`

`oterm` is the **Ollama-native TUI** of the catalog. Unlike every
other entry, it speaks exclusively to a local Ollama daemon at
`http://localhost:11434` (or a user-configured remote Ollama
endpoint) and treats that as the only inference backend that
exists. There is no OpenAI key field, no Anthropic key field, no
LiteLLM router, no provider abstraction. The tool's whole reason to
exist is to make a local Ollama installation pleasant to use from a
terminal — model picker, multi-chat tabs, image input, tool calls,
MCP — without touching any cloud provider.

```
ollama serve &
ollama pull llama3.2:3b
oterm
```

That is the entire onboarding. From there you press `Ctrl+N` for a
new chat, pick a model from whatever `ollama list` reports, and
start typing.

This is taxonomically distinct from [`aichat`](../aichat/) (which
treats Ollama as one of 20+ providers and is provider-agnostic at
its core), from [`gptme`](../gptme/) (which is a tool-using agent
REPL, with Ollama as one model option), and from [`tgpt`](../tgpt/)
(which is one-shot stdin/stdout, no chat tabs, free *cloud*
endpoints by default). `oterm`'s differentiator is **Ollama as the
only backend, with a real multi-chat TUI on top**.

## 1. Install footprint

- Pure Python, distributed on PyPI as `oterm`. Install with `pipx
  install oterm`, `uv tool install oterm`, or `brew install oterm`.
- Requires a reachable Ollama server. Default URL is
  `http://localhost:11434`; override with `OLLAMA_HOST` or
  `OLLAMA_URL` env vars.
- App data (chat history, settings) lives at
  `~/.local/share/oterm/` on Linux and the platform-equivalent
  application-support directory on macOS / Windows. Chats are
  persisted in a SQLite database, browseable across runs.
- TUI built on [Textual](https://github.com/Textualize/textual);
  works in any terminal that supports 24-bit color, falls back
  gracefully on 256-color.

## 2. License

MIT.

## 3. Models supported

**Ollama only.** Whatever `ollama list` shows on the configured
endpoint is what `oterm` sees in its model picker. This includes
anything Ollama can pull from its registry (Llama 3.x, Qwen 2.5 /
3, DeepSeek, Mistral, Gemma 3, Phi-4, Granite, etc.) plus any
custom model you have created with `ollama create -f Modelfile`.

There is no OpenAI / Anthropic / Gemini / Bedrock support and no
plan to add it — the project explicitly scopes itself to Ollama.
For multi-provider TUIs the catalog has [`aichat`](../aichat/),
[`opencode`](../opencode/), [`crush`](../crush/), and others.

Vision-capable Ollama models (`llava`, `llama3.2-vision`,
`qwen2.5vl`, etc.) get a working image-input affordance: drag-drop
or `/image <path>` attaches an image to the next user turn.

## 4. MCP support

**Yes, as a client.** `oterm` ships an MCP client and can attach to
arbitrary MCP servers configured in its settings file
(`~/.local/share/oterm/config.json`-equivalent). Tools, prompts,
and sampling are surfaced into the chat UI; tool calls are gated
behind a per-call confirmation toggle so a local model cannot
silently invoke a server-side action.

This is unusual among local-only TUIs — most tools that talk to
Ollama do not bother with MCP because local models historically had
weak tool-calling. `oterm` exposes it anyway, which becomes useful
once you point it at a tool-tuned Qwen 2.5 or a Llama 3.3 model
that can produce reliable JSON tool calls.

## 5. Sub-agent model

**None.** `oterm` is a chat-and-tool-call surface, not an agent
framework. Each conversation is a single model with a single
context window; tool calls happen in-line and the same model
handles the response. There is no planner / executor split, no
spawn-a-second-conversation primitive, no `Task`-tool equivalent.

If you want orchestration on top of local models, the catalog
options are [`gptme`](../gptme/) (single agent with a richer tool
set), [`opencode`](../opencode/) or [`continue`](../continue/)
pointed at an Ollama endpoint, or building on top of
[`goose`](../goose/) which exposes everything as MCP extensions.

## 6. Telemetry stance

**Off, with no opt-in.** `oterm` itself sends nothing. The Ollama
server it talks to runs on your machine (or a host you control).
There is no analytics endpoint, no crash reporter, no update
checker that pings home. Chat content stays in the local SQLite
file unless you explicitly export it.

This is a defining property — `oterm`'s entire customer is "I want
to chat with a local model and I want zero bytes leaving the box."

## 7. Prompt-cache strategy

Inherits whatever Ollama provides. Recent Ollama versions reuse the
KV cache across turns of the same chat session by keeping the model
loaded and the conversation tokens warm; `oterm`'s persistent
chat-tab model maps cleanly onto that. Switching models within a
tab triggers an Ollama unload / reload, which on a 7B–8B model is
single-digit seconds and on a 70B model can be tens of seconds —
something to know when you bounce between models.

There is no `oterm`-side disk cache of completions, no prompt
deduplication, no semantic-cache plugin. The persistence is "every
turn is in SQLite, you can scroll back."

## 8. Hot keybinds

`oterm` is keyboard-first. The defaults (configurable):

- `Ctrl+N` — new chat tab.
- `Ctrl+T` — list / switch chat tabs.
- `Ctrl+W` — close current tab.
- `Ctrl+E` — edit a previous user message in place; the chat
  forks from that point, dropping the subsequent turns.
- `Ctrl+R` — regenerate the last assistant response.
- `Ctrl+Q` — quit.
- `Ctrl+P` — open the model picker for the current tab.
- `Ctrl+S` — open settings (system prompt, temperature, parameters,
  tool/MCP toggles for this tab).
- `/image <path>` — attach an image (vision models only).
- `/clear` — wipe the current chat's history without closing the tab.
- `/export <file>` — write the current chat to a markdown file.

The "edit a previous turn and fork" affordance (`Ctrl+E`) is the
single best ergonomic feature — it makes iterative prompt
refinement on a slow local model bearable, because you can rewind
and retry without reissuing the whole conversation.

## 9. Killer feature, weakness, when to choose

**Killer feature.** A **first-class, persistent, multi-tab TUI
purpose-built for Ollama**, with MCP-client support, vision input,
and edit-and-fork on every turn. Nothing else in the catalog scopes
itself this tightly — by giving up provider-agnosticism, `oterm`
gets to make every UI affordance assume "the model is local, the
backend is Ollama, no API key exists" and the result is the
smoothest local-only chat experience in the catalog.

**Weakness.** Three:

1. **Ollama-only by design.** If you also need cloud access from
   the same TUI, this is the wrong tool. There is no provider
   abstraction; pull in [`aichat`](../aichat/) or
   [`opencode`](../opencode/) instead.
2. **Not an agent.** No planning, no sub-agents, no autonomous file
   editing. `oterm` is a chat client with tool calls, not an
   engineering assistant. For repo-level edits over local models,
   point [`aider`](../aider/) at an Ollama endpoint.
3. **Local-model quality is your problem.** `oterm` is only as
   smart as the Ollama model you load. A 3B model in `oterm` will
   feel dumb compared to a cloud frontier model in any other entry.
   Expectation management is on the user.

**When to choose.**

- You run a **local Ollama instance** as your primary inference
  backend and want a real terminal app on top of it instead of
  `curl`-ing the API.
- You need **air-gapped operation** with a chat UI; nothing here
  needs the public internet once Ollama has the weights.
- You want **multiple parallel chats** with different local models
  (a quick 3B for triage, a slower 32B for hard questions) without
  juggling shells.
- You want **vision input** to local multimodal models from a
  terminal.
- You want **MCP tool calls** routed through a local model for
  privacy-sensitive automation.

**When not to choose.**

- You need **OpenAI / Anthropic / Gemini** access. Use
  [`aichat`](../aichat/), [`mods`](../mods/), [`llm`](../llm/), or
  any agent CLI in the catalog.
- You want a tool that **edits files in your repo** based on the
  conversation. Use [`aider`](../aider/), [`opencode`](../opencode/),
  [`codex`](../codex/), or [`continue`](../continue/) pointed at a
  local endpoint.
- You want **one-shot stdin/stdout** for shell pipelines. `oterm`
  is interactive-first; reach for [`mods`](../mods/),
  [`tgpt`](../tgpt/), or [`llm`](../llm/) instead.
- You manage **many model providers per project** and want one TUI
  that handles all of them — this is explicitly *not* `oterm`'s
  scope.
