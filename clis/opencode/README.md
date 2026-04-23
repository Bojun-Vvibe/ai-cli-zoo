# opencode

> Snapshot date: 2026-04. Upstream: <https://github.com/anomalyco/opencode>

A terminal-first AI coding agent with a polished TUI, a plugin host, and
arguably the most thought-out **skill** system in the field.

## 1. Install footprint

- Single binary via `curl -fsSL opencode.ai/install | bash` or `npm i -g opencode-ai`.
- Binary ~40 MB (Bun-bundled).
- Runs on macOS, Linux, Windows. WSL is the recommended Windows path.
- No daemon, no background services. State lives in `~/.local/share/opencode/`.

## 2. License

MIT.

## 3. Models supported

- Anthropic (Sonnet/Opus/Haiku — full caching).
- OpenAI (o-series, GPT-4.1/5).
- Google Gemini.
- Local: Ollama, LM Studio, llama.cpp, vLLM (anything OpenAI-compatible).
- Bedrock, Vertex, OpenRouter, Groq, DeepSeek, xAI, Z.AI.
- Custom providers via `~/.config/opencode/opencode.json`.

## 4. MCP support

Yes. Both **client** (consume external MCP servers) and **plugin host**
(expose its own tools to other MCP clients). Servers are declared in
`opencode.json` under `mcp:`.

## 5. Sub-agent model

`Task` tool spawns a sub-agent with its own context window and an
optionally restricted tool set (e.g. read-only). Pattern: dispatch a
sub-agent to grep / explore, return a one-paragraph answer to the parent.
Saves tokens on long sessions.

## 6. Telemetry stance

**Off.** No outbound telemetry without explicit opt-in. Crash reports are
local-only unless you run `opencode telemetry enable`.

## 7. Prompt-cache strategy

Anthropic ephemeral cache on:
- The full system prompt
- The tools block
- The tail of the conversation (last 2 turns) when over a threshold

OpenAI: relies on the provider's automatic prefix cache.

## 8. Hot keybinds (TUI)

| Key | Action |
|-----|--------|
| `Ctrl+P` | Command palette (everything is here) |
| `Ctrl+C` then `Ctrl+C` | Cancel current LLM call |
| `Ctrl+L` | Clear screen, keep session |
| `Ctrl+R` | Compact (summarize) the conversation |
| `Esc` | Pop modal / cancel selection |
| `Tab` | Cycle focus between input / messages |
| `↑` in empty prompt | Recall previous prompt |

## 9. Killer feature, weakness, when to choose

- **Killer:** the **skill** system. A skill is a versioned markdown file with
  triggers, instructions, and bundled scripts. The agent loads skills on
  demand based on the user's intent — you can give it 50 skills without
  bloating the system prompt. No other CLI here ships this pattern.
- **Weakness:** documentation lags features. New tools land in code before
  they hit the docs site. You read source to discover capabilities.
- **Choose it when:** you want a long-term TUI driver that you can extend
  with project-specific knowledge (skills, plugins, custom MCP servers) and
  you're comfortable editing JSON config.
