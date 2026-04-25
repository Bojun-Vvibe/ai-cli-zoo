# CHOOSING.md — pick a CLI in 60 seconds

This is a decision tree, not a leaderboard. Start at the top and walk down.

## 1. Where does the model live?

- **Cloud-only is fine** → continue to step 2.
- **Must run locally / air-gapped** → [`aider`](clis/aider/) (Ollama),
  [`gptme`](clis/gptme/) (Ollama, llama.cpp), or
  [`continue`](clis/continue/) (any OpenAI-compatible local endpoint),
  or [`oterm`](clis/oterm/) for an Ollama-only multi-tab TUI with no
  cloud surface at all, or [`tlm`](clis/tlm/) for a single Go binary
  that delivers the natural-language → shell-command loop against a
  local Ollama with zero cloud egress, or
  [`ollama`](clis/ollama/) itself if you need the *runtime* layer
  first — `brew install ollama && ollama run llama3.1` gets you a
  local model in five minutes plus an OpenAI-compatible endpoint at
  `127.0.0.1:11434` that all of the above can point at, or
  [`ramalama`](clis/ramalama/) if you want the *runtime* layer
  (container-native local model server with hardware-aware image
  selection) that the other CLIs in this list can target via its
  OpenAI-compatible HTTP endpoint, or [`khoj`](clis/khoj/) for a
  long-lived
  local daemon that indexes a personal notes / PDF / code corpus and
  answers chat queries with citations (`--anonymous-mode
  --offline-chat` for a fully offline pipeline), or
  [`txtai`](clis/txtai/) if you want a *portable on-disk index file*
  (sentence-transformers + Faiss-CPU by default, no network) you
  can build on one machine and ship to an air-gapped box, or
  [`harbor`](clis/harbor/) if your real ask is "**the whole local
  LLM stack** — chat UI + runtime + web RAG + voice + image
  generation — wired up correctly in one command" rather than any
  single component (`harbor up open-webui ollama searxng speaches
  comfyui` brings the lot up on Docker Compose with cross-service
  URLs already threaded; hard prereq is Docker), or
  [`mlx-lm`](clis/mlx-lm/) if you are on Apple Silicon and want the
  *fastest* local decode the hardware can do — `mlx_lm.server --model
  mlx-community/Qwen3-Coder-30B-A3B-Instruct-4bit` gets you ~80 tok/s
  of a coding-grade model behind an OpenAI-compatible URL on a 64 GB
  M3 Max with no Docker and no GPU box; the Apple-blessed Metal
  kernels are 1.5–3× faster than llama.cpp on the same Mac, the
  small orthogonal CLI family (`generate`/`chat`/`server`/`convert`
  /`lora`/`fuse`) shell-scripts cleanly into a weights-to-endpoint
  pipeline, and ~4900 ready-to-run conversions live in the
  `mlx-community` HF org — Apple-Silicon-only is the hard catch
  (no x86 / NVIDIA / AMD path), or [`openllm`](clis/openllm/) if you
  have NVIDIA GPUs (one box or a cluster) and want a *production-
  shaped* OpenAI-compatible endpoint with one command — `openllm
  serve llama3.3:70b` pins backend (vLLM / Transformers / SGLang) +
  dtype + chat template + GPU shape in a reproducible bento
  manifest, ships a usable `/chat` UI at `:3000`, and `openllm
  deploy` reuses the same manifest against a managed BentoCloud
  cluster without rewriting serving config; the heavyweight
  counterpart to [`mlx-lm`](clis/mlx-lm/) on the cluster side.
  Skip the rest of this tree.

## 2. Where do you live?

- **Terminal, full-screen TUI** → [`opencode`](clis/opencode/),
  [`crush`](clis/crush/), [`codex`](clis/codex/),
  [`claude-code`](clis/claude-code/), [`gemini-cli`](clis/gemini-cli/),
  [`OpenHands`](clis/openhands/) (CLI mode).
- **Terminal, REPL-style** → [`aider`](clis/aider/), [`gptme`](clis/gptme/),
  [`mentat`](clis/mentat/), [`plandex`](clis/plandex/), or
  [`gpt-cli`](clis/gpt-cli/) if you want **named persona presets**
  (`gpt rust`, `gpt bash`, `gpt <your-name>`) defined in one
  `~/.config/gpt-cli/gpt.yml` — each preset bundles model +
  temperature + system prompt + predefined opening turns, plus
  first-class `--thinking N` for Claude extended-thinking, and a
  per-turn USD cost counter computed locally. No agent loop, no file
  edits — pure chat.
- **Terminal, multi-provider chat TUI with persistent searchable
  history (no agency, no tools)** → [`elia`](clis/elia/) — keyboard-
  driven Textual TUI, SQLite history with `/`-search, `Ctrl-J` to
  switch model mid-conversation. Pick this over [`oterm`](clis/oterm/)
  if you need cloud providers (OpenAI / Anthropic / Gemini / Groq),
  not just Ollama. Pick [`parllama`](clis/parllama/) instead if you
  want the same multi-provider chat **plus a full Ollama
  model-management console in the same TUI** — pull / delete / copy /
  Modelfile-build / GGUF-quantize without leaving the keyboard, with
  Fabric-pattern import and a persistent memory store as bonus
  screens. Pick [`tenere`](clis/tenere/) instead if you want **modal
  vim editing inside the prompt buffer**, multi-tab independent chats,
  and crash-resume of the last saved chat behind a single ~5 MB Rust
  binary — narrower backend matrix (OpenAI / `llama.cpp` / Ollama
  only) and no agency. Pick [`oatmeal`](clis/oatmeal/) if you want
  the chat TUI **glued to a running Neovim or VS Code**: yank a
  visual selection straight into the chat as a fenced block and
  accept the model's reply back into the buffer at the cursor — the
  clipboard middleman is gone, but the project has been quiet since
  2024-03 (no MCP, no agent loop).
- **Unix pipe primitive (no session, no UI)** → [`mods`](clis/mods/),
  or [`smartcat`](clis/smartcat/) (`sc`) if you want **named
  system prompts in a config file** swappable with one flag
  (`sc -p commit < diff.txt`) plus first-class glob input
  (`sc -g 'src/**/*.rs' "..."`),
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
- **Forget shell flags, want `Ctrl-L` to generate the command inline** →
  [`shell-gpt`](clis/shell-gpt/) (single answer) or
  [`gorilla-cli`](clis/gorilla-cli/) (multi-candidate picker; default
  endpoint logs prompts), or [`ai-shell`](clis/ai-shell/) if you want
  an `[E]xecute / [R]evise / [C]opy` loop where you can iterate on
  the suggestion (`r` to nudge, regenerate, repeat) without retyping
  the original intent, or [`llm-cmd`](clis/llm-cmd/) if you already
  use [`llm`](clis/llm/) and want the suggestion to land directly
  in your **readline buffer** (no TUI dialog, no menu — your shell's
  normal line editor handles edit-then-Enter), accepting that there
  is no `[A]bort` step and Enter executes immediately.
- **Already in `tmux` / `screen` and just want "what just
  happened?"** → [`wut`](clis/wut/). Reads the current pane's
  scrollback and explains the last command's output in one shot,
  optionally with a follow-up question (`wut "how do I fix this?"`).
  Pick this over `shell-gpt` / `tlm` / `gorilla-cli` / `ai-shell`
  when you want **zero copy-paste** — those translate intent into a
  command, `wut` interprets what already happened. Use Ollama
  (`OLLAMA_MODEL=...`) when scrollback may contain secrets.
- **Inside an IDE** → [`cline`](clis/cline/) (VS Code),
  [`continue`](clis/continue/) (VS Code + JetBrains).
- **GitHub-issue → PR, no local UI** → [`sweep`](clis/sweep/), or
  [`patchwork`](clis/patchwork/) if you want a **library of fix-shaped
  Patchflows** (`AutoFix` over Semgrep findings, `DependencyUpgrade`
  over a `depscan` run, `ResolveIssue` with chromadb RAG over the
  repo, `GenerateDocstring`, `GenerateUnitTests`, `PRReview`,
  `GenerateREADME`) declared as YAML + reusable Python Steps, where
  the *same* `patchwork <Patchflow>` invocation works on a laptop and
  in CI and the artifact is always an SCM pull request, not a chat
  log.
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
- **"Skip the diff preview, write the files now, let `git diff` be
  the review surface"** → [`promptr`](clis/promptr/). Reads a
  LiquidJS-templated prompt file plus a list of source paths from
  the command line, sends one OpenAI Chat Completions request, and
  **overwrites those files in place** with the model's response. No
  approval gate, no agent loop, no plan — built for Makefile / CI /
  git-hook use where the safety net is already `git`. Pick this
  over [`aider`](clis/aider/) when the workflow is "this prompt +
  this file list → done, in one shell command"; switch back to
  `aider` the moment you want to chat about the change before it
  lands.
- **"The failing test is the halting condition"** →
  [`micro-agent`](clis/micro-agent/). Bounds the agent to one file
  and rewrites it until `npm test` (or whatever `-t` is) passes,
  capped at `--max-runs N`. Observable stop condition in CI instead
  of "model says done". Pick this for leaf-module / pure-function
  TDD; switch to `aider` / `claude-code` / `opencode` the moment the
  change spans multiple files.
- **"Approve each tool call"** → `cline`, `opencode` (default).
- **"Run a sandbox, just don't break my host"** → `codex`, `OpenHands`.
- **"Run an agent in a bundled Docker sandbox with a built-in browser
  and a journal of every step"** → [`codel`](clis/codel/). One
  `docker run` brings up backend + headless browser + code editor +
  PostgreSQL journal; the agent spawns per-task worker containers via
  the host Docker socket. Lowest-friction "agent in a sandbox" demo
  in the catalog, but the project has been dormant since 2024 and the
  socket mount makes it root-equivalent on the host — pick
  [`OpenHands`](clis/openhands/) instead for the same shape with
  active maintenance.
- **"Run agent-generated code in a *hosted* Firecracker microVM that
  cold-starts in ~150 ms, scales to many concurrent sandboxes, and is
  built to be driven from `smolagents` / `crewai` / `langgraph` / your
  own framework"** → [`e2b`](clis/e2b/). The CLI manages auth, sandbox
  templates, and the lifecycle; the SDK (Python or TS) is what your
  agent actually calls. Pick this over [`container-use`](clis/container-use/)
  when you need cloud-scale, sub-second cold starts, and you are happy
  with hosted infra (or willing to self-host `e2b-dev/infra`); pick
  `container-use` instead when you want everything on your laptop with
  `git`-branch-per-agent semantics. Pick this over [`codel`](clis/codel/)
  when you need *many* concurrent sandboxes rather than one bundled
  all-in-one runtime. Skip it if you only want a single coding-CLI on
  your local machine — none of E2B's value materialises until you have
  an agent that programmatically spawns sandboxes.
