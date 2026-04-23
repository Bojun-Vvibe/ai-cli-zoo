# aider

> Snapshot date: 2026-04. Upstream: <https://github.com/Aider-AI/aider>

The granddaddy of terminal AI coders. REPL-style, git-native, and the only
tool here that ships a serious symbolic repo-map.

## 1. Install footprint

- `pipx install aider-chat` (recommended) or `pip install aider-chat`.
- Pulls in tree-sitter language packs (~80 MB total).
- Python 3.10+. macOS, Linux, Windows all first-class.
- No daemon. State in `~/.aider/` and `.aider*` files inside each repo.

## 2. License

Apache-2.0.

## 3. Models supported

Anything `litellm` supports — OpenAI, Anthropic, Gemini, Bedrock, Vertex,
Cohere, DeepSeek, Mistral, Groq, OpenRouter, Ollama, llama.cpp, vLLM, ...
Roughly 100+ providers. Aider keeps a model leaderboard at
<https://aider.chat/docs/leaderboards/>.

## 4. MCP support

**No.** Aider intentionally avoids MCP — its position is "the repo-map +
edit-block format already gives the model the context it needs; MCP adds
moving parts." You can wire external tools by `/run`-ing shell commands.

## 5. Sub-agent model

None. Single conversation, single model. There's a `/architect` mode that
lets you split planning (one model) from editing (another model) — this is
two sequential calls, not concurrent agents.

## 6. Telemetry stance

**Off.** Anonymous usage analytics are opt-in via `--analytics`. Nothing
leaves your machine without that flag.

## 7. Prompt-cache strategy

Anthropic ephemeral cache on the system prompt, the repo-map block, and
the chat history prefix. Aider was one of the first CLIs to wire this up
and it shows: long sessions with Sonnet stay cheap.

OpenAI: relies on automatic prefix cache.

## 8. Hot keybinds (REPL)

Aider is a readline REPL, so most "keybinds" are slash-commands:

| Command | Action |
|---------|--------|
| `/add <file>` | Add file to chat context |
| `/drop <file>` | Remove file from context |
| `/ls` | List files currently in chat |
| `/diff` | Show pending diff |
| `/undo` | Revert the last AI commit |
| `/run <cmd>` | Run shell, optionally feed output to chat |
| `/test` | Run configured test command |
| `/architect` | Two-model planner/editor mode |
| `Ctrl+C` | Cancel current LLM stream |
| `Ctrl+D` | Exit |

## 9. Killer feature, weakness, when to choose

- **Killer:** the **repo-map**. Tree-sitter parses every source file,
  extracts symbols, ranks them by graph centrality, and packs the most
  relevant ones into the system prompt as a compact symbol map. The model
  knows your repo's shape without you adding a single file. On a 500-file
  Python project this is night-and-day vs a CLI that only sees what you
  `/add`.
- **Weakness:** no full TUI. The REPL feels dated next to opencode/crush.
  Also, no MCP means external integrations are bring-your-own-shell.
- **Choose it when:** you're working on a large, established repo and you
  want a tool that respects git, shows you every diff, and never runs
  shell without `/run`.
