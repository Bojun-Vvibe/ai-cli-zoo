# crewai

> Snapshot date: 2026-04. Upstream: <https://github.com/crewAIInc/crewAI>
> Latest release: **v1.14.3** (2026-04-24). License: **MIT** ([`LICENSE`](https://github.com/crewAIInc/crewAI/blob/main/LICENSE)).

A role-based multi-agent framework that ships a real `crewai` CLI for
scaffolding, running, evaluating, and deploying "crews" (teams of LLM agents
with assigned roles, goals, and tools). Where most catalog entries are *one*
agent loop, crewai's whole product is **how multiple agents collaborate**, and
the CLI is the operator surface for that.

## 1. Install footprint

- `pip install 'crewai[tools]'` (or `uv tool install crewai`).
- Python 3.10+. Pulls in `litellm`, `pydantic`, `chromadb`, `instructor`,
  `opentelemetry-*`, plus the optional `crewai-tools` bundle (browser, RAG,
  file I/O, code interpreter). Roughly 300 MB resolved on a fresh venv.
- macOS, Linux, Windows. No daemon. State in `./.crewai/` per project +
  `~/.config/crewai/` for credentials.

## 2. License

MIT (file: `LICENSE`, SPDX `MIT`). The hosted "CrewAI Enterprise" backend
the CLI's `deploy` subcommand talks to is a separate commercial product —
the OSS framework + CLI itself are unambiguously MIT.

## 3. Models supported

Anything `litellm` routes — OpenAI, Anthropic, Gemini, Bedrock, Vertex,
Cohere, Mistral, Groq, DeepSeek, OpenRouter, Ollama, vLLM, llama.cpp.
Per-agent model assignment is first-class: the planner can be Sonnet, the
researcher Gemini Flash, the coder Qwen-Coder, all in the same crew.

## 4. MCP support

Yes (client). `crewai-tools` ships an `MCPServerAdapter` that mounts an
MCP server's tools as crewai `Tool` objects, usable by any agent in the crew.
No first-party MCP server mode — crewai is a consumer, not an exporter.

## 5. Sub-agent model

**The whole point.** A `Crew` is a list of `Agent`s plus a `Process`
(`sequential`, `hierarchical`, or a `Flow` graph). In `hierarchical` mode
a manager LLM dispatches `Task`s to worker agents and aggregates results.
Agents can themselves spawn sub-agents via the `delegate` tool.

## 6. Telemetry stance

**Opt-in.** OpenTelemetry hooks are present but disabled unless you set
`CREWAI_TELEMETRY=true` or wire your own OTel exporter. The hosted
"AgentOps"-style observability tier is opt-in and account-gated.

## 7. Prompt-cache strategy

Inherits whatever `litellm` does for the underlying provider — Anthropic
ephemeral cache markers are added on long system prompts when the agent
config sets `cache=True`. No crewai-side response cache; the
`chromadb`-backed memory is for *agent recall*, not request dedup.

## 8. CLI surface (the part this catalog cares about)

```
crewai create crew <name>          # scaffold a project (agents.yaml, tasks.yaml, crew.py)
crewai create flow <name>          # scaffold a Flow (graph-shaped, branching)
crewai install                     # uv-based dep install for the scaffold
crewai run                         # kickoff the crew defined in this project
crewai chat                        # interactive REPL against the crew
crewai test --n_iterations N       # run the crew N times, score outputs
crewai train --n_iterations N      # human-in-the-loop preference fine-tune
crewai replay --task_id <id>       # rerun from a checkpoint
crewai deploy create               # push to CrewAI Enterprise (optional)
crewai tool install <pkg>          # add a tool from the crewai-tools registry
```

`crewai chat` and `crewai run` are the two surfaces a human actually lives in.

## 9. Strengths

- **Role-based multi-agent is a first-class config concept**, not a pattern
  you reimplement — `agents.yaml` declares `role`, `goal`, `backstory`,
  `tools`, `llm`, and `Process` decides how they collaborate. Closest
  catalog cousin (forge) does YAML workflows but is single-process; crewai
  is built around the *team* abstraction.
- **`crewai test` / `crewai train` are real CLI verbs**: the framework
  treats agent quality as something you regress-test (run N times, score)
  and tune (human-rates outputs, preferences feed back into prompts).
  Almost no other agent framework in this catalog ships an evaluation
  loop in the CLI.
- **Per-agent model routing is trivial**: `llm=ChatOpenAI("gpt-4o")` on
  the planner, `llm=ChatAnthropic("claude-haiku")` on the worker — the
  cost-vs-quality knob is one line of Python per role.

## 10. Weaknesses / when not to use

- **Heavy stack for single-agent tasks.** If you want one model + one shell
  + one editor, `aider` or `codex` is two orders of magnitude less code
  and less moving parts. Crewai's value only kicks in past ~3 cooperating
  agents.
- **The framework moves fast and breaks APIs.** v0.x → v1.x reshaped
  imports, agent-config keys, and the `Flow` decorator surface; pin
  `crewai==1.14.3` and re-test before bumping. If you need API stability
  more than feature velocity, prefer `langgraph` or hand-rolled loops.

## 11. Comparison vs nearest neighbor in zoo

The closest catalog entry is **[`forge`](../forge/)** — both are
"multi-agent workflows declared in YAML/config" CLIs. The split: `forge`
treats sub-agents as *steps in a pipeline* (planner → editor → reviewer
on different models), single-process, code-editing-first. **`crewai`**
treats sub-agents as *roles in a team* with a manager loop arbitrating
delegation, and is general-purpose (research, content, analysis, code).
Pick `forge` for "the planner uses Claude, the editor uses GPT, on this
codebase". Pick `crewai` for "I have five distinct roles that need to
talk to each other and the work isn't just code edits".