- **"Agent loop with hard per-task budget caps in steps + tokens +
  USD"** → [`chatgpt-cli`](clis/chatgpt-cli/) `--agent`. ReAct or
  Plan/Execute, workdir sandbox enforced at the tool layer, allowlist
  policy for destructive shell commands, full ReAct-trace log per
  task on disk. The easiest catalog entry to *audit after the fact*
  — `cat ~/.chatgpt-cli/agent-logs/<task>.log` shows every thought,
  action, observation, token count, and final USD spend. Pick this
  over [`forge`](clis/forge/) when you want budget caps as a primary
  feature instead of a YAML field.
- **"Open a PR, I'll review on GitHub"** → `sweep`.

## 4. Do you need MCP (Model Context Protocol) servers?

- **Yes, must consume MCP servers** → `opencode`, `codex`, `cline`,
  `OpenHands`, `crush`, `continue`. All ship MCP clients.
- **Yes, want to *expose* a context-packing tool to an MCP-aware
  agent** → [`repomix`](clis/repomix/) (`repomix --mcp` exposes
  `pack_codebase`, `pack_remote_repository`, etc. over stdio).
- **Yes, want to *expose* IDE-grade symbolic edit tools (LSP-backed
  rename / find-references / replace-symbol-body) to an MCP-aware
  agent** → [`serena`](clis/serena/). `serena start-mcp-server`
  publishes `find_symbol`, `find_referencing_symbols`,
  `replace_symbol_body`, `insert_after_symbol`, etc. backed by real
  language servers (Pyright, gopls, rust-analyzer, JDT, omnisharp,
  clangd, …). Pick this when your MCP-capable agent
  ([`opencode`](clis/opencode/), [`codex`](clis/codex/),
  [`claude-code`](clis/claude-code/), [`crush`](clis/crush/),
  Cursor, Cline) is wasting tokens reading whole files to perform
  what should be one-symbol edits, especially on large polyglot
  monorepos.
- **No, prefer the CLI to bring its own tools** → `aider` (built-in repo-map),
  `gptme` (built-in shell/python/browser), `plandex` (built-in plan engine).
- **Want to *author* tools as plain shell / JS / Python scripts and
  expose them to a chat CLI without a Python framework** →
  [`llm-functions`](clis/llm-functions/). Each tool is a single
  executable file with a few `# @describe` / `# @option` argc-comment
  lines; `argc build` extracts a typed OpenAI `functions.json`
  schema from those comments. Canonical client is
  [`aichat`](clis/aichat/), but the JSON works with any
  tool-calling client. Bundled `mcp.sh` bridges any MCP server's
  tools into the same surface, so an aichat session can call MCP
  tools without aichat itself implementing MCP. Pick this over a
  Python agent framework when "tools are files in a directory" is
  exactly the model you want; pick a typed framework instead when
  you need isolation, observability, or hosted deployment.

## 5. Multi-file, multi-step plans?

- **Single edit at a time** → `aider`, `mentat`, `cline`.
- **Multi-step plan with branches** → [`plandex`](clis/plandex/).
- **Long-horizon agent loop with sub-agents** → [`opencode`](clis/opencode/)
  (Task tool), [`claude-code`](clis/claude-code/) (Task tool, hooks, skills),
  [`OpenHands`](clis/openhands/) (multi-agent), [`forge`](clis/forge/)
  (YAML workflows with per-step model routing).
- **Role-based team of cooperating agents (planner / researcher / coder /
  reviewer talking to each other)** → [`crewai`](clis/crewai/) — `Process="hierarchical"`
  puts a manager LLM over worker agents, per-agent `llm=` lets each role
  run on a different model (cheap planner, premium coder), and the CLI
  has real `crewai test --n_iterations N` / `crewai train` verbs for
  regressing and tuning crew quality. Pick this over `forge` when the
  shape is "five distinct *roles*" rather than "linear pipeline of steps".
  For a **batteries-included software-company SOP** that ships PM /
  Architect / Engineer / QA roles + a shared message bus and writes
  PRD / system-design / API-spec / task-plan as separate Markdown
  artifacts alongside the code, prefer [`metagpt`](clis/metagpt/) —
  per-role model assignment lives in three lines of `config2.yaml`.
  For the same multi-agent SDLC shape but with the **phase graph
  declared as JSON** (swap roles, phase order, loop stop conditions
  without touching Python) and the full role transcript saved as part
  of the deliverable, prefer [`chatdev`](clis/chatdev/).
- **Typed-Python agent framework with retries-on-validation-failure and a
  CLI you can chat the same agent from** → [`pydantic-ai`](clis/pydantic-ai/)
  — tools are typed Python, output is a Pydantic model, validation errors
  feed back into the loop, and `clai --agent module:var` chats with the
  *same* `Agent` your service runs (no drift between terminal and prod).
  Also publishes any agent as an MCP server via `Agent.to_mcp_server()`.
- **Code-as-action agent (model writes Python, sandbox executes it, loops
  on the output) as a framework + thin CLI** →
  [`smolagents`](clis/smolagents/) — `CodeAgent` emits `results =
  web_search(...); summary = llm(results[:3])` instead of structured
  tool-call JSON, cutting tokens and latency on multi-step tasks. Pick
  the E2B / Docker / Wasm sandbox before running anything you can't
  afford to lose; the local-Python default is fast and unsandboxed. For
  the *interactive REPL* version of the same idea, prefer
  [`open-interpreter`](clis/open-interpreter/).
- **No plan, just transform stdin** → [`mods`](clis/mods/), or
  [`llm`](clis/llm/) if you also want a queryable history of every call,
  or [`fabric`](clis/fabric/) if you want a versioned, shared library
  of prompt patterns to apply to that stdin (`pbpaste | fabric -p
  extract_wisdom`).
- **Chat with a folder of documents from a terminal** →
  [`aichat`](clis/aichat/) (built-in RAG, single binary, session-
  scoped index — best for ad-hoc "ask this folder once"), or
  [`khoj`](clis/khoj/) (long-lived daemon, persistent incremental
  index, citations back to source — best when the corpus is your
  ongoing notes vault / PDF library that you query repeatedly), or
  [`txtai`](clis/txtai/) (one level lower than `khoj`: build a
  portable on-disk index file with `python -m txtai.console`,
  `.index notes.txt`, `.save notes.index`, then query it offline
  with the same CLI — best when you want to *ship the index* to
  another machine, version-control it, or wrap it in a custom
  pipeline rather than chat with it interactively).
- **Need a written, sourced *research report* (not a chat thread) on a
  natural-language question** → [`gpt-researcher`](clis/gpt-researcher/)
  — Planner → parallel Researchers → Reviewer → Writer pipeline emits
  Markdown with inline citations plus a structured `{question,
  sub_questions, sources, findings}` JSON; pluggable search backends
  (Tavily / DuckDuckGo / SerpAPI / Searx / Bing) and a `--report-source
  local` mode that researches a folder of PDFs without web egress.
  Pick this over a generic coding agent when the deliverable is a
  cited report; pick it over `aichat` / `khoj` when you want breadth
  (10+ sources, fanned out concurrently) rather than chat over a
  pre-built index.

## 5b. Single-purpose workflow glue

Some niches are narrow enough that a dedicated CLI beats a general
agent. These are install-once-and-forget tools.

- **Auto-generate git commit messages from staged diff** →
  [`opencommit`](clis/opencommit/) (richer provider matrix,
  `prepare-commit-msg` hook story) or [`aicommits`](clis/aicommits/)
  (OpenAI-only by default, `-g N` multi-candidate picker UX)
  or [`gptcommit`](clis/gptcommit/) (single static Rust binary;
  **per-file diff summarisation** scales to 30-file commits that
  the single-prompt peers truncate).
  Pick `opencommit` if you want the hook installed once and
  `git commit` to "just work"; pick `aicommits` if you prefer
  choosing from N suggestions over editing one; pick
  `gptcommit` if you want a Rust binary and your real commits
  are routinely large.
- **Review the diff visually *and* draft / explain commits in the
  same binary** → [`lumen`](clis/lumen/). One Rust binary that
  combines a `tig`-shaped TUI diff viewer with `lumen draft`
  (conventional commit message from staged changes), `lumen explain
  <ref>` (prose explanation of any commit or range with `fzf` picker),
  and `lumen operate "<intent>"` (natural-language → `git` command).
  Pick this when you do code review *in the terminal* and want the
  LLM available without leaving it; pick `opencommit` / `aicommits`
  if you only want to draft messages and never review.
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
  code-modifying agent pipelines. Pick [`mentals-ai`](clis/mentals-ai/)
  instead if you want the *agent program itself* to be a Markdown
  file with `# Heading` mentals and `use:` / `delegate:` link syntax
  for recursive sub-agents — non-engineers can read and edit it in
  any Markdown previewer, and the runtime is a single C++ binary
  (the install tax is the C++ build; there are no prebuilt releases
  as of 2026-04).
- **Embed LLM calls as type-annotated Python functions in your own
  service or notebook** → [`magentic`](clis/magentic/). `def f(...)
  -> list[Recipe]:` with a `@prompt` decorator returns validated
  Pydantic objects, streaming-aware, native function-calling on
  OpenAI / Anthropic and ~100 more via LiteLLM. The cleanest
  structured-output API in the catalog. Closest cousin is
  [`chatblade`](clis/chatblade/) (`-e '.path' -j` extracts JSON
  sub-paths from a one-shot CLI call) — pick `chatblade` for shell
  pipelines, `magentic` when the consumer is Python code that wants
  real objects.
- **Regression-test a prompt across N providers and assert quality /
  cost / latency** → [`promptfoo`](clis/promptfoo/). Declarative
  `promptfooconfig.yaml`, cartesian eval, HTML diff viewer, optional
  red-team mode. The only entry in the catalog whose job is *grading*
  output rather than producing it; pair with [`llm`](clis/llm/) for
  exploration and [`fabric`](clis/fabric/) for prompt authoring.
- **AI-review a finished diff at PR time, no write access to your
  repo** → [`code-review-gpt`](clis/code-review-gpt/). Reads the
  diff, emits severity-tagged comments (`🔴 critical / 🟡 warning /
  🟢 nit`), optional `--ci=github` mode posts inline PR comments.
  This is the *inverse* of [`aider`](clis/aider/) /
  [`claude-code`](clis/claude-code/) (which start from a request and
  emit a patch); here you start from a finished patch and ask for
  review. Pair with [`symbex`](clis/symbex/) or
  [`repomix`](clis/repomix/) if you need cross-file reasoning that
  the diff alone cannot provide.
