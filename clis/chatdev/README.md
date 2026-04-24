# chatdev

> Snapshot date: 2026-04. Upstream: <https://github.com/OpenBMB/ChatDev>.
> Pinned version: **v2.2.0** (2026-03-23).
> License: **Apache-2.0** — `LICENSE` at the repo root
> (<https://github.com/OpenBMB/ChatDev/blob/main/LICENSE>).

A "virtual software company" multi-agent CLI: you type one product
description, and a roster of role-played agents (CEO, CPO, CTO,
Programmer, Reviewer, Tester, Designer) talks through requirements,
design, code, test, and docs in sequence — emitting a runnable project
directory with source files, README, requirements, and (optionally)
generated UI assets at the end.

## 1. Install footprint

- `git clone https://github.com/OpenBMB/ChatDev && pip install -r requirements.txt`
  is the documented path; no `pip install chatdev` on PyPI.
- Python 3.9+. ~200 MB of deps once the agent framework, tenacity,
  tiktoken, and the optional Pillow / openai-whisper extras land.
- Docker image is published; `docker run` for a self-contained run.
- macOS / Linux first-class; Windows works via WSL.

## 2. License

Apache-2.0 — `LICENSE` at the repo root
(<https://github.com/OpenBMB/ChatDev/blob/main/LICENSE>).

## 3. Models supported

OpenAI Chat Completions by default (GPT-4o / GPT-4 / o-series via
`OPENAI_API_KEY`); any OpenAI-compatible endpoint via `BASE_URL`
(Ollama, vLLM, LiteLLM, OpenRouter, DeepSeek, Together). The
`run.py --model` flag picks a model name; a `ChatDevConfig`
JSON lets you assign a different model per role if you want the
CEO to think on a smaller model than the Programmer.

## 4. MCP support

No. Tooling is an internal Python registry of "phases" and
"composed-phases"; external tool calls happen through hand-written
Python wrappers, not MCP servers.

## 5. Sub-agent model

Role-played multi-agent **as the product**. The pipeline is a
configurable directed graph of `Phase` objects, each phase pairs two
roles for a chat turn (e.g. `DemandAnalysis`: CEO ↔ CPO,
`LanguageChoose`: CTO ↔ CEO, `Coding`: CTO ↔ Programmer,
`CodeReviewComment`: CRO ↔ Programmer, `TestErrorSummary`: Programmer ↔
Tester). Composed phases (`CodeCompleteAll`, `CodeReview`, `Test`)
loop a phase until a stop condition (file complete / no review
comments / tests pass) or a max-iteration cap.

## 6. Telemetry stance

Off. The OSS codebase has no analytics call. Egress is exactly: your
configured LLM endpoint, plus PyPI when you `pip install` and the
local file system on output. Each run writes a full conversation log
plus a `manuals` directory with phase-by-phase transcripts to disk.

## 7. Prompt-cache strategy

None. Each phase is a fresh chat completion with a role-tailored
system prompt; no cross-phase caching beyond what the LLM provider
does on its own.

## 8. Hot keybinds

Non-interactive CLI — `python run.py --task "..." --name "MyApp"` runs
the whole pipeline to completion and writes the result to
`WareHouse/<name>_<org>_<timestamp>/`. No TUI; you watch streaming
logs.

## 9. Killer feature, weakness, when to choose

**Killer feature.** Software-process-as-prompt-graph. The agent
roles, the phase order, and the phase-loop stop conditions are
declarative `ChatChainConfig.json` + `PhaseConfig.json` files —
swap `Art` in for image generation, drop `Test` for a sketch run,
add a `Manual` phase for end-user docs, all without touching Python.
The catalog's clearest answer to "what does a multi-agent SDLC look
like if you write it down as data instead of code".

**Weakness.** Output is greenfield-only and small-scale: the pipeline
assumes one Python project from scratch and tops out around a few
hundred lines of code. No iterate-on-existing-repo mode. Quality is
heavily model-dependent — on smaller models the role play degrades
into agreement loops. The "Test" phase runs the generated code in
your local environment with no sandbox.

**When to choose.** Greenfield prototyping where you want a full
pretend-team conversation as the artifact, not just the code. Pick
this over [`gpt-engineer`](../gpt-engineer/) when the role transcript
*is* part of the deliverable (academic write-ups, multi-agent
research baselines, teaching materials), and over
[`smol-developer`](../smol-developer/) when you want phase-level
review and test loops instead of a single planner-then-coder pass.

## Quick example

```bash
# install
git clone https://github.com/OpenBMB/ChatDev
cd ChatDev
pip install -r requirements.txt

# minimal env
export OPENAI_API_KEY=sk-...

# run the full virtual-company pipeline
python run.py \
  --task "Design a terminal pomodoro timer with pause/resume and JSON history" \
  --name "PomoCLI" \
  --model "GPT_4O"

# output: WareHouse/PomoCLI_DefaultOrganization_<timestamp>/
#   ├── main.py
#   ├── requirements.txt
#   ├── manual.md
#   ├── meta.txt
#   └── PomoCLI.log         (full multi-agent transcript)
```
