# metagpt

> Snapshot date: 2026-04. Upstream: <https://github.com/FoundationAgents/MetaGPT>
> (formerly `geekan/MetaGPT`; the org redirected).
> Pinned version: **v0.8.1** (2024-04-22).
> License: **MIT** — `LICENSE` at the repo root
> (<https://github.com/FoundationAgents/MetaGPT/blob/main/LICENSE>).

A multi-agent framework that models a software company as a set of
typed roles (Product Manager, Architect, Project Manager, Engineer,
QA Engineer) communicating through a shared message bus. Ships a
`metagpt` CLI entrypoint: `metagpt "build a CLI snake game"` runs the
full SOP and drops a project directory plus PRD, system design, API
spec, and task plan as separate Markdown artifacts.

## 1. Install footprint

- `pip install metagpt` (Python 3.9–3.11) or
  `pipx install metagpt`. First run prompts you through `metagpt
  --init-config` to write `~/.metagpt/config2.yaml`.
- ~300 MB of deps once `pydantic`, `tenacity`, `tiktoken`, `tree-sitter`,
  `playwright`, `openai`, `anthropic`, and the document parsers land.
- Docker image is published; an optional `metagpt-ext-rag` extra adds
  the RAG pipeline.
- macOS / Linux first-class; Windows works.

## 2. License

MIT — `LICENSE` at the repo root
(<https://github.com/FoundationAgents/MetaGPT/blob/main/LICENSE>).

## 3. Models supported

Per-role model assignment is first-class in `config2.yaml`:

- **LLMs**: OpenAI, Anthropic, Google Gemini, Zhipu / GLM, Moonshot,
  Qwen / DashScope, DeepSeek, Mistral, Ollama, Groq, OpenRouter,
  Bedrock, any OpenAI-compatible endpoint via `base_url`.
- **Embeddings**: OpenAI, Gemini, Ollama, sentence-transformers via
  the optional RAG extra.
- **Tools**: web browser (Playwright), web search (Serper, DDG, Bing),
  code interpreter, image generation (DALL·E, Stable Diffusion via
  HuggingFace), Whisper STT.

## 4. MCP support

No first-party MCP client/server in the v0.8 line. External tools are
declared as Python `Action` subclasses with typed `run()` signatures;
composition is by message-bus subscription, not MCP.

## 5. Sub-agent model

**Role-based crew with a shared message bus**. Each `Role` has a
profile, a list of `Action`s it can take, and a watch list of
`Message` types it subscribes to. The default `SoftwareCompany`
team is `ProductManager → Architect → ProjectManager → Engineer →
QAEngineer`; a `Role.publish_message` to the env triggers whichever
roles subscribe to that type. You can declare custom teams with
arbitrary topology — the framework is the agent loop, the team
composition is a Python file.

## 6. Telemetry stance

Off in the OSS codebase. No analytics calls. Egress is exactly: your
configured LLM provider, the search backend you wired up, and any
URLs the browser action visits. The optional Reflyt / Phoenix
exporters for OTel tracing are opt-in.

## 7. Prompt-cache strategy

In-memory message-history pruning per role (configurable
`max_react_loop`, `max_history_tokens`); no on-disk prompt cache.
Anthropic prompt-caching headers are forwarded when you point a
role at Claude.

## 8. Hot keybinds

Non-interactive CLI: `metagpt "..."` streams role-by-role logs to
stdout and writes the project to `./workspace/<project_name>/`.
The optional `metagpt --interactive` REPL lets you steer mid-run by
injecting messages onto the bus.

## 9. Killer feature, weakness, when to choose

**Killer feature.** Standardized SOP artifacts as the deliverable.
Every run produces a PRD, system design, API spec, and task list as
separate Markdown / JSON files alongside the code, so the
"thinking" is reviewable independently of the diff. The
`config2.yaml` per-role model assignment is the cleanest in this
catalog: planner on a frontier model, engineer on an open model,
QA on a cheap model — three lines of YAML, no glue code.

**Weakness.** Greenfield-biased and small-to-medium scale; the SOP
assumes "build me X" not "fix bug Y in this 100k-LOC repo". The
v0.8 line has been quiet since 2024 with most active development
moving to `MetaGPT-X` / `agent-protocol` work in the same org —
treat this as a stable framework, not a fast-moving roadmap.
Engineer role runs generated code locally with no sandbox by
default.

**When to choose.** Multi-agent prototyping where the *process
artifacts* (PRD, design doc, task plan) matter as much as the code,
and where you want per-role model heterogeneity expressed as config,
not Python. Pick this over [`chatdev`](../chatdev/) when you want
typed Python `Action`s and a message-bus topology you can extend
programmatically, and over [`crewai`](../crewai/) when you want a
batteries-included software-company SOP out of the box rather than
a generic role-builder.

## Quick example

```bash
# install
pipx install metagpt

# first-run config (writes ~/.metagpt/config2.yaml)
metagpt --init-config
# edit ~/.metagpt/config2.yaml — set llm.api_type, llm.model, llm.api_key

# run the full SOP → writes to ./workspace/<project>/
metagpt "build a CLI snake game in Python with arrow-key controls"

# output:
# workspace/snake_game/
#   ├── docs/
#   │   ├── prd/         (Product Manager artifact)
#   │   ├── system_design/   (Architect artifact)
#   │   └── task/        (Project Manager task plan)
#   ├── snake_game/      (Engineer source files)
#   ├── tests/           (QA Engineer tests)
#   └── requirements.txt

# run a custom team in Python
python -c "
from metagpt.roles import ProductManager, Architect, Engineer
from metagpt.team import Team
team = Team()
team.hire([ProductManager(), Architect(), Engineer()])
team.invest(investment=3.0)
team.run_project('build a CLI markdown linter')
import asyncio; asyncio.run(team.run(n_round=5))
"
```
