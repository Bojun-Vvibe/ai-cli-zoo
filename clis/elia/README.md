# elia

> Snapshot date: 2026-04. Upstream: <https://github.com/darrenburns/elia>
> Binary name: `elia`

`elia` is a **keyboard-driven Textual TUI for chatting with multiple
LLM providers, with every conversation persisted to local SQLite by
default**. It is the closest thing in the catalog to "ChatGPT-style
chat, but in the terminal, fully offline-storable, multi-provider".

The niche distinction matters because two existing entries look
adjacent:

- [`oterm`](../oterm/) is a Textual TUI too, but **Ollama-only** —
  no cloud surface, no OpenAI/Anthropic/Gemini providers. `elia`
  is multi-provider (OpenAI, Anthropic, Gemini, Groq, plus any
  Ollama / LM Studio / OpenAI-compatible local endpoint) and was
  designed cloud-first.
- [`llm`](../llm/) also writes every prompt to SQLite, but it is a
  pipe-style CLI primitive with no UI; you replay history by
  querying SQLite or running `llm logs`. `elia` is the inverse:
  the SQLite log exists to power a **searchable, scrollable chat
  history pane** in the TUI, not as a primary CLI artifact.

So `elia` fills the slot: "I want a real chat-app feel, with
sidebar history and arrow-key navigation between conversations,
without leaving the terminal, and I want my history on my disk."

Three modes define the surface:

- `elia` — launches the full TUI with a conversation list on the
  left and a chat pane on the right. Arrow keys to switch
  conversations, `Ctrl-N` to start a new one, `/` to search.
- `elia "What is the difference between TCP and UDP?"` — opens the
  TUI with a fresh conversation seeded by that initial message,
  useful as a one-liner from `$EDITOR` or a key-binding.
- `elia --inline "..."` — non-TUI mode: emits a single response to
  stdout and exits, for pipe-style usage. (This mode exists but is
  intentionally minimal; if you want pipe-first ergonomics use
  `mods` or `llm`.)

## 1. Install footprint

- Pure Python, distributed via PyPI: `pipx install elia-chat`
  (the project ships as `elia-chat` on PyPI to avoid a name
  collision; the binary it installs is `elia`).
- Heavy on Python deps relative to one-shot tools: pulls in
  `textual` (TUI framework), `sqlmodel` / `sqlalchemy` (history
  store), and one provider client per supported backend (`openai`,
  `anthropic`, `google-generativeai`, etc.). First install is
  ~100 MB of wheels; not unreasonable for a TUI app, but heavier
  than `mods` or `tgpt`.
- Python 3.11+. The Textual dependency moves fast; older Python
  versions are not supported.
- Config lives at `~/.config/elia/config.toml`, written on first
  run with sensible defaults. The SQLite database lives at
  `~/.local/share/elia/elia.sqlite`. Both paths respect
  `XDG_CONFIG_HOME` / `XDG_DATA_HOME`.

## 2. License

Apache-2.0.

## 3. Models supported

Multi-provider, declared in `config.toml`:

- **OpenAI** — `OPENAI_API_KEY`. Models declared by name
  (`gpt-4o`, `gpt-4o-mini`, etc.).
- **Anthropic** — `ANTHROPIC_API_KEY`. Claude family.
- **Gemini** — `GOOGLE_API_KEY` (AI Studio path). Gemini 1.5/2.x.
- **Groq** — `GROQ_API_KEY`. Hosted open-weights at low latency.
- **Ollama** — local, no key. Auto-discovers models if the daemon
  is up; otherwise listed by name in `config.toml`.
- **OpenAI-compatible** generic provider: declare a `base_url` in
  config and any endpoint speaking the OpenAI Chat shape (vLLM,
  LM Studio, LiteLLM proxy, OpenRouter, Together, DeepSeek, etc.)
  is usable.

The model list is not auto-populated for cloud providers; you list
the models you want to expose in `config.toml`, and they appear in
the in-TUI model picker (`Ctrl-J` by default). This is intentional
— `elia` is a chat client, not a model directory, so it does not
try to enumerate every model your key can reach.

## 4. MCP support

None as of the snapshot date. There is no MCP client, no tool-call
loop, no function-calling surface exposed to the user. `elia` is
strictly a chat client: messages in, messages out, history stored.
If you need MCP, use `oterm` (Ollama + MCP) or any of the
agentic CLIs.

## 5. Sub-agent model

None. One conversation = one model = one linear message thread. No
sub-agents, no tool-call loops, no planner/builder split. The
"agency" boundary is the user pressing Enter; the model never
spawns a new conversation or invokes another model on its own.

