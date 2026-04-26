# dagger

> Snapshot date: 2026-04. Upstream: <https://github.com/dagger/dagger>

A **programmable container engine** with a typed function graph, content-addressed
caching, and — as of the 0.18+ line — a first-class **agent runtime**: any
Dagger Function (Python / TypeScript / Go / Java / PHP) can be exposed as an
LLM-callable tool, and Dagger ships an `LLM` primitive that drives a tool-use
loop over those Functions inside a sandboxed container graph. Same engine that
runs CI pipelines also runs the agent — caching, secrets, OCI image building,
and trace UI come for free.

## 1. Install footprint

- `curl -fsSL https://dl.dagger.io/dagger/install.sh | sh` drops a single
  static `dagger` binary (Go); no Docker socket required at install time.
- Engine runs as a per-user containerd-shaped process; auto-pulls
  `registry.dagger.io/engine:<version>` on first call.
- `dagger init --sdk=python` scaffolds a Function module; `dagger develop`
  generates typed bindings; `dagger functions` lists callable surface;
  `dagger call <fn> --arg=...` is the universal entry point.
- `dagger shell` opens an interactive Bash-like REPL where every command
  is a Function call and every result is a typed object reference.

## 2. Repo, version, license

- Repo: <https://github.com/dagger/dagger>
- Version checked: **`v0.20.6`** (released 2026-04-16)
- License: **Apache-2.0**. License file:
  <https://github.com/dagger/dagger/blob/main/LICENSE>
- Default branch: `main`
- CLI source under `cmd/dagger/`; engine under `engine/`; SDKs under `sdk/`.

## 3. Models supported

- Provider-agnostic via the `LLM` core type. Configured by environment:
  `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY`, plus any
  OpenAI-compatible base URL (`OPENAI_BASE_URL` / `OPENAI_MODEL` covers
  Ollama, vLLM, llama.cpp, OpenRouter, Together, Groq, DeepSeek, etc.).
- The `LLM` primitive is one node in the graph, not the controller —
  swap providers per call, A/B in the same module without touching
  Function code.

## 4. MCP support

Yes, both directions. `dagger mcp` exposes a module's Functions as an
MCP server (so [`opencode`](../opencode/) / [`claude-code`](../claude-code/) /
[`crush`](../crush/) can call any Dagger Function as a tool). The `LLM`
primitive can also consume external MCP servers as tool sources, so a
Dagger-driven agent can mix native Functions with remote MCP tools in
one loop.

## 5. Sub-agent model

`LLM` is a chainable graph node: `dag.llm().withPrompt(...).withTools(myMod)
.sync()`. Multiple `LLM` nodes can fan out in the same graph, each with a
distinct prompt / model / tool subset, and their outputs join back as
typed objects. Cache hits short-circuit the LLM call when prompt + tool
schema + inputs are unchanged — a property no Python-only agent framework
in this catalog has.

## 6. Telemetry stance

Off by default. The engine emits OTel spans to the exporter you configure
(`OTEL_EXPORTER_OTLP_ENDPOINT`); the optional **Dagger Cloud** trace UI is
opt-in via `DAGGER_CLOUD_TOKEN` and is the only path that sends data
off-machine. Function execution is fully local — every container the
engine spawns runs against your local cache, secrets stay in the engine's
secret store and never enter the LLM context.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **One engine, one trace UI, one cache for both CI and
the agent.** A `build-and-test` pipeline and an `agent-fixes-failing-test`
loop are both Dagger Functions calling the same containerized graph; a
cache hit on the build step is shared between the CI run and the agent
run; every LLM tool call is a span in the same trace as the rest of the
pipeline. Compared to bolting a Python agent framework onto an existing
CI system, the integration cost is roughly zero — they are the same
program.

**Weakness.** The agent surface is the newest part of the product and
is still evolving (the `LLM` type's exact shape moved between 0.15 and
0.20). For pure agent-framework work without a CI / build story,
[`pydantic-ai`](../pydantic-ai/) or [`openai-agents-python`](../openai-agents-python/)
have more agent-shaped ergonomics and a faster moving ecosystem of
recipes. The engine pulls a multi-hundred-MB image on first use and
expects to manage its own runtime — this is not a single-binary
no-deps tool like [`llm`](../llm/).

**When to choose.**
- You already use Dagger (or want to) for CI, and you want the agent to
  run on the *same* substrate so caching, secrets, OCI builds, and
  traces are shared.
- You want **typed, language-portable** tool surfaces — define a
  Function once in Python, call it from a Go module, expose it to an
  LLM, and run it in CI, all without rewriting.
- You need **content-addressed caching** of LLM-driven steps so reruns
  of the same prompt + tools + inputs are free.

**When to skip.**
- You want a pure-Python agent framework with no engine to install →
  [`pydantic-ai`](../pydantic-ai/) / [`smolagents`](../smolagents/).
- You want a one-binary local coding TUI → [`opencode`](../opencode/) /
  [`crush`](../crush/) / [`codex`](../codex/).
- You only need parallel sandboxes per agent on your laptop with
  git-branch semantics → [`container-use`](../container-use/) (which is,
  not coincidentally, built on top of Dagger).

## 8. Compared to neighbors

| Tool | Shape | Agent surface | CI integration |
|------|-------|---------------|----------------|
| dagger | Container engine + typed function graph + LLM node | Native `LLM` primitive over Functions | Same engine runs CI and agent |
| [container-use](../container-use/) | MCP server on top of Dagger | None — substrate for other agents | Indirect (via Dagger) |
| [e2b](../e2b/) | Hosted Firecracker microVM control plane | None — substrate for SDK callers | None |
| [modal](../modal/) | Hosted serverless GPU + sandbox runtime | None — substrate | None |
