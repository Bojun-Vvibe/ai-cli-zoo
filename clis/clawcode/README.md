# clawcode

- **Repo:** https://github.com/deepelementlab/clawcode
- **Version:** V0.1.2 (2026-04-15)
- **License:** GPL-3.0 (`LICENSE`)
- **Language:** Python + Rust
- **Install:** see upstream releases / wiki (Python install + Rust components)

## One-line summary

A coding-agent CLI organized around a **14-role virtual R&D team** and an
"experience capsule" learning loop, fronting 200+ models via OpenAI-compatible
APIs.

## What it does

ClawCode is a terminal-native agent that goes beyond the standard
read/write/exec loop with two distinguishing ideas:

- **`/clawteam`** — spins up specialist sub-agents (PM, architect, dev,
  reviewer, QA, etc., up to 14 roles) that collaborate on a task. Closest
  cousin in this catalog is the multi-agent style of crewai / metagpt,
  but here it's wired into a coding CLI rather than a framework.
- **ECAP / TECAP (Experience Capsules)** — a closed-loop learning system
  that writes back distilled "experience" from completed tasks. Capsules
  are portable, feedback-scored, and intended to make repeat work cheaper.
  `/clawteam --deep_loop` triggers automatic capture.
- **`/designteam`** — a separate specialist set focused on producing
  structured design specs (research / IxD / UI / product / visual) instead
  of ad-hoc UI suggestions.

Provider-wise it speaks Anthropic, OpenAI, Gemini, DeepSeek, GLM, Kimi,
Ollama, GitHub Models, and anything OpenAI-compatible.

Three invocation modes:

```
clawcode                          # interactive TUI
clawcode -p "Refactor this API"   # non-interactive prompt
clawcode -p "Summarize" -f json   # structured output
```

## When to choose it

- You want **persistent learning across sessions** without bolting your own
  vector store onto a generic agent.
- You actively want **multi-role orchestration** inside the CLI rather than
  juggling prompt templates yourself.
- You're comfortable with **GPL-3.0** and the obligations it imposes on
  derivative work.

## When NOT to choose it

- Your project cannot ship code derived from GPL-3.0 tooling — pick an
  Apache/MIT alternative.
- You want a small, single-loop agent — ClawCode's surface area
  (multi-role teams, capsule store, dual Python+Rust build) is large by
  design.
- You need a long track record — at v0.1.x the project is early, with
  rapid surface changes likely.