- **Refactor a multi-language codebase by *syntactic pattern* rather
  than by asking an LLM** → [`ast-grep`](clis/ast-grep/). Single
  static Rust binary, tree-sitter-backed, pattern language is
  isomorphic to the target language: `var $X = $Y` matches every
  `var` declaration in JS/TS regardless of identifier and rewrites
  to `let $X = $Y`. YAML rules (`all` / `any` / `not` / `inside` /
  `has` / `follows` + `fix`) compose into per-repo lint catalogs;
  `ast-grep scan --format github` posts to PRs. The deterministic
  counterpart to "have an agent refactor my codebase": when the
  change is expressible as a syntactic pattern, this is faster,
  cheaper, and correct by construction. Also exposes itself as an
  MCP server (separate `ast-grep-mcp` package) so an agent
  ([`opencode`](clis/opencode/) / [`claude-code`](clis/claude-code/) /
  [`goose`](clis/goose/)) can call it as a precise refactor primitive
  instead of doing token-expensive regex edits. Skip it when the
  refactor needs **type information** (use a TS-Morph / Roslyn /
  rust-analyzer driven script instead) or when the change is
  inherently semantic and better described in English (then reach
  for [`aider`](clis/aider/) or [`opencode`](clis/opencode/)).

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
- **Pack a tree from a single static Rust binary, with a
  Handlebars-templated export shape and an interactive TUI that
  shows a live token count as you toggle files** →
  [`code2prompt`](clis/code2prompt/). No Python, no Node; choose it
  over `files-to-prompt` when you want to **shape the wrapper
  prompt** with a custom template, and over `repomix` when you want
  hand-curation in a TUI rather than batch flags. Skip it if you
  need Tree-sitter compression or Secretlint scanning — those are
  `repomix` exclusives.
- **Pack a *remote* GitHub repo without cloning it first, and
  optionally let teammates grab the same digest by URL-renaming
  `hub` → `ingest` in their browser** →
  [`gitingest`](clis/gitingest/). Same engine as the
  `gitingest.com` web mirror, MIT, glob-filtered, single
  `digest.txt` out. Pick this over `repomix --remote owner/repo`
  when you want the **web-mirror symmetry** for non-CLI
  teammates; pick `repomix` when you need Tree-sitter compression
  or Secretlint secret-scanning that gitingest does not have.
- **Don't *pack* — *find* the relevant files first by semantic
  query, then pack only those** → [`seagoat`](clis/seagoat/).
  Local-first daemon (sentence-transformers + ChromaDB) plus a `gt`
  CLI that fuses semantic + literal + regex hits in one ranking.
  `gt "function that retries with exponential backoff" | head -20`
  gives you a file:line shortlist you can then feed into
  `files-to-prompt` / `repomix` / `aider`'s `/add` so you do not
  pack the whole repo when you only need three files. Pick this
  over the packers above when your problem is *finding* the right
  files, not packing them once you know which they are; skip it if
  `rg` is already fast enough on your repo, or if you need
  cloud-embedded multi-repo code search.

Decision shortcut:

- "Just give me one Python function": `symbex`.
- "A few selected files, glob-controlled, no Node": `files-to-prompt`.
- "The whole repo, with budget visibility and a safety net":
  `repomix`.
- "The repo lives on GitHub and I haven't cloned it (or my
  teammate hasn't installed any CLI)": `gitingest`.
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
- **Convert *messy* documents (multi-column scans, handwritten
  forms, merged-cell financial tables) where `marker` and
  `pdftotext` lose structure** → [`zerox`](clis/zerox/). Renders
  each page to PNG and asks a vision-capable LLM (GPT-4o, Claude
  3.5/3.7 Sonnet, Gemini 1.5 Pro, or local `llava` /
  `llama3.2-vision` via Ollama) to transcribe — the model *is* the
  OCR. `npx zerox doc.pdf -o ./out --maintain-format` keeps table
  / heading style consistent across long documents; `--schema
  schema.json` does typed structured extraction. Real money on
  frontier models (~$0.005–$0.02 per page) and non-deterministic;
  pick `marker` first when its output is usable, fall back to
  `zerox` when it isn't.
- **Convert a *heterogeneous corpus* (PDF + DOCX + PPTX + XLSX +
  audio + XBRL/JATS) behind one CLI, fully offline, with a typed
  IR you can re-export from** → [`docling`](clis/docling/). Linux
  Foundation AI & Data project (donated by IBM Research, MIT,
  active multi-contributor development); default pipeline is local
  layout + table + OCR (no LLM, no network); optional
  `--pipeline vlm --vlm-model granite_docling` mode runs a 258M
  document VLM with MLX acceleration on Apple Silicon. The killer
  trick is the unified `DoclingDocument` JSON IR: convert once,
  ship the JSON, then re-render Markdown / HTML / chunks tuned
  for your RAG without re-running the heavy ML pipeline. Also
  exposes itself as an MCP server (`docling mcp`) so coding agents
  can hand it a document directly. Pick `marker` if your inputs
  are PDF-only and you want the most polished single-format
  pipeline; pick `docling` if your corpus is mixed-format or you
  need the typed IR; pick `zerox` if your inputs are adversarial
  scans and you can afford a frontier vision model.

## 5d.1. Web pages → LLM-ingestible Markdown

Source-tree packers (§5c) and document converters (§5d) cover repos
and files. The third input shape is the open web — SPAs, JS-rendered
articles, paginated docs sites — which `curl | html2text` mangles.

- **Fetch a URL (or crawl a site) and get clean Markdown an LLM can
  reason over** → [`crawl4ai`](clis/crawl4ai/). Playwright-backed
  fetch handles SPAs and infinite scroll, then a `fit_markdown`
  content filter drops nav / footer / ads using heuristic main-
  article extraction (5–10× token reduction vs. raw `html2text`).
  `crwl <url>` for one-shot, `crwl <url> --depth 2 --max-pages 50`
  for BFS crawl, `crwl <url> -s schema.json` for typed extraction.
  Heavy install (~300 MB browser binaries) — use `curl |
  html2text` if your inputs are static and small. Doubles as an
  MCP server: `docker run unclecode/crawl4ai` exposes the same
  surface to opencode / codex / cline so a coding agent gets a
  browser without you wiring one.

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
- **Write LLM evals as ordinary pytest tests and run them in CI
  alongside unit tests** → [`deepeval`](clis/deepeval/). Same goal
  as `promptfoo` (gating quality, not producing it), opposite
  shape: instead of a YAML matrix, you write `test_*.py` files
  that build `LLMTestCase` objects and assert against metrics
  (G-Eval / DAG for custom criteria, plus a deep catalog of
  agentic — task completion, tool correctness, plan adherence,
  step efficiency — RAG, multi-turn, and MCP-specific metrics).
  `deepeval test run tests/llm/` reuses pytest collection,
  fixtures, parallelism, and JUnit XML, so a 50-case eval suite
  drops into existing CI in half a day. Pick `promptfoo` if your
  team prefers declarative YAML and an HTML diff viewer over a
  Python test file; pick `deepeval` if you live in pytest already
  and want one test command for unit, integration, and eval, or
  if you specifically need agentic / MCP metrics. Default
  telemetry is on — set `DEEPEVAL_TELEMETRY_OPT_OUT=YES` in your
  CI image.
