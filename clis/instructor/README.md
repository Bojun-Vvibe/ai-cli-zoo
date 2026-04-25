# instructor

> Snapshot date: 2026-04. Upstream: <https://github.com/567-labs/instructor>

"**Structured outputs for LLMs.**" `instructor` is a thin wrapper
around the major LLM SDKs (OpenAI, Anthropic, Gemini, Mistral,
Cohere, Groq, Bedrock, Vertex, Ollama, llama-cpp-python, litellm)
that lets you pass a Pydantic model as `response_model=` and get
back a validated instance instead of a free-form string. Under the
hood it picks the right structured-output mode per provider —
function/tool calling, JSON mode, JSON schema, or a re-ask loop with
the validation error fed back into the model — so the same Pydantic
class works against every backend without per-provider branching in
your code. Ships a CLI (`instructor jobs`, `instructor files`,
`instructor batch`, `instructor usage`, `instructor docs`) for
managing OpenAI batch / fine-tune jobs and reading docs offline.

## 1. Install footprint

- `pip install instructor` (Python ≥ 3.9). Core install pulls
  `pydantic`, `openai`, `tenacity`, `typer`, `rich`, `docstring-
  parser`, `jiter`. Per-provider extras: `instructor[anthropic]`,
  `instructor[google-genai]`, `instructor[mistralai]`,
  `instructor[cohere]`, `instructor[litellm]`,
  `instructor[bedrock]`, `instructor[vertexai]`, `instructor[groq]`,
  `instructor[xai]`, `instructor[perplexity]`, `instructor[writer]`.
- Single CLI binary entry point: `instructor` (Typer-based, batch /
  files / jobs / usage / docs subcommands).
- Also ships `examples/` covering ~80 patterns (validators, partial
  streaming, tool calls, multi-modal, RAG, knowledge graphs, agentic
  loops). Notebook-friendly — most users invoke as a library, the
  CLI is for OpenAI batch / fine-tune workflow management.
- TypeScript port: `@instructor-ai/instructor-js` (separate repo,
  same author).

## 2. Repo + version + license

- Repo: <https://github.com/567-labs/instructor>
- Latest release: **v1.15.1** (2026-04-03)
- License: **MIT** —
  <https://github.com/567-labs/instructor/blob/main/LICENSE>
- Default branch: `main`
- Language: Python

## 3. Models supported

Anything reachable through one of the wrapped SDKs. Native clients
for OpenAI (chat completions + responses API + batch + Azure-style
endpoints), Anthropic (messages + tools + Bedrock + Vertex),
Google (Gemini AI Studio + Vertex via `google-genai`), Mistral,
Cohere, Groq, xAI, Perplexity, Writer, Cerebras, Fireworks,
DeepSeek, Together, OpenRouter (via OpenAI-compatible endpoints),
plus a `litellm`-routed fallback for the long tail. Local: Ollama,
llama-cpp-python, vLLM, LM Studio — anything exposing an
OpenAI-compatible chat endpoint or implementing JSON mode. Mode
selection is automatic per provider — `Mode.TOOLS` for OpenAI/
Anthropic, `Mode.JSON_SCHEMA` for Gemini structured output,
`Mode.JSON` for backends without a native schema path — and
overridable via `mode=` on the `from_*` factory.

## 4. MCP support

No (not the protocol). `instructor` is one level below MCP — it's
the structured-output adapter you'd put *inside* an MCP server's
tool implementation to get typed args back from a model, not an MCP
client or server itself. Pair with [`fast-agent`](../fast-agent/) or
any MCP-aware host if you need MCP-shaped tool routing on top.

## 5. Sub-agent model

None at the framework level. Each `client.chat.completions.create
(response_model=Foo, ...)` call is one model round-trip plus up to
`max_retries` re-ask loops on `ValidationError`. The re-ask loop is
the only "agent" behaviour: when Pydantic raises, the error
messages are appended to the conversation as a user turn and the
model is asked to repair its output, up to N times (default 1, often
set to 2-3). Building a real agent on top of `instructor` is a few
dozen lines — return a discriminated-union `Action` model from each
turn, dispatch on the variant, append the result, loop — and the
`examples/agents/` directory shows several patterns (tool router,
plan-and-execute, ReAct-style, query rewrite). No persistent state,
no parallel sub-agents, no scheduler — that's intentional; the
project's stance is "give you typed I/O, get out of the way."

## 6. Telemetry stance

Off. The library makes no network calls of its own — egress is
exactly what your underlying SDK would have made plus the re-ask
turns on validation failure. Optional `instrumentation/` integrations
for OpenTelemetry, Logfire, Langfuse, Braintrust, and W&B Weave are
explicit opt-in imports (`from instructor.instrumentation.langfuse
import LangfuseInstrumentor`); nothing is registered by default. The
CLI's `instructor docs` command serves docs from a local copy
bundled in the wheel — no network fetch.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **One Pydantic class is your contract with every
LLM provider.** Define `class Order(BaseModel): item: str; qty:
PositiveInt; ship_by: datetime` once and `client.chat.completions.
create(response_model=Order, messages=[...])` works against OpenAI,
Anthropic, Gemini, Mistral, Cohere, Groq, Bedrock, Vertex, Ollama,
and the rest with no per-provider conditional in your code.
Validation errors trigger an automatic re-ask, so the model gets
told "qty must be positive integer, you gave 'three'" and corrects
itself; you never see a malformed response unless `max_retries` is
exhausted. Streaming is structured too — `client.chat.completions.
create_partial(response_model=Order, ...)` yields progressively-
filled Pydantic instances as tokens arrive, which is the right
primitive for any UI that wants to render fields live without
parsing JSON-in-progress yourself. Discriminated unions get you
typed agent action selection for free.

**Weakness.** **It's a library, not a runtime.** No agent loop, no
tool router, no memory, no eval harness, no observability dashboard
— all of that is on you (or on whatever framework you stack on
top). The structured-output mode picked per provider is the
provider's mode, with the provider's quirks: Gemini's JSON schema
won't accept some Pydantic features (recursive refs, certain
unions); Anthropic's tool mode requires a tool schema even when you
just want JSON; Ollama's structured output depends on the
underlying GGUF model actually following grammar (small models
drift). The re-ask loop costs tokens on every retry, and validators
that throw on semantically-wrong-but-syntactically-valid output
(e.g. "this date is in the past") can chew through `max_retries`
without the model ever fixing the underlying confusion. The CLI is
mostly OpenAI-batch-management surface; don't pick `instructor` for
its CLI.

**When to choose.** You're writing application code (RAG pipeline,
extraction service, chatbot turn handler, agent step) and you want
the model's output to land as a typed object you can pass into the
rest of your system without a `try: json.loads(... )` ladder. Pick
`instructor` when you'd otherwise write a per-provider JSON-parsing
wrapper, when you want to swap providers behind an A/B without
rewriting your output code, or when your output schema has real
constraints (positive ints, regex, enums, cross-field invariants)
that a Pydantic validator can catch and a model can repair on
re-ask. Pair with [`promptfoo`](../promptfoo/) or
[`deepeval`](../deepeval/) for eval, with [`marvin`](../marvin/) if
you want a higher-level "function-shaped" surface instead of
`response_model=`, or with [`fast-agent`](../fast-agent/) /
[`pydantic-ai`](../pydantic-ai/) if you need a full agent runtime
on top. Skip if you want an end-to-end agent framework out of the
box, or if your output is genuinely free-form text (summaries, prose
generation) where a schema would be a straitjacket.
