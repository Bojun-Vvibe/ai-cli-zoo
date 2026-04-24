# CHOOSING.md — pick a CLI in 60 seconds

This is a decision tree, not a leaderboard. Start at the top and walk down.

## 1. Where does the model live?

- **Cloud-only is fine** → continue to step 2.
- **Must run locally / air-gapped** → [`aider`](clis/aider/) (Ollama),
  [`gptme`](clis/gptme/) (Ollama, llama.cpp), or
  [`continue`](clis/continue/) (any OpenAI-compatible local endpoint),
  or [`oterm`](clis/oterm/) for an Ollama-only multi-tab TUI with no
  cloud surface at all.
  Skip the rest of this tree.

## 2. Where do you live?

- **Terminal, full-screen TUI** → [`opencode`](clis/opencode/),
  [`crush`](clis/crush/), [`codex`](clis/codex/),
  [`claude-code`](clis/claude-code/), [`gemini-cli`](clis/gemini-cli/),
  [`OpenHands`](clis/openhands/) (CLI mode).
- **Terminal, REPL-style** → [`aider`](clis/aider/), [`gptme`](clis/gptme/),
  [`mentat`](clis/mentat/), [`plandex`](clis/plandex/).
- **Unix pipe primitive (no session, no UI)** → [`mods`](clis/mods/),
  or [`llm`](clis/llm/) if you also want every prompt logged to SQLite
  for later replay, or [`shell-gpt`](clis/shell-gpt/) if you want the
  output to default to a *shell command* you can execute inline, or
  [`tgpt`](clis/tgpt/) if you want zero-config no-API-key operation,
  or [`fabric`](clis/fabric/) if you want to apply a curated
  prompt-pattern library to whatever you pipe in, or
  [`chatblade`](clis/chatblade/) if you want the model to reply as
  JSON/YAML and extract a sub-path in the same call (`chatblade -e
  '.commands[0]' -j "..."`), or
  [`files-to-prompt`](clis/files-to-prompt/) if you need to **pack a
  whole subtree** as deterministic context to pipe *into* one of the
  above (`files-to-prompt src/ --cxml | llm -m claude-sonnet '...'`),
  or [`symbex`](clis/symbex/) if you only need **specific Python
  symbols** instead of whole files (`symbex 'MyClass.*' | llm
  '...'`), or [`repomix`](clis/repomix/) if you want a **whole-repo
  pack with token counts, secret-scan, and optional remote fetch**
  (`repomix --remote owner/repo --compress`).
- **Multi-turn REPL where the model writes and runs code** →
  [`open-interpreter`](clis/open-interpreter/) (any language, real
  machine, no sandbox), [`gptme`](clis/gptme/) (sandboxed-ish via
  Docker tool), or [`codex`](clis/codex/) (sandboxed via
  Seatbelt/Landlock).
- **Inside an IDE** → [`cline`](clis/cline/) (VS Code),
  [`continue`](clis/continue/) (VS Code + JetBrains).
- **GitHub-issue → PR, no local UI** → [`sweep`](clis/sweep/).
- **Greenfield "build me an app from a paragraph"** →
  [`smol-developer`](clis/smol-developer/).

## 3. How much agency do you want to give the model?

```
low  ◄────────────────────────────────────────────────────────►  high
aider     mentat     continue     opencode     codex     OpenHands     sweep
(diff       (diff      (suggest)   (tool-call    (sandboxed   (full Linux   (writes
 preview,    preview)               with         exec,         sandbox,      to your
 you `y/n`)                        approval)    auto-loop)    long runs)    main branch)
```

- **"Show me the diff before any write"** → `aider`, `mentat`.
- **"Approve each tool call"** → `cline`, `opencode` (default).
- **"Run a sandbox, just don't break my host"** → `codex`, `OpenHands`.
- **"Open a PR, I'll review on GitHub"** → `sweep`.

## 4. Do you need MCP (Model Context Protocol) servers?

- **Yes, must consume MCP servers** → `opencode`, `codex`, `cline`,
  `OpenHands`, `crush`, `continue`. All ship MCP clients.
- **Yes, want to *expose* a context-packing tool to an MCP-aware
  agent** → [`repomix`](clis/repomix/) (`repomix --mcp` exposes
  `pack_codebase`, `pack_remote_repository`, etc. over stdio).
- **No, prefer the CLI to bring its own tools** → `aider` (built-in repo-map),
  `gptme` (built-in shell/python/browser), `plandex` (built-in plan engine).

## 5. Multi-file, multi-step plans?

- **Single edit at a time** → `aider`, `mentat`, `cline`.
- **Multi-step plan with branches** → [`plandex`](clis/plandex/).
- **Long-horizon agent loop with sub-agents** → [`opencode`](clis/opencode/)
  (Task tool), [`claude-code`](clis/claude-code/) (Task tool, hooks, skills),
  [`OpenHands`](clis/openhands/) (multi-agent), [`forge`](clis/forge/)
  (YAML workflows with per-step model routing).
