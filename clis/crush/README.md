# crush

> Snapshot date: 2026-04. Upstream: <https://github.com/charmbracelet/crush>

Charmbracelet's TUI coding agent. Single Go binary, gorgeous Bubble Tea UI,
fast cold-start.

## 1. Install footprint

- `brew install charmbracelet/tap/crush` or download a prebuilt binary
  from GitHub Releases.
- ~18 MB static Go binary.
- macOS, Linux, Windows native (no WSL needed).
- No daemon. State in `~/.config/crush/` and per-project `.crush/` dir.

## 2. License

FSL-1.1-MIT (Functional Source License — converts to MIT after 2 years).
Practically permissive for most uses; check terms if you're embedding.

## 3. Models supported

- Anthropic Sonnet/Opus/Haiku.
- OpenAI o-series, GPT-4.1/5.
- Google Gemini.
- Local: Ollama, LM Studio, any OpenAI-compatible endpoint.
- Configured per-provider in `crush.json`.

## 4. MCP support

Yes (client). Both stdio and SSE transports. Configure under `mcp` in the
project or user config.

## 5. Sub-agent model

None. Single agent, single loop. Tool calls run in parallel when the model
emits multiple in one turn.

## 6. Telemetry stance

**Off.** No telemetry sent by default. Crash reports stay local.

## 7. Prompt-cache strategy

Anthropic ephemeral cache on system prompt and tools block. Conversation
tail caching is not implemented as of this snapshot.

## 8. Hot keybinds (TUI)

Bubble Tea conventions, with vim flavor:

| Key | Action |
|-----|--------|
| `Ctrl+C` | Quit (confirm) |
| `Ctrl+S` | Save / commit current state |
| `Ctrl+P` | File picker / command palette |
| `Tab` | Switch panel focus |
| `j` / `k` (in message list) | Scroll messages |
| `g` / `G` | Top / bottom of message list |
| `i` | Enter insert mode (prompt input) |
| `Esc` | Leave insert mode / cancel modal |
| `?` | Help overlay |

## 9. Killer feature, weakness, when to choose

- **Killer:** the **TUI itself**. Crush is the prettiest, snappiest
  full-screen TUI in this catalog. Cold-start is sub-100ms. Mouse support,
  smooth scrolling, real syntax highlighting in diffs. If aesthetics affect
  your willingness to use a tool daily, this matters.
- **Weakness:** youngest project here, smallest feature surface. No
  sub-agents, no skills, no plan mode. What it does it does well, but it
  does less than opencode or OpenHands.
- **Choose it when:** you want a fast, beautiful, single-binary TUI agent
  with no Python, no Node, no Docker — just `crush` in your terminal.
