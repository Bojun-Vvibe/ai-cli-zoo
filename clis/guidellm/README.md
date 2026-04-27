# guidellm

> Snapshot date: 2026-04. Upstream: <https://github.com/vllm-project/guidellm>

A load-generation and benchmarking CLI for LLM inference servers,
maintained under the **vllm-project** umbrella (originally a Neural
Magic project, donated upstream). It hammers an OpenAI-compatible
endpoint with synthetic or replayed traffic, measures latency
percentiles + throughput + per-token economics across concurrency
levels, and emits a Pareto-shaped report you can hand to capacity
planning.

It is the catalog's reference for **"how fast / how cheap is my LLM
serving stack under realistic load?"** — answer this *before* you
pick a GPU SKU, before you size a deployment, and after every
material change to model / quantisation / engine / server config.

## 1. Install footprint

- `pip install guidellm` (Python 3.9+).
- Pulls `httpx`, `pydantic`, `loguru`, `rich`, `transformers`
  (for tokenizer-aware request shaping), `datasets` (HF datasets
  for replay), `numpy`. ~150 MB venv (mostly `transformers` +
  `torch` for the tokenizer).
- Talks to any OpenAI-compatible HTTP endpoint: vLLM, SGLang, TGI,
  LMDeploy, llama.cpp `--server`, Ollama, vLLM-OpenAI proxy,
  LiteLLM, hosted OpenAI / Anthropic / Together / Groq / Bedrock /
  Vertex (anywhere a Bearer-token plus `/v1/chat/completions` lives).
- No GPU dependency on the load-gen side — runs comfortably from a
  laptop or a single CPU pod against a remote inference server.

## 2. Repo, version, license

- Repo: <https://github.com/vllm-project/guidellm>
- Version checked: **v0.6.0** (released 2026-04-01).
- HEAD pinned at this snapshot:
  `41a3697ea9e98c806c99712ac1ef8a898ff8a9e7`.
