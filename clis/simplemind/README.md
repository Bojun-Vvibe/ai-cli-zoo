# simplemind

> Snapshot date: 2026-04. Upstream: <https://github.com/kennethreitz/simplemind>

A **deliberately small Python client for LLM providers** by
Kenneth Reitz (of `requests` fame) whose stated goal is to
"replace LangChain and LangGraph for most common use cases" by
collapsing the surface area to: pick a provider, send messages,
get text back, optionally bind a Pydantic schema for structured
output, optionally hold a conversation that remembers itself.
The README and the API are both about a page long. There is no
chain abstraction, no graph DSL, no agent runtime, no callback
manager, no document-loader registry — just `sm.generate_text`,
`sm.create_conversation`, and `sm.generate_data`.

It is the catalog's reference for **a Reitz-style minimal LLM
client** — the same design ethos as
[`mirascope`](../mirascope/) / [`ell`](../ell/) but optimised
for "least concepts to learn", and a deliberate counter-example
to the "every framework grows into LangChain" gravity well that
defines this corner of the catalog.

## 1. Install footprint

- `pip install simplemind` (Python 3.10+).
- Pulls `pydantic`, `httpx`, plus per-provider extras (`pip
  install 'simplemind[anthropic]'` adds the Anthropic SDK,
  `[openai]` adds OpenAI, `[gemini]` adds Google's, `[groq]`,
  `[ollama]`, `[xai]`). ~30 MB venv per provider.
- Models: OpenAI (`gpt-4o`, `gpt-4o-mini`, etc.), Anthropic
  (`claude-3-5-sonnet`, `claude-3-haiku`), Google (Gemini), xAI
  (Grok), Groq, Ollama (any local model). Each provider is a
  small adapter class; adding one is ~50 lines.
- No CLI binary of its own — `python -m simplemind` is not a
  thing in this version. The "CLI flavour" is via one-liner
  REPL use: `python -c "import simplemind as sm;
  print(sm.generate_text('hi', llm_provider='ollama'))"`.

## 2. Repo, version, license

- Repo: <https://github.com/kennethreitz/simplemind>
- Version checked: **v0.3.3** (latest tag,
  `b5a901efaf6cce9a63dabe8103cc02cd534323ba`; matches
  `pyproject.toml` `version = "0.3.3"`).
- HEAD pinned at this snapshot:
  `b5a901efaf6cce9a63dabe8103cc02cd534323ba`.
- License: Apache-2.0. License file at
  [`LICENSE`](https://github.com/kennethreitz/simplemind/blob/main/LICENSE).

## 3. What it actually does

Three primitives. That is the whole library.

**`generate_text`** is one round-trip:

```python
import simplemind as sm
sm.generate_text(
    prompt="Summarize this in one sentence.",
    llm_provider="anthropic",
    llm_model="claude-3-5-sonnet-latest",
)
```

**`create_conversation`** is a stateful chat that remembers the
turns:

```python
conv = sm.create_conversation(llm_provider="openai", llm_model="gpt-4o-mini")
conv.add_message("user", "What's the capital of France?")
print(conv.send())          # "Paris."
conv.add_message("user", "And of Germany?")
print(conv.send())          # remembers Paris context
```

The `Conversation` object is a Pydantic `BaseModel` with a
`messages: list[Message]` field — serialise it to JSON to
persist, load it back to resume. There is no vector store, no
summary memory, no automatic truncation; if context overflows,
the underlying provider raises and you handle it.

**`generate_data`** binds a Pydantic schema for structured output:

```python
class Recipe(BaseModel):
    name: str
    ingredients: list[str]
    steps: list[str]

r = sm.generate_data(
    prompt="A simple chocolate-chip cookie recipe.",
    llm_provider="openai",
    llm_model="gpt-4o",
    response_model=Recipe,
)
```

Internally this uses each provider's native structured-output
mode (OpenAI `response_format`, Anthropic `tool_use`,
Gemini's `response_schema`) — no Instructor-style retry-on-parse
loop. Validation failures raise; the caller decides whether to
retry.

A **plugin hook** lets you register pre / post hooks on
conversations (e.g. inject a system prompt, log every turn) but
the design pressure is to *not* grow the core surface — most use
cases are expected to be one of these three calls plus
host-language code.

## 4. MCP support

None. The library does not implement client or server MCP. It
also does not implement OpenAI-style function calling at the
public API level — tool use is per-provider native and not
abstracted. This is the explicit non-goal: simplemind is the
LLM-call hop, not the agent loop.

## 5. Sub-agent model

None. Multi-agent or multi-step orchestration is your loop in
host code. The library has no `Agent`, no `Runner`, no `Graph`.

## 6. Telemetry stance

Off. No analytics, no callback manager, no hosted control plane.
The provider SDKs do whatever they do; simplemind itself is
silent. Logging is opt-in via the plugin hooks.

## 7. Token / context strategy

None applied. Conversations accumulate raw `Message` objects
forever; the host is responsible for truncation, summarisation,
or rotation if the workload outgrows the model's context
window. This is consistent with the library's "do less"
posture — it does not silently drop the oldest message or
inject a hidden summary call.

## 8. Hot keybinds

N/A — library, not a CLI / TUI. The README explicitly suggests
"use it from a Jupyter notebook or a Python REPL" as the
intended interactive surface.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **An LLM client small enough to read in one
sitting**, with a one-page mental model: three top-level
functions, multi-provider parity, native Pydantic structured
output. The Reitz design tax — readable code, no clever
metaclasses, every public symbol does what its name says — is
unusual in this corner of the catalog and makes simplemind a
good "graduate from a notebook spike" target where
[`langchain`](../../) feels like overkill and
[`instructor`](../instructor/) feels too schema-heavy.

**Weakness.** Slow release cadence (last tag early 2025; the
LLM provider landscape has moved underneath it — newer
Claude / Gemini / GPT model ids may need a small adapter patch).
No tool / function calling abstraction at all, so any agent
loop is bring-your-own. No streaming token API in the public
surface (provider SDKs stream, but `sm.generate_text` is
non-streaming). No async API in v0.3.3 — every call is blocking
`httpx.Client`. Test coverage is light; production use should
treat this as "vendored code I can read and patch", not "managed
dependency I trust to be maintained for me".

**Choose simplemind when** the workload is "talk to an LLM,
maybe with a Pydantic schema, maybe stateful, definitely
multi-provider, and I want to read the framework's source in 20
minutes". **Choose something else when** you need streaming
([`mirascope`](../mirascope/), [`ell`](../ell/)), a
retry-on-validation-failure loop ([`instructor`](../instructor/),
[`baml`](../baml/)), real agent orchestration
([`pydantic-ai`](../pydantic-ai/),
[`langgraph`](../langgraph/)), tool-call abstractions across
providers ([`litellm`](../litellm/)), or a maintained,
release-cadenced commercial-friendly dependency.
