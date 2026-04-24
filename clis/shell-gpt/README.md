# shell-gpt

> Snapshot date: 2026-04. Upstream: <https://github.com/TheR1D/shell_gpt>
> Binary name: `sgpt`

`shell-gpt` (invoked as `sgpt`) is a **shell-command-generation
primitive**. The default mode of every other "pipe-style" entry in this
catalog is "give the model some text, get text back." `sgpt`'s default
mode is "give the model a description of a task, get *a shell command*
back, then optionally execute it." That sounds like a small framing
difference; in practice it is a different tool.

Three flags define the surface:

- `sgpt --shell "find every PDF modified this week"` — generates a
  shell command, prints it, asks `[E]xecute, [D]escribe, [A]bort`.
- `sgpt --code "regex to validate ipv6"` — generates code only, no
  prose, no fences, suitable for piping straight into a file.
- `sgpt --chat <name> "..."` / `sgpt --repl <name>` — multi-turn
  conversations stored under `~/.config/shell_gpt/chat_cache/`.

There is also a richer **function-calling** mode (`sgpt --functions`)
that lets the model call user-defined Python functions, but the tool's
identity sits firmly in the first three flags. It is the smallest
sensible answer to "I keep googling shell incantations; can the LLM
just write them for me?"

## 1. Install footprint

- `pipx install shell-gpt` (recommended), `pip install shell-gpt`,
  `brew install shell-gpt`, or run via the published Docker image.
- Pure Python, ~25 MB venv. Dependencies are deliberately small:
  `openai`, `typer`, `rich`, `distro`, `instructor`. No model SDKs
  beyond the OpenAI client (other providers go through
  OpenAI-compatible endpoints).
- Config at `~/.config/shell_gpt/.sgptrc` (an INI-ish key/value file).
  Roles, chat caches, and function definitions live next to it under
  `roles/`, `chat_cache/`, `functions/`.
- Optional shell integration: `sgpt --install-integration` adds a
  `Ctrl-L` binding to bash/zsh that converts the current command line
  buffer into a `sgpt --shell` call. This is the single most useful
  ergonomic in the tool.

## 2. License

MIT.

## 3. Models supported

Officially **OpenAI** out of the box. The configuration accepts an
`API_BASE_URL` and an arbitrary `DEFAULT_MODEL`, so anything that
implements an OpenAI-compatible chat-completions endpoint works:

- **OpenAI** — GPT-4.1, GPT-5, o-series (default).
- **Anthropic** — via a proxy (LiteLLM, OpenRouter) speaking the
  OpenAI dialect. Direct Anthropic SDK is not wired in.
- **Local** — Ollama (`API_BASE_URL=http://localhost:11434/v1`),
  LM Studio, llama.cpp server, vLLM. Works well; the
  shell-command-generation prompt is short enough that 7B–13B local
  models give usable answers for common Unix one-liners.
- **Azure OpenAI** — set `API_BASE_URL` to the Azure endpoint and
  prefix the model with `azure/`-style routing if you proxy through
  LiteLLM, otherwise use the Azure-compatible flag set.

Model can be overridden per-call via `--model` or per-role via the
role file's `model:` key.

## 4. MCP support

None. No client, no server, no plugin slot. `sgpt`'s extension surface
is its **functions** mechanism: drop a Python file in
`~/.config/shell_gpt/functions/`, decorate a function with `@function`,
restart the next call, and the model can invoke it with structured
arguments. This is OpenAI-function-calling, not MCP, and the two are
not interoperable. If you need to consume MCP servers, pick an
agent-class entry.

## 5. Sub-agent model

None. Each call is one OpenAI request (or N requests if functions are
called and the model loops). Chat sessions are flat: user / assistant
turns serialised as JSON in `chat_cache/`. There is no planner /
builder split, no Task tool, no recursive call.

The closest thing to "composition" is the **role** system. A role is
a YAML file with a system prompt, a model, and an `expecting:` field
(`shell` / `code` / `default`). You define roles for "translate to
SQL", "explain this regex", "sysadmin Mac-flavoured", etc., and call
them with `--role <name>`. Roles compose with `--chat`, so you can
have a long-running "sysadmin" conversation that always interprets
your prompts as shell-command requests.

## 6. Telemetry stance

