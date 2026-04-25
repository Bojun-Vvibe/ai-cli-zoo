# mirascope

> Snapshot date: 2026-04. Upstream: <https://github.com/Mirascope/mirascope>

"**The LLM Anti-Framework.**" `mirascope` is a Python library that
treats every LLM call as a typed Python function: you decorate a
function with `@llm.call(provider=..., model=..., response_model=...)`,
write the prompt as the docstring (or as a `messages` builder),
type-hint the arguments, and the return value is a validated Pydantic
object — or a streamed partial of one — instead of a free-form
string. Tools are just Python functions you pass in; agents are
loops over those decorated calls. Stated design goal: "no DSL, no
YAML, no chain object" — the prompt, the schema, the provider, and
the retries all live in normal typed Python so a static type checker
can see them.

## 1. Install footprint

- `pip install "mirascope[openai]"` (Python ≥ 3.10). Provider extras
  are independent: `[anthropic]`, `[google]` (Gemini AI Studio +
  Vertex), `[mistral]`, `[cohere]`, `[groq]`, `[xai]`, `[bedrock]`,
  `[azure]`, `[litellm]`, `[ollama]`, plus `[all]` for everything.
- Library-first: `import mirascope.llm as llm` and call decorated
  functions from any script, REPL, FastAPI handler, or notebook —
  there is no dedicated `mirascope` CLI binary.
- Optional integrations: `[langfuse]`, `[logfire]`, `[opentelemetry]`,
  `[hyperdx]` for tracing; `[tenacity]` is already a transitive dep
  for the retry / re-ask loop.
- Sister project `lilypad` (separate repo) layers prompt versioning
  and an evaluation UI on top — out of scope for this entry.

## 2. Repo + version + license

- Repo: <https://github.com/Mirascope/mirascope>
- Latest release: **v2.4.0** (2026-03-08)
- License: **MIT** —
  <https://github.com/Mirascope/mirascope/blob/main/LICENSE>
- Default branch: `main`
- Language: Python

## 3. Models supported

OpenAI (chat completions + responses + Azure-style endpoints),
Anthropic (messages + tools + Bedrock + Vertex pass-through),
Google Gemini (AI Studio + Vertex via `google-genai`), Mistral,
Cohere, Groq, xAI Grok, AWS Bedrock, Azure OpenAI, plus a
LiteLLM-routed long tail for OpenRouter / Together / Fireworks /
DeepSeek / Perplexity / Cerebras. Local: Ollama, vLLM, LM Studio,
llama-cpp-python — anything serving an OpenAI-compatible chat
endpoint. Provider/model selected per-call via
`@llm.call(provider="anthropic", model="claude-3-5-sonnet-latest")`,
swappable at runtime via `llm.override(...)`.

## 4. When to use it

- You want **typed prompts as functions**: the prompt template uses
  the function's parameters, the return type is a Pydantic model
  validated by the library, and your IDE / `mypy` / `pyright` lights
  up everywhere the call is consumed. No `prompt.yaml`, no
  `Chain(...)` graph, no callbacks.
- You want **provider portability with structured outputs and
  streaming**: same decorated function runs on Claude, GPT, Gemini,
  Llama-on-Ollama, etc., and `stream=True` yields partial Pydantic
  instances as they arrive — useful for token-by-token UI.
- You want **tools that are just Python functions**: pass
  `tools=[search_web, get_weather]` to the decorator; type hints
  become the JSON schema; the loop calls them and feeds results
  back. Compatible with MCP-served tools via the `mcp` extra.

## 5. When NOT to use it

- You want a **shrink-wrapped multi-agent framework** with planners,
  role assignment, shared memory, conversational UI — out of scope by
  design (the "anti-framework" stance). Use
  [`crewai`](../crewai/) / [`agno`](../agno/) /
  [`smolagents`](../smolagents/) / [`metagpt`](../metagpt/) instead.
- You want a **CLI binary** to talk to LLMs from the shell —
  `mirascope` is a library; reach for [`llm`](../llm/),
  [`aichat`](../aichat/), [`mods`](../mods/),
  [`shell-gpt`](../shell-gpt/) for that.
- You are not on Python — there is no Mirascope port to TypeScript /
  Go / Rust at the snapshot date.
- You want a **runtime-pluggable, RAG-first pipeline** with chunkers,
  retrievers, re-rankers shipped — pair Mirascope with
  [`txtai`](../txtai/) / [`llmware`](../llmware/) /
  [`db-gpt`](../db-gpt/) instead, or stay in
  [`dspy`](../dspy/) if you want optimisable signatures.

## 6. Closest alternatives

- [`instructor`](../instructor/) — same Pydantic-typed-output thesis;
  Mirascope folds the *whole call* (prompt + provider + tools +
  retries) into one decorator, instructor focuses on
  `response_model=` patching of provider SDKs.
- [`pydantic-ai`](../pydantic-ai/) — typed agents from the Pydantic
  team; a thicker abstraction (Agent + RunContext + dependency
  injection + graphs) than Mirascope's plain-function stance.
- [`magentic`](../magentic/) — decorator-driven Python wrapper with
  similar "function = LLM call" feel; smaller provider matrix.
- [`dspy`](../dspy/) — also signatures-as-code, but with prompt
  *optimisation* as the headline; heavier conceptual surface.

## 7. Repo health (snapshot)

- Active development on `main`; v2.x line is the current major (a
  breaking redesign vs v1.x — see migration notes in the docs).
- Latest release: **v2.4.0** on 2026-03-08; minor releases land on a
  steady cadence.
- Documentation site (mirascope.com) and the in-repo
  `examples/learn/` walkthrough are kept in sync with `main`.
