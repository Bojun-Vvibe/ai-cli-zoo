# gpt-cli

> Snapshot date: 2026-04. Upstream: <https://github.com/kharvd/gpt-cli>

A small Python REPL for chatting with OpenAI / Anthropic / Google
Gemini / Cohere / any OpenAI-compatible endpoint, configured by a
single YAML file at `~/.config/gpt-cli/gpt.yml`. The differentiator
is the *assistant* abstraction: each entry in the YAML is a named
preset (model + temperature + top-p + system prompt + predefined
opening messages) that you launch with `gpt <name>`. Out of the box
you get `general`, `dev`, and `bash`; you add your own by editing
the YAML.

If [`mods`](../mods/) is "LLM as a unix pipe" and
[`elia`](../elia/) is "LLM as a multi-tab TUI", `gpt-cli` is
"LLM as a config-driven REPL with named personas". It is the
smallest path in the catalog from "I have an API key" to "I have
five differently-tuned chat agents one keypress apart".

## 1. Install footprint

- `pip install gpt-command-line` (PyPI package name; the binary
  installed is `gpt`).
- Or zero-install via uv: `uvx --from gpt-command-line gpt`.
- Pure Python; ~15 MB venv plus the provider SDKs you actually use.
- Configuration: `~/.config/gpt-cli/gpt.yml`. API keys via
  environment variables (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`,
  `GOOGLE_API_KEY`, `COHERE_API_KEY`) **or** the same YAML file.
- Logs (optional): `--log_file path.jsonl --log_level INFO` writes
  every turn to a JSONL file you can grep later. There is no
  built-in SQLite store — pair with [`llm`](../llm/) if you want
  one.

## 2. Repo, version, license

- Repo: <https://github.com/kharvd/gpt-cli>
- Version checked: **v0.4.3** (released 2025-04-15; default branch
  `main` was active on 2025-04-21).
- License: MIT. License file at the repo root: `LICENSE`.
- PyPI package: `gpt-command-line`. Binary: `gpt`.

## 3. What it actually does

`gpt` opens an interactive readline prompt. Each line is a turn;
multi-line input is supported via a leading `"""` (or `Ctrl-Space`
toggle in some terminals). Useful keybinds:

- `Ctrl-C` — interrupt the current streaming response (without
  exiting).
- `Ctrl-D` — end the session.
- `Ctrl-R` — re-roll the last response with the same prompt
  (uses the model's regenerate path; cost counter still increments).

The `gpt.yml` file declares assistants:

```yaml
assistants:
  rust:
    model: anthropic/claude-3-7-sonnet-20250219
    temperature: 0.2
    messages:
      - { role: system, content: "You are a senior Rust engineer..." }
      - { role: user, content: "Always show the diff first." }
      - { role: assistant, content: "Understood." }
  bash:
    model: gpt-4o
    temperature: 0
    messages:
      - { role: system, content: "Reply with one POSIX shell command, no prose." }
```

`gpt rust` boots the first assistant with that system prompt
already in context; `gpt bash` boots the second. The
`general`, `dev`, and `bash` assistants ship as defaults.

Per-session controls:

- `--model claude-3-5-sonnet-20241022` — override the assistant's
  default model.
- `--temperature 0.7 --top_p 0.95` — override sampling.
- `--thinking 8000` — enable Claude's extended-thinking mode with
  an 8 000-token thinking budget; the reasoning trace is streamed
  inline, prefixed with a marker, before the final answer.
- `--no_markdown` — disable terminal markdown rendering (useful when
  piping output through other tools).
- `--prompt "..." --no_stream` — non-interactive one-shot, exit
  after the response (suitable for scripts).
- `--execute "..."` — special mode for the `bash` assistant: model
  emits a shell command, gpt-cli executes it directly. Use with
  care; there is no `[E]xecute / [A]bort` confirmation step the way
  [`shell-gpt`](../shell-gpt/) has.

A running token-cost counter prints after each turn (model-priced
USD, computed locally from the published per-1M rates baked into
the package — no separate API call).

## 4. MCP support

None. gpt-cli does not consume or expose Model Context Protocol
servers; tool use is limited to the built-in `--execute` shortcut
on the `bash` assistant. For MCP-aware chat, reach for
[`opencode`](../opencode/), [`crush`](../crush/), or
[`mcphost`](https://github.com/mark3labs/mcphost) outside this
catalog.

## 5. Telemetry

Off. gpt-cli makes no analytics calls of its own. Network egress is
exactly the configured provider's API endpoint plus, on Anthropic
extended-thinking sessions, the same endpoint with the
`anthropic-beta: extended-thinking` header. Token-cost numbers are
computed locally; no pricing-API roundtrip.

If you set `--log_file`, every turn (including system prompt and
provider response) is appended to that JSONL file on local disk
only.

## 6. Where it fits in the zoo

- vs [`mods`](../mods/) — `mods` is a single-shot Unix pipe (`echo
  | mods "..."`); `gpt-cli` is an interactive REPL with persona
  presets and a per-turn cost counter. Pick `mods` for shell
  pipelines; pick `gpt-cli` when the workflow is a multi-turn chat
  you want to launch with `gpt rust` and never re-paste the system
  prompt.
- vs [`llm`](../llm/) — `llm` has a vast plugin ecosystem and an
  SQLite-logged history of every call you can `llm logs` against
  later; `gpt-cli` has fewer providers, no plugins, no SQLite, but a
  much simpler "named assistants in one YAML" model. Pick `llm` for
  the long-term audit trail; pick `gpt-cli` when YAML personas plus
  the built-in cost counter are what you actually want.
- vs [`elia`](../elia/) — `elia` is a Textual TUI with tabs,
  searchable history, and `Ctrl-J` mid-conversation model switching;
  `gpt-cli` is a flat readline REPL. Pick `elia` for "find that
  conversation from last month"; pick `gpt-cli` for "spawn the same
  preset over and over from the shell".
- vs [`shell-gpt`](../shell-gpt/) — `shell-gpt` defaults to
  emitting *one shell command* with an `[E]xecute / [D]escribe /
  [A]bort` menu; `gpt-cli` defaults to free-form chat and exposes
  `--execute` as an opt-in for its `bash` assistant only. Pick
  `shell-gpt` for "translate intent → command", pick `gpt-cli` for
  "let me chat with my Rust persona about a refactor".
- vs [`chatblade`](../chatblade/) — both are minimal Python chat
  CLIs; `chatblade` is built around extracting structured sub-paths
  from JSON/YAML responses (`-e '.commands[0]' -j`), `gpt-cli` is
  built around persona presets and Anthropic extended-thinking.

## 7. When to pick gpt-cli

- You want **multiple named chat presets** (one per language /
  project / role) launched with one shell argument, configured in
  one YAML file.
- You use Claude 3.7 / 4 and want **extended-thinking** exposed as
  a flag (`--thinking 8000`) without writing your own SDK call.
- You want a per-turn **USD cost counter** without standing up a
  separate token-counting tool.
- You want a tiny, no-plugins-no-database REPL — one `pip install`,
  one config file, one binary.

## 8. When NOT to pick gpt-cli

- You need MCP / tool calling / sub-agents — use
  [`opencode`](../opencode/) / [`crush`](../crush/) /
  [`claude-code`](../claude-code/).
- You need every prompt logged to a queryable database — use
  [`llm`](../llm/).
- You need shell-pipe ergonomics (`cat file | <tool> "..."`) — use
  [`mods`](../mods/) or [`smartcat`](../smartcat/).
- You need multi-tab / cross-session search in a TUI — use
  [`elia`](../elia/).
- You want the model to actually edit files in your repo — use
  [`aider`](../aider/) or any of the agent CLIs.