## 6. Telemetry stance

Off. `elia` does not phone home, does not collect usage analytics,
does not send crash reports. The only network traffic is to the
provider endpoints you have configured, and only when you send a
message. Conversation history stays in your local SQLite file
forever unless you delete it from the TUI (`Ctrl-Backspace` on a
selected conversation, with confirmation) or wipe the database
file by hand.

## 7. Prompt-cache strategy

Provider-side only. `elia` does not implement client-side prefix
caching or system-prompt deduplication. When a provider supports
prompt caching natively (Anthropic's `cache_control`, OpenAI's
automatic prefix caching on long inputs), `elia` benefits from it
implicitly because it re-sends the same conversation history each
turn — but it does not annotate cache breakpoints itself. For
heavy-context workflows where caching cost matters, this is a
worse fit than `claude-code` or `gemini-cli`.

## 8. Hot keybinds

This is `elia`'s strongest surface; the whole product is keyboard
ergonomics. The most useful ones:

- `Ctrl-N` — new conversation.
- `Ctrl-J` — open the model picker (switch model mid-conversation;
  the next message goes to the newly selected model with the
  existing history as context).
- `/` — focus the conversation-search box; type to filter the
  history sidebar by substring.
- `Ctrl-S` — toggle the sidebar.
- `Ctrl-D` — open the conversation in your `$EDITOR` as Markdown
  for export / copy-paste.
- `Tab` / `Shift-Tab` — move focus between input and message list.
- `Up` / `Down` in the message list — navigate prior turns; press
  Enter on a turn to copy its content.
- `Ctrl-C` — cancel an in-flight generation (does not exit the TUI).
- `Ctrl-Q` — quit.

`elia` is one of two TUIs in the catalog (with `oterm`) where
"learn the keybinds" is a meaningful productivity step. The bindings
are documented in-app via `?`.

## 9. Killer feature, weakness, when to choose

**Killer feature.** Persistent, searchable, multi-provider chat
history in a real TUI, with a single command to launch and zero
config required to get started against OpenAI. The conversation
sidebar with `/` search is the feature you keep coming back for —
"what was that prompt I used last Tuesday for the Rust panic" is a
two-keystroke answer, not a SQL query against `llm`'s database.
Mid-conversation model switching (`Ctrl-J`) is also rare in this
category and genuinely useful for "ask GPT-4o for a draft, then
ask Claude to critique it without leaving the conversation".

**Weakness.** Four to call out:

1. **No tool calls, no MCP, no agency.** Pure chat. The model
   cannot read your files, run commands, or call tools. If you
   need any of that, this is the wrong category of tool — use
   `aider`, `opencode`, `codex`, or any agent.
2. **Heavy Python install** for what is conceptually a chat client.
   ~100 MB of wheels and Python 3.11+ requirement is steep next to
   a 10 MB Go binary like `tgpt` or `mods`.
3. **Manual model list.** You declare every model you want to use
   in `config.toml`. No "show me everything my key can reach". For
   power users who want to compare 12 OpenRouter models, this is
   tedious; for everyone else, it is fine.
4. **TUI-first means weak pipe ergonomics.** `--inline` exists but
   is minimal. For "dump stdin into an LLM and dump stdout to the
   next pipe stage", `mods` and `llm` are purpose-built; `elia` is
   not.

**When to choose.**

- You want **a chat-app experience in the terminal** with persistent
  history and search, against multiple providers.
- You want to **switch models mid-conversation** without manually
  copying history into a new session.
- You **resent leaving the terminal** for a browser-based chat UI
  but you do not need agency or tools — just chat.
- You want **all conversation data on local disk** in a format
  (`SQLite`) you can query, back up, or delete on your own terms.

**When not to choose.**

- You need the model to **edit files or run commands** → `aider`,
  `opencode`, `codex`, `claude-code`, `gptme`, `open-interpreter`.
- You want **Ollama-only with no cloud surface** → [`oterm`](../oterm/).
- You want a **pipe-first CLI** that logs to SQLite → [`llm`](../llm/).
- You want **MCP** → any of the MCP-client entries in the catalog.
- You need a **tiny binary** with no Python runtime → [`tgpt`](../tgpt/),
  [`mods`](../mods/), [`crush`](../crush/).

## Links

- Upstream: <https://github.com/darrenburns/elia>
- PyPI: <https://pypi.org/project/elia-chat/>
- Textual (the TUI framework `elia` is built on):
  <https://github.com/Textualize/textual>
