# ai-cli-zoo

A curated, deeply-annotated catalog comparing **54 AI coding CLIs**. Each entry
is hand-written from real usage, not marketing copy. The goal: help you pick
the right tool for the job in under five minutes.

> **Scope.** This repo only covers CLIs / TUIs that take natural-language
> instructions and edit code on disk. Browser-only assistants and IDE-locked
> plugins are out of scope.

## Contents

| Path | What it is |
|------|------------|
| [`clis/`](clis/) | One subdirectory per tool with a deep README |
| [`CHOOSING.md`](CHOOSING.md) | Decision tree: pick a CLI from your situation |
| [`ARCHITECTURE-PATTERNS.md`](ARCHITECTURE-PATTERNS.md) | Cross-cutting analysis: agent loops, context, tool call shapes, sandboxing |

## The catalog

| CLI | Lang | License | Models | MCP | Sub-agents | Telemetry default | Killer feature |
|-----|------|---------|--------|-----|-----------|--------|------|
| [opencode](clis/opencode/) | TS | MIT | Anthropic, OpenAI, local, OpenRouter, Bedrock | Yes (client + server) | Task tool spawns sub-agents | Off | Skill system + first-class plugin host |
| [codex](clis/codex/) | Rust | Apache-2.0 | OpenAI o-series, GPT-4.1/5 | Yes (client) | Single agent, parallel tool calls | Opt-in | Sandboxed exec via Seatbelt/Landlock |
| [aider](clis/aider/) | Python | Apache-2.0 | Anthropic, OpenAI, Gemini, Ollama, 100+ via litellm | No (built-in repo-map instead) | No | Off | Repo-map + git-native edit/commit loop |
| [cline](clis/cline/) | TS (VS Code) | Apache-2.0 | Anthropic, OpenAI, Bedrock, Vertex, OpenRouter, local | Yes (client + marketplace) | Plan/Act mode, no sub-agents | Off | Approve-each-step trust UX |
| [OpenHands](clis/openhands/) | Python | MIT | LiteLLM-routed (any) | Yes (client) | Multi-agent (CodeAct, browsing, etc.) | Opt-in | Full Linux sandbox + browser agent |
| [crush](clis/crush/) | Go | FSL-1.1-MIT | Anthropic, OpenAI, local | Yes (client) | No | Off | Beautiful TUI, fast Go binary |
| [continue](clis/continue/) | TS | Apache-2.0 | All major + custom | Yes (client) | No | Off | YAML config + reusable prompt blocks |
| [plandex](clis/plandex/) | Go | AGPL/MIT-dual | Anthropic, OpenAI | No | Internal planner / builder split | Opt-in | Long-horizon plans with branching |
| [mentat](clis/mentat/) | Python | Apache-2.0 | OpenAI, Anthropic | No | No | Off | Multi-file edits with diff preview |
| [gptme](clis/gptme/) | Python | MIT | Many via litellm + local | No (tool-based shell) | No | Off | Shell + Python + browser tools in one REPL |
| [smol-developer](clis/smol-developer/) | Python | MIT | OpenAI | No | No | Off | "One prompt → whole repo" greenfield generator |
| [sweep](clis/sweep/) | Python | Apache-2.0 | OpenAI, Anthropic | No | Issue → PR pipeline | Opt-in | GitHub-issue-driven auto-PRs |
| [goose](clis/goose/) | Rust | Apache-2.0 | Anthropic, OpenAI, Gemini, local, Bedrock, Vertex | Yes (client + server, all built-ins are MCP) | Recipe-based composition; subagent extension spawns child process | Opt-in | Everything-is-an-MCP-extension architecture |
| [gemini-cli](clis/gemini-cli/) | TS (Node) | Apache-2.0 | Gemini 2.x / 1.5 (OAuth, AI Studio, Vertex) | Yes (client) | Single agent, parallel tool calls | Opt-in (free tier opts into training) | 1M+ token context with aggressive prefix caching |
| [claude-code](clis/claude-code/) | TS (Node) | Source-available (Anthropic Commercial) | Claude Opus / Sonnet / Haiku (API, Pro/Max OAuth, Bedrock, Vertex) | Yes (client, deeply integrated) | True sub-agents via `Task` tool, configurable per project | Opt-out (no content training on API/Pro/Max) | Hooks + Skills + Sub-agents triangle |
| [mods](clis/mods/) | Go | MIT | OpenAI, Anthropic, Gemini, Cohere, Groq, local, Azure, any OpenAI-compatible | No | None — one-shot stdin→stdout | Off (no telemetry at all) | Unix-pipe primitive for LLMs |
| [forge](clis/forge/) | Rust (npm launcher) | Apache-2.0 | Anthropic, OpenAI, Gemini, OpenRouter, local, any OpenAI-compatible | Yes (client) | YAML-defined multi-agent workflows, per-step model | Off | Provider-mixing as a first-class concept (planner / editor / reviewer on different models) |
| [qwen-code](clis/qwen-code/) | TS (Node) | Apache-2.0 | Qwen3-Coder (OAuth + DashScope), any OpenAI-compatible endpoint | Yes (client) | Single agent, parallel tool calls | Opt-in | First-party Qwen3-Coder access with a real free-tier quota |
| [llm](clis/llm/) | Python | Apache-2.0 | OpenAI built-in; Anthropic, Gemini, Mistral, Cohere, Ollama, OpenRouter, Bedrock, etc. via plugins | Plugin-only (experimental `llm-tools-mcp`) | None — single conversation | Off (no telemetry) | SQLite log of every prompt/response + first-class plugin system |
| [aichat](clis/aichat/) | Rust | MIT/Apache-2.0 | 20+ providers built-in: OpenAI, Anthropic, Gemini, Mistral, Cohere, Groq, DeepSeek, Together, Ollama, Bedrock, Qwen/Zhipu/Moonshot/Ernie, any OpenAI-compatible | No (function-script convention instead) | None — serial function calls | Off (no telemetry) | Single static binary with built-in document RAG + shell-integration completion |
| [open-interpreter](clis/open-interpreter/) | Python | AGPL-3.0 | OpenAI, Anthropic, Gemini, local (Ollama/LM Studio/llama.cpp), any OpenAI-compatible via litellm | No | None — single REPL with code-exec loop | Off (opt-in) | Code-execution REPL: model writes Python/shell, runs it locally, sees output, loops |
| [shell-gpt](clis/shell-gpt/) | Python | MIT | OpenAI built-in; anything OpenAI-compatible (LiteLLM proxies, Ollama, LM Studio, vLLM) | No | None — flat chat sessions, OpenAI-style functions | Off (no telemetry) | `Ctrl-L` shell integration: natural-language → shell command with [E]xecute/[D]escribe/[A]bort prompt |
| [tgpt](clis/tgpt/) | Go | GPL-3.0 | Free providers (Pollinations, Phind, DuckDuckGo, Blackbox, Koboldai) by default; OpenAI/Anthropic/Groq/Ollama/OpenRouter via flags | No | None — one-shot or flat REPL | Off (no telemetry; free providers see prompts) | Zero-config no-API-key terminal LLM; `brew install tgpt && tgpt "..."` works in 30 seconds |
| [fabric](clis/fabric/) | Go | MIT | OpenAI, Anthropic, Gemini, Groq, Ollama, OpenRouter, Azure, Bedrock | No | None — one pattern = one call, compose via shell pipes | Off (no telemetry) | Shared, version-controlled, hand-curated 200+ pattern library as the product |
| [files-to-prompt](clis/files-to-prompt/) | Python | Apache-2.0 | None — emits text for downstream LLM CLI | No | None — context packer, not a model client | Off (no network) | Deterministic context packer for piping into any LLM CLI; honors `.gitignore`, emits Anthropic XML / markdown / line-numbered shapes |
| [oterm](clis/oterm/) | Python | MIT | Ollama only (whatever `ollama list` reports) | Yes (client) | None — chat tabs with in-line tool calls | Off (no network beyond Ollama) | Persistent multi-tab Textual TUI purpose-built for local Ollama, with edit-and-fork on every turn |
| [gorilla-cli](clis/gorilla-cli/) | Python | Apache-2.0 | Hosted Gorilla endpoint by default; any OpenAI-compatible endpoint via `GORILLA_API_URL` | No | None — one HTTP call returns N candidates | On (hosted endpoint logs prompts) | Multi-candidate shell-command picker with arrow-key selection; zero-key default, shell-only scope |
| [opencommit](clis/opencommit/) | TS (Node) | MIT | OpenAI, Anthropic, Azure, Gemini, Groq, Mistral, Ollama, any OpenAI-compatible | No | None — one LLM call per commit | Off (no analytics) | `prepare-commit-msg` git-hook integration: `git commit` opens `$EDITOR` with a Conventional-Commits-shaped message pre-filled from the staged diff |
| [aicommits](clis/aicommits/) | TS (Node) | MIT | OpenAI built-in; any OpenAI-compatible endpoint via `OPENAI_API_BASE` (Ollama, vLLM, LiteLLM, Groq, OpenRouter) | No | None — one diff → N suggestions | Off (no telemetry) | `-g N` multi-suggestion picker: arrow-key choose between N candidate commit messages instead of edit-or-confirm a single one |
| [aiac](clis/aiac/) | Go | Apache-2.0 | OpenAI, Azure, Anthropic, Bedrock, Ollama, any OpenAI-compatible | No | None — chat mode is single-thread | Off (no analytics) | IaC-shaped post-processing: emits clean Terraform / Pulumi / K8s / Dockerfile / GitHub Actions artifacts ready to `fmt && validate`, no markdown fences or prose |
| [symbex](clis/symbex/) | Python | Apache-2.0 | None — emits Python source for downstream LLM CLI | No | None — AST extractor, not a model client | Off (no network) | AST-correct surgical symbol extractor: `symbex 'MyClass.*'` prints exactly the matching Python methods, an order-of-magnitude smaller context than file-level packers |
| [repomix](clis/repomix/) | TS (Node) | MIT | None — emits text; tokenizes with `tiktoken` | Yes (server: `repomix --mcp`) | None — one-shot pack | Off (no analytics) | Remote-fetch + Tree-sitter compression + Secretlint secret-scan + per-directory token counts in one packer; `repomix --remote owner/repo --compress` works without cloning |
| [chatblade](clis/chatblade/) | Python | MIT | OpenAI built-in; any OpenAI-compatible endpoint via `OPENAI_API_BASE` (Ollama, vLLM, LM Studio, LiteLLM, OpenRouter, Azure) | No | None — one round-trip per call | Off (no analytics) | Inline structured-output extraction: `chatblade -e '.commands[0]' -j "..."` makes the model reply as JSON and pulls out a sub-path in one command, plus plain-text named sessions you can `git diff` |
| [promptfoo](clis/promptfoo/) | TS (Node) | MIT | OpenAI, Anthropic, Gemini, Bedrock, Ollama, Replicate, HuggingFace, custom HTTP / Python | No (custom-provider hook instead) | None — eval harness, not an agent | Off (eval cache local; `share` opt-in) | Cartesian-product prompt eval: `prompts × providers × tests` with assertions (`contains`, `is-json`, `latency`, `cost`, `llm-rubric`, `javascript`/`python`), HTML diff viewer, optional red-team mode |
| [marker](clis/marker/) | Python | GPL/commercial-dual | None for core (layout/OCR/table local models); optional `--use_llm` finishing pass via OpenAI / Anthropic / Gemini / Ollama | No | None — converter, not an agent | Off (network only if `--use_llm` configured) | PDF / EPUB / DOCX → clean Markdown with preserved tables, LaTeX equations, and image refs; the missing front-end for every other CLI in this catalog |
| [gptscript](clis/gptscript/) | Go | Apache-2.0 | OpenAI, Anthropic, Azure, any OpenAI-compatible | Yes (client; tools can be MCP servers, OpenAPI specs, or other `.gpt` files) | Sub-tools: scripts call other scripts as tools | Off | `.gpt` files: English instruction body + declared tools / args / output schema, runnable and composable like shell scripts |
| [shell-genie](clis/shell-genie/) | Python | MIT | OpenAI built-in; hosted free `free-genie` endpoint; any OpenAI-compatible endpoint via `OPENAI_API_BASE` | No | None — one HTTP call per ask | Off (no analytics; `feedback` writes locally only) | Aggressively minimal natural-language → one shell command → y/n; ships shell-aware prompt templates for bash / zsh / fish / PowerShell |
| [elia](clis/elia/) | Python | Apache-2.0 | OpenAI, Anthropic, Gemini, Groq, Ollama, any OpenAI-compatible (declared in `config.toml`) | No | None — linear chat threads | Off (no telemetry; history stays in local SQLite) | Multi-provider Textual chat TUI with persistent SQLite history, `/`-search across past conversations, and `Ctrl-J` mid-conversation model switching |
| [khoj](clis/khoj/) | Python | AGPL-3.0 | Embed: local sentence-transformers or OpenAI; Chat: local llama.cpp/Ollama, OpenAI, Anthropic, Gemini, any OpenAI-compatible | No (HTTP API only) | User-facing personas, not autonomous sub-agents | Off with `--anonymous-mode`; opt-in account telemetry otherwise | Long-lived local daemon that incrementally indexes a personal corpus (notes, PDFs, code) and answers chat queries with citations, fully offline-capable |
| [code-review-gpt](clis/code-review-gpt/) | TS (Node) | MIT | OpenAI, Anthropic, any OpenAI-compatible | No | None — one LLM call per file | Off (no analytics) | Diff-scoped PR-time AI code review with a CI mode that posts inline GitHub PR comments; custom rubrics encode team-specific lint policy that regex linters cannot express |
| [logai](clis/logai/) | Python | BSD-3-Clause | Local: TF-IDF / Word2Vec / sentence-transformers; LLM summarization (optional): OpenAI, any OpenAI-compatible | No | None — sequential pipeline stages | Off (no analytics; LLM egress only if summarization enabled) | Two-stage log analysis: classical Drain/IPLoM template extraction + Isolation Forest anomaly scoring collapses GBs to dozens of templates, then LLM summarizes only the residual anomalies |
| [txtai](clis/txtai/) | Python | Apache-2.0 | Embeddings: sentence-transformers, OpenAI, Cohere, HF; LLM: local llama.cpp/Ollama, OpenAI, Anthropic, any LiteLLM-routed | No (FastAPI surface available) | None — linear YAML workflows | Off (no analytics; fully offline by default) | Index-as-a-file portability: build an embeddings index on one machine, scp the tarball, query offline with the same CLI; covers vector search + graph-RAG + HF pipelines behind one shell surface |
| [k8sgpt](clis/k8sgpt/) | Go | Apache-2.0 | Optional `--explain` backend: OpenAI, Azure, Cohere, Bedrock, Vertex, HuggingFace, LocalAI, Ollama, `noopai` | No (gRPC `serve` instead) | None — analyzers run in parallel, one sweep per `analyze` | Off (no analytics; LLM egress only when `--explain` set) | Deterministic Kubernetes analyzer catalog (Pods, Services, Ingress, HPAs, webhooks, gateways…) finds breakage on its own; LLM is a post-hoc narration layer for on-call humans, not the decision-maker |
| [ttok](clis/ttok/) | Python | Apache-2.0 | None — `tiktoken` encoder only; `-m gpt-4o` selects a tokenizer, not an endpoint | No | None — one-shot tokenizer | Off (one-time vocab download from OpenAI's public CDN, then offline) | Cheap pre-flight token gate for every other catalog packer: `… \| ttok` answers "will this fit?" before paying for an API call, `… \| ttok --truncate N` caps the stream |
| [strip-tags](clis/strip-tags/) | Python | Apache-2.0 | None — emits text for downstream LLM CLI | No | None — HTML→text Unix filter | Off (no network; `strip-tags` does not fetch URLs itself) | HTML→text cleaner that drops `<script>`/`<style>`/chrome and supports `bs4` CSS selectors; the missing front-end for "I just `curl`'d a webpage and want to ask the model about it" without burning tokens on tracker JS |
| [tlm](clis/tlm/) | Go | Apache-2.0 | Ollama only (whatever `ollama list` reports) | No | None — one-shot `suggest`/`explain`/`chat` | Off (no telemetry; only egress is your local Ollama endpoint) | Single static Go binary for the local-only natural-language → shell-command loop, with a `tlm install` bootstrap that pulls and configures an Ollama model in one command |
| [smartcat](clis/smartcat/) | Rust | Apache-2.0 | OpenAI, Anthropic, Mistral, Groq, any OpenAI-compatible (Ollama / vLLM / LM Studio / LiteLLM / OpenRouter / DeepSeek) — declared per-block in `.api_configs.toml` | No | None — stdin→stdout filter | Off (no telemetry; conversation file local) | Named system prompts as a `prompts.toml` config file (`sc -p commit < diff.txt`, `sc -p review < diff.txt`) plus first-class glob input (`sc -g 'src/**/*.rs' "..."`) for a Unix-native LLM filter |
| [ramalama](clis/ramalama/) | Python (containerised `llama.cpp`) | Apache-2.0 | Any GGUF model, pulled from Ollama registry, Hugging Face, OCI artifact registries, or local files; no closed-weight cloud models | No (exposes OpenAI-compatible HTTP for MCP-aware clients to point at) | None — runtime layer, not an agent | Off (no telemetry; Podman/Docker pulls visible to those upstreams) | Hardware-aware container selection for local LLM inference: same `pull`/`run`/`serve` UX across CPU, NVIDIA CUDA, AMD ROCm, Intel oneAPI, Apple Metal — pick the matching `llama.cpp` image automatically |
| [magentic](clis/magentic/) | Python | MIT | OpenAI, Anthropic (built-in extras); ~100 more via LiteLLM extra | No | None — single round-trip per `@prompt` call; you drive any tool loop in Python | Off (no analytics; egress only to provider) | Type-annotated structured output as the primary API: `def f(...) -> list[Recipe]` returns validated Pydantic objects, streaming-aware |
| [mentals-ai](clis/mentals-ai/) | C++ | MIT | OpenAI built-in; any OpenAI-compatible endpoint (Ollama, vLLM, LiteLLM, OpenRouter, DeepSeek) | No | Sub-agents as Markdown link syntax (`use:`/`delegate:`) — recursive composition is a language construct | Off (no analytics; local `.log` traces) | Markdown-as-program: the agent definition *is* a `.md` file with `# Heading` mentals, runnable as a graph by a single C++ binary |
| [ai-shell](clis/ai-shell/) | TS (Node) | MIT | OpenAI built-in; any OpenAI-compatible endpoint via `OPENAI_API_ENDPOINT` (Ollama, LiteLLM, vLLM, OpenRouter, Groq, DeepSeek) | No | None — one-shot translator with optional `ai chat` REPL | Off (no analytics; OpenAI endpoint sees prompts) | `[E]xecute / [R]evise / [C]opy` revise loop on every suggestion: iterate the shell command without retyping the original intent |
| [code2prompt](clis/code2prompt/) | Rust | MIT | None — emits text for downstream LLM CLI | Yes (server: `code2prompt --mcp`) | None — one-shot packer | Off (no telemetry; only egress is the clipboard / output file) | Single static Rust binary that packs a tree into a prompt with **Handlebars templates** for the export shape, plus an interactive TUI with a live `tiktoken` counter so you can hand-curate context against a token budget before exporting |
| [wut](clis/wut/) | Python | MIT | OpenAI, Anthropic, Ollama (env-var configured) | No | None — one LLM call per invocation | Off (no analytics; provider sees the scrollback you just printed) | Reads your **current `tmux` / `screen` pane scrollback** and explains the last command's output — three keystrokes, zero copy-paste, optional follow-up question |
| [micro-agent](clis/micro-agent/) | TS (Node) | MIT | OpenAI, Anthropic, any OpenAI-compatible (Ollama, Groq, vLLM, LiteLLM, OpenRouter) | No | Two-agent in `--visual` mode (Claude critiques, OpenAI writes); single agent otherwise | Off (no analytics; provider sees source + tests + test output) | **The failing test is the halting condition**: agent rewrites a single file until `npm test` (or whatever `-t` is) passes, capped at `--max-runs N` — observable stop condition instead of "model says done" |

> **Caveat.** Licenses, model lists, and feature flags drift. Each subdirectory
> README pins a "as of" date — treat it as a snapshot, not a contract.

## How to read a per-CLI README

Every `clis/<name>/README.md` answers the same nine questions in the same order:

1. Install footprint (binary size, deps, OS support)
2. License
3. Models supported
4. MCP support
5. Sub-agent model
6. Telemetry stance (on by default? opt-out? opt-in?)
7. Prompt-cache strategy
8. Hot keybinds (TUI / REPL)
9. Killer feature, weakness, when to choose

This makes side-by-side reading trivial — open two READMEs, scroll in lockstep.

## Contributing

Issues and PRs welcome. Two rules:
- **No marketing copy.** If you can't run the CLI yourself, don't write the entry.
- **Cite versions.** "As of v0.x.y" beats "as of today" every time.

## License

Content under [CC-BY-4.0](LICENSE). Attribution = link back to this repo.
