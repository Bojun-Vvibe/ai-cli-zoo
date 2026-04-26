# agent-zero

> Snapshot date: 2026-04. Upstream: <https://github.com/agent0ai/agent-zero>

`agent-zero` is a general-purpose, **transparent, hackable** AI agent
framework. Unlike most agent frameworks that ship as a Python library
you `import` and configure, agent-zero ships as a **runnable
application** (Docker container or local Python) where every prompt,
tool, and behaviour lives as a **plain file on disk** that you are
expected to read and rewrite. Its prompts are not hidden behind a DSL;
they are markdown in `prompts/`. Tools are Python files in `python/tools/`.
Memory is JSON / vector files under `memory/`. The pitch: "the agent
*is* the filesystem; if you do not like how it behaves, edit it."

## 1. Install footprint

- **Recommended**: Docker — `docker pull agent0ai/agent-zero` then
  `docker run -p 50001:80 agent0ai/agent-zero`. Web UI at
  `http://localhost:50001`. The container ships its own runtime so the
  agent can freely `bash` / `python` / `pip install` without touching
  the host.
- **Local Python**: clone + `pip install -r requirements.txt`, then
  `python run_ui.py`. Works but the agent will execute commands on
  your real shell — Docker is strongly recommended for any
  non-trivial use.
- The agent's *own* code-execution sandbox runs **inside** the
  container; you do not need an additional sandbox layer.

## 2. License

MIT (file: [`LICENSE`](https://github.com/agent0ai/agent-zero/blob/main/LICENSE)).
GitHub's license classifier reports `NOASSERTION` because the file
includes a custom copyright header (`Copyright (c) 2025 Agent Zero,
s.r.o`) before the standard MIT body — the body itself is verbatim
MIT.

## 3. What is in the box

- **Web UI** (FastAPI + vanilla JS) for chat, plus a file browser into
  `prompts/`, `memory/`, `knowledge/`, `tmp/` so you can watch and
  edit the agent's state live while it runs.
- **Multi-agent recursion**: any agent can spawn a *subordinate*
  agent with a narrowed task and merge the result back. There is no
  fixed "planner / executor" split; hierarchy is emergent.
- **Tools as Python files**: `code_execution_tool`, `knowledge_tool`,
  `memory_tool`, `webpage_content_tool`, etc. Adding a tool = dropping
  a `.py` file with a `Tool` subclass into `python/tools/`.
- **MCP**: both **client** (call external MCP servers as tools) and
  **server** (exposes the agent itself over MCP so other agents can
  call it).
- **Voice / STT / TTS** built in (browser-side speech recognition +
  configurable TTS backends).
- **Speech-free, headless mode** via `python run_cli.py` for scripting.

## 4. One-liner usage

```bash
# Start the dockerised agent + web UI on port 50001
docker run --rm -p 50001:80 -v "$PWD/agent-zero-data:/a0" \
  agent0ai/agent-zero
# then open http://localhost:50001 and start chatting
```

For a non-interactive, scriptable invocation against a running
container, use the included `run_cli.py` inside the container or hit
the message API at `POST /message_async` with a JSON body.

## 5. Source-of-truth pin

- Verified default branch: `main`
- Latest release tag at snapshot: **`v1.9`**
- HEAD SHA at snapshot: `3fa8481ba252b3187036fecbbf5564bf05694ce5`
  (committed `2026-04-12T16:31:32Z`)
- License file path: `LICENSE` (SPDX-equivalent: MIT)
- Verified UTC: `2026-04-26T19:16:33Z` via `gh api repos/agent0ai/agent-zero`

## 6. Where it fits in the zoo

Closest neighbours: [`openhands`](../openhands/) (also a
container-first agent with a web UI), [`devika`](../devika/) (also a
"transparent" software-engineer agent), [`autogpt`-class frameworks].
Agent-zero is the right pick when you want **maximum control with
minimum abstraction** — when you would rather edit a markdown prompt
than learn a new agent DSL, and when you want the agent's filesystem
to *be* the configuration surface.
