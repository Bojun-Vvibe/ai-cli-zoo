# aichat

> Snapshot date: 2026-04. Upstream: <https://github.com/sigoden/aichat>

A single-binary Rust **terminal LLM client** that occupies an unusual
middle position in the catalog: more than a pipe primitive (it has
sessions, RAG, function calling, and a REPL), less than an agent
(it does not autonomously edit files in a project, does not loop on
tool calls without you pressing return, does not maintain a long-lived
project-state). It is what most engineers actually want from a
"general-purpose chat over many providers" tool: zero-runtime install,
twenty providers, sessions on disk, document-RAG built in, and a
shell-integration mode that turns it into an inline command-completion
helper.

It is the right tool when you want **chat with documents from a
terminal** without standing up a Python venv or a web UI.

## 1. Install footprint

- `brew install aichat`, `cargo install aichat`, `scoop install aichat`,
  or download a release binary.
- Single static Rust binary, ~12 MB, no runtime dependencies. Works
  identically on macOS / Linux / Windows.
- Config at `~/.config/aichat/config.yaml` (XDG-respecting). Sessions
  in `~/.config/aichat/sessions/<name>.yaml`, RAG indexes in
  `~/.config/aichat/rags/<name>/`, roles (system-prompt presets) in
  `~/.config/aichat/roles/`.
- No daemon, no project-local state by default. You can opt into a
  per-directory session with `aichat -s <name>`.

## 2. License

MIT or Apache-2.0 dual-licensed.

## 3. Models supported

Twenty-plus providers wired into core, no plugin install needed:

- OpenAI, Azure OpenAI, Claude (Anthropic), Gemini (AI Studio +
  Vertex), Mistral, Cohere, Groq, Perplexity, DeepSeek, Together,
  Fireworks, OpenRouter, Replicate, Ollama (local),
  llama.cpp / vllm / any OpenAI-compatible server, Bedrock,
  Cloudflare Workers AI, ZhipuAI, Moonshot, Qwen (DashScope), Ernie.
- `aichat --list-models` shows everything the current config
  resolves to. Switch model mid-session with `.model <name>`.

The wide provider matrix is the biggest single reason people pick
this CLI over `mods` or `llm`.

## 4. MCP support

No native MCP client as of 0.27. There is an active issue tracking
it; the `function_calling` field in config gives you a workable
substitute (you point it at a directory of executable scripts and
each becomes a callable function), but it is not MCP and tools
written for MCP servers will not work without re-wrapping.

If MCP is mandatory, `aichat` is not the pick — see `goose`,
`opencode`, or `claude-code`.

## 5. Sub-agent model

None. Single conversation, single agent. Function calls are
serial and require explicit user approval per call by default
(`function_calling: true; auto_call: false`). You can flip
`auto_call: true` to get an autonomous loop, but it is still
single-thread, single-context — no planner/builder split, no
parallel sub-agents.

## 6. Telemetry stance

**Off, no opt-in.** No analytics SDK in the binary. Outbound
network calls are exclusively to the model providers and (if you
enable it) to the search-engine plugin you configured for RAG /
web augmentation.

## 7. Prompt-cache strategy

Provider-pass-through. If you target Anthropic models, set
`extra: { cache_control: ephemeral }` on the client block to opt
into prompt caching for system prompts and large attached
documents. Gemini implicit prefix caching applies automatically.
There is no aichat-level cache layer.

The session file is not a cache — it is the literal message
history that gets re-sent on every turn (subject to
`max_input_tokens` truncation).

## 8. Hot keybinds

REPL mode (`aichat` with no args, or `aichat -s <name>`):

- `.help` — list commands
- `.model <name>` — switch model in-session
- `.role <name>` — apply a saved system-prompt role
- `.session <name>` — switch session (or create)
- `.rag <name>` — attach a RAG index
- `.file <path>` — attach a file to the next turn
- `.copy` — copy last assistant turn to clipboard
- `.edit` — open `$EDITOR` to compose a long prompt
- `.exit` / `Ctrl-D` — leave the REPL
- `Tab` — completion on commands and saved names

CLI mode (one-shot):

- `aichat 'one-line prompt'` — single response, exit
- `aichat -f path/to/log.txt 'summarise this'` — attach file
- `aichat -m claude-3.7-sonnet 'prompt'` — pick model
- `aichat -e 'find all png larger than 1MB'` — generate shell
  command, prompt-y / e / n before executing
- `aichat -c 'parse this csv into json'` — code-block-only mode

Shell integration (`aichat --shell-integration`) installs zsh /
bash / fish / nushell / pwsh hooks that bind a key (default `Alt-E`)
to "complete this command line via aichat -e".

## 9. Killer feature, weakness, when to choose

**Killer feature.** Built-in **document RAG** with the same single
binary. `aichat --rag <name>` walks you through chunking and
indexing a directory of PDFs / markdown / source files, embeds
them via your configured embedding model, and stores the index
locally. From then on `.rag <name>` inside a session attaches it
as retrievable context. No vector database to run, no Python venv
to maintain, no langchain to debug. The competition for "ask
questions over a folder of docs from a terminal" is mostly a
20-line Python script over a hosted vector DB; this is one binary
and a directory.

**Weakness.** Not an agent. Function calling exists but is not as
ergonomic as a real tool-loop CLI; the function-script convention
(executable in `~/.config/aichat/functions/bin/<name>` returning
JSON) works but is more friction than MCP. No file-editing
primitives — if you ask aichat to "fix a bug in src/foo.rs" it
will print a diff into the terminal, you copy-paste. No
sandboxing for the auto-call function loop. Conversation truncation
on long sessions can drop the wrong middle messages; manual session
pruning via `.save` / `.clear` is occasionally needed.

**When to choose.**

- You want a **single static binary** that handles "talk to many
  models from a terminal" with **sessions** and **RAG** out of the
  box, and you do not want a venv or a daemon.
- You routinely **switch providers mid-conversation** to compare
  outputs (`.model gpt-5` then `.model claude-3.7-sonnet`).
- You want **shell-integration command completion** as a daily
  driver for "I forgot the flag for `tar`".
- You want **document Q&A from your local filesystem** without
  setting up a separate RAG stack.
- You target **Chinese-ecosystem providers** (Qwen, Zhipu,
  Moonshot, Ernie) which several other CLIs in this catalog do
  not natively cover.

**When not to choose.** You need MCP, you want autonomous
multi-step file edits, you want a TUI with side-by-side diff
preview, or you want a multi-agent / planner-builder split. Pick
`opencode`, `claude-code`, `aider`, or `plandex` instead.
