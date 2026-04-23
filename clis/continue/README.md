# continue

> Snapshot date: 2026-04. Upstream: <https://github.com/continuedev/continue>

An open-source IDE extension (VS Code + JetBrains) plus a headless CLI mode
(`continue`). Highly configurable, YAML-driven, model-agnostic.

## 1. Install footprint

- **IDE**: install from VS Code Marketplace or JetBrains Plugin Repo
  (~15 MB).
- **CLI**: `npm i -g @continuedev/cli` for headless / CI usage (~30 MB).
- macOS, Linux, Windows.
- Config lives in `~/.continue/config.yaml` (also project-local
  `.continue/`).

## 2. License

Apache-2.0.

## 3. Models supported

Effectively all major providers — OpenAI, Anthropic, Gemini, Bedrock,
Vertex, Mistral, DeepSeek, Together, Fireworks, Groq, OpenRouter, Ollama,
LM Studio, vLLM, llama.cpp, SageMaker. Custom providers via JS plugin.

## 4. MCP support

Yes (client). Add MCP servers in `config.yaml` under `mcpServers:`.
Continue was one of the early adopters and the integration is mature.

## 5. Sub-agent model

No sub-agents. The architecture splits **chat / autocomplete / agent**
modes — each is its own configurable role with its own model and rules.
This is composition, not concurrency.

## 6. Telemetry stance

**Off by default.** Anonymous telemetry is opt-in via `allowAnonymousTelemetry`
in `config.yaml`. Easy to audit because the config is your source of truth.

## 7. Prompt-cache strategy

Anthropic ephemeral cache for Anthropic models. For OpenAI, relies on the
provider's automatic prefix cache. Continue keeps system prompts + tool
schemas stable across turns to maximize hits.

## 8. Hot keybinds (IDE)

| Key (default) | Action |
|---------------|--------|
| `Cmd+L` | Add selection to chat |
| `Cmd+Shift+L` | New chat session |
| `Cmd+I` | Inline edit on selection |
| `Cmd+J` | Toggle Continue side panel |
| `Tab` | Accept autocomplete |
| `Esc` | Reject autocomplete |
| `Cmd+Enter` | Send chat message |

CLI mode is a small REPL with `/help`, `/model`, `/exit`.

## 9. Killer feature, weakness, when to choose

- **Killer:** the **YAML config**. One file describes models, roles,
  context providers, MCP servers, custom slash-commands, and autocomplete
  behavior. You can check it into your team repo and everyone gets the
  same setup. No other entry here is this declaratively configurable.
- **Weakness:** the breadth is the curse — defaults are fine but to get
  the most out of Continue you end up writing a lot of YAML. New users
  often miss features simply because they didn't know to enable them.
- **Choose it when:** you want one tool for **autocomplete + inline edit
  + chat + agent**, configured once via YAML, working identically in
  VS Code, JetBrains, and CI.
