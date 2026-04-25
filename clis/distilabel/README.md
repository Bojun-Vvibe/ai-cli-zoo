# distilabel

> Snapshot date: 2026-04. Upstream: <https://github.com/argilla-io/distilabel>

"**Synthetic data and AI feedback for engineers.**" `distilabel`
is a pipeline framework for building reproducible synthetic-data
generation and labelling jobs against LLMs. You declare a DAG of
`Step`s — `LoadDataFromHub`, `TextGeneration`, `UltraFeedback`,
`SelfInstruct`, `EvolInstruct`, `MagPie`, `PrometheusEval`,
`PairRM`, `FormatTextGenerationDPO`, `PushToHub` — wire them with
`>>` operators, and `pipeline.run()` materialises the DAG with
batching, retries, caching, distributed execution (Ray), and a
resumable on-disk checkpoint so a 12-hour generation job survives
a crash. Most of the published "synthetic SFT / DPO / KTO" datasets
on HuggingFace from the Argilla / HuggingFace H4 teams (UltraFeedback,
Distilabel-Capybara, OpenHermesPreferences, etc.) were built with it.

## 1. Install footprint

- `pip install distilabel` (Python ≥ 3.9). Provider extras:
  `distilabel[openai]`, `distilabel[anthropic]`,
  `distilabel[vertexai]`, `distilabel[mistralai]`,
  `distilabel[cohere]`, `distilabel[groq]`, `distilabel[ollama]`,
  `distilabel[vllm]`, `distilabel[hf-transformers]`,
  `distilabel[hf-inference-endpoints]`, `distilabel[litellm]`,
  `distilabel[llama-cpp]`, `distilabel[mlx]`. Infra extras:
  `distilabel[ray]` (distributed exec), `distilabel[redis]`
  (cross-process cache).
- CLI binary: `distilabel` (Typer). Subcommands: `pipeline run`,
  `pipeline info`, `pipeline view`, plus a `dataset`/`hf` helper.
- Pipelines are `.py` scripts that call `Pipeline(name="...")` —
  you run them with `python my_pipeline.py` *or* `distilabel
  pipeline run --config pipeline.yaml` for the YAML-defined variant.
- Heavy by default — pulls scientific-Python stack (`pandas`,
  `numpy`, `pyarrow`, `datasets`, `multiprocess`, `fsspec`).

## 2. Repo + version + license

- Repo: <https://github.com/argilla-io/distilabel>
- Latest release: **1.5.3** (2025-01-28). Default branch active
  through 2026-04 (`develop` cuts continue).
- License: **Apache-2.0** —
  <https://github.com/argilla-io/distilabel/blob/main/LICENSE>
- Default branch: `main`
- Language: Python

## 3. Models supported

Wide. First-party LLM classes for OpenAI (chat completions +
Azure-style endpoints), Anthropic, Google Vertex (PaLM + Gemini),
Mistral, Cohere, Groq, Together, Anyscale, OpenRouter via the
generic `OpenAILLM(base_url=...)`, plus `LiteLLM` for the long
tail. Local: `OllamaLLM`, `vLLM` (in-process, with batched
generation across the whole step batch — usually the fastest path
for synthetic-data jobs on a GPU box), `TransformersLLM`
(HuggingFace `AutoModelForCausalLM`), `LlamaCppLLM`, `MlxLLM`
(Apple Silicon), `InferenceEndpointsLLM` (HF dedicated endpoints).
Reward / preference models: `PairRMMagpie`, `ArmoRMRewardModel`
classes for the labelling steps. Embedding models for
deduplication: `SentenceTransformersEmbeddings`,
`vLLMEmbeddings`.

## 4. MCP support

