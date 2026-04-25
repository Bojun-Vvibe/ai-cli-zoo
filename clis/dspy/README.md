# dspy

> Snapshot date: 2026-04. Upstream: <https://github.com/stanfordnlp/dspy>

"**Programming — not prompting — language models.**" `dspy` is a
Python framework that treats LLM behaviour as a *compiled artifact*:
you declare a typed `Signature` (input fields → output fields), pick
a `Module` (`Predict`, `ChainOfThought`, `ReAct`, `ProgramOfThought`,
`Refine`, `BestOfN`), wire modules together as ordinary Python, then
hand the resulting program to a `Teleprompter` / `Optimizer`
(`BootstrapFewShot`, `MIPROv2`, `BootstrapFinetune`, `COPRO`,
`SIMBA`) that *automatically* searches for prompts and few-shot
demonstrations that maximise a metric on your training set. The
prompt strings the model actually sees are an output of the compile
step, not a hand-tuned input — and when you swap the underlying
model (Claude → Llama → DeepSeek) you re-compile against the new
backend instead of re-writing prompts.

## 1. Install footprint

- `pip install dspy` (Python ≥ 3.9; project moved from
  `pip install dspy-ai` to `pip install dspy` with the 2.5 release).
  Pulls `litellm`, `openai`, `pydantic`, `tqdm`, `optuna`,
  `joblib`, `datasets`, `requests`. Optional retrieval extras:
  `dspy[chromadb]`, `dspy[qdrant]`, `dspy[weaviate]`,
  `dspy[milvus]`, `dspy[pinecone]`, `dspy[mongodb]`,
  `dspy[snowflake]`. Anthropic / Cohere / Mistral / Together /
  Groq / DeepSeek / Bedrock / Vertex routing rides on `litellm` so
  no per-provider extra is required.
- No dedicated CLI binary — `dspy` is library-first. `python
  my_program.py` runs your DSPy module; `dspy.inspect_history()`
  prints the last N completions; the optimisers expose `compile()`
  / `save()` / `load()` for the compiled prompt + demo bundle (a
  JSON file you check in alongside the source).
- Configuration is a single `dspy.configure(lm=..., rm=...)` call
  per process; everything else is constructor kwargs. No daemon, no
  background process, no on-disk state besides the compiled bundle.

## 2. Repo + version + license

- Repo: <https://github.com/stanfordnlp/dspy>
- Latest release: **3.2.0** (2026-04, `main` active through
  snapshot date)
- License: **MIT** —
  <https://github.com/stanfordnlp/dspy/blob/main/LICENSE>
- Default branch: `main`
- Language: Python

## 3. Models supported

Anything `litellm` can route — OpenAI (Chat + Responses + Azure),
Anthropic (Claude 3.5 / 3.7 / 4 family with extended thinking),
Gemini (AI Studio + Vertex), Bedrock, Mistral, Cohere, Groq,
DeepSeek (V3, R1, Coder), Together, Fireworks, OpenRouter,
Cerebras, xAI, Perplexity, plus local Ollama, vLLM, llama.cpp,
LM Studio, mlx-lm, openllm via their OpenAI-compatible endpoints
(`dspy.LM("openai/...", api_base="http://127.0.0.1:11434/v1")`).
A first-party `dspy.LM` class supports caching, retries, structured
output (per-provider best mode auto-picked), and per-call
temperature / max-tokens / response-format overrides. Embedding /
retrieval models routed via the `dspy.Embedder` class plus the
`dspy.<vector store>RM` retrievers.

## 4. MCP support

Yes (client). `dspy.Tool.from_mcp(server)` mounts an MCP server's
tools as DSPy `Tool` objects callable by `ReAct` / `Refine` modules
mid-program; the agent's tool surface composes DSPy-native Python
tools and MCP tools transparently. No first-party server mode —
expose a compiled DSPy program as MCP by wrapping `program.forward()`
in your own server (the program's signature carries enough type info
that the wrapper is ~20 lines).

## 5. Sub-agent model

Module composition is the only "sub-agent" model: a `dspy.Module`
calls other modules in its `forward()` like ordinary Python. `ReAct`
runs a tool loop with the LM as the planner; `BestOfN` /
`Refine` / `MultiChainComparison` fan out N parallel module calls
and synthesise; `ProgramOfThought` emits Python that runs in a
sandbox and feeds output back to the LM. No manager-LLM-spawning-
worker-LLMs crew abstraction; per-module `lm=` lets a planner
module run on Claude Opus while a worker module runs on a cheap
local Qwen-Coder.

## 6. Telemetry stance

