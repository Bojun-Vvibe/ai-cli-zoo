# cline

> Snapshot date: 2026-04. Upstream: <https://github.com/cline/cline>

A VS Code extension (formerly Claude Dev) that turns the editor into an
agentic coding environment. Strictly speaking not a "CLI" — but it's the
canonical IDE-resident agent and worth comparing on the same axes.

## 1. Install footprint

- VS Code Marketplace install (~12 MB extension).
- Requires VS Code 1.84+. Also runs in Cursor, Windsurf, and other VS Code
  forks.
- No separate binary, no daemon. Talks to model APIs directly from the
  extension host.

## 2. License

Apache-2.0.

## 3. Models supported

- Anthropic (Sonnet/Opus/Haiku — full caching).
- OpenAI o-series and GPT-4.1/5.
- Google Gemini.
- AWS Bedrock, GCP Vertex.
- OpenRouter, Together, Fireworks, DeepSeek, Mistral, xAI, Groq.
- Local: Ollama, LM Studio, any OpenAI-compatible endpoint.

## 4. MCP support

Yes. Strong story here — Cline ships an **MCP marketplace** inside the
extension UI. One-click install, auto-detected tool schemas, per-server
enable/disable.

## 5. Sub-agent model

No sub-agents, but a distinctive **Plan / Act mode toggle**:
- **Plan mode**: the model can only read and ask questions. No writes,
  no shell. Use this to discuss before doing.
- **Act mode**: full tool access (subject to approval).

## 6. Telemetry stance

**Off by default.** Optional anonymous error reporting; opt-in.

## 7. Prompt-cache strategy

Anthropic ephemeral cache on:
- System prompt
- Tools block
- The most recent user message and prior assistant message

This last bit (caching the conversation tail, not just the head) is unusual
and meaningfully cheaper for back-and-forth sessions.

## 8. Hot keybinds

Cline lives in a VS Code side panel; most actions are buttons. Keybinds
worth knowing:

| Key (default) | Action |
|---------------|--------|
| `Cmd+Shift+P` → "Cline: New Task" | Start fresh task |
| `Cmd+Shift+P` → "Cline: Toggle Plan/Act Mode" | Switch modes |
| `Cmd+L` (in chat) | Clear chat / new task |
| `Esc` | Cancel running tool call |
| `Cmd+Enter` | Send message (Enter alone = newline) |

Approve / reject buttons are mouse-driven by default; you can rebind them
to `Cmd+Y` / `Cmd+N` via VS Code's keybinding JSON.

## 9. Killer feature, weakness, when to choose

- **Killer:** the **approval UX**. Every tool call shows a diff or command
  with big Approve/Reject buttons. You can build an "auto-approve" allowlist
  per tool (e.g. always allow `read`, never auto-approve `bash`). It's the
  most legible "human in the loop" experience here.
- **Weakness:** tied to VS Code. If you live in vim/emacs/Zed, this isn't
  for you. Also, the extension's process model means context (open editors,
  selection) is shared with the editor in ways that occasionally surprise.
- **Choose it when:** you're already a VS Code user, you want the model's
  edits visible in your normal diff UI, and you want to gradually grant
  trust per-tool rather than flip a single "auto-pilot" switch.
