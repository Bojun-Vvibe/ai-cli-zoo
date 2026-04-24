# open-interpreter

> Snapshot date: 2026-04. Upstream: <https://github.com/OpenInterpreter/open-interpreter>

`open-interpreter` (the `interpreter` command, often abbreviated OI) is a
**code-execution REPL** for language models. You type natural language at
a chat prompt, the model writes code (Python, shell, JavaScript,
AppleScript, R, PowerShell, HTML), the tool runs it locally, the model
sees the output, and the loop continues until the task is done.

That single sentence hides what makes OI ecologically distinct from the
rest of the catalog. It is **not an editor** — it does not target a
codebase, it does not produce diffs, it does not commit. It is **not a
sandboxed agent** — the code runs against your real machine, your real
files, your real shell, with the same permissions you have.  And it is
**not a unix primitive** — every interaction is a multi-turn REPL with
output-feedback, not a one-shot pipe.

What it *is* is the canonical "give the model a Python and let it just
do the thing" surface. Renaming a stack of files by EXIF date,
extracting tables from a PDF, plotting a CSV, scraping a website,
controlling iTunes via AppleScript, exfil-pulling a wifi-passwords list
from your own Keychain — all are one-prompt operations because the
model can write *and run* the code.

This power comes with the obvious foot-gun, and OI is unusually upfront
about it: every code block prompts for `y/n` approval by default, and
the README does not pretend the safety story is solved.

## 1. Install footprint

- `pip install open-interpreter` (most common), `pipx install
  open-interpreter`, or via the official `setup.sh`.
- Pure Python, ~80 MB venv after install (heavy because it pulls
  `litellm`, `tiktoken`, `rich`, `inquirer`, plus optional ML deps for
  the local-model path).
- macOS / Linux / Windows / WSL all supported. The OS-control features
  (AppleScript on macOS, `pyautogui`/`pynput` for screen/mouse) are
  opt-in extras (`pip install open-interpreter[os]`).
- Config lives at `~/Library/Application Support/Open Interpreter/`
  (macOS) or the XDG equivalent. Profiles are YAML/Python files in
  `profiles/`. There is no project-local state by default — start it in
  any directory and it will treat that directory as cwd.

## 2. License

AGPL-3.0. This is the strongest copyleft in the catalog and matters if
you want to embed OI inside a product: distributing a service that
exposes OI's interface arguably triggers AGPL's network clause. Use it
as a personal CLI freely; before bundling it into a SaaS, read the
license.

## 3. Models supported

Anything `litellm` can route. The official defaults:

- **OpenAI** — GPT-4.1, GPT-5, o-series (default if `OPENAI_API_KEY`
  is set).
- **Anthropic** — Claude 3.x / 4.x via `--model claude-...`.
- **Gemini** — `gemini/...` model strings.
- **Local** — `interpreter --local` walks you through Ollama / LM
  Studio / llama.cpp setup interactively. The Local Model wizard is
  unusually well-built; it auto-detects running Ollama and lists
  available tags.
- **Anything OpenAI-compatible** — set `--api_base` and pass any model
  name your endpoint serves.

Switching mid-session is supported: `%model claude-sonnet-4` reroutes
the next turn.

## 4. MCP support

No native MCP client and no server. There is a community discussion
about adding one but as of the snapshot date it is not in core. If your
workflow centres on consuming MCP servers, this is the wrong tool.

OI's tool surface is fixed and small: `execute(code, language)` is
essentially the only tool. Everything else (file IO, web fetch, OS
control) is something the model writes Python for, not something OI
exposes as a structured tool call.

## 5. Sub-agent model

None in the classic sense. There is no `Task` tool, no planner/builder
split, no recipe engine. The "agency" is the loop:

```
user → model → code → exec → output → model → ...
```

Profiles can pre-load a system message, model choice, custom tools,
and auto-run policy, which lets you spin up purpose-specific OIs
(`interpreter --profile data-cleaner.yaml`), but each profile is still
a single conversation with a single model. No fan-out.

## 6. Telemetry stance

**Off by default, opt-in.** Telemetry is gated by an explicit
`--telemetry` flag and a config setting; nothing is sent on a vanilla
install. The project uses Posthog when telemetry is enabled and the
events captured are documented in the source.

