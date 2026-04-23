# gptme

> Snapshot date: 2026-04. Upstream: <https://github.com/gptme/gptme>

A personal AI assistant that lives in your terminal as a REPL. Think
"bash + Python + browser, with an LLM holding the reins." Less of a
dedicated coding agent, more of a Swiss-army assistant that happens to
be excellent at code tasks.

## 1. Install footprint

- `pipx install gptme` or `pip install gptme`.
- Python 3.10+. Optional extras for browser (`gptme[browser]`) and TTS.
- ~40 MB without optional extras.
- macOS, Linux, Windows.

## 2. License

MIT.

## 3. Models supported

Anything `litellm` supports — OpenAI, Anthropic, Gemini, Groq,
DeepSeek, OpenRouter, plus local via Ollama, llama.cpp, vLLM. Model is
selected by `--model` or `gptme.toml`.

## 4. MCP support

**No** as of this snapshot. Tools are built-in: `shell`, `python`,
`browser`, `read`, `save`, `patch`, `tmux`. Adding new tools is a Python
class, not an MCP server.

## 5. Sub-agent model

No sub-agents. Long-running shell commands can be parked in a `tmux`
session via the built-in `tmux` tool — closest thing to "delegate this
in the background".

## 6. Telemetry stance

**Off.** No telemetry.

## 7. Prompt-cache strategy

No explicit annotations. Anthropic models work but caching isn't actively
managed. For long sessions, expect normal per-turn cost.

## 8. Hot keybinds (REPL)

`gptme` is a `prompt-toolkit` REPL with multi-line input:

| Key / command | Action |
|---------------|--------|
| `Esc Enter` (or `Alt+Enter`) | Submit multi-line prompt |
| `Ctrl+C` | Cancel current LLM call |
| `Ctrl+D` | Exit |
| `/help` | List slash-commands |
| `/tools` | Toggle which tools are enabled |
| `/edit` | Edit the conversation (yes, the conversation itself) |
| `/undo` | Drop the last turn |
| `/save <path>` | Save conversation to file |
| `/load <path>` | Resume from file |

## 9. Killer feature, weakness, when to choose

- **Killer:** **`/edit` the conversation**. You can open the entire
  conversation history in `$EDITOR`, hand-edit any past turn (including
  the model's), and resume. Wildly powerful for prompt iteration and
  for "rolling back" without losing later context. No other CLI here
  exposes the conversation as a directly-editable buffer.
- **Weakness:** generalist by design. The shell + python + browser tools
  are all useful, but it doesn't have aider's repo-map or opencode's
  skill system. On a 500-file repo it gets confused.
- **Choose it when:** the "task" is fuzzy — research, scripting, exploring
  an API, generating a one-off chart — and you want one REPL that can
  shell out, run Python, hit a URL, and edit a file, all in one place.
