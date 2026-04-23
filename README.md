# ai-cli-zoo

A curated, deeply-annotated catalog comparing **16 AI coding CLIs**. Each entry
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