No. `distilabel` is a batch / pipeline runtime, not an
interactive-agent host — there is no live tool-call surface for an
MCP server to plug into. If you need MCP-shaped tools available to
the model during generation (e.g. a "look up the current value in
this DB" tool the model can call mid-completion), wrap an MCP host
behind an OpenAI-compatible endpoint and point a `LLM` step at it,
or use [`fast-agent`](../fast-agent/) / [`crewai`](../crewai/) for
the agent surface and persist the resulting traces into a
`distilabel` pipeline for the labelling/scoring step.

## 5. Sub-agent model

Step-graph, not agent-graph. Each `Step` is a stateless transform
over batches of dictionaries; steps with a `LLM` attribute (e.g.
`TextGeneration`, `UltraFeedback`, `EvolInstruct`) call the model
once per input row, with configurable batch size and retry policy.
The "agent-like" patterns are constructed by chaining steps:
`LoadDataFromHub >> SelfInstruct >> TextGeneration >>
UltraFeedback >> KeepColumns >> PushToHub`. Steps run in parallel
across CPU cores by default and across a Ray cluster with
`use_ray=True`; within a step, the LLM call is batched (especially
useful with vLLM, which can saturate a GPU on one step's batch
without per-call HTTP overhead). No long-lived agent state, no
mid-generation tool calls — it's a Spark-style synthetic-data ETL,
not a chat loop.

## 6. Telemetry stance

Off (no analytics from the library itself). Egress is exactly the
LLM endpoints you configure, plus optional `PushToHub` /
`PushToArgilla` writes when you wire those steps in. Pipelines emit
detailed structured logs (`distilabel.pipelines.local`) and write
a per-run cache directory under `~/.cache/distilabel/pipelines/
<name>/<signature>/` for resume-on-crash; nothing in the cache
leaves the box. The HuggingFace Hub integration is opt-in via
explicit `PushToHub(repo_id=..., token=...)` steps.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **The published-paper recipes are first-class
steps.** UltraFeedback, EvolInstruct, MagPie, SelfInstruct, Genstruct,
APIGen, DEITA, PairRM, PrometheusEval — the recent literature on
synthetic instruction generation, response ranking, and preference
labelling lands in `distilabel` as named, parameterised steps you
compose with `>>`. You don't reimplement the paper to use the
recipe; you import it and wire it. The Ray + vLLM combination
makes the actually-expensive step (running a 70B reward model over
100k pairs) tractable on a small GPU cluster, and the on-disk
cache means a crashed 8-hour run resumes from the last checkpoint
instead of starting over. The output of a pipeline is a HuggingFace
`Dataset` ready to push, which is the format the rest of the
post-training stack ([`axolotl`-style trainers, `trl`-style RLHF
harnesses](../) or [`lighteval`-style evals](../)) consumes
natively.

**Weakness.** **It's a heavy, opinionated framework, not a
script-and-go.** Everything goes through `Pipeline` / `Step` /
`StepInput` / `LLM` abstractions; the smallest meaningful program
is ~30 lines of class wiring. The DAG is static (no dynamic
fan-out based on row content without writing a custom step), and
debugging a failing step usually means re-reading the cache
directory by hand. The release cadence is uneven (1.5.3 in early
2025, with active `develop` work since but no tagged 1.6 at this
snapshot) and the public API has historically broken between
minors — pin your version. If your synthetic-data job is "call
GPT-4o on 5000 prompts and write the results to a JSONL," a
20-line `asyncio.gather` over the OpenAI SDK is faster to write,
faster to debug, and adequate. `distilabel` earns its weight only
above ~10k rows or once you're chaining ≥3 model-calling steps.

**When to choose.** You're building a real synthetic-SFT or
preference dataset (DPO/KTO/ORPO), you have a published recipe in
mind (UltraFeedback-style, MagPie-style, EvolInstruct-style), and
you want the pipeline to survive crashes, scale to a Ray cluster,
push to the Hub at the end, and be reproducible from a YAML. Pair
with a vLLM-served generator on the GPU side, with
[`deepeval`](../deepeval/) or [`promptfoo`](../promptfoo/) for the
post-hoc quality bar, and with a downstream trainer that consumes
HF Datasets. Skip for ad-hoc one-shot generation jobs (write a
30-line script), skip if your data flow is fundamentally
interactive / agent-shaped (use [`crewai`](../crewai/) /
[`fast-agent`](../fast-agent/) and dump traces), skip if you need
mid-generation tool calls (no MCP / tool surface).
