# langgraph

> Snapshot date: 2026-04. Upstream: <https://github.com/langchain-ai/langgraph>

**Stateful, graph-shaped runtime for long-running agents.** LangGraph
models an agent as a typed `StateGraph` of nodes (LLM calls, tools,
routers, human-in-the-loop checkpoints) sharing a typed state object;
the runtime persists every step to a `Checkpointer` (in-memory,
SQLite, Postgres, Redis), so a graph that crashed at step 17 of 30
resumes from step 17 against the same state. The `langgraph` CLI
(packaged separately as `langgraph-cli`) scaffolds projects, runs the
local dev server with hot reload, and ships graphs to LangGraph
Platform.

## Repo + version + license

- Repo: <https://github.com/langchain-ai/langgraph>
- Latest core release: **`langgraph` 1.1.9** (PyPI, 2026-04)
- CLI release: **`langgraph-cli` 0.4.24** (PyPI, 2026-04)
- HEAD on `main`: `53a9806`
- License: **MIT** —
  <https://github.com/langchain-ai/langgraph/blob/main/LICENSE>
- License path in repo: `LICENSE`
- Default branch: `main`
- Language: Python (TS port `@langchain/langgraph` parallel)

## Install

```bash
pip install -U langgraph langgraph-cli langgraph-prebuilt
# project scaffold
langgraph new my-agent --template react-agent-python
langgraph dev    # local server + Studio UI on :2024
langgraph build  # docker image
langgraph up     # docker compose with Postgres + Redis
```

## Niche

Graph-shaped agent orchestration with **first-class durability**.
Where most frameworks model an agent as a chat loop with retries,
LangGraph models it as a directed graph whose every edge traversal is
a checkpoint write, so resumption, human-in-the-loop pauses,
time-travel debugging ("what if we retried node 12 with a different
prompt?"), and parallel node execution are runtime primitives, not
bolt-ons. Pairs with [langfuse](../langfuse/) / [opik](../opik/) for
trace inspection.

## Why it matters

- The graph + state separation makes long-horizon multi-step agents
  (research, multi-day support tickets, cron-driven pipelines) actually
  resumable across process restarts — `Checkpointer(Postgres)` plus a
  `thread_id` is the entire durability story.
- `interrupt()` makes human-in-the-loop a graph node, not a hack: the
  graph pauses, persists, returns control to the caller, and resumes
  on `Command(resume=...)` — the cleanest "approve this tool call"
  pattern in this catalog after [humanlayer](../humanlayer/).
- `langgraph dev` + Studio give you a live visual graph debugger
  (state diff per step, fork-and-rerun, prompt edit) without leaving
  the local repo, and the same graph deploys unchanged to LangGraph
  Platform or any container runtime.
- Provider-mixing is per-node: planner on Claude Opus, worker tools on
  Gemini Flash, structured-extraction node on a local Qwen via Ollama,
  declared inline as `model=...` per node.