Off (no analytics endpoint; the library makes no calls of its own
besides what your `dspy.LM` is configured to call). Optional OTel
hooks via `dspy.settings.callbacks = [...]` — first-class
adapters for MLflow, Phoenix / Arize, Langfuse, Weights & Biases.
Compiled-prompt bundles save to local JSON; nothing leaves the
process unless you explicitly push to one of the tracing backends.

## 7. Prompt-cache strategy

Two layers. Per-process: `dspy.settings.cache` (default `True`)
hashes `(model, prompt, kwargs)` and stores responses in an LRU
in-memory dict — re-running an optimiser does not re-pay for
identical sub-calls. On-disk: `dspy.LM(..., cache=True,
cache_dir="...")` writes a SQLite-backed cache that survives
process restart, which is the load-bearing piece for
`MIPROv2` / `BootstrapFewShot` runs that fan out tens of thousands
of LM calls during the search. Per-call opt-out with
`dspy.context(cache=False)`. No prompt-prefix-cache awareness yet
(the underlying provider's cache hits still apply, DSPy just does
not surface them as a metric).

## 8. Hot keybinds

N/A — library, not a TUI. Common runtime helpers:

- `dspy.inspect_history(n=1)` — print the last completion
  (prompt + response + token counts) to stdout for debugging.
- `program.save("compiled.json")` / `dspy.load("compiled.json")` —
  ship a compiled program (prompts + demos + per-module config) as
  one JSON file you can review and check into git.
- `dspy.Evaluate(devset=..., metric=..., num_threads=N)` —
  parallel eval over a dev set, returning per-example scores.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Prompts are an output of compilation, not
an input.** You write a typed program (`Signature("question ->
answer: str"`, `ChainOfThought(...)`, retrieval module, judge
module), declare a metric (`exact_match`, custom Python, or an
LLM-as-judge `dspy.Suggest`), point an optimiser at a small
training set (50–500 examples is enough for `BootstrapFewShot`,
`MIPROv2` scales further), and the framework searches the joint
space of *system prompt wording* + *which few-shot demos to
include* + *per-module decoding params* against your metric. The
output is a `compiled.json` you check into git. Six months later
you swap GPT-4o for DeepSeek-V3, re-run `compile()`, and the new
prompts that fall out are tuned to *that* model's quirks instead
of the prompt you originally wrote for OpenAI. The 3.x line adds
`Refine` and `BestOfN` as first-class modules so the same compile
loop optimises inference-time-compute decisions, not just static
prompts. `ReAct` + MCP tools means the agent loop is also
optimised, not just the chat layer.

**Weakness.** **The compile step is slow and the abstractions
are non-trivial.** A `MIPROv2` run against a 200-example dev set
can fan out 5–20 k LM calls — typically minutes-to-hours wall
clock and tens to hundreds of dollars depending on provider.
Worth it for a production system you ship and re-compile
quarterly; overkill for a prototype where one
hand-written prompt would do. The mental model
(`Signature` → `Module` → `Optimizer`) and the mid-tier concepts
(`Predict` vs `ChainOfThought` vs `ReAct`, demo bootstrapping,
proposal models, `Suggest` vs `Assert`) take a few days of
reading and an example project to internalise — there is no
"one-line agent" entry point. Debugging optimised prompts is
awkward: you read a JSON blob of demos to figure out *why* the
compiled program scored well, and the prompts can be quite long
(tens of thousands of tokens of demos for a strong few-shot
configuration). Documentation has improved sharply in 2.5 / 3.x
but the optimiser zoo (`BootstrapFewShot`, `BootstrapFewShotWithRandomSearch`,
`COPRO`, `MIPROv2`, `MIPROv2-Zero`, `BootstrapFinetune`, `SIMBA`,
`Ensemble`) is still a "read the paper" moment for choosing.

**When to choose.** You have a measurable LLM task — RAG accuracy
on a curated eval set, classification F1, agent task-completion
rate from a benchmark like SWE-bench / Big-Bench / HotpotQA — and
prompt regressions when you switch provider or model version are
costing you real time. Pair with [`promptfoo`](../promptfoo/) or
[`deepeval`](../deepeval/) for the eval rubric layer (deepeval's
metrics are directly usable as `dspy.Evaluate` metric callables);
pair with [`distilabel`](../distilabel/) if you need to first
*generate* the labelled training set the optimiser will compile
against. Skip if your model integration is "one prompt, one call,
ship it" — the compile loop is overhead with no payoff. Skip if
you want a multi-agent crew abstraction with manager-LLM
delegation — DSPy gives you typed module composition, not a
[`crewai`](../crewai/) / [`metagpt`](../metagpt/) / [`agno`](../agno/)
role hierarchy. Reach for [`instructor`](../instructor/) instead
if all you need is "Pydantic class out of an LLM call" without
the optimisation loop.
