# mentat

> Snapshot date: 2026-04. Upstream: <https://github.com/AbanteAI/mentat>

A diff-first terminal coder. The model proposes edits as unified diffs;
you accept or reject each one. Conservative by design.

## 1. Install footprint

- `pipx install mentat` (recommended) or `pip install mentat`.
- Python 3.10+.
- ~50 MB with deps.
- macOS, Linux, Windows.

## 2. License

Apache-2.0.

## 3. Models supported

OpenAI (default), Anthropic, Azure-hosted OpenAI endpoints, and any other
provider you wire up via env vars. Model selection is per-session via
`--model`.

## 4. MCP support

**No.** Mentat's tool surface is intentionally narrow: read files, propose
edits. No shell, no MCP, no browser.

## 5. Sub-agent model

None. Single-model, single-loop.

## 6. Telemetry stance

**Off.** No telemetry, no phone-home.

## 7. Prompt-cache strategy

No explicit caching annotations. Relies on OpenAI's automatic prefix
cache when present. Mentat's per-turn context is small enough that this
matters less than for tool-heavy CLIs.

## 8. Hot keybinds (REPL)

| Key / command | Action |
|---------------|--------|
| `Ctrl+C` | Cancel current LLM call |
| `Ctrl+D` | Exit |
| `/help` | Slash-command reference |
| `/include <path>` | Add file or directory to context |
| `/exclude <path>` | Remove from context |
| `/undo` | Revert the last applied edit |
| `/commit` | Commit applied edits to git |
| `y` / `n` (on each edit prompt) | Accept / reject |
| `e` (on edit prompt) | Open editor to tweak before applying |

## 9. Killer feature, weakness, when to choose

- **Killer:** the **per-edit accept/reject loop**. Mentat shows each diff
  hunk separately and asks. You can accept hunks 1, 3, 4 from a five-hunk
  proposal and reject 2 and 5. This granularity is unique here.
- **Weakness:** small project, low velocity, narrow feature set. No MCP,
  no shell, no sub-agents. The community is much smaller than aider's.
- **Choose it when:** you want the most paranoid possible workflow — every
  hunk reviewed individually — and you don't need the model to do anything
  beyond proposing edits.
