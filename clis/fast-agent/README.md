# fast-agent

> Snapshot date: 2026-04. Upstream: <https://github.com/evalstate/fast-agent>
> Pinned version: **v0.6.22** (PyPI: `fast-agent-mcp`).

A Python CLI + library for building and running MCP-native agents. Aimed
squarely at people who want the agent definition to be a short Python
file (decorators + types) with MCP servers as the tool surface, and a
real terminal chat to drive it.

## 1. Install footprint

- `uv tool install fast-agent-mcp` (recommended) or
  `pip install fast-agent-mcp`.
- Pulls in `mcp`, `pydantic`, `rich`, `prompt-toolkit`, plus per-provider
  extras (`openai`, `anthropic`, `google-genai`, `boto3`, ...).
- Installs two console scripts: `fast-agent` and `fast-agent-mcp`
  (aliases) plus an `apply_patch` helper.
- Python 3.10+. macOS, Linux, Windows.
- No daemon; config in `fastagent.config.yaml` + `fastagent.secrets.yaml`
  in the project dir.

## 2. License

Apache-2.0 (verified from the upstream `LICENSE` file).

## 3. Models supported

OpenAI (Chat + Responses), Anthropic, Google Gemini (AI Studio + Vertex),
Amazon Bedrock, Azure OpenAI, DeepSeek, Groq, OpenRouter, TensorZero,
HuggingFace Inference, Ollama and any OpenAI-compatible endpoint via
explicit `base_url`. Per-agent model assignment is first-class (`@agent(
model="anthropic/claude-sonnet-4-5")` on one agent, `model="openai/o4-mini"`
on the next).

## 4. MCP support

**Yes — MCP is the entire tool model.** Tools, prompts and resources all
come from MCP servers declared in `fastagent.config.yaml`. Stdio,
HTTP/SSE and streamable-HTTP transports are all supported. ACP (the
agent-communication protocol) is also supported as a peer of MCP, so a
fast-agent can both consume and expose ACP endpoints.

## 5. Sub-agent model

Multiple. The framework ships first-class composition primitives:

- `@agent` — a single agent
- `@chain` — sequential pipeline of agents
- `@parallel` — fan-out / fan-in
- `@router` — pick one of N agents per turn (LLM-routed)
- `@orchestrator` — planner LLM coordinates worker agents
- `@evaluator_optimizer` — generator + critic loop until score ≥ rating

Each is a Python decorator over a function; composition is by passing
agent names into another decorator's `agents=[...]` arg.

## 6. Telemetry stance

**Off.** No analytics in the OSS codebase. Egress = your configured
model provider plus whatever MCP servers you mount. Optional OTel hook
is opt-in via `otel.enabled: true` in the config.

## 7. Prompt-cache strategy

Anthropic ephemeral cache wired on the system prompt and conversation
prefix when the provider is Claude. OpenAI / Bedrock rely on automatic
prefix cache. No fast-agent-specific cache layer.

## 8. Hot keybinds (REPL)

`fast-agent go` opens an interactive prompt-toolkit REPL:

| Key / command | Action |
|---------------|--------|
| `Enter` | Send turn |
| `Alt+Enter` / `Esc` then `Enter` | Newline in prompt |
| `Ctrl+C` | Cancel current LLM stream |
| `Ctrl+D` | Exit |
| `***SAVE_HISTORY <file>` | Dump the turn history to disk |
| `***STOP` | End the conversation |
| `/list` | List configured agents |
| `/agent <name>` | Switch active agent |
| `/clear` | Reset conversation |

## 9. Killer feature, weakness, when to choose

- **Killer:** **MCP + ACP as the only integration shape, and composition
  primitives as decorators.** A 30-line `agent.py` declaring `@chain`,
  `@parallel`, `@evaluator_optimizer` becomes a runnable `fast-agent go`
  CLI, a `fast-agent send` one-shot, and an MCP/ACP server that other
  agents can call — same definition, three surfaces. No bespoke tool
  registries; if it isn't an MCP server, fast-agent doesn't see it.
- **Weakness:** opinionated. If you don't want to define your tools as
  MCP servers, this isn't the framework. Also young; expect the surface
  to keep moving (PyPI name is still `fast-agent-mcp`, not `fast-agent`).
- **Choose it when:** you've already invested in MCP servers and want
  the smallest typed-Python harness that turns them into an agent crew
  with a real terminal you can drive.
