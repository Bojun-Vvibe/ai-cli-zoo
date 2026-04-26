# agentstack

> Snapshot date: 2026-04. Upstream: <https://github.com/AgentOps-AI/AgentStack>

"**The fastest way to build robust AI agents.**" AgentStack is a
**scaffolder + project CLI** for multi-agent applications, in the same
shape that `create-react-app` / `cargo new` occupy in their ecosystems.
You run `agentstack init my-agent`, pick a framework
([`crewai`](../crewai/), [`langgraph`](../langgraph/), or
[`openai-agents-python`](../openai-agents-python/)), and the tool
generates a runnable project tree with agents, tasks, tools, env
config, and a `pyproject.toml` already wired together. Subsequent
commands (`agentstack generate agent`, `agentstack tools add`,
`agentstack run`) extend the project without you re-learning each
framework's boilerplate.

## 1. Install footprint

- `pip install agentstack` (or `uv tool install agentstack`,
  `pipx install agentstack`).
- Python ≥ 3.10. Brings the chosen framework as an extra at `init`
  time; the AgentStack CLI itself is small.
- One-shot scaffolder: `agentstack init my-agent --framework crewai`.
- Project commands (run inside a generated project):
  - `agentstack generate agent <name>` — append a new agent to
    `agents.yaml`.
  - `agentstack generate task <name>` — append a new task to
    `tasks.yaml`.
  - `agentstack tools add <tool>` — install + wire an integration
    (Composio, MCP servers, web search, file IO, custom Python).
  - `agentstack run` — execute the crew / graph end-to-end.
  - `agentstack templates list / use <name>` — start from a
    pre-built template (research crew, content crew, code-review
    crew, etc.).
- Configure via env: `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`,
  `AGENTOPS_API_KEY` (optional observability), per-tool integration
  keys.

## 2. Repo + version + license

- Repo: <https://github.com/AgentOps-AI/AgentStack>
- Latest release: **v0.3.7** (2025-07-10)
- License: **MIT** —
  <https://github.com/AgentOps-AI/AgentStack/blob/main/LICENSE>
- Default branch: `main`
- Language: Python

## 3. Models supported

Inherited from the chosen framework. CrewAI projects route through
LiteLLM (OpenAI, Anthropic, Gemini, Bedrock, Vertex, Cohere, Mistral,
Groq, DeepSeek, OpenRouter, Ollama, vLLM, llama.cpp); LangGraph
projects through `langchain_openai` / `langchain_anthropic` / etc.;
OpenAI Agents projects through the OpenAI SDK plus `litellm` extra for
non-OpenAI providers. Per-agent model assignment is first-class in the
generated `agents.yaml` / `agents.py`.

## 4. MCP support

Yes (client). `agentstack tools add` includes MCP server integrations
out of the box; the generated project mounts the MCP client surface of
the chosen framework (CrewAI's `MCPServerAdapter`, OpenAI Agents'
`MCPServerStdio` / `MCPServerSse` / `MCPServerStreamableHttp`,
LangGraph's tool-binding helpers). AgentStack itself is not an MCP
server.

## 5. Sub-agent model

Inherited from the chosen framework. CrewAI projects ship with the
hierarchical `Process` model and per-agent model assignment;
LangGraph projects ship with the typed-state-machine subgraph
pattern; OpenAI Agents projects ship with handoffs and `as_tool()`
delegation. AgentStack does not impose its own crew abstraction — it
generates the framework's native one and lets you edit the YAML / Python.

## 6. Telemetry stance

Off by default in the OSS CLI itself (no `agentstack` analytics ping).
The generated project has an **opt-in** `agentops` integration —
setting `AGENTOPS_API_KEY` ships traces to the AgentOps dashboard
(same vendor as AgentStack), and the project still runs fully without
it. Egress otherwise is whatever the chosen framework + tool
integrations make.

## 7. Killer keybinds / commands

```sh
agentstack init my-agent --framework crewai      # scaffold project
agentstack generate agent researcher             # add an agent
agentstack generate task summarize_findings      # add a task
agentstack tools add tavily                      # wire a search tool
agentstack tools add mcp_filesystem              # wire an MCP server
agentstack templates use research_crew           # start from template
agentstack run                                   # execute end-to-end
```

## 8. Killer feature, weakness, when to choose

**Killer feature.** **Framework-agnostic project scaffolding for
multi-agent apps with a real `generate` subcommand.** Most agent
frameworks ship a quick-start tutorial and stop; AgentStack ships a
project CLI that knows how to extend an existing project file tree
without you remembering each framework's preferred layout. The
result is that adding a third agent or a new tool to a six-month-old
project is `agentstack generate agent foo` + edit a YAML, not
"re-read the docs and copy-paste from the example repo." The
template gallery (`research_crew`, `content_crew`, `code_review_crew`,
…) means greenfield starts from a real working topology, not from
`hello_world`.

**Weakness.** It is a scaffolder, not a runtime — the moment you
need to debug, you are debugging the underlying framework
([CrewAI](../crewai/) / [LangGraph](../langgraph/) /
[openai-agents-python](../openai-agents-python/)), and the
AgentStack abstraction adds a layer of indirection between your
problem and the framework's docs. The `tools` registry is curated,
not exhaustive — niche integrations still need hand-rolled Python.
Pace of releases is moderate (v0.3.x as of 2025-07), so do not
expect every framework's newest feature to be wired up day-one.
Locks you to one of the three supported frameworks at `init` time;
swapping later is a re-init.

**When to choose.** You are bootstrapping a new multi-agent project
in Python and you have already decided which framework you want — you
just do not want to copy a template repo and rename files.
AgentStack is the boring CLI productivity layer on top.
Pair with [`agentops`](../agentops/) for tracing if you want the
matching observability story. Skip if you have an existing agent
codebase you are extending (the value is mostly at `init` time), if
you want a chat-shaped coding agent ([`opencode`](../opencode/),
[`claude-code`](../claude-code/), [`aider`](../aider/)), or if your
target framework is not in the supported set.