- **No plan, just transform stdin** → [`mods`](clis/mods/), or
  [`llm`](clis/llm/) if you also want a queryable history of every call,
  or [`fabric`](clis/fabric/) if you want a versioned, shared library
  of prompt patterns to apply to that stdin (`pbpaste | fabric -p
  extract_wisdom`).
- **Chat with a folder of documents from a terminal** →
  [`aichat`](clis/aichat/) (built-in RAG, single binary).

## 5b. Single-purpose workflow glue

Some niches are narrow enough that a dedicated CLI beats a general
agent. These are install-once-and-forget tools.

- **Auto-generate git commit messages from staged diff** →
  [`opencommit`](clis/opencommit/) (richer provider matrix,
  `prepare-commit-msg` hook story) or [`aicommits`](clis/aicommits/)
  (OpenAI-only by default, `-g N` multi-candidate picker UX).
  Pick `opencommit` if you want the hook installed once and
  `git commit` to "just work"; pick `aicommits` if you prefer
  choosing from N suggestions over editing one.
- **Generate Terraform / K8s / Dockerfile / GitHub Actions from a
  one-line description** → [`aiac`](clis/aiac/). Output is a clean
  IaC artifact (no markdown fences, no prose), ready to
  `terraform fmt && validate` or `kubectl apply`. Use it for
  greenfield scaffolding; for editing an existing IaC repo, switch
  to `aider` or `claude-code`.
- **Ship a runnable, version-controlled agent script alongside your
  code** → [`gptscript`](clis/gptscript/). A `.gpt` file is an
  English instruction body plus declared tools / args / output
  schema; scripts call other scripts as sub-tools. Closer to a
  shell script than to an interactive session — review it in PRs,
  run it in CI. Closest cousin in the catalog is
  [`forge`](clis/forge/) (heavier, multi-agent YAML for coding);
  pick `gptscript` for general-purpose scripting and `forge` for
  code-modifying agent pipelines.
- **Regression-test a prompt across N providers and assert quality /
  cost / latency** → [`promptfoo`](clis/promptfoo/). Declarative
  `promptfooconfig.yaml`, cartesian eval, HTML diff viewer, optional
  red-team mode. The only entry in the catalog whose job is *grading*
  output rather than producing it; pair with [`llm`](clis/llm/) for
  exploration and [`fabric`](clis/fabric/) for prompt authoring.

## 5c. Context-packing pipeline (LLM-input shaping)

These produce text *for* a downstream LLM CLI; they do not call
models themselves. Compose with shell pipes
(`<packer> | <llm-cli> '<question>'`).

- **Pack a directory tree, language-agnostic, deterministic, no
  network** → [`files-to-prompt`](clis/files-to-prompt/) (Python,
  honors `.gitignore`, emits Anthropic-XML / markdown / line-numbered
  shapes).
- **Pack a *Python* codebase down to specific symbols, not whole
  files** → [`symbex`](clis/symbex/) (AST-correct,
  `symbex 'MyClass.*'`, ~10× smaller context than file-level
  packers, Python-only).
- **Pack a *whole repo* with token counts, secret-scan, optional
  Tree-sitter compression, and the ability to fetch a remote repo
  you do not have cloned** → [`repomix`](clis/repomix/) (Node, MIT,
  `--mcp` server mode for MCP-aware agents).

Decision shortcut:

- "Just give me one Python function": `symbex`.
- "A few selected files, glob-controlled, no Node": `files-to-prompt`.
- "The whole repo, with budget visibility and a safety net":
  `repomix`.
- "It's a PDF / EPUB / DOCX, not source code":
  [`marker`](clis/marker/) first, then one of the above on the
  resulting `.md` tree.

## 5d. Document conversion (PDF → LLM-ingestible text)

Source-tree packers (§5c) assume your inputs are already text. They
are useless on PDFs of papers, contracts, scanned manuals, or office
docs. For those, run a converter first.

- **Convert PDF / EPUB / DOCX / PPTX → Markdown with preserved
  tables, LaTeX equations, and image refs** →
  [`marker`](clis/marker/). Local layout + OCR + table-recognition
  pipeline; optional `--use_llm` finishing pass for ambiguous
  regions. Output is a directory of `.md` + images you can pipe into
  any other CLI here:
  `marker_single paper.pdf out/ && cat out/paper/paper.md | llm
  '...'`. Skip it only if `pdftotext` is good enough for your input
  (one-column, no tables, no math).

## 5e. Evaluating prompts (gating quality, not producing it)

