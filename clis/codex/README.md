# codex

> Snapshot date: 2026-04. Upstream: <https://github.com/openai/codex>

OpenAI's official terminal coding agent. Rust binary, sandbox-first design,
opinionated about safety boundaries.

## 1. Install footprint

- Single static binary: `brew install codex` or `npm i -g @openai/codex`
  (the npm package wraps the binary).
- ~25 MB.
- Runs on macOS, Linux. Windows via WSL.
- No daemon. State in `~/.codex/`.

## 2. License

Apache-2.0.

## 3. Models supported

- OpenAI o-series (o3, o4-mini, etc.)
- GPT-4.1 / GPT-5 family
- Through API key or ChatGPT login (uses your subscription).
- Routing to non-OpenAI providers requires a wrapper (community LiteLLM
  configs exist; not officially supported).

## 4. MCP support

Yes, **client only**. Declare servers under `[mcp_servers]` in
`~/.codex/config.toml`. Stdio and HTTP transports both work.

## 5. Sub-agent model

Single agent. Tool calls run in parallel when the model emits multiple in
one turn. No spawnable sub-agents.

## 6. Telemetry stance

**Opt-in**, prompted on first run. Sends anonymized command + error metrics
when enabled. Easy to flip off in `~/.codex/config.toml`.

## 7. Prompt-cache strategy

Relies on the OpenAI API's automatic prefix cache. Codex keeps the system
prompt and tools schema byte-stable across turns to maximize hit rate.
No `cache_control`-style annotations needed (OpenAI doesn't expose them).

## 8. Hot keybinds (TUI)

| Key | Action |
|-----|--------|
| `Esc Esc` | Cancel running call |
| `Ctrl+J` | Insert newline (vs Enter to send) |
| `Ctrl+T` | Toggle thinking-tokens visibility |
| `Ctrl+R` | Recompose / show last reasoning |
| `/approval` | Switch approval mode (suggest/auto-edit/full-auto) |
| `/diff` | Show pending diff |
| `Up Arrow` | Recall previous prompt in empty input |

## 9. Killer feature, weakness, when to choose

- **Killer:** **sandboxed shell execution**. On macOS uses Seatbelt
  (`sandbox-exec`); on Linux uses Landlock + seccomp. The model can run
  `cargo build`, `pytest`, etc. with no network and read-only access
  outside the project dir. This is the safest "let it loop" experience
  in this catalog.
- **Weakness:** locked to OpenAI for first-class support. If your shop is
  on Anthropic or local models, you're swimming upstream.
- **Choose it when:** you're already on OpenAI, you want autonomous
  multi-step runs (build → test → fix → re-test), and you don't want to
  babysit each tool call.
