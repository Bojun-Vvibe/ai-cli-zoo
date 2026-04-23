# OpenHands

> Snapshot date: 2026-04. Upstream: <https://github.com/All-Hands-AI/OpenHands>
> (formerly OpenDevin)

A research-grade, multi-agent platform that gives the model a full Linux
sandbox plus a browser. The most "open-ended autonomous agent" entry in
this catalog.

## 1. Install footprint

- Recommended: `docker run -p 3000:3000 ...` with the
  `docker.all-hands.dev/all-hands-ai/openhands` image (~2.5 GB).
- Local Python install also supported (`pip install openhands-ai`) but the
  full experience needs the runtime container too.
- macOS, Linux, Windows (via Docker Desktop).
- Web UI on `:3000`, optional CLI mode (`openhands cli`).

## 2. License

MIT.

## 3. Models supported

Anything routed through `litellm` — same long list as aider. The recommended
config uses Anthropic Sonnet for the main agent loop.

## 4. MCP support

Yes (client). Configure under `agent.mcp_servers` in `config.toml`.

## 5. Sub-agent model

True multi-agent. The platform defines distinct **agent classes**:
- **CodeActAgent** — primary coder, writes and executes code in the sandbox.
- **BrowsingAgent** — drives a Chromium instance for web tasks.
- **DummyAgent**, **DelegatorAgent**, etc. — orchestration helpers.

Agents communicate via an event bus and can delegate tasks to each other.
This is closer to a multi-agent OS than a single-loop CLI.

## 6. Telemetry stance

**Opt-in**, prompted on first run. Anonymous error reports + feature usage
when enabled.

## 7. Prompt-cache strategy

Anthropic ephemeral cache on system prompt + tools + recent history.
For OpenAI, relies on automatic prefix cache.

Long sessions inside one Docker runtime can accumulate large histories;
OpenHands has a built-in **condenser** that summarizes old turns to keep
context bounded.

## 8. Hot keybinds

Primarily a web UI; CLI mode has a minimal REPL:

| Key / command | Action |
|---------------|--------|
| `Ctrl+C` (CLI) | Cancel current step |
| `/exit` | Stop the agent |
| `/status` | Show runtime + agent state |
| Web UI: `Cmd+Enter` | Send message |
| Web UI: `Esc` | Pause agent |

## 9. Killer feature, weakness, when to choose

- **Killer:** the **runtime sandbox**. The model gets a real Linux box with
  shell, Python, Node, and a headless browser, all isolated in a container.
  Tasks like "scrape this docs site, then update our SDK to match" are
  realistic here in a way they aren't anywhere else in this catalog.
- **Weakness:** heavy. 2.5 GB image, multiple processes, a web UI you have
  to keep open. Cold-start is slow vs `aider`/`codex`. Debugging when an
  agent loop goes wrong is harder because there are more moving parts.
- **Choose it when:** the task is bigger than "edit these 3 files" — e.g.
  reproduce a benchmark, run a migration that needs to query a live API,
  or anything where a browser in the loop is genuinely necessary.
