# llm-functions

> Snapshot date: 2026-04. Upstream: <https://github.com/sigoden/llm-functions>
> License file: <https://github.com/sigoden/llm-functions/blob/main/LICENSE>
> Pinned: commit `616d8d8` (2025-06-25). The repo has no tagged releases —
> it is consumed as a `main`-tracking checkout under `~/.config/aichat/functions/`.

A toolkit for building **LLM tools and agents in plain Bash, JavaScript, or
Python** — no Python framework, no DSL, no class hierarchy. Each tool is a
single executable file whose CLI flags double as the function schema; a
`build.sh` script scrapes those flags into a `functions.json` that any
OpenAI-style tool-calling client can consume.

It is the same author as [`aichat`](../aichat/) and is the canonical way to
extend aichat's tool-call surface, but the JSON it emits works with any
client that follows the OpenAI function-calling shape.

## 1. Install footprint

- `git clone https://github.com/sigoden/llm-functions ~/.config/aichat/functions`
  then `cd` in and run `./scripts/build.sh`. There is no package on PyPI,
  npm, or Homebrew — it is a checkout, not an installable.
- Each tool is implemented as a single file under `tools/` (`*.sh`,
  `*.js`, `*.py`); each agent is a directory under `agents/` containing
  `index.yaml` + tools.
- Runtime deps: bash, `jq`, plus whatever language(s) your tools are
  written in. No daemon, no global state.
- Build output: `functions.json` (the schema bundle) and `bin/`
  (executable shims). aichat picks both up automatically.

## 2. License

MIT.

## 3. Models supported

Indirectly: **any model whose client supports OpenAI-style function
calling.** llm-functions itself is just a toolbox + schema generator —
the calling LLM is whatever you point your client at. In practice that
means OpenAI, Anthropic (via tool_use), Gemini, Mistral, DeepSeek,
Groq, OpenRouter, Ollama (recent versions), llama.cpp, vLLM. If your
client speaks tool-calls, your tools work.

The bundled agents in `agents/` (e.g. `coder`, `todo`, `ner`) are
demonstrated against aichat but are model-agnostic.

## 4. MCP support

**Yes, as a *consumer*.** Ships an `mcp.sh` bridge tool that exposes any
running MCP server's tools through the same `functions.json` surface, so
an aichat (or other) session can call MCP tools without the client
itself implementing MCP. It is not an MCP *server* — it does not expose
your llm-functions tools over MCP.

## 5. Sub-agent model

The `agents/` directory is the closest thing: each agent is a YAML file
that pins a system prompt, a tool subset, and (optionally) a
documents folder for RAG. Calling `aichat -a coder` enters that agent.
Agents do not call other agents — there is no orchestration layer. If
you want hierarchical agents you compose them in shell.

## 6. Telemetry stance

**Off, with no opt-in.** The build is offline shell. Tools execute
locally; nothing is uploaded by the framework itself. Whatever a given
tool does (a `web_search.sh` will obviously hit a search API) is
visible in its source.

## 7. Prompt-cache strategy

N/A at this layer — caching is the calling client's concern. The
`functions.json` schema is small and stable, so it slots into the
client's system-prompt prefix and benefits from whatever prefix-cache
the client/provider already does.

## 8. Hot keybinds

There is no TUI. The relevant surface is the build commands and the
calling convention:

- `argc build` — regenerate `functions.json` + `bin/` shims after editing tools
- `argc check` — lint tool definitions and agent YAMLs
- `argc test@tool <tool> '<args-json>'` — invoke a tool directly from CLI to debug
- `argc test@agent <agent>` — load an agent and dry-run it
- `aichat --role %functions%` or `aichat -a <agent>` — actually use the tools

Each tool file follows the **argc** comment-DSL, e.g.:

```bash
#!/usr/bin/env bash
# @describe Get current weather for a city
# @option --city! The city name
# @option --unit[=celsius|fahrenheit] Temperature unit
```

Those comments become the JSON schema; that is the entire authoring
contract.

## 9. Killer feature, weakness, when to choose

**Killer feature.** The **argc-comment → JSON-schema pipeline**. You
write a normal shell/JS/Python script with a few comment lines; the
build extracts a fully-typed OpenAI function definition from those
comments. No decorators, no Pydantic models, no FastAPI — just a file
that is both a CLI and a tool definition. Adding a new capability to
your agent stack is "drop a file in `tools/`, run `argc build`."

**Weakness.** No package, no versioning, no test suite that the
framework runs for you, and the docs assume you already know aichat.
The project is one developer's vision of "tools should be files," and
if that vision does not match yours (you want types, you want
isolation, you want a server) you will fight it. Also: error messages
when the LLM sends bad arguments are whatever your shell prints, which
can be cryptic.

**When to choose.**

- You are using [`aichat`](../aichat/) and want real tool-calls without
  switching CLIs.
- You want to **wrap existing shell scripts as LLM tools** with near
  zero ceremony — no Python service, no Docker.
- You need an **MCP-to-OpenAI-functions bridge** for a client that
  lacks native MCP.
- You like the **"tools are files in a directory"** model and want a
  reference implementation of it.

**When not to choose.** You want a framework with strong typing,
isolation, observability, or hosted deployment — pick a Python agent
framework instead. You want the tools themselves to be portable across
many clients with minimal coupling — write them as MCP servers
directly. You want commercial support: this is a one-author project.