- License: Apache-2.0. License file at
  [`LICENSE`](https://github.com/vllm-project/guidellm/blob/main/LICENSE)
  (blob sha `261eeb9e9f8b2b4b0d119366dda99c6fd7d35c64`).

## 3. What it actually does

The core command:

```
guidellm benchmark \
  --target http://localhost:8000/v1 \
  --model meta-llama/Llama-3.1-8B-Instruct \
  --rate-type sweep \
  --max-seconds 300 \
  --data "prompt_tokens=512,output_tokens=128"
```

What that does:

1. **Synthesises requests.** The `--data` selector either generates
   synthetic prompts at the requested input/output token shape
   (token-aware via the model's tokenizer, so prompts hit the actual
   token count, not the character count), or replays from a HF
   dataset (`--data hf_dataset_id`), or from a JSONL of completions.
2. **Shapes load.** `--rate-type` choices: `synchronous` (one in-
   flight at a time), `throughput` (saturate the server), `constant`
   (fixed RPS), `poisson` (Poisson-distributed arrivals at given
   RPS), `concurrent` (fixed in-flight count), and `sweep`
   (automatically walks RPS from 1 → throughput-saturation across
   N levels). Sweep is the killer mode — one command, one report,
   the whole latency-vs-throughput curve.
3. **Measures.** Per-request: TTFT (time-to-first-token), ITL
   (inter-token latency), end-to-end latency, prompt-tokens,
   output-tokens, status. Aggregated: p50 / p90 / p95 / p99 / mean
   for each, plus requests/sec, output-tokens/sec, prompt-tokens/sec,
   request-success rate.
4. **Reports.** Console summary (rich-formatted table), HTML report
   (per-rate-level latency CDFs + scatter), JSON / YAML for
   machine-readable downstream pipelines.

The intended workflow is "stand up your inference server, run
`guidellm benchmark --rate-type sweep`, eyeball the knee of the
TTFT-vs-RPS curve, deploy at 80% of that".

## 4. MCP support

None — it's a load generator, not an agent. The integration boundary
is "any HTTP endpoint that speaks OpenAI Chat Completions".

## 5. Sub-agent model

N/A. Concurrency unit is the **in-flight HTTP request**, driven by
an asyncio event loop with a configurable max-concurrency cap.

## 6. Telemetry stance

Off. No analytics, no opt-in callbacks. Egress is the inference
endpoint under test plus, if you `--data hf_dataset_id`, the HF Hub
on first dataset pull (cached locally afterward).

## 7. Token / context strategy

Token-aware request shaping is the load generator's job: when you
say `--data "prompt_tokens=512,output_tokens=128"`, guidellm uses
the model's actual tokenizer (downloaded from HF Hub) to construct
prompts whose token count *as the server will tokenize them* equals
512 — not 512 characters, not 512 words. This matters because TTFT
and prefill cost scale with the *real* token count; benchmarking
against character-shaped prompts produces numbers that don't survive
contact with production traffic.

For replay data sources (HF datasets), the same tokenizer pass
classifies prompts into bucketed input-token-count distributions you
can target with `--data-args 'input_tokens=512,output_tokens=128'`.

## 8. Hot keybinds

None — single-shot CLI. The relevant flags are:

- `--rate-type sweep` — auto-walk the RPS axis (the killer mode)
- `--max-seconds N` / `--max-requests N` — stopping criterion
- `--warmup-percent 0.1` — discard the first 10% of measurements
- `--cooldown-percent 0.1` — discard the last 10%
- `--output-path report.html` — emit the HTML latency report

## 9. Killer feature, weakness, when to choose

**Killer feature.** A **rate-sweep mode that automatically walks
the request-rate axis** and emits the full latency-vs-throughput
curve in one command — most ad-hoc Locust / k6 / wrk benchmarks pick
one RPS, miss the knee, and tell you nothing about where the server
saturates. Combined with token-aware request shaping (uses the
*real* tokenizer to construct prompts at the requested input-token
count) this gives you numbers that survive the deploy. The HTML
report includes per-rate-level latency CDFs which are the right
shape for capacity-planning conversations.

**Weakness.** OpenAI-Chat-Completions-API-shaped only — it doesn't
benchmark embedding endpoints, reranker endpoints, image-generation
endpoints, or audio endpoints out of the box (extension points
exist; no built-in profiles). The synthetic data generator produces
tokenizer-correct but semantically random prompts; for cache-hit-
sensitive workloads (a server with prefix caching) you want a
realistic prompt distribution from a `--data hf_dataset_id` replay,
not synthetic. The report is a single point in time — there's no
built-in regression-tracking story; pair with [`promptfoo`](../promptfoo/)
or store the JSON output and diff in CI yourself.

**When to choose.**
- You stood up [`vllm`](../vllm/) / [`sglang`](../sglang/) /
  [`text-generation-inference`](../text-generation-inference/) /
  [`lmdeploy`](../lmdeploy/) / [`llama.cpp`](../llama.cpp/) `--server`
  and need to know the **latency-vs-throughput curve** before
  picking a deployment SKU.
- You changed the engine config (max-batch-size, max-num-seqs,
  PagedAttention block size, tensor-parallel degree, quantisation)
  and need to confirm the change actually helped at production
  traffic shape.
- You're comparing **two engines** (vLLM vs SGLang, Q4 vs FP8) on
  the same model and want apples-to-apples numbers — guidellm's
  rate-sweep mode produces directly comparable curves.
- You're sizing **per-token economics** for a hosted-API
  alternative — divide $/hour-of-GPU by tokens/sec at the rate-level
  you'd actually deploy at.

**When to skip.**
- You want a **functional eval** (does the model give correct
  answers?) — guidellm measures latency / throughput / cost only;
  use [`lm-evaluation-harness`](../lm-evaluation-harness/) /
  [`lighteval`](../lighteval/) / [`inspect-ai`](../inspect-ai/) /
  [`promptfoo`](../promptfoo/) for correctness.
- You need **non-LLM** load generation — use Locust / k6 / wrk.
- You want to benchmark **embedding** / **reranker** /
  **image generation** endpoints — guidellm is chat-completions-only
  out of the box; for embedding throughput specifically, the [`text-embeddings-inference`](../text-embeddings-inference/)
  upstream ships its own benchmarking script.

## 10. Compared to neighbors in the catalog

| Tool | Primary job | Measures | Auto-sweep RPS | Tokenizer-aware |
|------|-------------|----------|----------------|-----------------|
| guidellm | LLM serving load test | Latency / throughput / cost | **Yes** | **Yes** |
| [promptfoo](../promptfoo/) | Functional eval | Correctness / quality | No | N/A |
| [lm-evaluation-harness](../lm-evaluation-harness/) | Academic-bench eval | Accuracy on tasks | No | N/A |
| [inspect-ai](../inspect-ai/) | Eval framework | Correctness / safety | No | N/A |
| [llmperf](https://github.com/ray-project/llmperf) (out of catalog) | LLM load test | Latency / throughput | No (manual) | Yes |

Decision shortcut:
- "How fast does my LLM server go?" → **guidellm**.
- "Does my LLM server give correct answers?" →
  [`promptfoo`](../promptfoo/) /
  [`lm-evaluation-harness`](../lm-evaluation-harness/) /
  [`inspect-ai`](../inspect-ai/).
- "Both, in the same CI gate" — run them in series; guidellm gates
  on p99 TTFT + tokens/sec, promptfoo gates on correctness rubrics.
