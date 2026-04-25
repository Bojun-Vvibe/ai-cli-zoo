# ms-agent

> Snapshot date: 2026-04. Upstream: <https://github.com/modelscope/ms-agent>
> License file: <https://github.com/modelscope/ms-agent/blob/main/LICENSE>
> Pinned: **v1.6.0** (2026-03-23, PyPI: `ms-agent`).
> The repo was previously known as `modelscope-agent`; the rename to
> `ms-agent` happened with the v1.x line that consolidated the
> framework around a `Workflow` + `Tool` + `Skill` triad. The `ms-`
> prefix here refers to **ModelScope** (Alibaba's HF-equivalent model
> hub), not anything else.

ms-agent is a **lightweight Python agent framework** out of the
ModelScope team. The pitch is "agentic execution of complex tasks
without the dependency surface of a Letta or a DB-GPT": one Python
package, a typed `Workflow` DAG, a small built-in tool catalog, an
optional sandbox, and a `ms-agent` CLI that runs a workflow file
from the terminal. Where lagent (the InternLM cousin) is a pure
library, ms-agent ships an actual command-line driver and a workflow
file format you can hand to a non-Python user.

## 1. Install footprint

- `pip install -U ms-agent` (or `uv pip install ms-agent`).
- Core deps are deliberately small: `pydantic`, `httpx`, `openai`,
  `dashscope` (for ModelScope-hosted Qwen models), `rich`, `typer`.
  Heavier features (RAG, sandbox, knowledge search) gate behind
  extras: `ms-agent[rag]`, `ms-agent[sandbox]`, `ms-agent[all]`.
- Python 3.9+. Linux + macOS + Windows.
- No daemon. The `ms-agent` CLI runs a workflow to completion and
  exits; long-running modes are implemented as a workflow that loops
  on stdin, not as a server process.
- Workspace footprint: a `.ms_agent/` directory at the cwd holding
  per-workflow trace files, downloaded skills, and (if used) the
  knowledge-search SQLite. Add it to `.gitignore`.

## 2. License

Apache-2.0 (verified: repo root `LICENSE`, 11416 bytes, full Apache-2.0
text).

## 3. Models supported

ModelScope-hosted models are first-class via the bundled `dashscope`
adapter (Qwen3-Coder, Qwen3-Max, Qwen2.5-72B, QwQ, plus the open-
weight Qwen embedding family). OpenAI-compatible endpoints are the
generic interop path, so anything that speaks `/v1/chat/completions`
works: OpenAI, Anthropic via a translation proxy, DeepSeek, Moonshot,
Zhipu, Groq, OpenRouter, Together, Ollama, vLLM, llama.cpp,
SGLang. Per-step model selection is supported in the workflow YAML
(`model: qwen3-coder-plus` on the planner step,
`model: qwen2.5-7b-instruct` on cheap executor steps).

## 4. MCP support

**Yes — client.** A `MCPTool` adapter in `ms_agent/tools/` mounts an
external MCP server (stdio or HTTP/SSE) and exposes its tools to the
workflow runtime alongside built-in tools. Tool-call results flow
back into the workflow trace the same as any native tool. Server
mode (exposing an ms-agent workflow over MCP for an external client)
is not yet shipping in v1.6.0; the documented external surface is the
Python API + the CLI.

## 5. Sub-agent model

**Workflow DAG with typed steps.** The `Workflow` primitive is the
composition unit: a YAML or Python file declares a list of `steps`,
each step is one of `LLMStep`, `ToolStep`, `BranchStep`, `LoopStep`,
`SubWorkflowStep`. Sub-agents are plain steps with their own
`system_prompt`, `model`, and `tools` subset, addressed by name.
There is no manager-LLM that decides who runs next at runtime — the
DAG is static, the LLM picks tool calls within a step. The deliberate
trade-off: predictable execution and easy tracing in exchange for
losing dynamic routing. For dynamic routing you write a
`BranchStep` whose condition is itself an LLM call.

## 6. Telemetry stance

**Off.** No analytics in the OSS codebase. The CLI does emit a
trace JSON to `.ms_agent/traces/<run-id>.json` for every run (purely
local; useful for replay + debugging); disable with
`--no-trace` or `MS_AGENT_TRACE=off`. Egress = your configured model
provider plus whatever tool steps reach out (web fetch, MCP servers,
shell commands).

## 7. Prompt-cache strategy

DashScope (Qwen) provider-side prefix cache is honored automatically
when the system prompt is stable across turns within a step. OpenAI-
compatible endpoints rely on the upstream provider's automatic
prefix cache. ms-agent's own contribution is **skill-template
caching**: the `Skill` abstraction (a reusable system-prompt + tool
subset bundle) is hashed, and the LLM call layer reuses the cached
hash key in the request so providers that key cache by hash see hits
across workflow runs.

## 8. Hot keybinds (TUI / REPL)

ms-agent is a CLI driver, not a TUI. The default mode renders the
workflow as a `rich` live tree (current step highlighted, tool
calls expanded inline, token-stream of LLM steps printed in place).
Useful invocations:

| Command | Action |
|---------|--------|
| `ms-agent run workflow.yaml` | Execute a workflow file |
| `ms-agent run workflow.yaml --input "explain this repo"` | Pass an initial input |
| `ms-agent run workflow.yaml --model qwen3-coder-plus` | Override the model on every `LLMStep` |
| `ms-agent run workflow.yaml --trace ./trace.json` | Custom trace path |
| `ms-agent skill list` | List installed skills |
| `ms-agent skill install <name>` | Install a skill from the registry |
| `ms-agent tool list` | Show built-in tools (`web_search`, `code_interpreter`, `file_io`, `shell`, `mcp`, ...) |
| `ms-agent --help` | Subcommand catalog |

Inside an interactive workflow step, `Ctrl+C` cancels the current
LLM stream (the workflow can declare an `on_cancel` handler);
`Ctrl+D` ends the run. There is no in-process REPL; the design
preference is "edit the YAML, re-run the workflow."

## 9. Killer feature, weakness, when to choose

- **Killer:** **YAML-defined typed workflows that a non-Python user
  can write and a CLI can run unmodified.** A 30-line YAML declares
  the steps, the per-step models, the tool subsets, the branch
  conditions; `ms-agent run that.yaml` executes it with a live tree
  view, durable trace, and the same MCP/tool plumbing the Python API
  uses. The skill registry lets you ship a reusable agent recipe as
  one file. Compared to the heavyweight alternatives in this
  catalog ([letta](../letta/), [db-gpt](../db-gpt/)), the install is
  measured in seconds and the surface fits in a tab of documentation.
- **Weakness:** the model adapter list is **deliberately Qwen / OpenAI-
  compatible-only.** No Anthropic / Gemini / Bedrock first-class
  adapters — you reach those through translation proxies, which is
  fine but means you're responsible for your own tool-call schema
  translation. Documentation is primarily Chinese; the English README
  exists but trails the Chinese side. Workflow primitives are
  static-DAG only; if you need a manager-LLM that decides at runtime
  which sub-agent to invoke based on the conversation, [letta](../letta/)
  or [agno](../agno/) are better fits.
- **Choose it when:** you want a **small, well-typed, ModelScope-
  native agent runner** for Qwen-family models, especially if the
  artifact you want to ship is "a workflow YAML my team can read,
  edit, and run from a CLI." Particularly strong for Qwen3-Coder
  pipelines, where the same dashscope account that hosts the model
  also hosts the tool-call cache and the skill registry, so the
  end-to-end stack is one provider deep.
