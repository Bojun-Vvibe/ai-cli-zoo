# gorilla-cli

> Snapshot date: 2026-04. Upstream: <https://github.com/gorilla-llm/gorilla-cli>
> Binary name: `gorilla-cli` (also installed as `gorilla`)

`gorilla-cli` is the **shell-command suggester** of the catalog,
narrow-scope edition. Its only job is: take an English description
of a task, return a numbered list of *candidate shell commands*
that might do it, and let you pick one with the arrow keys to
execute (or edit before executing). Unlike [`shell-gpt`](../shell-gpt/)
or [`tgpt`](../tgpt/), which return a single command and prompt
[E]xecute / [D]escribe / [A]bort, `gorilla-cli` always returns
**multiple options** and treats picking-among-candidates as the
core interaction.

```
$ gorilla list all PDFs modified in the last week
1. find . -type f -name "*.pdf" -mtime -7
2. fd -e pdf --changed-within 7d
3. find . -name "*.pdf" -newermt "$(date -d '7 days ago' '+%Y-%m-%d')"
> _
```

Use the arrow keys to highlight, Enter to execute, `e` to edit
inline, or `Ctrl+C` to bail. The result is taxonomically distinct
from a single-answer LLM-shell tool: `gorilla-cli` *expects you to
choose*, which makes it well-suited to "I think there are three
ways to do this and I want to see them" rather than "just give me
the rsync flags."

