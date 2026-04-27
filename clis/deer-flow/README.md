# deer-flow

> Snapshot date: 2026-04. Upstream: <https://github.com/bytedance/deer-flow>
> Pinned HEAD: `9dc25987e05e71ae87db0da22a63b4290c5e9747` (default branch
> `main`, last upstream activity 2026-04-27; project ships from `main`,
> no tagged releases at this snapshot).
> License file: [`LICENSE`](https://github.com/bytedance/deer-flow/blob/main/LICENSE)
> (sha `9dc98a4a6b9b549447a4fc314e4cf50529c184c7`, MIT).

`deer-flow` is ByteDance's open-source long-horizon **deep-research
SuperAgent harness**: a planner LLM decomposes a research goal
into a tree of sub-tasks, a coder / writer / researcher
subagent fan-out executes the leaves against a sandboxed
runtime (Python + browser + filesystem) with a shared memory
tier, and a synthesis pass collapses the artefacts back into
a final report or codebase. The CLI / web UI surface
(`deer-flow.tech`) is the catalog's reference for **a
LangGraph-shaped multi-agent research harness with sandboxed
tool-use baked in** — sibling to
[`gpt-researcher`](../gpt-researcher/),
[`local-deep-research`](../local-deep-research/),
[`langgraph`](../langgraph/) (which is the underlying graph
runtime), [`crewai`](../crewai/),
[`autogen`](../ag2/) / [`metagpt`](../metagpt/), and a
peer to general agent stacks like
[`openhands`](../openhands/), [`devon`](../devon/),
[`smol-developer`](../smol-developer/).

## 1. Install footprint

- `git clone https://github.com/bytedance/deer-flow && cd
  deer-flow && uv sync` (Python 3.12+, the project pins
  `uv` for dependency management).
- `pnpm install && pnpm dev` in `web/` boots the React /
  Next.js front-end at `127.0.0.1:3000`; the FastAPI
  back-end runs at `:8000`.
- `make langgraph-dev` boots the LangGraph dev server
  (`langgraph dev`) for graph-level inspection / time-travel.
- LLM providers are configured through
  [`litellm`](../litellm/) — OpenAI, Anthropic, Gemini,
  DeepSeek, Qwen, Doubao, any OpenAI-compatible endpoint
  (Ollama, vLLM, [`sglang`](../sglang/)) all work via the
  same `conf.yaml` shape.
- A self-hosted **sandbox** is required for the coder
  subagent's tool-use; the project bundles a Docker
  Compose flavour (`docker compose up sandbox`) and an
  E2B-shaped remote-sandbox flavour for cloud runs.

## 2. Repo, version, license

- Repo: <https://github.com/bytedance/deer-flow>
- No GitHub releases tagged; project distributes from `main`.
- HEAD pinned at this snapshot:
  `9dc25987e05e71ae87db0da22a63b4290c5e9747`.
- License: MIT. Pinned LICENSE blob:
  [`LICENSE`](https://github.com/bytedance/deer-flow/blob/main/LICENSE)
  (sha `9dc98a4a6b9b549447a4fc314e4cf50529c184c7`).
- Homepage: <https://deerflow.tech>.

## 3. What it actually does

The end-to-end flow for a single research run:

```
deer-flow run "Compare RISC-V's vector extension to AVX-512
                for inference workloads on small LLMs"
```

drives a graph that looks roughly like:

```
coordinator
  ├─ planner             # decomposes the goal into a step tree
  ├─ researcher          # web search + crawl + read
  ├─ coder               # writes Python in a sandbox, runs it,
  │                      # iterates on output
  ├─ reporter / writer   # final synthesis (Markdown report,
  │                      # podcast script, presentation outline)
  ├─ background-investigator  # parallel exploration thread
  └─ memory / RAG        # shared long-term store across runs
```

Each subagent is a typed LangGraph node with its own prompt,
tool whitelist, and recovery policy; the coordinator handles
retry, replan, and human-in-the-loop interrupts. Outputs
land in a per-run workspace directory plus the shared memory
tier, and the web UI streams the graph state in real time so
a human can interrupt, edit a step, and resume.

## 4. MCP support

Yes (client). `deer-flow` mounts external MCP servers as tool
sources — filesystem, web search, browser, GitHub, Slack, any
MCP server in the host's `mcp.json` becomes available to the
coder / researcher subagents alongside the built-in tools. No
first-party MCP server flavour exposes a deer-flow run
outward — that is the host agent's job (e.g. wrap the FastAPI
endpoint in [`fastmcp`](../fastmcp/)).

## 5. Sub-agent model

First-class. The harness *is* a multi-agent shape: each role
(planner, researcher, coder, reporter, background-investigator)
is a typed LangGraph subagent with its own model selection,
prompt, tool list, and budget. Sub-runs are also supported —
a researcher subagent can spawn a nested deer-flow subgraph
for a focused sub-question — with the parent run's memory
tier shared across the tree. Concurrency is the LangGraph
runtime's job; `deer-flow` adds the role definitions, the
coordinator policy, and the human-in-the-loop interrupt
surface.

## 6. Telemetry stance

Off by default in self-host — the project does not phone home;
LLM provider, search provider, and sandbox provider see
their respective traffic per their own posture. LangGraph
runs land in the local LangGraph dev server unless the user
opts into LangSmith. Pair with self-hosted SearXNG +
Ollama + the local Docker sandbox for an air-gapped flavour.

## 7. Token / context strategy

The harness leans on **role specialisation + a shared memory
tier** to keep any single LLM call's context bounded. The
planner sees the goal + plan tree, the researcher sees the
current sub-question + retrieved snippets, the coder sees the
current sub-task + sandbox state, the reporter sees the
collected artefacts. Long-horizon coherence comes from the
memory store (vector + key-value) and from the coordinator's
plan tree — not from stuffing one giant transcript into every
turn. Per-role models are configurable so the planner can be
a frontier model while the coder runs on a cheaper local one.

## 8. Hot keybinds

None for the FastAPI back-end (programmatic surface). The web
front-end is mouse-driven (graph view, chat panel, run
inspector); LangGraph's dev UI provides the time-travel
debugger. Headless invocations go through the CLI / API.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **A LangGraph-shaped, role-specialised
deep-research harness with a real sandbox tier and a real web
UI, shipped as one repo, MIT, with `litellm` swappability for
every model decision.** The category has many "deep research"
demos that are one-shot prompt chains; `deer-flow` is the
first open release that ships the planner + researcher +
coder + reporter split with sandboxed tool-use, shared
memory, human-in-the-loop interrupts, and a graph-level
debugger together — the same shape ByteDance presumably runs
internally, scrubbed and open-sourced.

**Weakness.** It is **a heavy harness for a 30-minute
question** — Docker Compose, FastAPI, Next.js, a sandbox
runtime, a memory tier, LangGraph's dev server. For "fetch
five sources and write a paragraph" the right tool is a
one-shot deep-research script ([`gpt-researcher`](../gpt-researcher/),
[`local-deep-research`](../local-deep-research/),
[`paper-qa`](../paper-qa/)). The runtime also assumes a real
sandbox; running coder steps on the host directly is
explicitly unsupported and a security risk.

**Choose `deer-flow` when** the workload is multi-hour,
multi-tool, multi-source research or codebase generation
that needs role specialisation, sandboxed tool-use, shared
memory across steps, and a human-in-the-loop interrupt
surface. **Choose something else when** the question is
single-pass research ([`gpt-researcher`](../gpt-researcher/),
[`local-deep-research`](../local-deep-research/)), the goal
is interactive coding rather than research
([`aider`](../aider/), [`openhands`](../openhands/),
[`devon`](../devon/)), or the team wants to assemble its
own multi-agent topology from primitives rather than adopt
a pre-built role split ([`langgraph`](../langgraph/),
[`crewai`](../crewai/), [`autogen`](../ag2/),
[`metagpt`](../metagpt/)).
