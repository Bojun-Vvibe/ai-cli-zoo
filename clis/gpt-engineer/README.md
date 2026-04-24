# gpt-engineer

> Snapshot date: 2026-04. Upstream: <https://github.com/AntonOsika/gpt-engineer>

The original "one prompt → whole repo" greenfield generator from May
2023, still maintained as the **CLI entry point** to what eventually
became the (now-hosted) Lovable.dev product. Unlike most catalog
entries, `gpt-engineer` is not a chat REPL and not a per-file editor:
you write a `prompt` file describing the application you want, run
`gpte projects/myapp`, and it scaffolds the repo from scratch in a
sandboxed working directory, asking clarifying questions only when
explicitly invoked.

It earns a catalog slot for being **the canonical reference for
greenfield code generation as a one-shot workflow**: every later
"natural language → starter project" tool ([`smol-developer`](../smol-developer/),
[`micro-agent`](../micro-agent/), [`sweep`](../sweep/)) was either
inspired by it or explicitly positioned against it.

## 1. Install footprint

- `pip install gpt-engineer` (or `pipx install gpt-engineer` to keep
  it isolated). Pure Python, ~30 MB venv.
- Single `gpte` console-script entry point. No daemon, no GUI, no
  background process.
- API key via `OPENAI_API_KEY` (default), or any OpenAI-compatible
  endpoint via `OPENAI_API_BASE` and `MODEL_NAME` (Ollama, vLLM,
  LiteLLM, Azure OpenAI, OpenRouter).
- Project skeleton convention: each run lives in its own directory
  (`projects/myapp/`) containing a `prompt` file. Generated code,
  memory, and improvement logs land alongside it.

## 2. Repo, version, license

- Repo: <https://github.com/AntonOsika/gpt-engineer>
- Latest release checked: **v0.3.1** (tag `v0.3.1`, published
  2024-06-06; still the latest tagged release as of 2026-04, with
  ongoing `main`-branch commits).