**Off, with no opt-in.** `sgpt` itself sends nothing. The OpenAI (or
configured-provider) endpoint sees prompts; that is unavoidable. The
chat cache and function definitions never leave disk.

## 7. Prompt-cache strategy

None at the framework level. Each call is stateless from the
provider's perspective unless you are inside a `--chat` session, in
which case the prior turns are re-sent each time. There is no
attempt to use Anthropic-style prompt caching (which is moot anyway
because the default provider is OpenAI and Anthropic's cache headers
go through the LiteLLM-style proxy path the user configures, not the
core code).

The system prompt for the shell mode is short and stable, which means
provider-side automatic prefix caching (OpenAI's, when enabled on the
account) does kick in for free across many calls.

## 8. Hot keybinds

There is no TUI. The relevant ergonomics are:

- `Ctrl-L` (after `--install-integration`) — convert current shell
  buffer to a `sgpt --shell` call. Most-used feature; type the natural
  language directly on the prompt, hit `Ctrl-L`, get the command,
  press Enter to run.
- `[E]xecute, [D]escribe, [A]bort` — the three-way prompt that
  follows every `--shell` generation. `D` re-asks the model to
  explain the command in prose; `E` runs it via your `$SHELL`.
- `sgpt --repl temp` — disposable REPL session named `temp`.
- `sgpt --repl temp --shell` — REPL that returns shell commands
  every turn (great for an iterative "fix that, now also handle
  symlinks, now print null-delimited" loop).
- `sgpt --list-chats` / `sgpt --show-chat <name>` — inspect saved
  sessions.

## 9. Killer feature, weakness, when to choose

**Killer feature.** `Ctrl-L` shell integration plus the
[E]xecute/[D]escribe/[A]bort prompt. It collapses the
"google → stack overflow → copy → re-edit for your filenames →
paste → run → re-google because flags differ on macOS" loop into a
single keystroke and a 3-key decision. Nothing else in the catalog
targets exactly this workflow at exactly this size — `mods` is more
general-purpose, `llm` is more concerned with logging, and the
agent-class entries are far too heavy for "what's the tar flag again."

**Weakness.** Three:

1. **OpenAI-shaped.** The defaults, the docs, the function-calling
   surface, and the proxy story all assume an OpenAI-compatible
   endpoint. Routing to Anthropic / Gemini natively requires an
   external proxy (LiteLLM, OpenRouter), and the `--functions`
   surface only works on providers that implement OpenAI's
   `tools=` schema.
2. **No project awareness.** `sgpt` does not look at your repo, your
   shell history, your environment beyond `$SHELL`/`$OS`, or any
   open files. You re-type any context that matters every time. For
   "fix this failing test in this repo", use anything else.
3. **Execute-by-default risk.** The `[E]xecute` option runs the
   generated command in your live shell with your privileges. The
   prompt is one keystroke. There is no dry-run sandbox. Treat it
   like any other clipboard-paste-and-run pattern: read before `E`.

**When to choose.**

- You **forget shell flags constantly** and want the LLM to generate
  the exact `find -newer` / `rsync --info=progress2` / `awk -F`
  invocation, with the option to run it inline.
- You want a **stateless code-snippet generator** for pipelines:
  `sgpt --code "python regex for E.164" >> validators.py` is the
  intended idiom.
- You want a **role-driven prompt library** that lives in
  `~/.config/shell_gpt/roles/` and can be invoked from any
  directory.
- You want the **smallest possible footprint** for "LLM in my
  terminal" — `pipx install shell-gpt` and a single `OPENAI_API_KEY`
  is the entire setup.
- You run a **local model** and want a fast, opinionated wrapper
  that asks for shell commands by default rather than chat.

**When not to choose.**

- You want anything to **edit your project files**. Use `aider`,
  `opencode`, `codex`, `claude-code`, or `cline`.
- You want a **queryable, durable log** of every call. Use
  [`llm`](../llm/) — its SQLite log is exactly that, and `sgpt`'s
  chat cache is per-session JSON, not searchable.
- You want **MCP** integration. `sgpt` does not have it.
- You want a **non-OpenAI-shaped function-calling** experience.
  `sgpt`'s `--functions` mode tracks OpenAI's schema closely; pick
  an agent CLI with native multi-provider tool routing instead.
- You want **multi-turn agent loops** with tool calls and execution
  feedback over many turns. That is `open-interpreter`'s job, not
  this one.
