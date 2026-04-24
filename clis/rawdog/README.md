# rawdog

> Snapshot date: 2026-04. Upstream: <https://github.com/AbanteAI/rawdog>
> (canonical redirect: <https://github.com/granawkins/rawdog>)

A Python CLI that **generates a Python script, runs it, sees the
output, and loops** until your task is done — "Recursive
Augmentation With Deterministic Output Generations". Sits between
`open-interpreter` (full code-exec REPL with rich shell tooling) and
`llm-cmd` (one-shot NL → one shell command): rawdog is one-shot
NL → one Python script → execute → feed stdout back → loop, on the
**local interpreter**, in **your cwd**, with **no sandbox**.

## 1. Install footprint

- `pip install rawdog-ai` (PyPI package name differs from the binary).
- Pulls only the OpenAI Python SDK + `litellm` for provider routing.
- Config at `~/.rawdog/config.yaml`. Conversation log at
  `~/.rawdog/log.jsonl`.
- Runs as your user, in your cwd, against your live filesystem — no
  containerisation, no Docker, no chroot.

## 2. Repo + version + license

- Repo: <https://github.com/AbanteAI/rawdog>
  (HTTP 301 → <https://github.com/granawkins/rawdog>)
- Latest release: **v0.1.6**
- License: **Apache-2.0** —
  <https://github.com/granawkins/rawdog/blob/main/LICENSE>
- Default branch: `main`

## 3. Models supported

Anything `litellm` routes — OpenAI (default), Anthropic, Gemini,
Bedrock, Azure, Ollama, any OpenAI-compatible endpoint. Pick with
`rawdog --llm-model gpt-4o` or `model:` in the config file.

## 4. MCP support

None. The single tool the model has is "emit a Python script that
will be `exec`'d in a fresh subprocess and whose stdout becomes the
next turn's user message".

## 5. Sub-agent model

None. One model, one rolling conversation. The agentic loop is
`generate Python → exec → append stdout → repeat`, capped by
`--leash` (require human confirmation per script) or by the model
emitting `CONTINUE = False`.

## 6. Telemetry stance

Off, no opt-in. Egress is exclusively to the configured LLM provider.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **Iteration on real stdout** as the agent loop's
ground truth. Ask `rawdog "summarise the largest 5 files in this
tree"` and it writes a `find … | sort` Python script, runs it, sees
the actual paths, then writes a follow-up script to `head` each one
— all without you reviewing intermediate code if `--leash` is off.
For one-off "bash this tree into shape" tasks, the dev-loop overhead
of `aider` / `opencode` is overkill; rawdog is a single-pip-install
disposable agent.

**Weakness.** **No sandbox.** A bad model output can `rm -rf` your
home directory; `--leash` mitigates but is off by default. No
project memory between invocations beyond the JSONL log. Python-only
— if the right tool is `jq` or `awk`, the model will reach for
`subprocess.run` instead. Repo activity is light; treat as "stable
small tool", not "actively shipping".

**When to choose.**

- You want a **disposable Python-loop agent** for ad-hoc "do this
  in my filesystem" tasks and you accept the no-sandbox risk.
- You already trust the LLM with your shell (you'd run
  `open-interpreter` too) but want something **simpler and
  Python-only**.
- You want the **simplest possible** "see-stdout-and-iterate" loop
  to wrap in a shell alias for one-off data wrangling.

**When not to choose.** You need filesystem isolation
(→ `OpenHands` sandbox, `codex` Seatbelt/Landlock), you want
multi-language tool access (→ `gptme`, `open-interpreter`), or you
want a code-editing agent rather than a script-running agent
(→ `aider`, `opencode`).

## 8. Quick usage

```sh
pip install rawdog-ai
export OPENAI_API_KEY=sk-...

# one-shot
rawdog "count lines of Python in this repo grouped by top-level dir"

# interactive REPL
rawdog
>>> rename every screenshot in ~/Desktop to its EXIF date

# require confirmation per script
rawdog --leash "delete all .DS_Store under here"

# pick a non-default model via litellm
rawdog --llm-model anthropic/claude-3-5-sonnet-latest "..."
```
