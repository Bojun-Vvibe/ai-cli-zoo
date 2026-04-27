# local-deep-research

> Snapshot date: 2026-04. Upstream:
> <https://github.com/LearningCircuit/local-deep-research>
> Pinned release: `v1.6.2`
> (HEAD `84448acd1af804866ab4358eb26179e966643498`).
> License file:
> [`LICENSE`](https://github.com/LearningCircuit/local-deep-research/blob/main/LICENSE)
> (sha `81f8a883f07aef808e54b10df542a9fb29602487`, MIT).

`local-deep-research` (often shortened to `ldr`) is a Python
research-agent runtime: feed it a question, it spawns a
multi-step search-plan-read-cite loop across 10+ sources
(arXiv, PubMed, Wikipedia, SearXNG, the open web, Semantic
Scholar, GitHub, plus user-attached PDF / Markdown corpora),
and emits a cited Markdown report. The CLI / web UI is the
primary surface; an MCP server flavour exposes the same agent
to MCP clients. It is the catalog's reference for **a fully
local "deep-research" agent** — the pattern that
[`gpt-researcher`](../gpt-researcher/) popularised, but with
local-Ollama as the default LLM, SQLCipher-encrypted history,
and a self-hosted SearXNG search front-end so no query leaves
the box.

## 1. Install footprint

- `pip install local-deep-research` (Python 3.11+) installs the
  `ldr` CLI plus the Flask web UI on `127.0.0.1:5000`.
- Docker: `docker run -p 5000:5000 -v deep-research:/data
  localdeepresearch/local-deep-research:v1.6.2`. The repo also
  ships a `docker-compose.yml` that stands up Ollama + SearXNG
  + LDR together with one command.
- Hard runtime deps it expects to talk to: an LLM endpoint
  (Ollama, LM Studio, llama.cpp, vLLM, OpenAI, Anthropic,
  Google, OpenRouter — anything LiteLLM routes) plus a search
  back-end (SearXNG self-hosted is recommended; Tavily,
  Serper, DuckDuckGo, Brave also wired in).
- Local index: SQLCipher-encrypted SQLite at `~/.config/ldr/`
  holds research history, citations, and the optional personal
  document corpus.

## 2. Repo, version, license

- Repo: <https://github.com/LearningCircuit/local-deep-research>
- Latest release: `v1.6.2`.
- HEAD pinned at this snapshot:
  `84448acd1af804866ab4358eb26179e966643498`.
- License: MIT. Pinned LICENSE blob:
  [`LICENSE`](https://github.com/LearningCircuit/local-deep-research/blob/main/LICENSE)
  (sha `81f8a883f07aef808e54b10df542a9fb29602487`).

## 3. What it actually does

The agent loop is "iterative deep research": plan sub-
questions, dispatch parallel searches, read the top-N results
per sub-question, synthesise a partial answer with citations,
update the plan, repeat until convergence or the iteration
budget is hit. Two modes ship out of the box:

- `quick` — single search round, summarise top results, ~2 min
  on a local 7-8B model.
- `detailed` — multi-round iterative research with a planner-
  executor split, runs 10–60 min depending on the model and
  the question; the upstream README cites ~95 % on SimpleQA
  with GPT-4.1-mini and competitive numbers on local models.

Surfaces:

```
ldr                              # web UI on :5000
ldr-cli "question"               # one-shot research from the shell
ldr-mcp                          # stdio MCP server flavour
```

The web UI is the friendliest path (multi-tab research,
inline citation hovers, live progress); the CLI is the
script / cron path; the MCP server lets a host agent
([`claude-code`](../claude-code/), [`opencode`](../opencode/),
[`goose`](../goose/), etc.) outsource the deep-research step.

## 4. MCP support

**Yes — server-side.** `ldr-mcp` exposes the research engine
as an MCP server with two primary tools (`research_quick`,
`research_detailed`); plug it into any MCP-aware client and
the model can request a citation-backed research report
without leaving the session.

## 5. Sub-agent model

Internal — the planner LLM and the per-sub-question executor
LLM are configurable to different models (e.g. cheap local 8B
for sub-question reads, larger model for synthesis). Not
exposed as a user-facing sub-agent CLI; configured via the web
UI's settings page or the YAML config under `~/.config/ldr/`.

## 6. Telemetry stance

Off / not applicable to the agent runtime itself. The history
DB is SQLCipher-encrypted on disk; the LLM and search back-
ends inherit whatever telemetry posture they ship with
(Ollama / SearXNG self-hosted = no egress; OpenAI / Tavily =
their respective endpoints). The repo emphasises the
"everything local & encrypted" stance and the
`localdeepresearch/local-deep-research` Docker image runs with
no analytics by default.

## 7. Token / context strategy

Per-LLM — `ldr` does not impose its own ceiling. The planner
truncates retrieved-document context to fit the configured
model's window before each synthesis call; long documents are
chunked + re-ranked rather than naively concatenated. For
local 8 k-context models the iterative loop is what compensates
for the small window: many small contexts in series rather
than one big context.

## 8. Hot keybinds

None — the primary surface is a Flask web UI in the browser,
secondary surfaces are non-interactive CLI / MCP. The web UI
uses standard chat affordances (`Enter` to send a research
query); long-running research can be backgrounded and resumed
from the History tab.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **A `gpt-researcher`-class deep-research
agent that runs end-to-end against a local Ollama + a self-
hosted SearXNG with no per-query egress and an SQLCipher-
encrypted history DB.** The MCP server flavour means the same
engine plugs into any MCP-aware host agent, so a Claude Code /
OpenCode / Goose session can offload "go research X for 20
minutes" without losing privacy or paying per-token search
fees.

**Weakness.** Quality on a small local 7-8B model is
noticeably below the GPT-4.1-mini / Claude / Gemini number the
upstream benchmark cites; the iterative loop helps but cannot
fully close the gap. Operationally the project assumes Docker
or a Python venv plus an Ollama daemon plus a SearXNG instance
— a 3-container stack to set up before the first query, vs. a
single hosted SaaS call. Web UI is the most polished surface;
the CLI is functional but less ergonomic for interactive work.

**Choose `local-deep-research` when** the requirement is a
private, citation-producing research agent that owns its own
search and history, and the team is happy to run a small Docker
stack to keep queries off third-party servers. **Choose
something else when** SaaS speed and quality matter more than
privacy ([`gpt-researcher`](../gpt-researcher/) with a frontier
model), the deliverable is a single shell pipe rather than a
multi-minute research run ([`fabric`](../fabric/),
[`mods`](../mods/), [`shell-gpt`](../shell-gpt/)), or the host
agent already does its own web research
([`opencode`](../opencode/),
[`claude-code`](../claude-code/) with the WebFetch tool,
[`gemini-cli`](../gemini-cli/)).
