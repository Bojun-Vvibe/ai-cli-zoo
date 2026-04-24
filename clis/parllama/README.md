# parllama

> Snapshot date: 2026-04. Upstream: <https://github.com/paulrobello/parllama>

A full-screen Textual TUI built primarily around an Ollama daemon, with
optional cloud providers bolted on. Where [`oterm`](../oterm/) and
[`elia`](../elia/) treat the terminal chat as the central object,
`parllama` treats the *model registry* as the central object: pulling,
deleting, copying, creating, and quantizing local models is a first-class
screen, alongside chat sessions, custom prompts, a persistent memory
store, and Fabric-pattern import.

It is the only entry in the catalog whose TUI doubles as an Ollama
**model-management console** — you can browse the local registry, pull
new tags, delete old ones, copy/rename, and run `Modelfile` builds
without leaving the keyboard, then drop into a chat session against any
of them in the same window.

## 1. Install footprint

- `pipx install parllama` (recommended) or `uv tool install parllama`.
- Pure Python (Textual + Rich + the `par_ai_core` companion library);
  ~30 MB venv plus whichever provider SDKs you actually wire up.
- Runs on macOS / Linux / Windows (including WSL); single binary
  entrypoint `parllama`.
- Configuration lives at `~/.config/parllama/` (theme, provider keys,
  per-provider model cache TTL, custom prompts, memory store).
- API keys via environment variables — `OLLAMA_HOST` defaults to
  `http://localhost:11434`; `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`,
  `GROQ_API_KEY`, `XAI_API_KEY`, `OPENROUTER_API_KEY`,
  `DEEPSEEK_API_KEY` are all read on startup if present and the
  matching provider screen unlocks.

## 2. Repo, version, license

- Repo: <https://github.com/paulrobello/parllama>
- Version checked: **v0.8.4** (released 2026-01-28; default branch
  `main` was active as of 2026-04-23).
- License: MIT. License file at the repo root: `LICENSE`.
- PyPI package: `parllama`.

## 3. What it actually does

`parllama` opens a multi-screen TUI:

- **Models screen** — lists everything `ollama list` would return, with
  size, parameter count, quantization, and modified date as columns.
  Keybinds for `pull` (with progress bar), `delete`, `copy`, and
  `create from Modelfile`. A separate sub-screen handles
  `llama.cpp`-style GGUF quantization for HuggingFace models you have
  pulled locally (this is the optional `huggingface` extra).
- **Chat screen** — a tabbed conversation view. Each tab is a
  persistent session saved to disk; you can switch model mid-tab,
  fork a session, and export to JSON / Markdown.
- **Prompts screen** — CRUD over a library of system prompts and
  reusable templates. Supports importing
  [Fabric](../fabric/) patterns directly so a Fabric pattern becomes a
  Parllama "prompt" you can attach to any session with one keypress.
- **Memory screen** — a persistent user-context store. Items written
  here are AI-summarized into a context block injected at the top of
  every new session; intended as "things the model should know about
  me" rather than RAG over documents.
- **Templates screen** — the secure-execution side: declared shell
  commands with an allowlist (`config.allowed_commands`) that the
  model may invoke from inside a session. This is the closest thing
  parllama has to an "agent" loop, and it is opt-in per-template.

Vision-capable models (LLaVA, Qwen-VL, GPT-4o vision, etc.) accept
image attachments in chat — drag-and-drop a path or paste an image
filename and the model sees it.

Provider matrix (as of v0.8.4): Ollama, OpenAI, Anthropic, Groq,
XAI, OpenRouter, DeepSeek, plus anything LiteLLM can route to (the
underlying `par_ai_core` uses LiteLLM for the cloud half).

## 4. MCP support

None. parllama does not consume or expose Model Context Protocol
servers; tool use is limited to the allow-listed shell commands of
the Templates screen, which is a parllama-specific mechanism, not
MCP.

If you need an Ollama-only TUI **with** MCP, use [`oterm`](../oterm/)
instead — it is purpose-built for that combination.

## 5. Telemetry

Off. parllama makes no analytics calls of its own. Network egress
is exactly what the active screen requires:

- Models screen → your Ollama daemon (default `127.0.0.1:11434`)
  and, on `pull`, `registry.ollama.ai`.
- Chat screen → whichever provider endpoint backs the active
  session (Ollama / OpenAI / Anthropic / Groq / XAI / OpenRouter /
  DeepSeek / any LiteLLM-routable URL).
- Quantization sub-screen → HuggingFace Hub for the source GGUF if
  not already cached.

No crash reporters, no usage pings.

## 6. Where it fits in the zoo

- vs [`oterm`](../oterm/) — `oterm` is *only* Ollama and ships
  built-in MCP client support; pick `oterm` when the workflow is
  "local Ollama + MCP servers, nothing else". Pick `parllama` when
  the workflow is "I manage a local Ollama registry **and** want to
  fan out to cloud providers in the same TUI", or when you need the
  Fabric-pattern / persistent-memory workflows.
- vs [`elia`](../elia/) — `elia` is a chat TUI with SQLite history
  and `/`-search across past conversations; it has no model-management
  UI and no shell-tool execution. Pick `elia` when the value is
  "find that thing I asked the model three weeks ago"; pick
  `parllama` when the value is "manage my Ollama models without
  shelling out".
- vs [`aichat`](../aichat/) — `aichat` is a single-binary REPL with
  built-in RAG and an OpenAI-compatible local server mode; no TUI,
  no model registry browser. Pick `aichat` when you want a Unix-pipe
  CLI plus folder RAG; pick `parllama` when you want a
  multi-screen TUI.
- vs [`ramalama`](../ramalama/) / [`ollama`](../ollama/) — those are
  *runtimes* (the daemon serving the model); `parllama` is a
  *client* against them. They compose: run `ramalama` or `ollama`
  for the daemon, then point `parllama` at `OLLAMA_HOST` for the
  TUI.

## 7. When to pick parllama

- You already run Ollama locally and find yourself shelling out to
  `ollama pull` / `ollama list` / `ollama rm` constantly — parllama
  collapses all of that into one keyboard-driven screen.
- You want to occasionally fan a session out to a cloud model
  (Anthropic, OpenAI, Groq, OpenRouter) without leaving the TUI.
- You have a [Fabric](../fabric/) prompt library and want to use
  those patterns interactively, with session history, instead of as
  one-shot pipes.
- You want vision-model chat against local LLaVA / Qwen-VL without
  building your own UI.

## 8. When NOT to pick parllama

- You only ever talk to Ollama and you need MCP — use
  [`oterm`](../oterm/).
- You want a Unix-pipe primitive — parllama has no stdin/stdout
  mode worth using; reach for [`mods`](../mods/) /
  [`llm`](../llm/) / [`smartcat`](../smartcat/).
- You want the model to actually *edit your repo* — parllama is a
  chat client with allow-listed shell snippets, not a coding agent.
  Use [`aider`](../aider/), [`opencode`](../opencode/), or
  [`claude-code`](../claude-code/).
- You want SQLite-backed cross-session search — use
  [`elia`](../elia/).
