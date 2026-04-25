# outlines

> Snapshot date: 2026-04. Upstream: <https://github.com/dottxt-ai/outlines>

"**Structured outputs for LLMs.**" `outlines` is a Python library
that *constrains* an LLM's token sampling so the generated string
is provably valid against a JSON Schema, a Pydantic model, a
Python type (`list[int]`, `dict[str, float]`), a regex, or a
context-free grammar (Lark `.lark` files). Unlike "ask the model
nicely for JSON and re-prompt on failure" approaches, the
constraint is enforced at the logits level: the next-token
distribution is masked so only tokens that keep the partial output
on a valid prefix of the target language can ever be sampled, which
means **valid output on the first try, every try**, even from small
local models that would otherwise hallucinate brackets or commas.

## 1. Install footprint

- `pip install outlines` (Python ≥ 3.10). Backend extras:
  `outlines[transformers]` (HuggingFace local models),
  `outlines[vllm]` (vLLM-served models with logits processors),
  `outlines[llamacpp]` (`llama-cpp-python`),
  `outlines[mlx]` (Apple Silicon via `mlx-lm`),
  `outlines[exllamav2]`. Cloud / OpenAI-compatible providers
  (OpenAI, Anthropic, Gemini, Mistral, Cohere, Groq, Together,
  Bedrock, Vertex, OpenRouter, DashScope, DeepSeek, llama.cpp
  HTTP) ride through generic OpenAI-compatible adapters and need
  no per-provider extra. Pulls `pydantic`, `jsonschema`,
  `interegular`, `lark`, `numpy`, `numba` (FSM compilation).
- No dedicated CLI binary — `outlines` is library-first. Programs
  are `python my_extractor.py` scripts using `outlines.from_*` to
  build a typed model wrapper.
- Heavy on first run: the regex / JSON-schema → finite-state-machine
  compilation step is `numba`-jitted and can take a few seconds for
  a complex schema, then cached on disk under `~/.cache/outlines/`
  for instant reuse on subsequent runs against the same tokenizer.

## 2. Repo + version + license

- Repo: <https://github.com/dottxt-ai/outlines>
- Latest release: **1.2.12** (2026-04)
- License: **Apache-2.0** —
  <https://github.com/dottxt-ai/outlines/blob/main/LICENSE>
- Default branch: `main`
- Language: Python

## 3. Models supported

Two tiers. **Local with full constraint enforcement** (logits
masking, the strongest mode): any HuggingFace `transformers`
causal LM (Llama 3.x/4, Qwen 2.5/3, Mistral, Phi 3/4, Gemma 2/3,
DeepSeek, Granite, Yi, Command-R), `llama.cpp` (any GGUF), `vLLM`
(any model vLLM serves), `mlx-lm` (Apple Silicon), `ExLlamaV2`. In
this mode `outlines` owns the sampling loop and guarantees valid
output. **Remote / API-only with best-effort enforcement**: OpenAI
(via the provider's native `response_format` / `tool_choice`
modes), Anthropic, Gemini, Mistral, Cohere, Groq, OpenRouter,
DeepSeek, Bedrock, Vertex — here the enforcement is whatever the
provider exposes (which is strong for OpenAI/Anthropic structured
output and weak for vanilla chat-completions providers without
`response_format`); `outlines` adapts the schema and post-validates
with an automatic re-prompt loop on failure. Embedding models out
of scope (this is a generation library, not a retrieval one).

## 4. MCP support

No (one level below MCP — use `outlines` *inside* an MCP tool
implementation to get a typed dict back from a model, then return
the validated dict as the tool's response). The `outlines.Generator`
returned by `outlines.Generator(model, schema)` is callable like a
plain Python function, which makes wrapping it as an MCP tool a
~10-line `@mcp.tool` decorator on top.

## 5. Sub-agent model

None — `outlines` is the constraint layer, not an agent loop.
Composition happens in the *output type*: you define
`class Plan(BaseModel): steps: list[Step]; estimated_minutes:
int` and the model emits the full structure in one call, instead
of a manager-LLM dispatching to per-step worker LLMs. For agent-
style tool loops, pair with [`pydantic-ai`](../pydantic-ai/),
[`smolagents`](../smolagents/), [`fast-agent`](../fast-agent/),
or call `outlines` inside your own loop to constrain each turn's
output.

## 6. Telemetry stance

Off (no analytics endpoint, no opt-in pings; the library makes no
calls of its own besides what your configured backend / provider
makes). FSM-cache writes go to `~/.cache/outlines/` and can be
disabled with `OUTLINES_CACHE_DIR=/dev/null`. For local backends
the only egress is HuggingFace Hub on first model pull; for remote
backends the provider sees your prompts as it would for any other
client.

## 7. Prompt-cache strategy

Two non-overlapping caches. **FSM cache** (the load-bearing one):
`(tokenizer, schema)` → compiled finite-state-machine, persisted
to `~/.cache/outlines/`. First call against a new schema +
tokenizer pair takes seconds (numba-compiled DFA build); every
subsequent call is instant. **Prompt cache**: not built-in — the
library does not memoise `(prompt, schema)` → completion, on the
assumption that you want sampling diversity on re-runs. For prompt
memoisation use the underlying backend's cache (vLLM prefix cache,
provider-level prompt caching) or wrap the generator yourself.

