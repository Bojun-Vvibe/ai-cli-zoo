# patchwork

> Snapshot date: 2026-04. Upstream: <https://github.com/patched-codes/patchwork>
> Latest release: `v0.0.124` (2025-04-16). License file:
> [`LICENSE`](https://github.com/patched-codes/patchwork/blob/main/LICENSE).

A self-hosted CLI **agentic workflow runner** for development gruntwork —
PR reviews, library upgrades, vulnerability remediation, docstring
generation, README generation, issue resolution. The unit of work is a
**Patchflow**: a YAML-declared graph of reusable **Steps** (call an LLM,
open a PR, run Semgrep, query a vector index, post to Slack…) glued
together with versioned **Prompt Templates**.

Patchflows run identically from a developer laptop, a CI job, or a hosted
runner. The product surface is "ship a fix, not a chat reply".

## 1. Install footprint

- Python 3.10+. Distributed on PyPI as `patchwork-cli`.
- `pip install 'patchwork-cli[all]' --upgrade` for the full kit. Optional
  extras keep core install lean:
  - `[security]` — pulls `semgrep` + `depscan` (needed for `AutoFix` and
    `DependencyUpgrade`).
  - `[rag]` — pulls `chromadb` (needed for `ResolveIssue`).
  - `[notifications]` — Slack / etc. step backends.
  - bare install (`pip install patchwork-cli`) supports `GenerateDocstring`,
    `PRReview`, `GenerateREADME` only.
- No daemon. Each `patchwork <Patchflow>` invocation is a one-shot pipeline.
- Built-in Patchflow defaults live inside the package; project-level
  overrides go in [`patchwork-configs`](https://github.com/patched-codes/patchwork-configs)
  and are pointed at via `--config /path/to/patchwork-configs/patchflows`.

## 2. License

AGPL-3.0. See [`LICENSE`](https://github.com/patched-codes/patchwork/blob/main/LICENSE).
Strong copyleft — running a modified Patchwork as a network service to
others triggers the AGPL source-disclosure obligation. There is also a
hosted commercial offering at `app.patched.codes` for users who need a
managed alternative.

## 3. Models supported

LLM step is provider-agnostic. Supported out of the box:

- **OpenAI** (`openai_api_key` + `model`).
- **Google AI Studio / Vertex** (`google_api_key` + `model=gemini-1.5-pro`
  etc. — the docs explicitly call out Gemini's 1M-token context as the
  recommended choice for whole-repo Patchflows).
- **Anthropic** (`anthropic_api_key` + `model=claude-3-…`).
- **Hosted Patched endpoint** (`patched_api_key`) — managed routing.
- **Any OpenAI-compatible endpoint** via `client_base_url=…` —
  documented for Groq, Together, Hugging Face Inference API,
  `llama.cpp` server, `ollama`, `vllm`, `tgi`. A `config.yml` file can
  pin all three of `openai_api_key` / `client_base_url` / `model`
  together.

Per-step model overrides are supported, so a Patchflow can run cheap
classification on Groq / Llama-3.1-8B and the final patch generation on
Claude or Gemini-1.5-Pro.

## 4. MCP support

**No.** Patchwork's tool surface is its own **Step library** (Python
classes registered under `patchwork.steps.*`), not MCP. Steps are
imported and chained inside a Patchflow YAML; you write a new Step in
Python rather than mounting an MCP server. Any MCP integration is
out-of-band — wrap an MCP-aware client as a Step.

## 5. Sub-agent model

Pipeline-style, not crew-style. A Patchflow is a **declarative directed
graph of Steps**; LLM calls are individual Steps, and the "agency" lives
in the loop logic *between* Steps (Semgrep finds 12 issues → fan out 12
LLM calls → collect → open one PR with 12 fix commits). There is no
manager-LLM-spawning-worker-LLMs orchestra in the [`crewai`](../crewai/)
sense — orchestration is Python control flow plus YAML, not a planner
agent's tool call.

## 6. Telemetry stance

**Off in the OSS codebase.** No analytics in the `patchwork` Python
package. Network egress is whatever the configured Steps require: the
LLM provider, the SCM REST API (GitHub / GitLab / Bitbucket / Azure
DevOps), Semgrep / depscan registries on first use, Slack webhook if
configured. The hosted `patched_api_key` path obviously routes prompts
through `app.patched.codes` and that is the explicit trade.

## 7. Prompt-cache strategy

None at the framework layer. Patchflows are batch pipelines, not long
chats — the cache surface that helps a REPL ([`aider`](../aider/),
[`opencode`](../opencode/)) does not apply here. Whatever automatic
prefix caching the configured provider gives you (Anthropic ephemeral
cache on a long shared system prompt, OpenAI prefix cache, Gemini
implicit caching) is what you get. The reusable artifact is the
**Prompt Template + Step graph**, version-controlled in
[`patchwork-configs`](https://github.com/patched-codes/patchwork-configs),
not a prompt cache.

## 8. Hot keybinds (CLI)

Patchwork is non-interactive — there is no TUI and no REPL. The "keybinds"
are subcommand shapes. The CLI is invoked as `patchwork <Patchflow> <key=value> …`.

| Patchflow | What it does |
|-----------|--------------|
| `AutoFix` | Run Semgrep, ask LLM to patch each finding, open a PR |
| `DependencyUpgrade` | Run `depscan`, bump vulnerable deps, regenerate lockfile, open a PR |
| `PRReview` | Pull PR diff from SCM, post inline review comments |
| `GenerateDocstring` | Add docstrings to every function in the working tree, open a PR |
| `GenerateREADME` | Generate / refresh `README.md` from the codebase |
| `ResolveIssue` | Take a GitHub issue ID, RAG over the repo (chromadb), produce a fix-PR |
| `GenerateUnitTests` | Generate pytest / jest / go-test cases for a target file |
| `LogAnalysis` | Summarize a log file, classify anomalies |
| custom | Compose your own Patchflow YAML from registered Steps |

Every Patchflow accepts `--config <path>` (or `--config <yaml>`) to
override defaults, and `key=value` arguments to override individual
template variables.

## 9. Killer feature, weakness, when to choose

- **Killer:** **PR is the primary surface, chat is not.** Most agent
  CLIs in the catalog produce a chat transcript and trust the human to
  apply changes; Patchwork produces an actual SCM pull request via a
  declarative Patchflow that is the same artifact in dev (`patchwork
  AutoFix …`) and in CI (`patchwork AutoFix …` inside a workflow). The
  Step library is genuinely reusable across Patchflows — security scan,
  RAG query, LLM call, open PR are not glued to a specific Patchflow.
- **Weakness:** **AGPL-3.0** rules out a lot of internal corporate use
  if legal is strict. No MCP, so you cannot mount it inside an
  [`opencode`](../opencode/) / [`claude-code`](../claude-code/) chat as
  a tool — Patchwork is the orchestrator, not a tool another orchestrator
  consumes. Steps are Python-only; extending the catalog means writing
  Python and re-publishing the wheel, not dropping a YAML or a binary
  shim.
- **Choose it when:** you want **agentic workflow automation that ends
  in a PR**, you are happy declaring it as YAML + Python Steps, and you
  want the same workflow definition to work on a laptop and in CI.
  Compare with [`pr-agent`](../pr-agent/) (review-shaped slash-command
  bot across many SCMs, more chat-shaped) and [`sweep`](../sweep/)
  (issue → PR pipeline, narrower scope). Pick Patchwork over those when
  you want a *library of fix-shaped flows* (security, deps, docs, tests)
  rather than a single review bot.
