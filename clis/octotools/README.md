# octotools

> Snapshot date: 2026-04. Upstream: <https://github.com/octotools/octotools>

A training-free agentic reasoning framework from a Stanford-led
academic project that builds **task-specific tool plans** out of
a registry of self-describing tool *cards*. You install the
library, point it at a multi-step problem ("solve this
high-school physics word problem with diagrams", "answer this
medical question using the literature", "extract the table from
this PDF and compute the totals"), and the planner picks a
small ordered subset of tool cards from the registry to invoke
in sequence — generating reasoning, calling Python / image
classifiers / web search / domain solvers, and aggregating into
a final answer.

It is the catalog's reference for **academically-published
agentic-reasoning-with-tools framework targeting structured
domains** (math, science, medical) rather than coding or chat.

## 1. Install footprint

- `pip install octotools` (Python 3.10+).
- Pulls `openai`, `anthropic`, `google-generativeai`, `pillow`,
  `requests`, plus heavier scientific extras (`scipy`, `numpy`,
  `sympy`) on demand. ~250 MB venv depending on which tool
  cards you enable.
- Models: OpenAI (default), Anthropic Claude, Google Gemini,
  Together, vLLM (any OpenAI-compatible) — set via the
  `--llm_engine_name` flag or `OPENAI_API_KEY` /
  `ANTHROPIC_API_KEY` / `GOOGLE_API_KEY` env vars.
- CLI entrypoint: `octotools --task "your question" --tools
  Python_Code_Generator_Tool,Wikipedia_Knowledge_Searcher_Tool`.
- Programmatic: `from octotools.solver import Solver` then
  `solver.solve(question)`.

## 2. Repo, version, license

- Repo: <https://github.com/octotools/octotools>
- Version checked: **1.0** (latest on PyPI as of 2026-04). The
  GitHub repo has no tagged releases; pin commits.
- HEAD pinned at this snapshot:
  `1bae2d3e0854b4878bcd192cf20b668195f4d5f6`.
- License: MIT. License file at
  [`LICENSE`](https://github.com/octotools/octotools/blob/main/LICENSE).
- Paper: "OctoTools: An Agentic Framework with Extensible Tools
  for Complex Reasoning" (Lu et al., 2025).

## 3. What it actually does

Three roles, all driven by the same LLM:

1. **Planner.** Reads the question + the metadata of every
   enabled tool card, emits a high-level plan: which tools to
   call, in which order, with what intermediate sub-goals.
2. **Executor.** For each step in the plan, generates the
   concrete arguments for the tool call, invokes the tool
   (Python interpreter / vision model / arXiv search /
   Wikipedia lookup / URL fetcher / Pubmed query / image
   captioner / generalist solver), captures the result.
3. **Verifier / Aggregator.** After execution, re-reads the full
   trajectory and either declares the answer, asks the planner
   to revise, or invokes one more tool. Bounded by
   `max_steps`.

Tool cards are Python classes that declare their capabilities,
input schema, and example usage in plain text — the planner reads
the cards in-context and decides which apply. Adding a new tool =
write a new card class. The shipped registry includes ~16 tools
spanning math (`Python_Code_Generator_Tool`,
`Generalist_Solution_Generator_Tool`), retrieval
(`Wikipedia_Knowledge_Searcher_Tool`,
`ArXiv_Paper_Searcher_Tool`,
`URL_Text_Extractor_Tool`,
`Pubmed_Search_Tool`,
`Google_Search_Tool`), vision
(`Image_Captioner_Tool`,
`Object_Detector_Tool`, `Text_Detector_Tool`), and domain solvers
(`Relevant_Patch_Zoomer_Tool`, `Path_Generator_Tool`).

## 4. MCP support

None as of v1.0. Tools are first-party Python tool-card classes,
not MCP servers. Adding MCP would mean writing an MCP-client tool
card; not in the box.

## 5. Sub-agent model

Single-agent, three-role (planner / executor / verifier) loop.
No spawned sub-agents, no parallel tool calls — the plan is
sequential by construction (the paper argues sequential planning
is what makes the trajectories *auditable* for science / medical
QA). Each step's output is logged to disk for replay.

## 6. Telemetry stance

Off in the OSS library (no analytics in `octotools` itself).
Egress = the configured LLM provider + whichever tool cards you
enable (web search → Google / DuckDuckGo / Wikipedia, arXiv
fetch → arxiv.org, PubMed → ncbi.nlm.nih.gov, URL extractor →
arbitrary URLs the planner picks). Trajectories are written to
the local `logs/` directory by default.

## 7. Token / context strategy

The planner sees `question + every enabled tool's card text`
each time it plans. With ~16 cards this fits comfortably in a
32k-context model; if you author large registries (50+ cards),
selectively whitelist with `--enabled_tools` to keep the planner
prompt small. Trajectories accumulate per-step text + per-step
tool output; long medical / multi-figure tasks can run into
20k–40k tokens of context by step 8. Set `--max_steps` (default
~10) defensively.

## 8. Hot keybinds

None — `octotools` is a CLI / library, not a TUI. The interactive
surface is shell + log files in `logs/`.

## 9. Killer feature, weakness, when to choose

**Killer feature.** Agentic-reasoning framework with **published
benchmarks across 16 diverse tasks** (MathVista, MMLU-Pro,
Game of 24, MedQA, PathVQA, SciFiBench, …) showing measurable
lifts over GPT-4o + CoT and over LangChain / AutoGen baselines —
plus a **tool-card abstraction explicitly designed for adding
domain solvers without prompt-engineering the planner**. Every
other "general agent" entry in the catalog
([`smolagents`](../smolagents/),
[`crewai`](../crewai/),
[`autogen`](../ag2/),
[`agno`](../agno/)) is built for *coding* or *business*
workflows; OctoTools is built for *structured-domain reasoning
with traceable, replay-able trajectories*.

**Weakness.** Academic-style code: `1.0` on PyPI but no GitHub
tags, no semver discipline yet, README assumes you read the
paper. Single-threaded sequential planning is by design but
limits throughput. The shipped tool registry leans
math/science/medical — for "build me a SaaS agent", you would
re-implement most of the work that
[`agno`](../agno/) /
[`pydantic-ai`](../pydantic-ai/) /
[`openai-agents-python`](../openai-agents-python/) ship
out-of-the-box.

**When to choose.**
- You are doing **scientific / medical / math QA** with mixed
  inputs (text + figure + diagram + table) and want a framework
  whose primitives match that domain.
- You want **auditable trajectories** (every plan, every tool
  call, every output, on disk) for paper-writing or
  domain-expert review.
- You have one or two domain solvers (a custom physics engine, a
  pathology classifier) and want to plug them in *as tool cards*,
  not as bespoke chains.

**When to skip.**
- You want to **build a chat / coding agent** → use
  [`smolagents`](../smolagents/) /
  [`pydantic-ai`](../pydantic-ai/) /
  [`agno`](../agno/) — broader provider matrix, MCP support,
  serving primitives.
- You need **parallel tool execution / multi-agent fan-out** →
  [`crewai`](../crewai/) /
  [`langgraph`](../langgraph/) /
  [`agno`](../agno/) (Team mode).
- You want **production HTTP serving** out of the box → OctoTools
  has none; wrap it in your own FastAPI.
- You want **MCP tool ecosystem** → use any catalog entry tagged
  with MCP-client support; OctoTools has none.

## 10. Compared to neighbors in the catalog

| Tool | Domain focus | Tool surface | Trajectory replay | Parallel agents |
|------|--------------|--------------|-------------------|-----------------|
| octotools | Science / math / medical multi-modal QA | Python tool cards (16 shipped) | Yes (per-step logs) | No (sequential by design) |
| [smolagents](../smolagents/) | General coding / web | Python `Tool` + MCP | Via OTel | Yes (managed agents) |
| [pydantic-ai](../pydantic-ai/) | General typed agents | Function tools + MCP | Logfire / OTel | Yes (graphs) |
| [agno](../agno/) | General + production serving | Functions + MCP | OTel + AgentOS UI | Yes (Team / Workflow) |
| [crewai](../crewai/) | Role-based business workflows | Functions + crewai-tools | Built-in | Yes (Crew) |
| [langgraph](../langgraph/) | Graph-controlled agents | LangChain tools + MCP | LangSmith | Yes (graph) |

Decision shortcut:

- "Reproduce a paper-style multi-modal reasoning benchmark" →
  `octotools`.
- "Add an LLM agent to a Python app, structured outputs, MCP" →
  [`pydantic-ai`](../pydantic-ai/).
- "Code-as-action general agent with sandbox menu" →
  [`smolagents`](../smolagents/).
- "Production agent with built-in HTTP / UI" → [`agno`](../agno/).
- "Role-based crew" → [`crewai`](../crewai/).