## 8. Hot keybinds

N/A — library, not a TUI. Common runtime helpers:

- `outlines.from_transformers(model, tokenizer)` /
  `outlines.from_vllm(...)` / `outlines.from_llamacpp(...)` /
  `outlines.from_openai(client, model_name)` — wrap a backend.
- `gen = outlines.Generator(model, MyPydanticModel)` then
  `gen("Extract...")` returns a validated `MyPydanticModel`
  instance directly.
- `outlines.Generator(model, regex_str)` — regex constraint;
  `outlines.Generator(model, lark_grammar)` — CFG constraint;
  `outlines.Generator(model, list[Choice])` — categorical pick.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Token-level constraint masking turns "model
sometimes returns valid JSON" into a mathematical impossibility
of returning anything else.** Define
`class Invoice(BaseModel): vendor: str; total_cents: int;
line_items: list[LineItem]`, build a generator against a local
Llama-3.1-8B (a model that's mediocre at "please return JSON"
prompts), and **every** completion is a syntactically valid
`Invoice` because the sampler literally cannot emit a token that
breaks the schema. Same idea for regex (`r"\d{3}-\d{2}-\d{4}"`
guarantees a SSN-shaped string), for context-free grammars (parse a
DSL the model invents), for categorical choice (the model picks
exactly one of `["positive", "neutral", "negative"]`, never
"positive!" or "Positive."). The benefit compounds at the small-
model end: a 4 B local model with `outlines` constraints often
beats a 70 B unconstrained model on extraction tasks because the
small model's "knowledge of the answer" survives intact when it
does not also have to spend capacity on JSON syntax. The 1.x line
brings the public API into stability, ships first-class adapters
for the major OpenAI-compatible cloud providers, and folds in
streaming-aware FSMs so partial structured output is renderable
mid-generation.

**Weakness.** **Constraint enforcement requires *owning the
sampling loop*, which means the strong guarantees only land on
local backends.** Remote providers (OpenAI, Anthropic, Gemini)
expose a fraction of the constraint surface — JSON Schema with
many caveats, no arbitrary regex, no full CFGs — so for cloud
endpoints you fall back to the provider's native structured-output
mode plus a re-prompt loop on validation failure, which is the
same thing [`instructor`](../instructor/) does and not really
"outlines magic". The FSM compile step for very large schemas
(deeply-nested Pydantic with many `Union` branches) can be slow
the first time and produce DFAs that mask aggressively enough to
hurt sampling diversity — choose schema shape with that in mind.
The library leans Pythonic; bindings for other languages (Rust
`outlines-core`, JS) exist but are less feature-complete than the
Python surface.

**When to choose.** You are doing structured extraction (PDFs →
typed records, log lines → events, free-form chat → API calls,
HTML → catalog rows) and the cost of a malformed JSON downstream
is high (re-runs, dropped records, on-call pages). Especially
choose `outlines` if you are running a **local** model — that is
where the token-level constraint pays off most, both in
correctness and in unlocking small-model use for tasks that would
otherwise need a 70 B+ class model. Pair with
[`vllm`](../vllm/) or [`mlx-lm`](../mlx-lm/) on the inference side
to combine "fast local" with "always valid"; pair with
[`distilabel`](../distilabel/) if you are generating a synthetic
labelled dataset (constraint-enforced outputs make the labelling
step trivially correct). Reach for
[`instructor`](../instructor/) instead if you are exclusively on
hosted providers and want a thinner integration that just wraps
each provider's native structured-output endpoint with a Pydantic
re-ask loop. Reach for [`dspy`](../dspy/) if your real problem is
*prompt* selection across providers, not output structure.
