# marvin

> Snapshot date: 2026-04. Upstream: <https://github.com/PrefectHQ/marvin>

"**An ambient intelligence library**" from the team behind Prefect.
Marvin 3.x is a Python framework that wraps LLM calls as **typed
functions, classifiers, extractors, and small autonomous agents**, with
a thin CLI for one-shot use. The pitch is "Pydantic + LLM = boring,
reliable AI features": instead of writing prompts that return strings
you parse, you declare a Python signature and Marvin turns it into a
validated round-trip. Pairs naturally with Prefect for orchestration,
but the CLI / library work standalone.

## 1. Install footprint

- Recommended: `pip install marvin` (or `uv pip install marvin`,
  `pipx install marvin` for the CLI).
- Python ≥ 3.10. Brings Pydantic v2, OpenAI SDK, and (optionally)
  Anthropic / Gemini extras.
- One-shot CLI: `marvin "summarize this changelog" < CHANGELOG.md`.
- Library entry points: `marvin.fn`, `marvin.classify`, `marvin.extract`,
  `marvin.cast`, `marvin.generate`, `marvin.run`, `marvin.Agent`,
  `marvin.Task`, `marvin.Thread`.
- Configure via env: `OPENAI_API_KEY`, `MARVIN_AGENT_MODEL=openai:gpt-4o-mini`
  (LiteLLM-style provider prefix), `ANTHROPIC_API_KEY` etc. for other
  providers.

## 2. Repo + version + license

- Repo: <https://github.com/PrefectHQ/marvin>
- Latest release: **v3.2.7**
- License: **Apache-2.0** —
  <https://github.com/PrefectHQ/marvin/blob/main/LICENSE>
- Default branch: `main`
- Language: Python

## 3. Models supported

Anything routable via Pydantic-AI / LiteLLM under the hood: OpenAI
(default), Anthropic, Gemini, Groq, Mistral, Bedrock, any
OpenAI-compatible endpoint (Ollama, vLLM, LM Studio, OpenRouter,
DeepSeek). Provider/model selected with the `openai:gpt-4o-mini`-style
prefix in `MARVIN_AGENT_MODEL` or per-call `model=` kwarg.

## 4. MCP support

None first-party. Marvin is not designed as an MCP client/server; if you
need MCP, mount the underlying Pydantic-AI `Agent` instead. Tooling in
Marvin happens through plain Python callables passed as `tools=[...]`.

## 5. Sub-agent model

`marvin.Agent` is the unit. A `Task` can be assigned to one agent or to
a list of agents; `marvin.run("...", agents=[planner, coder, reviewer])`
runs a small orchestrated team where each agent has its own
instructions, model, and tools. Threads (`marvin.Thread`) carry shared
memory across multi-agent / multi-task runs. No auto-spawn of
subprocess agents — composition is explicit Python.

## 6. Telemetry stance

Off in the OSS library. No analytics ping. Egress is exactly the LLM
provider you configure plus any tool calls your code makes. Optional
Prefect / Logfire integration is opt-in via env vars.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **Pydantic-typed LLM round-trips as the primary
API.** `@marvin.fn def extract_invoice(text: str) -> Invoice: ...` turns
a Python signature into a validated structured-output call: the model's
response is parsed into the declared Pydantic model and re-prompted on
validation error, so caller code gets typed objects (or a real
exception) instead of `json.loads(...)` boilerplate. The same idea
extends to `marvin.classify(text, labels=["bug","feature","question"])`,
`marvin.extract(text, target=list[Citation])`, and
`marvin.cast(value, target=DateTime)` — small, boring, deterministic
LLM primitives that compose cleanly with normal Python.

**Weakness.** Not a coding agent and not a TUI: there is no file-edit
loop, no `git diff` review, no MCP client, no shell sandbox. The CLI is
a thin one-shot wrapper, not a chat REPL. v3 is also a hard rewrite of
the v1/v2 API — older tutorials and StackOverflow answers are obsolete,
and the Prefect-aligned `marvin.run` / `marvin.Task` surface is the
only one that gets active development. If your stack is Node/Go,
nothing here is reusable.

**When to choose.** You are writing Python application code and want
LLM calls to look like normal typed functions, not free-form strings —
classification, extraction, casting, summarization, structured
generation. Pair with Prefect if you need orchestration, with FastAPI
if you need a service layer, or with [`pydantic-ai`](../pydantic-ai/)
if you need lower-level control over the agent loop. Skip if you want
a terminal coding agent that edits files
([`opencode`](../opencode/), [`aider`](../aider/),
[`claude-code`](../claude-code/)) or if you are not in Python.