- **Run a prompt against N providers × M test rows and fail CI on
  regressions** → [`promptfoo`](clis/promptfoo/). Declarative
  `promptfooconfig.yaml` with `assert:` blocks (`contains`,
  `is-json`, `latency`, `cost`, `llm-rubric`, `javascript`,
  `python`, `model-graded-*`), HTML diff viewer (`promptfoo view`),
  optional `redteam` mode for adversarial generation. The only
  *grader* in this catalog; everything else *produces* output.
  Compose with [`llm`](clis/llm/) (replay logged sessions as
  `promptfoo` rows) and [`fabric`](clis/fabric/) (author the
  patterns you then regress).

## 6. Team / org constraints

- **Telemetry must be off by default** → all entries marked "Off" in the
  matrix: `opencode`, `aider`, `cline`, `crush`, `continue`, `mentat`,
  `gptme`, `smol-developer`, `mods` (no telemetry at all), `forge`,
  `tgpt` (no telemetry; note that free providers see prompts), `fabric`,
  `opencommit`, `aicommits`, `aiac`, `symbex` (no network at all),
  `repomix` (no analytics; only egress is the optional `--remote` git
  clone), `chatblade`, `promptfoo` (eval cache local; `share` is
  opt-in), `marker` (no network unless `--use_llm` configured),
  `gptscript`.
- **Permissive license required (no AGPL, no GPL)** → avoid `plandex`
  (AGPL core), `open-interpreter` (AGPL-3.0), and `tgpt` (GPL-3.0).
  **Source-available, not OSI-approved** → `claude-code` is excluded if
  you require an OSI license.
  Prefer Apache-2.0 / MIT entries.
- **Air-gapped CI** → `aider` + Ollama, or `continue` headless mode pointed
  at a local vLLM.

## TL;DR cheat-sheet

| Situation | Pick |
|-----------|------|
| First-time user, want a clean TUI | `opencode` |
| Want a single binary, no Python | `crush` or `codex` |
| Already live in VS Code | `cline` |
| Editing a large existing repo, git-native | `aider` |
| Need a sandboxed agent that can run shell | `codex` or `OpenHands` |
| Building a new project from scratch | `smol-developer`, then switch |
| Triaging a backlog of GitHub issues | `sweep` |
| Want plans, not just edits | `plandex` |
| Want a Python REPL with an LLM in it | `gptme` |
| Multi-tool config in one YAML | `continue` |
| Diff-first conservative edits | `mentat` or `aider` |
| Need hooks + sub-agents on Claude | `claude-code` |
| Want a max-context model with caching | `gemini-cli` |
| LLM as a unix utility, no agent loop | `mods` |
| Want different models for plan vs edit vs review | [`forge`](clis/forge/) |
| Want a serious coder model with a free tier and no credit card | [`qwen-code`](clis/qwen-code/) |
| Want SQLite-logged history of every LLM call | [`llm`](clis/llm/) |
| Want chat + RAG over local docs from one binary | [`aichat`](clis/aichat/) |
| Want the LLM to write and *run* code in a REPL on your real machine | [`open-interpreter`](clis/open-interpreter/) |
| Forget shell flags, want `Ctrl-L` to generate the command inline | [`shell-gpt`](clis/shell-gpt/) (single answer) or [`gorilla-cli`](clis/gorilla-cli/) (multi-candidate picker; default endpoint logs prompts) |
| Want an LLM in a fresh terminal with no API key, right now | [`tgpt`](clis/tgpt/) |
| Want a team-shared, version-controlled prompt-pattern library | [`fabric`](clis/fabric/) |
| Want to pack a whole subtree as deterministic context for any LLM CLI | [`files-to-prompt`](clis/files-to-prompt/) |
| Want a chat TUI that talks *only* to a local Ollama, with multi-tab + MCP | [`oterm`](clis/oterm/) |
| Want a shell-command suggester that shows *multiple* candidates to pick from | [`gorilla-cli`](clis/gorilla-cli/) |
| Auto-fill commit messages via a git hook, no new command in your workflow | [`opencommit`](clis/opencommit/) |
| Pick from 3 candidate commit messages with arrow keys instead of editing one | [`aicommits`](clis/aicommits/) |
| Scaffold a fresh Terraform / K8s / Dockerfile from a one-line description | [`aiac`](clis/aiac/) |
| Pack a Python codebase down to *specific symbols* instead of whole files | [`symbex`](clis/symbex/) |
| Pack a whole repo (or a remote repo you have not cloned) with token counts, secret-scan, and optional compression | [`repomix`](clis/repomix/) |
| Make the model reply as JSON/YAML and extract a sub-path in one shell command | [`chatblade`](clis/chatblade/) |
| Convert a PDF / EPUB / DOCX into Markdown that preserves tables and equations | [`marker`](clis/marker/) |
| Ship a runnable, version-controlled `.gpt` agent script alongside your code | [`gptscript`](clis/gptscript/) |
| Regression-test a prompt across providers and fail CI when quality / cost / latency budget breaks | [`promptfoo`](clis/promptfoo/) |
