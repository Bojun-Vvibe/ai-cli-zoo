# sidekick-cli

- **Repo:** https://github.com/geekforbrains/sidekick-cli
- **Version:** v0.5.1 (2025-05-23)
- **License:** MIT (`LICENSE`)
- **Language:** Python
- **Install:** `pip install sidekick-cli`

## One-line summary

A small, BYO-key agentic coding CLI for Anthropic / OpenAI / Gemini with MCP
support and JIT system-prompt injection.

## What it does

Sidekick is a terminal agent loop in the same family as the better-known
coding CLIs. You point it at a project, it reads/edits files, runs commands,
and iterates against a model you choose. Notable design choices:

- **No vendor lock-in.** Pick your model per session; switch mid-session.
- **JIT system prompt.** It re-injects the system prompt at strategic points
  rather than hoping the model remembers it after 80k tokens — a pragmatic
  hack against context decay.
- **MCP client.** Configure servers in `~/.config/sidekick.json` like most
  modern agent CLIs.
- **Cost / token tracking** per session, surfaced in the TUI.
- **`/yolo` toggle** to skip per-tool confirmations once you trust the run.

It is explicitly self-described as beta. The codebase is small enough to read
in an afternoon, which is most of the appeal.

## When to choose it

- You want a **hackable Python coding agent** you can fork without drowning
  in framework code.
- You want **JIT prompt injection** and per-session model switching without
  writing your own loop.
- You want something **smaller and more transparent** than the flagship
  agent CLIs already in this catalog.

## When NOT to choose it

- You need production stability — it is in active beta with a self-admitted
  thin test suite.
- You need sub-agents, planner/executor split, or sandboxed exec — Sidekick
  is a single-loop agent without those.
- You want a polished TUI to match the Go-based agents — the UI is
  functional, not a showpiece.