- License: MIT. License file at the repo root:
  [`LICENSE`](https://github.com/AntonOsika/gpt-engineer/blob/main/LICENSE).
- PyPI package: `gpt-engineer`.

## 3. What it actually does

Two primary modes, both scoped to a single project directory:

1. **`gpte <project-dir>`** — *new project*. Reads
   `<project-dir>/prompt` as the spec, asks an optional clarification
   round (`--clarify`), then emits a tree of files plus a runnable
   `run.sh` entry script. The model is asked to *also* explain the
   architecture in plain prose, which is saved into
   `<project-dir>/.gpteng/memory/`.
2. **`gpte <project-dir> --improve` (`-i`)** — *iterate on an
   existing repo*. Loads every file under the project dir into the
   model's context, asks for the desired change in `prompt`, and
   emits **unified diffs** that get applied locally. This is the
   only mode that approximates an editor loop.

Other useful flags: `--lite` (skip the architecture-explainer pass,
cheaper), `--no-execution` (don't run `run.sh` at the end),
`--azure <endpoint>` (Azure OpenAI), `--use-custom-preprompts`
(override the system-prompt set under `~/.gpte/preprompts/` to
specialise the agent's persona).

## 4. MCP support

None. `gpt-engineer` predates MCP and the project has not adopted
the protocol. Tool calls are not part of its loop — the only
"tools" are the file-write operation and the optional `run.sh`
execution.

## 5. Sub-agent model

None as a structured concept. Internally there is a small pipeline
(clarify → generate spec → generate code → generate entry script →
optional execution-and-self-debug step), but each stage is a
sequential single-model call, not a delegated sub-agent.

The optional **self-healing run** (`--use-custom-preprompts` plus
`run.sh` failure capture) is the closest thing to a loop: if
`run.sh` exits non-zero and the user opts in, the captured stderr is
fed back into the model for one repair pass. There is no
configurable max-iterations cap — it's one shot.

## 6. Telemetry stance

**Opt-in collection.** On first run the CLI prompts a yes/no for
anonymous "consent" telemetry that records prompts, models used,
and outcomes for upstream improvement. Default is **No** — answer
the prompt accordingly or pre-set `COLLECT_LEARNINGS_OPT_IN=false`
in the environment. The OpenAI / configured endpoint always sees
the prompt + project files by construction.

## 7. Token / context strategy

- **`--improve` mode loads every file in the project directory into
  one prompt.** There is no AST extraction, no embedding-based
  retrieval, no chunking. On a project bigger than ~2k LOC, you
  will hit the model's context limit; the recommended workaround is
  to delete or `.gpteignore` files that aren't relevant to the
  current change before invoking.
- New-project mode is bounded only by the size of the model's
  output, not its input — `prompt` files are typically <1k tokens.
- Prompts and responses are written to `<project-dir>/.gpteng/memory/`
  for replay and debugging; nothing is sent to a remote store
  unless telemetry is opted in.

## 8. Hot keybinds

None. `gpte` is a non-interactive CLI: it runs to completion (or
asks one clarification round), then exits. Iteration is "edit
`prompt`, re-run `gpte … -i`".

## 9. Killer feature, weakness, when to choose

**Killer feature.** **The `prompt`-file workflow is the cleanest
greenfield primitive in the catalog.** A `prompt` is plain text, in
your project directory, version-controllable. The whole "what did I
ask for" history of a project lives in git diffs of one file.
Compared to chat-REPL tools where the spec evaporates with the
session, this is auditable: you can grep your project history for
*intent* changes, not just *code* changes.

**Weakness.** **Improve-mode does not scale past small projects.**
The "load every file into one prompt" architecture means a 5k-LOC
codebase needs aggressive `.gpteignore` curation, and a 50k-LOC one
is hopeless. There is no repo-map ([`aider`](../aider/)), no AST
extractor ([`symbex`](../symbex/)), no embedding retrieval. Also:
the project is in maintenance mode — the active commercial
descendant is the hosted Lovable.dev product, and the CLI's
roadmap (multi-agent, browser-based clarification, integrated
testing) has been static since 2024. Single-author governance with
infrequent releases.

**Choose it when.**

- You're starting a **new** project from a paragraph-sized spec and
  want a runnable scaffold in one command, not a chat session.
- You want the spec itself (`prompt` file) to be a **git-tracked
  artifact** that future contributors read first, not buried in
  chat logs.
- You want a tool that **runs the generated code at the end**
  (`run.sh`) so you immediately know whether the scaffold works,
  not just whether it compiles.

**Skip it when.**

- The project already exists and is bigger than a few thousand
  lines — use [`aider`](../aider/) (repo-map + git-native diff
  loop) or [`claude-code`](../claude-code/) /
  [`opencode`](../opencode/) (full agent with sub-tasks) instead.
- You want **interactive clarification** mid-generation — use a
  chat-shaped tool ([`crush`](../crush/), [`codex`](../codex/),
  [`continue`](../continue/)) where you can steer turn-by-turn.
- You need **non-OpenAI providers as a first-class concern** —
  `gpt-engineer` supports them via env vars but the system prompts
  are tuned for GPT-4-class models and quality drops noticeably on
  smaller open-weight ones.

## 10. Compared to neighbors in the catalog

| Tool | Greenfield primitive | Improve-existing | Spec storage | Runs the result |
|------|----------------------|------------------|--------------|-----------------|
| gpt-engineer | `prompt` file → whole repo | Yes (loads all files) | Git-tracked `prompt` file | Yes (`run.sh`) |
| [smol-developer](../smol-developer/) | One CLI prompt → whole repo | No | Ephemeral arg | No |
| [micro-agent](../micro-agent/) | Failing test → single file | Yes (one file at a time) | Test file | Yes (test runner is the loop) |
| [aider](../aider/) | N/A (repo-anchored from turn 1) | Yes (repo-map + diffs) | Chat history | No (you run things) |
| [sweep](../sweep/) | GitHub issue → PR | Yes (PR cycle) | GitHub issue | CI runs it |

Decision shortcut:

- **"I have one paragraph and I want a working scaffold by lunch"**
  → `gpt-engineer`.
- "I have a failing test and want a tiny implementation" →
  `micro-agent`.
- "I have an existing repo and want to make a focused change" →
  `aider`.
- "I want to chat my way through the design first" → `crush` /
  `codex` / `continue`.
