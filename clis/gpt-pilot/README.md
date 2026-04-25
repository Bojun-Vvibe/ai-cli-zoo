# gpt-pilot

> Snapshot date: 2026-04. Upstream: <https://github.com/Pythagora-io/gpt-pilot>
> Pinned version: **v0.2.13**. License file: [`LICENSE`](https://github.com/Pythagora-io/gpt-pilot/blob/main/LICENSE).

Spec-driven, multi-agent app generator. You describe an app in plain English,
gpt-pilot interviews you to fill the gaps, then a chain of role-named agents
(Product Owner → Architect → Tech Lead → Developer → Code Monkey → Reviewer)
scaffolds the project, writes files, runs the test loop, and asks you to
verify each step in the browser.

## 1. Install footprint

- Python 3.9+ project; clone the repo and `pip install -r requirements.txt`,
  or use the bundled `Dockerfile` to avoid host Python drift.
- Persists per-project state to a local PostgreSQL (default) or SQLite DB —
  the agent re-reads "what we already built" from the DB on every run, so
  the same project resumes across sessions.
- Optional VS Code extension wraps the same backend; the CLI entry point is
  `python main.py` from the repo root.

## 2. License

**FSL-1.1-MIT** (Functional Source License, version 1.1, MIT Future License).
Free for any non-competing use; converts to plain MIT two years after each
release. License text lives at [`LICENSE`](https://github.com/Pythagora-io/gpt-pilot/blob/main/LICENSE)
in the repo root. Same family as `crush`, so the practical posture is "treat
as permissive unless you're building a competitor".

## 3. Models supported

- OpenAI GPT-4 / GPT-4o / GPT-4.1 family (default and best-tested path).
- Anthropic Claude (Sonnet / Opus) via the same provider config block.
- Any OpenAI-compatible endpoint (Azure OpenAI, local vLLM / Ollama,
  OpenRouter) by overriding `OPENAI_ENDPOINT` and `MODEL_NAME` in `.env`.
- Provider is set once per project; sub-agents do **not** swap models per role
  out of the box — you'd patch `core/llm/` to do that.

## 4. MCP support

No. gpt-pilot predates MCP and uses its own JSON-schema "agent commands"
protocol between the convo manager and each role agent.

## 5. Sub-agent model

Hard-coded role pipeline: Product Owner clarifies the spec, Architect picks
the stack, Tech Lead breaks the spec into epics + tasks, Developer drafts
each task, Code Monkey writes the actual file diffs, Reviewer runs / asks
you to verify. Agents communicate by appending to a structured "project state"
that lives in the DB, not by free-form chat.

## 6. Telemetry stance

**Opt-in.** A telemetry block in `config.json` is commented out by default;
when enabled it pings Pythagora's analytics endpoint with anonymized event
counts (no source code). Egress at runtime is whatever LLM provider you
configured plus your own DB.

## 7. Prompt-cache strategy

No first-class prompt-cache integration. Each agent rebuilds its prompt from
the project-state DB every turn, so the cache hit rate depends entirely on
how stable that state is. If you point it at Anthropic, you get the
provider-side prompt cache for the system block "for free".

## 8. Hot keybinds (TUI / REPL)

CLI is line-driven, not a TUI. The interaction loop is: agent prints a
question or proposes a diff → you type a free-text answer or `continue` /
`change <thing>`. The richer UX lives in the VS Code extension; the CLI is
deliberately bare so it scripts cleanly.

## 9. Killer feature, weakness, when to choose

**Killer feature.** Stateful, role-decomposed app build. The DB-backed
project state means you can kill the process, come back tomorrow, and the
Tech Lead picks up at the exact next task — there is no "lost context"
failure mode that plagues single-loop agents on multi-day greenfield work.

**Weakness.** Greenfield-shaped. Pointing it at an existing 200k-LOC repo
and saying "add a feature" is not what the role pipeline was designed for;
the Architect step assumes it's choosing a stack, not adopting one. For
brownfield work, [`aider`](../aider/) or [`opencode`](../opencode/) fit
better.

**When to choose.** You're building a new web app from a paragraph of spec,
you want the agent to ask clarifying questions instead of guessing, and you
care that the work survives a laptop reboot. Don't choose it for one-off
edits to an existing codebase.
