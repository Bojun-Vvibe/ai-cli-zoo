# claude-engineer

> Snapshot date: 2026-04. Upstream: <https://github.com/Doriandarko/claude-engineer>
> Binary name: `ce3` (and `python ce3.py`)

`claude-engineer` is a self-extending Anthropic-Claude CLI agent whose
distinguishing trick is **tool authorship at runtime**: when the model
hits a capability gap mid-conversation, it writes a new Python tool,
drops it into `tools/`, hot-loads it into the live registry, and
continues using it on the very next turn. The tool is then permanent
across future sessions. The repo ships a small starter set
(`createfolderstool`, `filecontentreadertool`, `webscrapertool`,
`uvpackagemanager`, `lintingtool`, etc.) and a `toolcreator` meta-tool
that authors the rest on demand.

This is a different design point from the rest of the catalog:
[`aider`](../aider/) ships a fixed edit-loop, [`opencode`](../opencode/)
and [`claude-code`](../claude-code/) extend via MCP servers and skills,
[`gptme`](../gptme/) bundles a fixed shell+python+browser tool kit.
`claude-engineer` instead treats the **tool set itself as a mutable,
agent-grown surface**, which is the right shape when the bottleneck is
"I keep wanting one more 30-line tool but I do not want to ship a new
MCP server for it".

## 1. License

MIT (declared in `pyproject.toml`; no top-level `LICENSE` file as of
the `main` branch snapshot — vendored as text in the package metadata).

## 2. Latest version

No tagged GitHub release. Track `main` at the snapshot SHA; the
`Claude-Eng-v2` directory holds the current rewrite. Pin via commit
SHA in `requirements.txt` if reproducibility matters.

## 3. Install

```bash
git clone https://github.com/Doriandarko/claude-engineer
cd claude-engineer
pip install -r requirements.txt   # or: uv pip install -r requirements.txt
export ANTHROPIC_API_KEY=sk-ant-...
python ce3.py                     # CLI REPL
# or:
python app.py                     # web UI on http://localhost:5000
```

Models: Claude 3.5 Sonnet by default (configurable in `config.py`).

## 4. When to choose

- You want an agent that **grows its own tool surface as you use it**,
  with the new tools committed back into the repo as plain Python.
- You are comfortable with `main`-branch-only, no semver, no release
  cadence — this is a research-velocity project, not a packaged CLI.
- You are Anthropic-only and do not need MCP, sub-agents, or sandboxing
  ([`opencode`](../opencode/), [`claude-code`](../claude-code/),
  [`openhands`](../openhands/) are better picks if you do).
- Avoid if you need a stable tool allowlist for compliance — the whole
  point here is the tool list mutates at runtime.