The model providers obviously see your prompts (and the contents of
files you ask the model to read). OI itself does not phone home in the
default configuration.

## 7. Prompt-cache strategy

None at the framework level beyond what the underlying provider does
automatically. Anthropic prompt caching can be enabled by passing
provider-specific kwargs through `litellm`, but OI does not insert
cache markers for you and does not chunk the system message into a
cache-friendly prefix.

OI's context-management story is also weak: long sessions truncate
older turns with a simple FIFO once the model's window fills, with
no summarisation or scratchpad. For long-running data tasks, expect
to start fresh sessions per logical unit of work.

## 8. Hot keybinds

REPL with `rich` styling. Useful in-session commands:

- `%verbose` — show the raw model messages and tool call JSON
- `%reset` — wipe conversation, keep config
- `%save_message <path>` / `%load_message <path>` — persist / replay
  a conversation as JSON
- `%undo` — drop the last user+assistant pair
- `%tokens` — show running token + cost estimate
- `%model <name>` — switch model mid-conversation
- `%profile <name>` — switch profile mid-conversation
- `%help` — list all magic commands

CLI-side useful flags:

- `interpreter -y` — auto-approve every code block (DANGEROUS, see
  "When not to choose")
- `interpreter --os` — enable screen/mouse control extras
- `interpreter --vision` — turn on vision (GPT-4o / Claude image
  inputs); pasted screenshots become tool input
- `interpreter --conversations` — list saved conversations and
  resume one

## 9. Killer feature, weakness, when to choose

**Killer feature.** The "any task you'd otherwise solve with a quick
Python script" surface. OI compresses the time from problem statement
to working one-off automation to the time it takes the model to write
30 lines of Python and see the traceback. For exploratory data work
in a notebook-like flow without a notebook, and for ad-hoc OS chores
that would otherwise require remembering twelve `find -exec` flags,
nothing else in the catalog touches it.

**Weakness.** Three big ones, in order of severity:

1. **Safety surface.** Code runs on your real machine. The default
   `y/n` approval is your only line of defence; `-y` removes even
   that. There is no sandbox, no allowlist, no syscall filter. A
   prompt-injected web page returned by a tool-written `requests.get`
   can talk the model into running `rm -rf` and you will see the
   prompt go by once before it executes. Treat OI like sudo: only run
   it when you're paying attention.
2. **Not a code editor.** OI will *write a script that edits a file*
   but it will not give you a diff against your repo, will not stage
   commits, and has no notion of "the project I am in". For
   refactoring an existing codebase, every other entry in the catalog
   is better.
3. **Context handling.** Long sessions hit the window and start
   forgetting silently. There is no summarisation pass and no
   scratchpad-on-disk pattern; you are expected to start fresh
   sessions per task.

**When to choose.**

- You have a **one-off computational task** that you would otherwise
  solve with a throwaway Python script — analyse a CSV, rename a
  thousand files, OCR a folder of PDFs, convert audio formats.
- You want **REPL-driven OS control** — "open every PDF in
  ~/Downloads from this week and rename it to its first-line title"
  is a one-prompt task here and a 20-minute exercise anywhere else.
- You want a **local-model code-executor** without writing harness
  code — `interpreter --local` is the fastest way to a working Ollama
  + code-exec loop.
- You want to **show non-engineers what a tool-using LLM actually
  feels like.** The REPL + auto-run-with-approval loop is the most
  legible demo of agency-with-guardrails in the catalog.

**When not to choose.**

- You are editing a real codebase. Use `aider`, `opencode`, `codex`,
  `claude-code`, or `cline`.
- You need a **sandbox**. Use `codex` (Seatbelt/Landlock) or
  `OpenHands` (Docker).
- You need **MCP servers**. Use any of the MCP-client entries.
- You need **headless / CI** execution. OI is REPL-shaped; the
  one-shot mode (`interpreter -i "prompt"`) works but is not the
  ergonomic happy path.
- You require a **permissive license** for embedding. AGPL is a
  hard no for many product surfaces; pick an Apache-2.0 / MIT entry.
