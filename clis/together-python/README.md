# together-python

> Snapshot date: 2026-04. Upstream: <https://github.com/togethercomputer/together-python>
> Pinned release: `v1.5.35` (HEAD `cc9f25369987e73bcd037be955833c37ac8a9109`).
> License file: [`LICENSE`](https://github.com/togethercomputer/together-python/blob/main/LICENSE)
> (Apache-2.0).

`together-python` is the official Python SDK + `together` CLI
for the Together AI inference platform — a hosted multi-tenant
service that serves 200+ open-weight LLMs (Llama, Qwen, DeepSeek,
Mixtral, Gemma, Stable Diffusion, FLUX) behind an OpenAI-shaped
chat / completions / embeddings / images API plus an explicit
fine-tuning + dedicated-endpoints surface. The CLI (`together
chat.completions create`, `together files upload`, `together
fine-tuning create`, `together endpoints start`) is what scripts,
batch jobs, and CI pipelines reach for when they want pay-per-token
access to open models without standing up [`vllm`](../vllm/) /
[`sglang`](../sglang/) / [`tgi`](../text-generation-inference/)
themselves. It is the catalog's reference for **a CLI/SDK over a
hosted open-weight inference + fine-tune marketplace** — sibling
to the [`groq-python`](../groq-python/),
[`replicate-cli`](../replicate-cli/),
[`modal`](../modal/) lanes.

## 1. Install footprint

- `pip install together` (Python 3.8+). Single dependency tree:
  `httpx`, `pydantic`, `click`, `rich`, ~12 transitive deps.
- The CLI is `together <resource> <verb>`; auth is via
  `TOGETHER_API_KEY` env var or `together --api-key=...`.
- No local model weights, no GPU requirement — all inference
  is remote. Streaming chat responses use SSE; structured
  output uses JSON-mode + JSON Schema constraints.
- OpenAI compatibility is two lines: `OpenAI(base_url='https://api.together.xyz/v1', api_key=...)`
  works against the same endpoints, so existing OpenAI-shaped
  client code (including [`litellm`](../litellm/) /
  [`instructor`](../instructor/) / [`openai-agents-python`](../openai-agents-python/))
  routes to Together with a base-URL swap.

## 2. Repo, version, license

- Repo: <https://github.com/togethercomputer/together-python>
- Latest release: `v1.5.35` (2026-01-21).
- HEAD pinned at this snapshot:
  `cc9f25369987e73bcd037be955833c37ac8a9109`.
- License: Apache-2.0. License file at
  [`LICENSE`](https://github.com/togethercomputer/together-python/blob/main/LICENSE).
- The SDK is OSS; the inference / fine-tuning service it talks
  to is a paid hosted product.

## 3. What it actually does

CLI verbs map 1:1 to the API resources:

```
together chat.completions create -m meta-llama/Llama-3.3-70B-Instruct-Turbo -p "..."
together completions create -m mistralai/Mixtral-8x22B -p "..."
together embeddings create -m togethercomputer/m2-bert-80M-8k-retrieval -i "..."
together images generate -m black-forest-labs/FLUX.1-schnell -p "a cat"
together files upload training.jsonl
together fine-tuning create --training-file file-... --model meta-llama/Llama-3.1-8B-Instruct
together fine-tuning list / retrieve / cancel
together endpoints create --model my-ft-model --hardware-type 1x_a100_80gb
together models list                      # browse the 200+ catalog
```

Two surfaces beyond plain inference matter:

1. **Fine-tuning.** Upload a JSONL of conversations, kick off
   a LoRA or full-parameter fine-tune on a base model, and
   either serve it on the shared serverless tier (cheap, cold
   starts) or pin it to a dedicated endpoint (no cold start,
   metered by GPU-hour).
2. **Dedicated endpoints.** `together endpoints` provisions a
   reserved deployment of any catalog model on a chosen GPU
   class (H100 / A100 / L40S / A100-80GB) so latency-sensitive
   prod traffic doesn't share a queue with the burst tier.

## 4. MCP support

None first-party in the SDK. The hosted models are accessed via
the chat completions endpoint, so any MCP-aware client
([`fast-agent`](../fast-agent/),
[`mcp-agent`](../mcp-agent/),
[`pydantic-ai`](../pydantic-ai/)) pointed at the Together
base URL gets MCP-tool support via the client side.

## 5. Sub-agent model

None — `together` is a thin SDK + CLI over a stateless inference
API. Tool calling / function calling is supported on models that
expose it (Llama-3.x-Instruct, Qwen-2.5, DeepSeek-V3,
Mistral-Large) via the OpenAI-shaped `tools=[...]` parameter;
multi-step agent loops are the caller's responsibility.

## 6. Telemetry stance

The SDK itself emits no telemetry. The hosted service obviously
logs API requests for billing + abuse + latency monitoring;
Together's published policy is "no training on customer data"
(the inference + fine-tune-output retention policy is documented
in their TOS). The `TOGETHER_LOG=debug` env var is a
client-side debug knob, not telemetry.

## 7. When it is the right answer

- You want OpenAI-shaped pay-per-token access to **open-weight**
  models (Llama-3.3-70B, DeepSeek-V3, Qwen-2.5-72B, Mixtral-8x22B,
  FLUX.1) without renting GPUs yourself.
- You want a serverless fine-tune flow: `files upload` →
  `fine-tuning create` → `chat.completions` against the resulting
  model name, no infra.
- You need predictable low-latency prod traffic and are willing
  to pay for a dedicated endpoint instead of the shared serverless
  tier.

## 8. When to reach for something else

- You only need inference of one specific model with sub-300ms
  TTFT — [`groq-python`](../groq-python/) is faster on the
  models it carries.
- You want to host the weights yourself for compliance / cost
  reasons — [`vllm`](../vllm/) / [`sglang`](../sglang/) /
  [`text-generation-inference`](../text-generation-inference/) /
  [`bentoml`](../bentoml/) / [`truss`](../truss/) /
  [`modal`](../modal/) are the self-host lanes.
- You need a router that fans the same call across many
  providers (Together + Groq + Anthropic + OpenAI + local) —
  [`litellm`](../litellm/) sits in front of all of them.
- You want a generic multi-modal hosted inference marketplace
  (audio, video, custom Cog containers) — [`replicate-cli`](../replicate-cli/)
  is broader-purpose.

## 9. Cross-references

- [`groq-python`](../groq-python/) — sibling SDK over a
  latency-optimised LPU inference service.
- [`replicate-cli`](../replicate-cli/) — broader hosted
  inference marketplace (Cog containers, multi-modal).
- [`litellm`](../litellm/) — OpenAI-shaped router that lists
  Together as one of 100+ providers.
- [`modal`](../modal/) — serverless GPU compute for hosting
  your own weights instead of using Together's catalog.
- [`vllm`](../vllm/) / [`sglang`](../sglang/) — the OSS
  inference engines Together itself runs internally.
