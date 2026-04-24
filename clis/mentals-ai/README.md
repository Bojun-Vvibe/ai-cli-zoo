# mentals-ai

> Snapshot date: 2026-04. Upstream: <https://github.com/turing-machines/mentals-ai>

A C++ agent runtime where **the agent program is a Markdown file**.
You write a `.md` file with `# Heading` blocks; each heading is a
"mental" (a reusable instruction unit) and link-style references
(`[goal: do X]`, `use: search`) define the control flow. The runtime
parses the Markdown into a graph and executes it as an agent loop.

It earns its slot because the *program-is-a-document* angle is unique
in this catalog: every other framework here either codes agents in a
real programming language (`magentic`, `gptscript`) or wires them
through YAML (`forge`, `continue`). `mentals-ai` makes the program a
human-readable Markdown spec that you can hand to a non-programmer.

## 1. Install footprint

- Build from source: `git clone && make` against a C++17 toolchain
  with libcurl and yaml-cpp; or use the provided Dockerfile.
- No prebuilt brew tap as of 2026-04 — packaging is upstream's main
  open issue. Linux x86_64 / aarch64 and macOS Apple Silicon both
  build cleanly; Windows requires WSL.
- Single static binary (~5 MB) once built. Per-user config at
  `./config.toml` next to the binary (no XDG support yet).
- `agents/` directory of Markdown agent definitions ships with the
  repo — examples include `web_search`, `coder`, `researcher`.

## 2. License

MIT.

## 3. Models supported

- OpenAI (built-in): GPT-4.x, o-series, omni — selected in
  `config.toml` via `model = "..."`.
- Any OpenAI-compatible endpoint by overriding `endpoint = "..."`
  (Ollama, vLLM, LM Studio, LiteLLM, Groq, OpenRouter, DeepSeek).
- Native Anthropic / Gemini integrations are *not* shipped as of
  2026-04 — the workaround is a LiteLLM proxy.

## 4. MCP support

**No.** Tools are C++-implemented native functions registered in the
runtime (`search`, `read_file`, `write_file`, `bash`, `python_eval`),
exposed to the model as OpenAI-style function calls. There is no
plugin loader for external MCP servers; adding a new tool means
recompiling.

## 5. Sub-agent model

**Yes — and this is the point.** A "mental" can `use:` another
mental, which the runtime executes as a child agent with its own
context and memory. Composition is by Markdown link: `[delegate:
research(topic="ant colonies")]` invokes the `research` mental as a
sub-agent and substitutes its return value back into the parent.
This makes mentals-ai one of the few catalog entries (along with
[`gptscript`](../gptscript/) and [`forge`](../forge/)) where
sub-agents are first-class language constructs rather than
runtime-spawned tasks.

## 6. Telemetry stance

**Off.** No analytics. Network egress is exclusively to the model
provider configured in `config.toml`. Conversation traces are written
to local `.log` files for debugging.

## 7. Prompt-cache strategy

None at the runtime layer. The runtime re-sends the full mental
context on each tool-call round, which interacts poorly with
Anthropic prompt caching (no cache breakpoints) but works fine with
OpenAI's automatic prefix caching when the system block is stable.
This is a known limitation tracked upstream.

## 8. Hot keybinds

N/A — non-interactive. Invoke `./mentals path/to/agent.md` and the
runtime executes to completion (or until a tool call requires user
confirmation, where it prompts on stdin).

## 9. Killer feature, weakness, when to choose

**Killer feature.** Markdown-as-program. An agent looks like:

```markdown
# main
goal: research ant colonies and write a 200-word summary
use: research, write

# research
prompt: search the web for {{topic}} and return the 5 best sources
use: search

# write
prompt: write a 200-word summary of {{research_output}}
```

A non-programmer can read, edit, and version this file. Compared to
[`gptscript`](../gptscript/) (`.gpt` files: English instruction +
declared tools/args, also runnable as agents), `mentals-ai` is
denser — Markdown link syntax instead of `tools:` / `args:` blocks
— and the runtime is a single C++ binary with no Go toolchain
required. Compared to [`forge`](../forge/) (YAML workflows with
per-step model routing), `mentals-ai` is editable in any Markdown
preview pane.

**Weakness.** The C++ build is the friction point. No prebuilt
binaries means every user pays the libcurl / yaml-cpp /
nlohmann-json install tax. The tool catalog is also fixed at
compile time — you cannot drop a new MCP server in. Documentation
is sparse; most users learn the link syntax by reading the
example agents.

**When to choose.**

- You want agent definitions to live as Markdown files in your repo,
  reviewable by non-engineers.
- You want recursive sub-agents as a *language* construct, not a
  runtime API.
- You are comfortable building a C++ project (or running it in
  Docker).

**When NOT to choose.**

- You need a polished out-of-box install — pick [`gptscript`](../gptscript/)
  for the same composability angle with a Go binary release pipeline.
- You need MCP server support — pick [`opencode`](../opencode/),
  [`crush`](../crush/), [`continue`](../continue/), [`goose`](../goose/),
  or [`gptscript`](../gptscript/).
- You need native Anthropic / Gemini support without proxying through
  LiteLLM.
- You want a TUI or interactive REPL — `mentals-ai` is batch-style.

## Representative invocation

```bash
# build once
$ git clone https://github.com/turing-machines/mentals-ai
$ cd mentals-ai && make
$ cp example_config.toml config.toml   # set OPENAI_API_KEY, model

# run a shipped example
$ ./mentals agents/web_search.md -d   # -d = debug trace to stdout
[main]   goal received: "What is the latest LLM context-length record?"
[search] tool_call: web_search(query="largest LLM context window 2026")
[search] -> 3 hits
[main]   summary: As of 2026-04, ...
```