The project comes out of the [Gorilla LLM](https://gorilla.cs.berkeley.edu/)
research line at UC Berkeley — the same group that publishes the
Berkeley Function Calling Leaderboard. The CLI itself is a thin
client; the smart part is a hosted endpoint at
`cli.gorilla-llm.com` that the project maintains for free public
use.

## 1. Install footprint

- Pure Python, single small dependency set (`requests`,
  `prompt-toolkit`, `halo`). Install with `pipx install gorilla-cli`,
  `uv tool install gorilla-cli`, or `pip install --user
  gorilla-cli`.
- One executable on `$PATH` (`gorilla` or `gorilla-cli`). No config
  file required for default usage.
- A small history file at `~/.gorilla-cli-history` records every
  prompt + selected command for later auditing. Inspect or wipe it
  any time.
- First invocation generates an anonymous user ID stored in
  `~/.gorilla_cli_user_id` for the hosted endpoint's rate limiting.

## 2. License

Apache-2.0.

## 3. Models supported

By default, the **hosted Gorilla endpoint** at
`cli.gorilla-llm.com`, which routes to a fine-tuned Gorilla model
specifically trained on shell-command generation across common
Unix toolchains, AWS / `gcloud` / `kubectl`, `git`, `docker`, and
package managers. The endpoint is free to use; the project funds
it via research grants.

You can override the endpoint with `GORILLA_API_URL` to point at a
self-hosted Gorilla model, or at any OpenAI-compatible chat
completion endpoint that returns a numbered list. There is no
in-binary support for picking among "OpenAI vs Anthropic vs
Gemini" — the contract is "an endpoint that returns command
candidates," and BYO endpoint is your escape hatch.

This is taxonomically narrow. `gorilla-cli` will not help you write
prose, summarize a transcript, or chat about a paper. It is
specialized to **shell**.

## 4. MCP support

None. No client, no server, no tool-call surface. The contract is
single-shot HTTP: prompt in, candidate list out, user picks, shell
executes locally.

## 5. Sub-agent model

None. One HTTP request per prompt. No planning, no retry-on-error
loop, no follow-up "did that work?" turn. If the picked command
fails, you re-invoke `gorilla` with a refined description.

The composition primitive is **the user's eyeballs and the up/down
arrows**. The "agent" is the operator at the keyboard.

## 6. Telemetry stance

**Opt-in by inspection, on by default in practice.** Because the
default endpoint is hosted by the project, every prompt + the
selected-command index goes to `cli.gorilla-llm.com` and is logged
for model improvement (the project is open about this in the
README — the data informs Gorilla's continued training).

If that is unacceptable, the only recourse is to set
`GORILLA_API_URL` to a self-hosted endpoint. There is no
"telemetry off" toggle that keeps the hosted endpoint and disables
logging — using the hosted endpoint *is* the telemetry. Treat the
default as "the prompts are public-ish."

The local history file (`~/.gorilla-cli-history`) is purely local
and never uploaded.

## 7. Prompt-cache strategy

None at the CLI level. Each invocation is a fresh HTTP round-trip;
there is no on-disk response cache and no session reuse.
Server-side caching may exist but is not exposed to the client.

## 8. Hot keybinds

`gorilla-cli` is interactive at the moment of selection but not a
TUI. The relevant ergonomics:

- `gorilla <natural language description>` — submit a prompt; the
  rest of the argv is treated as the description, no quoting needed.
- `↑` / `↓` — move the cursor among candidate commands.
- `Enter` — execute the highlighted command in your current shell
  (it is `exec`'d, so it inherits cwd, env, and aliases — modulo
  the usual subprocess rules).
- `e` — open the highlighted command in `$EDITOR` for tweaks before
  execution.
- `Ctrl+C` — bail without executing anything.
- `gorilla --history` — print the local history file (prompts +
  picked commands, timestamped).
- `gorilla -p` / `--print-only` — print candidates and exit; do not
  enter the interactive picker. Useful in pipelines or for shell
  functions that wrap `gorilla` with custom UX.

## 9. Killer feature, weakness, when to choose

**Killer feature.** A **multi-candidate, arrow-key-pickable shell
command suggester** backed by a model fine-tuned specifically on
shell + cloud CLI usage, with **zero API-key setup** for the
default endpoint. The "show me three plausible commands and let
me pick" interaction model is genuinely different from
`shell-gpt`'s single-answer prompt and frequently more useful when
you half-know what you want and want the model to remind you of
the exact flag.

**Weakness.** Three:

1. **Hosted endpoint = sent prompts.** The default UX involves
   shipping every prompt to a Berkeley-run endpoint that logs them
   for training. Convenient, but a non-starter for prompts that
   contain hostnames, paths, or anything proprietary. The
   `GORILLA_API_URL` workaround puts the burden of self-hosting on
   you.
2. **Narrow scope.** Shell commands only. It will not write code,
   summarize a doc, or chat about a paper. For general-purpose
   one-shot prompting use [`mods`](../mods/), [`tgpt`](../tgpt/),
   [`shell-gpt`](../shell-gpt/), or [`llm`](../llm/).
3. **No correction loop.** If the chosen command fails, the tool
   does not see the error and offer a fix. You re-prompt. Compare
   with [`open-interpreter`](../open-interpreter/) or
   [`gptme`](../gptme/), where the model sees the failure and
   iterates.

**When to choose.**

- You want **multiple plausible commands to choose from**, not a
  single answer you have to trust.
- Your day is full of "what's the `find` invocation for X" / "what
  `kubectl` flag does Y" moments and the friction of opening a
  browser to ChatGPT is the actual cost.
- You are willing to **send prompts to the hosted Gorilla
  endpoint** in exchange for zero-config installation, or you are
  willing to self-host the Gorilla model behind `GORILLA_API_URL`.
- You want a **separate tool for shell suggestions** so it does
  not get tangled with your "real" LLM workflows in
  [`opencode`](../opencode/) / [`codex`](../codex/) /
  [`claude-code`](../claude-code/).

**When not to choose.**

- You need **prompt privacy** by default. Use
  [`shell-gpt`](../shell-gpt/) with your own OpenAI-compatible key
  (including a local Ollama endpoint), or [`tgpt`](../tgpt/) with a
  self-hosted backend.
- You want a **single best answer** with E/D/A keys. Use
  [`shell-gpt`](../shell-gpt/) (`Ctrl-L` shell integration) or
  [`tgpt -s`](../tgpt/).
- You want the tool to **see command output and self-correct**.
  Use [`open-interpreter`](../open-interpreter/) or
  [`gptme`](../gptme/) — `gorilla-cli` is one-shot per prompt.
- You want **general-purpose prompting** (prose, code review,
  summarization). `gorilla-cli` is shell-only by training; reach
  for [`llm`](../llm/), [`mods`](../mods/), or [`fabric`](../fabric/).
