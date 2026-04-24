# llm

> Snapshot date: 2026-04. Upstream: <https://github.com/simonw/llm>

Simon Willison's `llm`. Like `mods`, it is a **pipe-style CLI primitive**
for talking to language models — but with two characteristics that put it
in a different ecological niche entirely:

1. **Every prompt and response is logged to SQLite by default.** The
   default database lives at `~/.config/io.datasette.llm/logs.db` and is
   queryable from `llm logs` or directly via `sqlite3` / Datasette. You
   get a free, durable, local audit log of every model call you ever
   make from this tool.
2. **A first-class plugin system.** New model providers, embeddings
   backends, fragment loaders, command extensions, and template
   handlers all ship as pip-installable plugins. The list of
   supported models is effectively unbounded, and you can write a new
   provider plugin in ~50 lines.

It is not an agent. It does not edit files, it does not loop, it does
not call tools (though there is experimental tool support behind a
flag). Its job is to be the smallest possible surface for "talk to an
LLM from a script, and remember what was said."

## 1. Install footprint

- `pipx install llm` (recommended), `pip install llm`, or
  `brew install llm`.
- Pure Python, ~10 MB venv after install. Plugins add 1–50 MB each
  depending on whether they pull in heavy SDKs.
- Config and SQLite log live under
  `~/.config/io.datasette.llm/` (XDG-respecting). Templates,
  fragments, and aliases are also stored here.
- No daemon, no project-local state. Each invocation is a fresh
  process; conversations are reconstructed from the SQLite log on
  demand via `--cid` / `-c`.

## 2. License

Apache-2.0.

## 3. Models supported

OpenAI is built-in. Everything else is a plugin:

- **Anthropic** — `llm-anthropic`
- **Gemini** — `llm-gemini`
- **Mistral** — `llm-mistral`
- **Ollama / local** — `llm-ollama`, `llm-llama-cpp`, `llm-gpt4all`
- **Cohere** — `llm-command-r`
- **OpenRouter** — `llm-openrouter`
- **Bedrock / Vertex** — community plugins
- **Embeddings backends** — `llm-sentence-transformers`,
  `llm-cluster`, `llm-embed-jina`, etc.

`llm models` lists every model the current install can address.
`llm keys set <provider>` stores credentials in
`~/.config/io.datasette.llm/keys.json` (mode 600).

## 4. MCP support

No native client. There is a community plugin (`llm-tools-mcp`) that
exposes MCP servers as tools, but it is experimental and depends on
the still-stabilising `--functions` / `--tool` surface in core. If
MCP is a hard requirement, pick a different tool.

## 5. Sub-agent model

None. `llm` is single-shot or single-conversation. Conversations
chain via `-c` (continue last) or `--cid <id>` (continue a specific
logged conversation). There is no planner/builder split, no
tool-loop, no recursive call. If you need agency, compose `llm`
inside a shell loop you write yourself — that is the intended usage
pattern.

## 6. Telemetry stance

**Off, with no opt-in.** `llm` itself sends no telemetry to anyone.
The model providers obviously see your prompts (that is what you
asked for); the tool itself does not phone home and has no analytics
SDK.

The SQLite log is local-only. Nothing is uploaded.

## 7. Prompt-cache strategy

None at the framework level. `llm` is a thin shim over each provider's
SDK; if the provider supports prompt caching (e.g. Anthropic's
`cache_control` blocks), you can pass it through with
`-o cache_control '{"type":"ephemeral"}'` on a per-call basis, but
`llm` itself will not insert cache markers for you.

The SQLite log is the closest thing to a cache — you can replay a
prior conversation cheaply by piping `llm logs --cid <id>` and
re-feeding the assistant turns.

## 8. Hot keybinds

There is no TUI. `llm` is a CLI verb. The relevant ergonomics are
the subcommands themselves:

- `llm "prompt"` — one-shot
- `llm -c "follow-up"` — continue most-recent conversation
- `llm -m claude-3.7-sonnet "prompt"` — pick a model
- `llm chat` — interactive REPL (still single agent, single thread)
- `llm logs -n 5` — last 5 prompts as JSON
- `llm logs -q "<keyword>"` — full-text search the SQLite log
- `llm templates` / `llm -t <name>` — saved prompt templates
- `llm embed-multi <collection> --files src/ '*.py'` — index files
  into a local embeddings collection
- `llm similar <collection> -q "regex parser"` — semantic search

## 9. Killer feature, weakness, when to choose

**Killer feature.** The SQLite log + plugin system combination. You
get a queryable history of every model call across every project
across every machine you sync the log to, and the tool will outlive
any single provider because new providers ship as third-party
plugins. Six months from now when you ask "wait, what was the prompt
I used to debug that auth thing in March?", `llm logs -q auth` is
the answer.

**Weakness.** Not an agent. If you want anything to actually happen
on disk — file edits, tool calls, multi-step reasoning — you have to
build that orchestration yourself in shell or Python. The tool
support that exists is experimental and not yet a drop-in for what
codex / opencode / aider give you. MCP is plugin-only and not
production-grade. Conversation-state model is "last conversation
wins by default", which is convenient for one user at a terminal but
foot-gun-prone inside scripts (always pass `--cid` explicitly in
automation).

**When to choose.**

- You want a **persistent, queryable record** of every LLM call you
  make — for cost analysis, prompt iteration, or just memory.
- You write a lot of **shell pipelines that include an LLM step**
  and you want each step to be addressable and replayable later.
- You need to **support obscure or new model providers** quickly —
  writing or installing a plugin is faster than waiting for a big
  agent CLI to add a provider.
- You are doing **embeddings + semantic search** and want the same
  CLI surface for chat and for indexing/querying vectors.
- You want a **scriptable LLM primitive** that is more capable than
  `mods` (sessions, embeddings, plugins) but still nowhere near an
  agent.

**When not to choose.** You want code edits done for you, you want
tool-call loops, you want sandboxed execution, you want a TUI with
diff preview, or you need rock-solid MCP support today. Pick an
agent-class tool from the catalog instead — `llm` is a primitive,
not an assistant.