- **Lift accuracy on a specific reasoning step *without* changing
  client code, by spending more inference-time compute** →
  [`optillm`](clis/optillm/). Drop a single OpenAI-compatible proxy
  in front of your provider, prefix the model name on just the call
  that matters (`model="moa-gpt-4o-mini"` for Mixture-of-Agents,
  `model="mcts-claude-3-5-sonnet"` for Monte-Carlo tree search,
  `model="cepo-llama-3.3-70b"` for Cerebras' CePO planning loop),
  and the proxy runs N parallel upstream calls + synthesis behind
  one response your client sees as plain `chat.completions`.
  Published gains are real (MARS over Gemini 2.5 Flash Lite: +30 pts
  on AIME 2025; CePO over Llama 3.3 70B: +18.6 on Math-L5) but the
  cost is N× tokens / N× latency / N× $, so use it *selectively* —
  route only the architectural-plan / root-cause / hard-math step
  through `optillm`, not every chat turn, and A/B with `promptfoo`
  or `deepeval` to confirm the technique wins on your distribution.
  Pair upstream with [`mlx-lm`](clis/mlx-lm/) or
  [`openllm`](clis/openllm/) for a fully self-hosted "cheap small
  model + inference-time compute = strong reasoning" stack. Skip if
  you'd rather pay per-token for a natively reasoning-tuned model
  (DeepSeek-R1, OpenAI o-series, Claude with extended thinking) than
  pay per-N-samples for ensemble inference, or if your bottleneck is
  *tool use* rather than *reasoning quality* (that's an agent-loop
  problem, not an inference-proxy problem).

## 5f. Operational data → AI (log analysis)

Source-tree packers and document converters assume your input is
something a human *wrote*. For machine-emitted streams — application
logs, syslog, k8s events — the right move is to mine first and only
then summarize.

- **Find the needle in a multi-GB log haystack and get a plain-
  English explanation** → [`logai`](clis/logai/). Two-stage
  pipeline: classical log mining (Drain / IPLoM template extraction,
  TF-IDF / Word2Vec embeddings, Isolation Forest / DBSCAN anomaly
  scoring) collapses millions of lines to dozens of templates, then
  an *optional* LLM stage summarizes only the residual anomalies in
  natural language. Run pure-classical with no API key, or enable
  the LLM stage for narrative incident reports. The only "ops data
  → AI" entry in the catalog; nothing else here touches log volume
  at this shape.
- **Triage a misbehaving Kubernetes cluster, get a one-line-per-
  issue list of broken objects with optional plain-English
  paragraphs** → [`k8sgpt`](clis/k8sgpt/). Deterministic Go
  analyzers walk the live cluster API (Pods, Services, Ingress,
  HPAs, PDBs, webhooks, gateways…) and emit findings on their own;
  `--explain` is a *post-hoc* narration layer powered by OpenAI /
  Azure / Bedrock / Vertex / Cohere / LocalAI / Ollama. Pick this
  over an exploratory agent (`OpenHands`, `claude-code` with
  `kubectl`) when you need bounded token cost and reproducible
  findings; pick the agents instead when you need *remediation*,
  not just diagnosis. Pick [`kubectl-ai`](clis/kubectl-ai/)
  instead when you want a **chat-shaped** Kubernetes assistant
  that runs `kubectl` itself in an agent loop (Gemini / OpenAI /
  Anthropic / Vertex / Ollama / llama.cpp), with the bonus that
  `kubectl-ai --mcp-server` mounts a curated kubectl toolset
  inside [`opencode`](clis/opencode/) /
  [`claude-code`](clis/claude-code/) / [`crush`](clis/crush/) /
  [`continue`](clis/continue/) so your generic coding agent
  inherits safe cluster powers without you hand-rolling an MCP
  server. Pick [`holmesgpt`](clis/holmesgpt/) instead when the
  shape is **SRE investigation across multiple stacks** —
  Kubernetes *plus* Prometheus / Grafana / Loki / Tempo / Datadog
  / NewRelic / Sentry / Elasticsearch / Postgres / Kafka /
  AWS / GCP / ArgoCD / Helm / Confluence / GitHub / Jira /
  PagerDuty / OpsGenie under one CNCF-sandbox agent, with
  petabyte-aware tool-output handling and write-back to the
  source channel; the install tax is heavier than `k8sgpt` /
  `kubectl-ai`, but it is the only catalog entry that ingests
  alerts, runs the investigation, and posts the RCA back where
  the on-call human looks.

## 5g. Token budgeting and HTML cleaning (front-ends to the LLM call)

The packers in 5d/5e produce text. Two more upstream steps are
worth their own slot: **measuring** the cost of that text before
spending it, and **cleaning** raw web HTML so it is worth measuring
in the first place.

- **Measure exactly how many tokens a packer just emitted (and
  optionally cap the stream)** → [`ttok`](clis/ttok/). One-shot
  Unix filter wrapping `tiktoken`. `… | ttok` prints the count;
  `… | ttok --truncate 100000` caps it. Counts are exact for
  OpenAI tiktoken-vocab models, useful upper-bound estimates for
  Anthropic / Gemini / Llama. The cheap pre-flight gate that makes
  every `files-to-prompt` / `symbex` / `repomix` / `strip-tags` /
  `marker` invocation steerable instead of "discover the bill via
  an API error".
- **Strip a webpage to text before piping it into an LLM CLI** →
  [`strip-tags`](clis/strip-tags/). `bs4`-backed HTML→text filter
  with CSS-selector targeting (`strip-tags article` keeps only the
  `<article>` content) and a `--minify` mode that preserves
  structure but drops chrome. Pair with `curl` for static pages,
  with a headless-browser pre-renderer for SPAs. The web-content
  analogue to `marker` (PDF → markdown) and `files-to-prompt`
  (source tree → packed prompt).

## 6. Team / org constraints

- **Telemetry must be off by default** → all entries marked "Off" in the
  matrix: `opencode`, `aider`, `cline`, `crush`, `continue`, `mentat`,
  `gptme`, `smol-developer`, `mods` (no telemetry at all), `forge`,
  `tgpt` (no telemetry; note that free providers see prompts), `fabric`,
  `opencommit`, `aicommits`, `aiac`, `symbex` (no network at all),
  `repomix` (no analytics; only egress is the optional `--remote` git
  clone), `chatblade`, `promptfoo` (eval cache local; `share` is
  opt-in), `marker` (no network unless `--use_llm` configured),
  `gptscript`, `shell-genie` (no analytics; `feedback` writes locally
  only), `elia` (no telemetry; history stays in local SQLite),
  `khoj` (off when run with `--anonymous-mode`),
  `code-review-gpt` (no analytics; only egress is the LLM provider),
  `logai` (no analytics; LLM egress only if summarization stage
  enabled), `txtai` (no analytics; fully offline by default),
  `k8sgpt` (no analytics; LLM egress only when `--explain` set),
  `ttok` (one-time vocab download, then offline), `strip-tags`
  (no network at all), `tlm` (no telemetry; only egress is your
  local Ollama endpoint), `smartcat` (no telemetry; conversation
  file local),   `ramalama` (no telemetry; container/model pulls
  visible to the OCI/Hugging Face/Ollama upstreams), `magentic`
  (no analytics; egress only to the configured provider),
  `mentals-ai` (no analytics; local `.log` traces), `ai-shell`
  (no analytics; configured OpenAI-compatible endpoint sees prompts),
  `ollama` (no telemetry; only egress is `registry.ollama.ai` on
  `pull` and your own clients on `:11434`), `llm-cmd` (inherits
  `llm`'s no-telemetry posture; logs locally to SQLite),
  `gitingest` (CLI does not phone home; remote fetches go directly
  to GitHub), `parllama` (no analytics; egress is your Ollama daemon
  plus the active provider plus `registry.ollama.ai` on `pull`),
  `gpt-cli` (no analytics; per-turn USD cost computed locally;
  `--log_file` writes JSONL to local disk only),   `promptr` (no
  analytics; egress is exactly one OpenAI Chat Completions request
  per invocation), `kubectl-ai` (no analytics; egress = Kubernetes
  API + LLM provider + any MCP servers you mount), `holmesgpt`
  (off in OSS codebase; egress = LLM provider + configured toolsets
  + opt-in write-backs),   `harbor` (no analytics in CLI or App;
  egress = Docker image pulls + the active backend's model
  registry), `serena` (no analytics; egress = locally-spawned
  LSP processes + the connected MCP client; optional dashboard
  binds `127.0.0.1` only), `seagoat` (no analytics; daemon binds
  locally, embedding is local, queries hit `127.0.0.1`),
  `kit` (no analytics; egress = the OCI registries you push/pull
  plus HuggingFace on `kit import` plus your Cosign / KMS endpoint),
  `agno` (off in OSS; egress = your model providers + your DB;
  AgentOS UI and OTel export are opt-in),
  `claude-code-router` (no analytics; egress = the providers you
  declare; proxy + UI bind to `127.0.0.1` only).
- **Permissive license required (no AGPL, no GPL)** → avoid `plandex`
  (AGPL core), `open-interpreter` (AGPL-3.0), `khoj` (AGPL-3.0),
  `patchwork` (AGPL-3.0), `tenere` (GPL-3.0), and
  `tgpt` (GPL-3.0).
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
| Want a single Go binary that gives you natural-language → shell command against a *local* Ollama, no cloud egress, with a `tlm install` bootstrap | [`tlm`](clis/tlm/) |
| Want the simplest possible "natural language → one shell command → y/n" with shell-aware prompt templates and no flags to learn | [`shell-genie`](clis/shell-genie/) |
| Want a **multi-candidate shell-command picker** that you can flip between OpenAI / Azure / Groq / Ollama / Mistral with one env var, with optional context-mode that pipes recent shell output into the prompt | [`shell-ai`](clis/shell-ai/) |
| Want an LLM in a fresh terminal with no API key, right now | [`tgpt`](clis/tgpt/) |
| Want a team-shared, version-controlled prompt-pattern library | [`fabric`](clis/fabric/) |
| Want to pack a whole subtree as deterministic context for any LLM CLI | [`files-to-prompt`](clis/files-to-prompt/) |
| Want a chat TUI that talks *only* to a local Ollama, with multi-tab + MCP | [`oterm`](clis/oterm/) |
| Want a shell-command suggester that shows *multiple* candidates to pick from | [`gorilla-cli`](clis/gorilla-cli/) |
| Auto-fill commit messages via a git hook, no new command in your workflow | [`opencommit`](clis/opencommit/) |
| Pick from 3 candidate commit messages with arrow keys instead of editing one | [`aicommits`](clis/aicommits/) |
| TUI diff viewer + AI commit-draft + AI commit-explain in one Rust binary | [`lumen`](clis/lumen/) |
| Boot an autonomous agent (LLM + browser + editor + journal) in one `docker run` | [`codel`](clis/codel/) |
| Multi-step agent loop with hard per-task budget caps in steps + tokens + USD | [`chatgpt-cli`](clis/chatgpt-cli/) |
| Scaffold a fresh Terraform / K8s / Dockerfile from a one-line description | [`aiac`](clis/aiac/) |
| Pack a Python codebase down to *specific symbols* instead of whole files | [`symbex`](clis/symbex/) |
| Pack a whole repo (or a remote repo you have not cloned) with token counts, secret-scan, and optional compression | [`repomix`](clis/repomix/) |
| Make the model reply as JSON/YAML and extract a sub-path in one shell command | [`chatblade`](clis/chatblade/) |
| Convert a PDF / EPUB / DOCX into Markdown that preserves tables and equations | [`marker`](clis/marker/) |
| Ship a runnable, version-controlled `.gpt` agent script alongside your code | [`gptscript`](clis/gptscript/) |
| Regression-test a prompt across providers and fail CI when quality / cost / latency budget breaks | [`promptfoo`](clis/promptfoo/) |
| Multi-provider terminal chat TUI with persistent SQLite history and `/`-search across past conversations | [`elia`](clis/elia/) |
| Long-lived local daemon that indexes your notes / PDF / code corpus and answers chat queries with citations, fully offline-capable | [`khoj`](clis/khoj/) |
| AI-review a finished diff at PR time, post inline GitHub PR comments, no write access to the repo | [`code-review-gpt`](clis/code-review-gpt/) |
| Find the needle in multi-GB application logs, plain-English summary of anomaly clusters, classical mining + optional LLM summarization | [`logai`](clis/logai/) |
| Build a portable on-disk embeddings index (sentence-transformers + Faiss-CPU), ship it offline, query it with the same CLI | [`txtai`](clis/txtai/) |
| Triage a misbehaving Kubernetes cluster — deterministic analyzer sweep over the live API, optional plain-English explanations | [`k8sgpt`](clis/k8sgpt/) |
| Count tokens of any packer's output before paying for the API call (and optionally truncate to a hard cap) | [`ttok`](clis/ttok/) |
| Strip a `curl`'d webpage to clean text (or minified HTML) for piping into any LLM CLI | [`strip-tags`](clis/strip-tags/) |
| Named system prompts in a TOML config file, swappable with one flag in a Unix pipeline (`sc -p commit < diff.txt`), plus glob input (`sc -g 'src/**/*.rs' "..."`) | [`smartcat`](clis/smartcat/) |
| Run local GGUF models with hardware-aware container selection (CPU / CUDA / ROCm / Metal / Vulkan, same command line) and an OpenAI-compatible HTTP endpoint other catalog CLIs can target | [`ramalama`](clis/ramalama/) |
| On Apple Silicon, fastest local decode the hardware can do — `mlx_lm.server` exposes any MLX-converted model (Llama 3.x/4.x, Qwen 2.5/3, DeepSeek-R1, Mistral, Phi, Gemma) behind an OpenAI-compatible URL with no Docker; ~4900 ready-to-run conversions in the `mlx-community` HF org; small orthogonal CLI family (`generate`/`chat`/`server`/`convert`/`lora`/`fuse`) | [`mlx-lm`](clis/mlx-lm/) |
| Production-shaped self-hosted LLM serving on NVIDIA GPUs — `openllm serve llama3.3:70b` pins backend (vLLM / Transformers / SGLang) + dtype + chat template + GPU shape in one reproducible bento manifest, ships a `/chat` UI, same manifest deploys to a managed BentoCloud cluster | [`openllm`](clis/openllm/) |
| Lift accuracy on hard reasoning steps with no client change — drop an OpenAI-compatible proxy in front of your provider, prefix the model name (`moa-gpt-4o-mini`, `mcts-claude-3-5-sonnet`, `cepo-llama-3.3-70b`) to switch between ~20 inference-time-compute techniques (MoA, MCTS, BoN, Self-Consistency, CePO, MARS, AutoThink, LongCePO) | [`optillm`](clis/optillm/) |
| Embed LLM calls as type-annotated Python functions, get validated Pydantic objects back (streaming-aware), no agent loop | [`magentic`](clis/magentic/) |
| Write the agent program as a Markdown file with `# Heading` mentals and `use:` / `delegate:` links — readable by non-engineers, runnable by a single C++ binary | [`mentals-ai`](clis/mentals-ai/) |
| One-shot natural-language → shell command with an iterative `[R]evise` loop so you can nudge the suggestion without retyping the original intent | [`ai-shell`](clis/ai-shell/) |
| Local LLM runtime: `brew install ollama && ollama run llama3.1` in five minutes, then OpenAI-compatible HTTP at `:11434` for the rest of the catalog to point at | [`ollama`](clis/ollama/) |
| Suggestion lands in your *readline buffer* (no TUI dialog, no menu) so your shell's normal line editor handles edit-then-Enter — for users already on [`llm`](clis/llm/) | [`llm-cmd`](clis/llm-cmd/) |
| Pack a remote GitHub repo without cloning, with a hosted web mirror so non-CLI teammates can grab the same digest by URL-renaming `hub` → `ingest` | [`gitingest`](clis/gitingest/) |
| Hand a non-technical user (or an air-gapped machine) one file that boots a local LLM with no install — same byte-for-byte executable on macOS / Linux / Windows / *BSD, x86-64 and arm64, weights included | [`llamafile`](clis/llamafile/) |
| Point an existing OpenAI-SDK app at one local server that serves chat **and** embeddings **and** Whisper transcription **and** image generation **and** TTS **and** reranking under the same `/v1/...` routes | [`localai`](clis/localai/) |
| Run one PR-review / `/describe` / `/improve` / `/ask` bot across more than one SCM (GitHub + GitLab + Bitbucket + Azure DevOps + Gitea + Gerrit) from one `.pr_agent.toml` | [`pr-agent`](clis/pr-agent/) |
| Multi-screen Textual TUI whose *Models* screen doubles as a full Ollama registry console (pull / delete / copy / Modelfile-build / GGUF-quantize) plus chat tabs against any local or cloud provider | [`parllama`](clis/parllama/) |
| Named persona presets (`gpt rust`, `gpt bash`, `gpt <your-name>`) defined in one YAML, with first-class Claude `--thinking N` and a per-turn USD cost counter — pure chat REPL, no agent loop | [`gpt-cli`](clis/gpt-cli/) |
| Templated prompt files in the repo + a file list on the command line; the model overwrites those files in place with `git diff` as the only review surface — built for Makefile / CI / git-hook use, not chat | [`promptr`](clis/promptr/) |
| `prompt`-file → whole-repo greenfield scaffold from one paragraph of intent, runnable `run.sh` at the end, spec lives as a git-tracked artifact (not a chat log) | [`gpt-engineer`](clis/gpt-engineer/) |
| Single Go binary that fuses **shell-command exec mode + chat mode** in one Bubble Tea TUI, `Tab` to toggle, `[E]xecute/[A]nswer/[C]opy/[X]cancel` confirm bar | [`yai`](clis/yai/) |
| Natural-language → shell command but with **prerequisite install commands separated as first-class output** (`s`=setup / `d`=desired / `a`=all hotkey menu) — fixes "the suggestion assumed I had `jq` installed" | [`cmdh`](clis/cmdh/) |
| Chat-shaped Kubernetes assistant that runs `kubectl` itself in an agent loop, **and** doubles as an MCP server so [`opencode`](clis/opencode/) / [`claude-code`](clis/claude-code/) / [`crush`](clis/crush/) inherit safe `kubectl` powers without you hand-rolling an MCP server | [`kubectl-ai`](clis/kubectl-ai/) |
| Cross-stack SRE investigation agent — Kubernetes + Prometheus + Grafana + Loki + Tempo + Datadog + NewRelic + Sentry + Postgres + Kafka + AWS + GCP + ArgoCD + Helm + Confluence + GitHub + Jira + PagerDuty + OpsGenie — under one CNCF-sandbox agent that ingests alerts and posts RCAs back to the source channel, with petabyte-aware tool-output handling | [`holmesgpt`](clis/holmesgpt/) |
| One command brings up the **whole local LLM stack** (chat UI + runtime + web RAG + voice + image generation) on Docker Compose with cross-service URLs already correct, instead of hand-maintaining a Compose file forever — `harbor up open-webui ollama searxng speaches comfyui` | [`harbor`](clis/harbor/) |
| Want a **transparent shell wrapper** that records every command + output and lets `!` prefix turn the next line into NL→suggested-command (or `!!` into autonomous multi-step) inside your normal interactive shell, with a hotkey to ask about the last output | [`butterfish`](clis/butterfish/) |
| Want a disposable **generate-Python → exec → feed-stdout-back** loop in your real cwd (no sandbox) for one-off "do this in my filesystem" tasks, single `pip install` away | [`rawdog`](clis/rawdog/) |
| Want `mods`-shaped Unix-pipe ergonomics but with a **built-in glob/URL context feeder** (`clai -i 'src/**/*.go' query "..."`, `-i URL` for inline web fetch) so you do not have to chain a separate packer first, plus re-enterable named conversations | [`clai`](clis/clai/) |
| Want a chat TUI **glued to a running Neovim or VS Code** — yank a visual selection straight into the chat as a fenced block, accept the model's reply back into the editor buffer at the cursor, no clipboard middleman that breaks indentation; project quiet since 2024-03 | [`oatmeal`](clis/oatmeal/) |
| Want **modal vim editing inside the prompt buffer** plus multi-tab independent chats with crash-resume of the last saved chat, behind a single ~5 MB Rust binary — narrower backend matrix (OpenAI / `llama.cpp` / Ollama only), no agency, no MCP | [`tenere`](clis/tenere/) |
| Want **agentic workflow automation that ends in a PR** (`AutoFix` over Semgrep findings, `DependencyUpgrade` over `depscan`, `ResolveIssue` with chromadb RAG, `GenerateDocstring`, `GenerateUnitTests`, `PRReview`, `GenerateREADME`) — declared as YAML + reusable Python Steps, identical invocation on a laptop and in CI | [`patchwork`](clis/patchwork/) |
| Want to **run several agents in parallel on the same prompt without trashing your working tree** — each agent lands in its own container on its own git branch, exposed as plain MCP tools so any MCP-capable client ([`opencode`](clis/opencode/), [`claude-code`](clis/claude-code/), [`crush`](clis/crush/), [`goose`](clis/goose/)) inherits the sandbox without you writing Docker plumbing | [`container-use`](clis/container-use/) |
| Want a **research-grade autonomous agent with reproducible trajectories** — every action + observation + tool output saved as typed JSON, the whole agent (prompts, tools, parser, demos) is one diffable YAML, fan-out batch mode for SWE-bench-class evals; trajectory replay via `sweagent inspect` | [`swe-agent`](clis/swe-agent/) |
| Want **a typed multi-role pipeline (File Picker → Planner → Editor → Reviewer) checked into your repo** instead of one heroic agent with opaque sub-agent escape hatches — `.agents/types/*.ts` declares each role's tool allowlist and which other roles it may spawn, with per-role model assignment | [`codebuff`](clis/codebuff/) |
| Want a **cited long-form research report** (not a chat thread) — Planner → parallel Researchers → Reviewer → Writer pipeline, swappable search backends (Tavily / DDG / SerpAPI / Searx / Bing), `--report-source local` for offline PDF research, structured JSON of sources + findings dumped alongside the Markdown | [`gpt-researcher`](clis/gpt-researcher/) |
| Want a **declarative SDLC pipeline as JSON** — phase graph (`ChatChainConfig.json`) pairs roles per turn (CEO ↔ CPO, CTO ↔ Programmer, CRO ↔ Programmer, Programmer ↔ Tester) and composed phases loop until file-complete / no-comments / tests-pass; full multi-agent transcript saved as part of the deliverable | [`chatdev`](clis/chatdev/) |
| Want a **batteries-included software-company SOP** — PM → Architect → ProjectManager → Engineer → QA roles on a shared message bus, PRD + system design + API spec + task plan as separate Markdown / JSON artifacts alongside the code, per-role model assignment in three lines of `config2.yaml` | [`metagpt`](clis/metagpt/) |
| Multi-candidate (`-g`-style) **shell-command picker** that switches between OpenAI / Azure / Groq / Ollama / Mistral with one env var (`SHAI_API_PROVIDER`), with an opt-in `CTX=true` mode that pipes recent shell output into the prompt for follow-up disambiguation | [`shell-ai`](clis/shell-ai/) |
| **Single static Rust binary** for the `prepare-commit-msg` git-hook niche, with **per-file diff summarisation** so a 30-file commit becomes 30 small prompts + one rollup instead of one giant prompt that truncates | [`gptcommit`](clis/gptcommit/) |
| Iterate on **prompts as code** with automatic local-SQLite versioning + autogenerated commit messages per `@ell.simple` decorated function, plus a local web UI (`ell-studio`) with a side-by-side diff viewer for every version of every "language model program" you have ever called | [`ell`](clis/ell/) |
| Expose **IDE-grade symbolic edit tools** (LSP-backed rename / find-references / replace-symbol-body) as MCP tools to your existing MCP-capable coding agent ([`opencode`](clis/opencode/) / [`codex`](clis/codex/) / [`claude-code`](clis/claude-code/) / [`crush`](clis/crush/) / Cursor / Cline), so it stops doing line-number text surgery on large polyglot monorepos | [`serena`](clis/serena/) |
| **Find the right files first by semantic + literal + regex fusion** (local sentence-transformers index, no cloud upload of source) before handing them to a packer or an agent — `gt "where do we round numbers"` is the workflow | [`seagoat`](clis/seagoat/) |
| **Hosted Firecracker microVM sandbox per LLM-tool-call** for agents you build in `smolagents` / `crewai` / `langgraph` / your own framework — ~150 ms cold start, many concurrent sandboxes, none of the docker-socket-mount footguns of laptop runners | [`e2b`](clis/e2b/) |
| Want LLM calls **as type-annotated Python functions** that return validated Pydantic objects (`@marvin.fn def extract_invoice(text) -> Invoice`) for app code / notebooks / glue scripts — typed `classify` / `extract` / `cast` / `generate` primitives, no agent loop, no framework forest | [`marvin`](clis/marvin/) |
| Want a **self-hostable end-to-end agent stack** — inline completion + chat + autonomous agent + PR opening behind one Docker image, integrations (`postgres` / `chrome` / `github` / `jira` / `slack`) wired up as YAML, agent loop on-prem while still reaching cloud LLMs for the chat tier (telemetry on by default — flip off explicitly) | [`refact`](clis/refact/) |
| Want a **readable ~500-line dependency-light Python chat client** with streaming + structured Pydantic-schema output + session save/load + parameter control — no LangChain, no agent loop, no hidden prompts; the right substrate for glue scripts and notebooks where pulling a 500 MB framework to make one chat call is the actual problem (project in maintenance mode — last v0.2.2) | [`simpleaichat`](clis/simpleaichat/) |
| Want **one immutable, signable, content-addressable bundle for the *entire* AI/ML thing** — model weights + datasets + prompts + agent specs + MCP server bundles — stored in the OCI registry you already operate, with full Cosign signing, AI SBOM generation, ModelPack interop, and layered storage that diff-shares unchanged weights across versions; CNCF-governed substrate, not an agent or model client | [`kit`](clis/kit/) |
| Want a Python **agent framework where the agent you build *is* the production API in ~20 lines** — declare `Agent` / `Team` / `Workflow`, get a stateless session-scoped FastAPI runtime (`AgentOS`) with per-user memory + streaming + a stable HTTP+WS surface for free, with per-agent model assignment across 30+ providers and an opt-in hosted observability UI | [`agno`](clis/agno/) |
| Want **per-task model routing for an agent harness that hardcoded one provider** — keep the upstream agent's UX (sub-agents, hooks, MCP) but bind `default` / `think` / `longContext` / `background` / `webSearch` to different upstream models (DeepSeek for cheap edits, GPT-5 for hard planning, Gemini 2.5 Pro for 1M-context, local Qwen-Coder via Ollama for background summaries), with a transformer registry rewriting request shapes per provider | [`claude-code-router`](clis/claude-code-router/) |
| Want an Anthropic agent that **grows its own tool surface as you use it** — when it hits a capability gap mid-conversation, the `toolcreator` meta-tool writes a new Python tool into `tools/`, hot-loads it into the live registry, and uses it on the next turn, with the new tool persisting across future sessions instead of needing a fresh MCP server per gap | [`claude-engineer`](clis/claude-engineer/) |
| Want a **persistent local code-search daemon** that fuses regex + tree-sitter symbol search + semantic search into one ranked list across many checkouts, with line-anchored citations into source, exposed both as a desktop UI *and* a documented HTTP API so you can also wire an external coding agent at the same backend (project in maintenance mode — last v0.6.5) | [`bloop`](clis/bloop/) |
| Want a **self-hosted inline-completion + chat backend for a team** — one `tabby serve` brings up an OpenAI-compatible endpoint plus IDE-plugin protocol plus a `tabby scheduler` repo-indexing daemon for retrieval-augmented completion on open-weights coder models (StarCoder / DeepSeek-Coder / Qwen-Coder), reusable as the upstream for [`aider`](clis/aider/) / [`continue`](clis/continue/) / [`shell-gpt`](clis/shell-gpt/) via `--openai-api-base` | [`tabby`](clis/tabby/) |
| Want a **stateful, role-decomposed greenfield app build from a paragraph of spec** — Product Owner → Architect → Tech Lead → Developer → Code Monkey → Reviewer pipeline whose project state lives in PostgreSQL/SQLite, so killing the process mid-build and resuming tomorrow picks up at the exact next task instead of "lost context"; spec-driven with the PO agent asking clarifying questions instead of guessing — greenfield only, not for adding a feature to a 200k-LOC repo | [`gpt-pilot`](clis/gpt-pilot/) |
| Want **editor-agnostic AI through the protocol every modern editor already speaks** — one shared TOML config gives Helix / Neovim / VS Code / Emacs the same inline completion + chat + code-actions, with named model pools so inline completion can route to a fast local model while "refactor this" routes to Claude in the same session, and zero outbound sockets beyond the LLM provider you configured | [`lsp-ai`](clis/lsp-ai/) |
| Want **per-task model routing in two-line tools** for the boring-but-frequent git + i18n chores — `lobe-commit` (gitmoji-style AI commit messages with interactive `e`/`r` reword) and `lobe-i18n` (LLM-driven JSON locale-file translation, diff-only updates) ship from one monorepo + one config style + one OpenAI-compatible endpoint env var, so you can point commits at a $0.10/M-token model and translation at GPT-4-class without standing up a full agent harness | [`lobe-cli-toolbox`](clis/lobe-cli-toolbox/) |
| Want a **decorator-defined MCP-native agent crew in ~30 lines of Python** — `@chain` / `@parallel` / `@router` / `@orchestrator` / `@evaluator_optimizer` over typed Python functions, the same `agent.py` becomes a REPL (`fast-agent go`), a one-shot (`fast-agent send`), and an MCP/ACP server other agents can call; opinionated — if it isn't an MCP server, this framework doesn't see it | [`fast-agent`](clis/fast-agent/) |
| Want an **agent-agnostic human-approval inbox** that gates every shell / write call from `claude-code` + `codex` + `opencode` + `amp` workers running in parallel on the same task, routable to Slack / email / a desktop TUI queue, with a per-worker audit log — the human-decision counterpart to [`container-use`](clis/container-use/)'s sandbox layer (codelayer desktop app is nightly-only at snapshot, no stable v1 tag yet) | [`humanlayer`](clis/humanlayer/) |
| Want **AST-anchored slices instead of line-bound `grep` output** — `gast 'BACKOFF_MS' src/` returns each enclosing function or method body that mentions the symbol with the signature on top, the shape an LLM actually needs to answer "what changes if I bump this constant?"; sits between `rg` (one-line hits) and a full repo-map, narrower than [`files-to-prompt`](clis/files-to-prompt/) (whole files), broader than [`symbex`](clis/symbex/) (Python-only exact symbols) | [`grep-ast`](clis/grep-ast/) |
| Want an agent that **genuinely remembers you across weeks of separate sessions** — `core_memory_replace` lets the LLM rewrite its own system prompt, `archival_memory_*` tool calls page unbounded long-term memory in and out of the window, every block durably persisted to Postgres/SQLite; the MemGPT successor with the durability story baked into the architecture instead of bolted on as a vector-store side-table; trade-off is the heaviest dependency surface in the catalog (FastAPI + Postgres + Alembic + OTel + Sentry + Datadog + temporalio) and a separate `letta-code` npm CLI for the terminal client | [`letta`](clis/letta/) |
| Want a **production-grade text-to-SQL agent stack** — schema introspection → schema-aware retrieval → SQL synthesis → execution → result interpretation → chart rendering wired as one AWEL DAG with safe-mode SQL filtering, row-cap injection, dry-run plan preview, per-database privilege scoping, and a connector catalog (MySQL/PG/ClickHouse/Snowflake/BigQuery/Doris/StarRocks/Hive/Spark/Oracle/MSSQL/DuckDB/Milvus/Chroma/Weaviate) long enough that "let analysts ask questions in natural language" is a one-day install instead of a six-month project; web UI is the primary surface, CLI boots the server | [`db-gpt`](clis/db-gpt/) |
| Want a **lightweight YAML-defined workflow runner for Qwen-family models** — 30 lines of YAML declare typed steps (`LLMStep` / `ToolStep` / `BranchStep` / `LoopStep` / `SubWorkflowStep`), per-step model + tool-subset + system prompt; `ms-agent run that.yaml` executes with a live `rich` tree view + durable local trace + the same MCP plumbing the Python API uses, install in seconds, surface fits in a tab of docs; deliberate trade-off vs. [`letta`](clis/letta/) — static DAG and no runtime manager-LLM in exchange for predictable execution and easy tracing | [`ms-agent`](clis/ms-agent/) |
| Want **typed structured outputs from any LLM provider via one Pydantic class** — `client.chat.completions.create(response_model=Order, ...)` works against OpenAI, Anthropic, Gemini, Mistral, Cohere, Groq, Bedrock, Vertex, Ollama with the right structured-output mode picked per provider, automatic re-ask on `ValidationError`, and `create_partial(...)` streams progressively-filled instances for live UI rendering — the right substrate when your output schema has real constraints (positive ints, regex, enums, cross-field invariants) and you'd otherwise write a per-provider JSON-parsing wrapper | [`instructor`](clis/instructor/) |
| Want a **reactive Python notebook with no hidden state** — cells are a dataflow DAG (not an ordered list), so editing one cell re-runs every dependent cell and deleting one GC's its variables everywhere; the file on disk is `.py` (readable `git diff`, works with `pre-commit` / `pytest`); `marimo run nb.py` turns the same file into a hosted web app with no rewrite; in-cell AI + an MCP-aware chat panel operate on the same typed-cell substrate you do | [`marimo`](clis/marimo/) |
| Want a **synthetic-data + AI-feedback pipeline framework** that ships published-paper recipes (UltraFeedback, EvolInstruct, MagPie, SelfInstruct, DEITA, PrometheusEval, PairRM) as named `Step`s you compose with `>>` — Ray + vLLM scales the expensive labelling step (e.g. 70B reward model over 100k pairs) across a cluster, on-disk cache resumes a crashed 8h run from the last checkpoint, output is a HuggingFace `Dataset` ready to push to the Hub for downstream SFT / DPO / KTO trainers | [`distilabel`](clis/distilabel/) |
| Want a framework where **prompts are an output of compilation, not an input** — declare a typed `Signature` + `Module` + metric, hand to a `Teleprompter` (`BootstrapFewShot`, `MIPROv2`, `COPRO`, `SIMBA`) that searches the joint space of system-prompt wording + few-shot demos + per-module decoding params on a small dev set, ship the resulting `compiled.json` to git, and re-compile (instead of re-prompt) every time you swap providers or model versions | [`dspy`](clis/dspy/) |
| Want **token-level constraint masking** so a small local model emits provably valid JSON Schema / Pydantic / regex / Lark-grammar / categorical output on the first try, every try — the sampler literally cannot produce a token that breaks the schema, which lets a constrained 4 B local model beat an unconstrained 70 B on extraction tasks; pair with [`vllm`](clis/vllm/) or [`mlx-lm`](clis/mlx-lm/) for "fast local + always valid" | [`outlines`](clis/outlines/) |
| Want the **production performance floor for self-hosted LLM serving on NVIDIA GPUs**, exposed as one OpenAI-compatible URL — PagedAttention + continuous batching + automatic prefix caching turn an A100 80 GB into 50–100 req/s on Llama-3.1-70B under realistic chat load (vs. ~5 req/s on `transformers.generate`); wide model + quantisation support (Llama / Qwen / Mixtral / DeepSeek / Phi / Gemma / VLMs / embeddings / rerankers; AWQ / GPTQ-Marlin / FP8 / INT8 / BitsAndBytes); the URL the rest of this catalog points at when the box has real GPUs | [`vllm`](clis/vllm/) |
| Want to **give an LLM agent hands on the web** — real Chromium via Playwright, with the DOM extracted as a numbered list of interactive elements *and* the screenshot annotated with the same indices so vision and structure agree on what "element 23" means; sub-agent fan-out gives each child its own browser context (separate cookies / tab / viewport) for parallel scraping; the right pick when the task lives behind a UI with no usable API (internal admin consoles, vendor portals, account-management flows), not for sites reachable via REST | [`browser-use`](clis/browser-use/) |
| Want to **eval a RAG pipeline with the canonical, peer-reviewed metrics** (`faithfulness` / `answer_relevancy` / `context_precision` / `context_recall` / `context_entity_recall` / `noise_sensitivity`) — the library RAG papers cite their numbers against, plus a `TestsetGenerator` that synthesises eval triples (simple / multi-context / reasoning / conversational) from your own corpus when you have no labelled set; pair with [`promptfoo`](clis/promptfoo/) for the CI-runner side and [`deepeval`](clis/deepeval/) when you also need pytest-shaped agentic / tool-correctness metrics | [`ragas`](clis/ragas/) |
| Want **OTel-shaped LLM observability you can self-host** — one-line OpenInference auto-instrumentation makes every chat-completion call a span with `llm.input_messages` / `llm.output_messages` / `llm.token_count.*`, framework instrumentors (LangChain / LlamaIndex / DSPy / CrewAI / smolagents / Pydantic-AI) land agent-shaped span trees (`AGENT` / `CHAIN` / `RETRIEVER` / `RERANKER` / `TOOL`), an MCP instrumentor records every `tools/call` round-trip with payloads, and the same span store drives a sample-and-eval loop with [`ragas`](clis/ragas/) / [`deepeval`](clis/deepeval/)-class evaluators writing scores back as span annotations; the only OSS tool that ships the whole instrument → trace → sample → eval → debug loop in one install (ELv2 — internal use is effectively Apache-equivalent, "don't resell as a hosted service") | [`arize-phoenix`](clis/arize-phoenix/) |
| Want a **single OpenAI-shaped call that fans out to every major provider via a `"provider:model"` string** — `client.chat.completions.create(model="anthropic:claude-3-5-sonnet", ...)` and `model="ollama:llama3.1"` go through the same code path; the deliberately minimal pedagogical / glue layer for "benchmark same prompt across N providers" in notebooks, classes, and eval harnesses; gives up streaming + tool-call + structured-output parity across providers in exchange for a tiny dependency footprint and a one-page mental model — reach for [`instructor`](clis/instructor/) / [`mirascope`](clis/mirascope/) / [`magentic`](clis/magentic/) when you need full feature surface | [`aisuite`](clis/aisuite/) |
| Want **typed prompts as plain Python functions, with no DSL / YAML / chain-graph object** — decorate a function with `@llm.call(provider=..., model=..., response_model=Pydantic)`, the docstring is the prompt, the type-hinted args are the variables, the return value is the validated Pydantic instance (or a stream of partials); tools are normal Python functions whose type hints become the JSON schema; provider portability + structured outputs + tool calls + retries + streaming all live in one decorator stack so `mypy` / `pyright` actually see them — the "anti-framework" stance, the right pick when you want a typed-Python aesthetic without the conceptual surface of [`pydantic-ai`](clis/pydantic-ai/) or the prompt-optimisation surface of [`dspy`](clis/dspy/) | [`mirascope`](clis/mirascope/) |
| Want a **batteries-included enterprise RAG framework that bets on small specialised models, not one giant general one** — bundles ingestion (PDF / DOCX / PPTX / XLSX / HTML / OCR via native parser shims) + chunker + embeddings + vector-store abstraction (Postgres / Mongo / SQLite / Milvus / Qdrant / Chroma / Redis / pgvector / Neo4j / LanceDB / FAISS) + retriever + prompt catalog + fact-checker + the `LLMfx` router-over-small-models in one import, with a 100+ HF-published model fleet (BLING 1–3B RAG instruct, DRAGON 6–7B production RAG, SLIM 1–3B function-call / classification / NER / SQL, Industry-BERT embeddings, all in safetensors / GGUF / ONNX) shipped in step with library releases — the framework-shaped foil to assembling [`crawl4ai`](clis/crawl4ai/) + [`docling`](clis/docling/) + [`txtai`](clis/txtai/) + [`instructor`](clis/instructor/) à la carte | [`llmware`](clis/llmware/) |
| Want **long-term memory bolted onto an agent you already have**, without adopting a stateful-agent server — `m.add(messages, user_id="alice")` extracts facts via one LLM call, embeds them, dedupes against existing memories, persists to a pluggable vector store (Qdrant default; Chroma / Pinecone / Weaviate / pgvector / Milvus / LanceDB / Redis / Elasticsearch / Vertex / OpenSearch / Azure AI Search / MongoDB / FAISS / Supabase); `m.search(query, user_id="alice")` does vector recall + LLM rerank; agent framework, prompt template, and runtime do not change; the lightweight counterpart to [`letta`](clis/letta/) for "I just want my chat assistant to remember the user" — note PostHog telemetry is on by default, opt-out via `MEM0_TELEMETRY=False` before first client construction | [`mem0`](clis/mem0/) |
| Want **OpenTelemetry-shaped LLM observability that lands in the dashboards you already have** — `Traceloop.init(...)` plus one auto-instrumentation package per provider (~30 in the matrix: OpenAI / Anthropic / Cohere / Mistral / Bedrock / Vertex / Together / Groq / Watsonx / Ollama / Replicate / SageMaker / VoyageAI / Transformers / OpenAI Agents SDK + vector stores Chroma / Pinecone / Qdrant / Milvus / Weaviate / Marqo / LanceDB + orchestrators LangChain / LlamaIndex / Haystack / [`crewai`](clis/crewai/) / [`agno`](clis/agno/) + an MCP instrumentor) emit OTel GenAI-convention spans (`gen_ai.usage.input_tokens` / `gen_ai.completion`) into any OTLP collector (Jaeger / Tempo / SigNoz / Datadog / Honeycomb / Grafana / NewRelic), so a multi-component RAG pipeline shows up as one trace next to the HTTP request that triggered it; pick this over [`arize-phoenix`](clis/arize-phoenix/) when you already run an OTel pipeline and want LLM spans to land in the *same* backend instead of a parallel UI; SDK self-telemetry is on by default — set `TRACELOOP_TELEMETRY=false` before deploying to a regulated environment | [`openllmetry`](clis/openllmetry/) |
| Want **typed structured data extracted from heterogeneous web pages** (every page lays out the same data differently) — declare a Pydantic schema and a natural-language prompt, the graph fetches via Playwright → semantic-filters the irrelevant DOM → runs LLM extraction on the filtered subset → validates against the schema → re-prompts on validation failure, emitting `list[BlogPost]` directly instead of Markdown you parse a second time; per-page cost is high (~3–5 LLM calls) and install is heavy (~600 MB after Chromium) so use it on heterogeneous targets where the schema is the deliverable, not as a default web scraper — pick [`crawl4ai`](clis/crawl4ai/) instead when the deliverable is clean Markdown for downstream LLM reasoning, or a static `requests` + `selectolax` scraper when one CSS selector handles every page on the site | [`scrapegraphai`](clis/scrapegraphai/) |
| Want **deterministic message-passing multi-agent wiring** where agent A always forwards to B and escalates to C only on a typed failure — explicit `Task` graphs over `ChatAgent` mailboxes (`addSubTask` is the wiring primitive) instead of a manager LLM picking the next worker; ships its own retrieval / tool-calling / chunking primitives and `DocChatAgent` / `SQLChatAgent` / `TableChatAgent` / `LanceRAGTaskCreator` as separate typed Agent subclasses rather than one mega-prompt — the explicit-graph counterpart to [`crewai`](clis/crewai/)'s role-play crews and [`metagpt`](clis/metagpt/)'s software-team idiom; pick when the org chart is *your* responsibility and the framework should not auto-delegate | [`langroid`](clis/langroid/) |
| Want **traces + evals + prompt management + datasets in one Apache-2.0 self-hostable UI** instead of stitching Langfuse + Promptfoo + Argilla — `@track` decorator on any function turns it into a span, online evaluation rules score every new trace automatically with results back-annotated to spans, `Dataset` → `Experiment` → comparison loops compare N prompt or model variants with statistical significance, and the same install ships prompt versioning + LLM-as-judge metrics; framework integrations cover LangChain / LlamaIndex / [`crewai`](clis/crewai/) / [`dspy`](clis/dspy/) / [`pydantic-ai`](clis/pydantic-ai/) / [`smolagents`](clis/smolagents/) / Haystack / AG2 / ADK; the bundled-stack Apache-2.0 alternative to [`arize-phoenix`](clis/arize-phoenix/)'s ELv2 trap — pick `arize-phoenix` instead when you want OTel-native ingest into Tempo / Jaeger / Datadog, pick `opik` when you want one self-hostable UI with prompt management + dataset curation built in | [`opik`](clis/opik/) |
| Want **typed Artifacts + off-prompt Task Memory by default** so a tool that returns 50 KB of CSV writes it to a configured store (file / Redis / Postgres / Mongo / OpenSearch) and the LLM sees only a handle the next tool dereferences — "tool returned 50 KB → context limit → halt" stops happening without manual chunking code; `Workflow` is a real DAG (`task_a >> [task_b, task_c] >> task_d` runs the middle two in parallel and waits on both) rather than a manager LLM picking the next worker; pairs with a mature driver matrix (10+ vector stores, 10+ prompt drivers) under one Apache-2.0 install — the right pick when the agent's data contract is "typed Artifacts moving through a task graph" rather than "Pydantic models from one decorator" ([`mirascope`](clis/mirascope/)) or "role-play crew under a manager LLM" ([`crewai`](clis/crewai/)) | [`griptape`](clis/griptape/) |
| Want a **typed pipeline DAG with named sockets** as the composition contract for production RAG — `splitter.documents` → `embedder.documents` → `writer.documents` validates the wire shape at `pipeline.connect(...)` time; the same `Pipeline.run({...})` works in a notebook, as a `hayhooks` HTTP service, and as an MCP server (`hayhooks-mcp` republishes any deployed pipeline on an MCP transport); component matrix covers converters / splitters / embedders / retrievers (BM25 + dense + hybrid + sparse SPLADE) / rankers / prompt builders / generators / routers / joiners / agents under one Apache-2.0 install with stable v2 APIs since early 2024 — pick over [`langroid`](clis/langroid/) when you want named-socket validation over freeform `Task` mailboxes, over [`griptape`](clis/griptape/) when you want a deeper integration package ecosystem in the `haystack-integrations` org, and over LangChain when you want a Pydantic-v2-typed framework that does not pull LangChain underneath | [`haystack`](clis/haystack/) |
| Want a **vector search engine where embedding lives inside the engine** — one `docker run -p 8882:8882 marqoai/marqo:latest` brings up inference + vector store + REST in 2 GB; you POST text or an image URL and Marqo embeds server-side at index *and* query time, so you never wire a separate embedder service or ship 1024-dim float arrays over the wire; `searchMethod="HYBRID"` fuses BM25 and dense scoring server-side in one call; bundled OpenCLIP variants and first-party `marqo-fashionCLIP` / `marqo-ecommerce-embeddings` checkpoints make text-to-image and product-catalog search a one-day install instead of standing up a vision tower service alongside the vector DB; pick over Pinecone / Qdrant / Weaviate when "embed server-side" matters more than "managed serverless"; default branch is `mainline` (not `main`) — minor papercut for drive-by contributors | [`marqo`](clis/marqo/) |
| Want a **visual canvas as the authoring surface for an agent / RAG pipeline** so PMs and prompt engineers iterate by drag-drop while engineers consume the same JSON via the runtime API or `langflow export` to a `.py` script — `langflow run` boots the React UI + FastAPI backend at `http://localhost:7860`, the bundled ~200 component nodes cover generators (OpenAI / Anthropic / Gemini / Bedrock / Azure / Groq / Mistral / Cohere / DeepSeek / Together / Fireworks / OpenRouter / Ollama / vLLM / LM Studio / llama.cpp), embedders, vector stores (Astra DB / Pinecone / Chroma / Qdrant / Weaviate / Milvus / OpenSearch / pgvector / Mongo / Redis), document loaders (PDF / DOCX / Notion / Confluence / GitHub / Drive / S3), agents, and MCP tools, and the same flow runs as a chat playground, an HTTP endpoint (`POST /api/v1/run/<flow-id>`), and an MCP server publishing flows as callable tools to MCP-aware agents ([`opencode`](clis/opencode/) / [`claude-code`](clis/claude-code/) / [`crush`](clis/crush/) / [`fast-agent`](clis/fast-agent/)) — pick over Flowise when you prefer Python over Node, over [`haystack`](clis/haystack/) when the canvas IS the value (handing flows to non-Python users), over [`fast-agent`](clis/fast-agent/) when you want the most-starred (~147k) drag-drop entry rather than code-first MCP-native crews; bundled component package paths churn occasionally so pin the Langflow version in production | [`langflow`](clis/langflow/) |
| Want **traces + evals on an OTel-shaped span model with a bundled local Streamlit dashboard** in one Apache-style install — wrap the app once (`TruApp` / `TruChain` / `TruLlama` / `TruCustomApp`), attach `Feedback(provider.groundedness_measure_with_cot_reasons).on(Select.RecordCalls.retrieve.rets[:].page_content.collect()).on_output()` style selectors that score *any* sub-span of the recorded run (rate the retriever's context independently from the synthesizer's groundedness), and `run_dashboard(session)` brings up a leaderboard + per-record trace viewer + A/B compare view at `localhost:8484`; broad feedback-function catalog (groundedness, answer-relevance, context-relevance, harmfulness, language-match, comprehensiveness, ~20 more) plus first-class Snowflake Cortex provider for in-warehouse scoring with zero data egress; pick over [`deepeval`](clis/deepeval/) when you want a dashboard + selector grammar (not pytest), over [`ragas`](clis/ragas/) when you need a trace store and per-sub-span scoring (not just RAG metrics), over [`opik`](clis/opik/) when OTel-native span shape matters more than bundled prompt-management UI, and over [`arize-phoenix`](clis/arize-phoenix/) when you want straightforward Apache-licensed sub-packages instead of ELv2 server | [`trulens`](clis/trulens/) |
| Want a **semantic cache for LLM calls as a one-import drop-in** over existing SDK call sites — `from gptcache.adapter import openai` is a literal namespace replacement so cache lookup happens without touching call site logic; the cache embeds the request prompt (bundled ONNX `paraphrase-albert` for offline / CPU, or OpenAI / Cohere / sentence-transformers), looks for a semantically-similar cached response in your scalar + vector store (SQLite + FAISS local, Postgres + Milvus / Mongo + Qdrant / Redis production), and returns the cached response when similarity beats your threshold; on chatbot / RAG workloads with repetitive question patterns, hit rates of 30–70% cut latency to sub-100 ms (vs. seconds) and provider bills by the same factor; `Session("tenant-X")` namespacing + per-route similarity thresholds + LRU/LFU eviction + post-hit reranking are first-class. Caveat: upstream is in slow-maintenance mode (last release 2024-08), pin the version, use a *tight* threshold (≤0.05) for outputs where semantic-near-miss is dangerous (legal / medical / numerical), and reach for a real AI gateway (LiteLLM / Helicone / Portkey) when you also need rate-limiting + key-rotation + audit logging on top of caching | [`gptcache`](clis/gptcache/) |
| Want an **embedded vector + full-text + hybrid DB with no server to run** — `pip install lancedb` + `lancedb.connect("./data")` give you a million-row store with sub-millisecond mmap'd ANN queries (IVF_HNSW_SQ / HNSW / IVF_PQ), versioned commits for time-travel reproducibility (`db.open_table("docs", version=42)` lets an evaluator reproduce exactly what a retriever saw at scoring time), multimodal storage (image bytes + text + vector + label as Arrow columns in one row), and the same directory queryable from Rust / Node / Java / Go / Python / DuckDB / Polars; object-store-native (`./data` ≡ `s3://bucket/data` ≡ `gs://bucket/data` in client code) so the prototype-to-production path is one connection-string change, not a code rewrite; pick over [`marqo`](clis/marqo/) when you want a library not a Docker service (no port to secure, no image to keep current), over `pgvector` when storage is multimodal (images + structured columns alongside vectors) rather than a side-table on an existing Postgres, over Pinecone / Qdrant Cloud when you want the data to live in the directory you control, and over Milvus / Weaviate clusters when "embedded single-process" is fine and the operational simplicity is the value | [`lancedb`](clis/lancedb/) |
| Want a **server-shaped Rust vector engine** where payload filtering, sparse vectors, multitenancy, and quantisation are first-class query primitives (not bolt-ons) — `docker run -p 6333:6333 qdrant/qdrant` brings up a process with its own RBAC + snapshots + Raft cluster membership, the new `query` API chains prefetch (binary-quantised ANN over millions of vectors) → rescore (full-precision) → fusion (RRF / DBSF for hybrid dense + sparse) → group_by ("one result per `doc_id`") in one HTTP round-trip, multitenancy via `group_id` payload + tenant-key index scales past hundreds of thousands of tenants on one collection, and binary quantisation shrinks billion-vector tables 32× to fit in 64 GB RAM with negligible recall loss when paired with `rescore=true`; official clients exist for Python / TS / Rust / Go / Java / Kotlin / .NET; cluster mode (Raft + sharding + replication) ships in the OSS image with no feature gate, and the official `mcp-server-qdrant` exposes `qdrant-store` / `qdrant-find` to MCP-aware agents; pick over [`lancedb`](clis/lancedb/) when the operations contract is "we run a database" not "we ship a directory", over [`chroma`](clis/chroma/) when you need production-shaped hybrid + payload-filter quantisation + Raft cluster mode, over [`marqo`](clis/marqo/) when you want a vector engine without the bundled embedder service (your client embeds and uploads), and over `pgvector` when scale or multitenancy outgrows a side-table on an existing Postgres; default branch is `master` (not `main`) — minor papercut for drive-by contributors | [`qdrant`](clis/qdrant/) |
| Want the **shortest distance from "I have a list of documents" to "I have working RAG"** — `pip install chromadb` → `chromadb.PersistentClient(path="./db")` → `collection.add(documents=[...], ids=[...])` → `collection.query(query_texts=["..."], n_results=5)` is four lines and the default `all-MiniLM-L6-v2` sentence-transformers ONNX embedder (~80 MB) auto-downloads on first add, so there is no embedder configuration to think about until you outgrow it; the same Python / JS API works in `EphemeralClient` (in-memory tests + CI fixtures), `PersistentClient` (laptop dev), `HttpClient` (team-shared `chroma run` server), and `CloudClient` (managed) — the prototype-to-production path is one constructor name change, not a code rewrite; metadata (`where=`) + document substring (`where_document=`) filters compose with vector search in one round-trip; the 1.x core is a Rust rewrite (single binary, mmap-friendly segment store) under the unchanged user-facing API, and the official `chroma-mcp` exposes collection / query / add tools to MCP-aware agents; pick over [`lancedb`](clis/lancedb/) when "default embedder bundled, no setup" matters more than time-travel reads, over [`qdrant`](clis/qdrant/) when API ergonomics matter more than production-shaped hybrid + cluster mode, and over [`marqo`](clis/marqo/) when you want a Python library not a Docker service; caveat — anonymous PostHog telemetry is on by default in OSS (one-line opt-out via `Settings(anonymized_telemetry=False)` or `ANONYMIZED_TELEMETRY=False`), and the 1.x rewrite changed on-disk format from 0.5.x so pin the version + migrate on a planned window | [`chroma`](clis/chroma/) |
| Want **one OpenAI-shaped call site that survives every vendor swap, plus a self-hostable AI gateway you actually own** — `from litellm import completion` is a drop-in for `openai.chat.completions.create` that translates the same call to 100+ providers (Anthropic / Gemini / Vertex / Bedrock / xAI / DeepSeek / Groq / Together / Fireworks / Cohere / Mistral / Ollama / vLLM / any OpenAI-compatible URL) via the `model="provider/model-id"` prefix, killing the "rewrite the call site to migrate from OpenAI to Anthropic" tax (`model="anthropic/claude-sonnet-4"` is the diff); `litellm --config config.yaml --port 4000` boots a FastAPI proxy that downstream apps hit at `http://localhost:4000/v1/chat/completions` as if it were OpenAI while you own the actual provider keys + routing (`routing_strategy: latency-based-routing` / `cost-based-routing` / `usage-based-routing` / `least-busy`) + per-key + per-tenant + per-model RPM/TPM limits + per-virtual-key budgets + audit logging + per-tenant cost attribution + observability fan-out (`callbacks: ["langfuse", "langsmith", "helicone", "openllmetry", "prometheus", "datadog", "sentry", "slack", "s3"]`); MCP is first-class on the proxy (expose MCP tools to downstream agents *and* call out to MCP servers as tools in LLM tool-call rounds, with per-virtual-key allow-lists); response caching at the gateway layer (Redis / S3 / disk, optional `mode: semantic`) is the orthogonal counterpart to [`gptcache`](clis/gptcache/)'s SDK-side caching; pick over Helicone / Portkey / OpenRouter when you want self-hostable not managed, over the Bedrock / Vertex front door when multi-vendor matters, and over a single-provider SDK when "one team, many providers" is the shape; caveat — MIT for the everyday codebase but the `enterprise/` directory carries a separate commercial licence (`enterprise/LICENSE`); also pin the version because translation-layer drift hits when providers ship new features faster than the adapters | [`litellm`](clis/litellm/) |

