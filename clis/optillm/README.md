# optillm

> Snapshot date: 2026-04. Upstream: <https://github.com/codelion/optillm>

"**Optimizing inference proxy for LLMs.**" `optillm` is a single
process you put *between* your OpenAI-compatible client and any
upstream model endpoint. It speaks `/v1/chat/completions` on both
sides; on the way through, it transparently swaps in one of ~20
inference-time reasoning techniques — Mixture-of-Agents (`moa-`),
Best-of-N (`bon-`), Monte-Carlo Tree Search (`mcts-`), Self-
Consistency (`self_consistency-`), Chain-of-Code (`coc-`),
Reflection (`re2-`), CePO, MARS, AutoThink, LongCePO, RTO, PVG, RStar,
PlanSearch, LEAP, and others — by reading a prefix on the model name
(`moa-gpt-4o-mini`, `mcts-claude-3-5-sonnet`). Pointed at a cheap
upstream model, the proxy can deliver substantially higher accuracy
on math / coding / reasoning benchmarks at the cost of more tokens
per request — without touching the client code.

## 1. Install footprint

- `pip install optillm` (Python ≥ 3.10). Pulls `litellm`, `openai`,
  `flask`, `numpy`, `tiktoken`. Optional `optillm[mlx]` /
  `optillm[mcp]` extras.
- Single binary entry point: `optillm` (HTTP server, default port
  8000). Also Docker: `ghcr.io/algorithmicsuperintelligence/optillm`.
- Configuration is env-driven: `OPENAI_API_KEY`, `OPTILLM_API_KEY`
  (for client auth), per-technique tuning via env or
  `optillm.config.toml`.
- HuggingFace Space and a Colab notebook published for zero-install
  smoke tests.

## 2. Repo + version + license

- Repo: <https://github.com/codelion/optillm>
- Latest release: **v0.3.14** (2026-03-19)
- License: **Apache-2.0** —
  <https://github.com/codelion/optillm/blob/main/LICENSE>
- Default branch: `main`
- Language: Python

## 3. Models supported

Anything with an OpenAI-compatible chat-completions endpoint —
OpenAI, Anthropic (via litellm `claude-*`), Google (Gemini AI Studio
+ Vertex), Cerebras, Groq, DeepSeek, Together, Fireworks, OpenRouter,
xAI, Mistral, Cohere, AWS Bedrock, Azure OpenAI, plus any local
runtime exposing the same API ([`ollama`](../ollama/),
[`mlx-lm`](../mlx-lm/) `mlx_lm.server`, [`openllm`](../openllm/),
`vllm`, `llama.cpp`, `localai`). Provider routing is delegated to
`litellm`, so any model string `litellm` understands works with the
optimizer prefix in front of it. Some techniques (CePO, MARS,
LongCePO) explicitly target reasoning-tuned models (DeepSeek-R1,
Llama 3.3 70B, Gemini 2.5 Flash); others (BoN, MoA, Self-
Consistency) work with anything.

## 4. MCP support

Yes (client). The proxy can be configured to mount MCP servers and
expose their tools to the upstream model on each call (`optillm[mcp]`
extra), so a request that comes in as plain `chat.completions` can
have tools transparently injected before it goes upstream — useful
for stitching deterministic tool calls into a base model that your
client wasn't going to enable tools for. No server mode (the proxy
is the server, but it speaks OpenAI, not MCP).

## 5. Sub-agent model

Per-request, technique-defined. Most prefixes spawn N parallel
upstream calls to the same model (`bon-`: N samples, return best;
`moa-`: N agent samples → aggregator; `self_consistency-`: N
samples → majority vote; `mcts-`: tree of rollouts under a UCB
selector). The proxy is the manager; the upstream model is the
worker; the client sees one response. No persistent agent state
across requests — each call is independent (which is what makes
`optillm` safe to drop into an existing client without retraining
the rest of the system).

## 6. Telemetry stance

Off (no analytics, no usage reporting in the OSS proxy). Egress is
your configured upstream provider plus, optionally, any MCP servers
you mount. The HuggingFace Space is a hosted demo — don't send
real traffic through it. Local logs go to stdout; structured trace
output to a file via `--log-file`.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **A model-name prefix gives you a different
inference-time algorithm with no client change.** `model="moa-gpt-
4o-mini"` runs three Mixture-of-Agents samples through GPT-4o-mini
and returns a synthesized answer often comparable to GPT-4o on
reasoning tasks; `model="mcts-claude-3-5-sonnet"` runs a Monte-
Carlo tree search over Sonnet completions; `model="cepo-llama-3.3-
70b"` runs Cerebras' CePO planning-and-edit loop. The published
benchmarks are striking — MARS over Gemini 2.5 Flash Lite gains 30
points on AIME 2025, CePO over Llama 3.3 70B gains 18.6 on Math-
L5 — and the trick is that *your application code does not change*.
You're not adopting a new agent framework; you're moving an env var
and a model string. The catalog of techniques (BoN, MoA, MCTS, Self-
Consistency, CoC, ReRead, CePO, MARS, AutoThink, LongCePO, RTO, PVG,
RStar, PlanSearch, LEAP, Z3-solver-augmented) is the largest
collection of plug-and-play inference-time-compute methods in any
single OSS tool.

**Weakness.** **N× tokens, N× latency, N× cost.** The improvements
are real but not free — a 3-sample MoA call sends 3× the upstream
tokens; a deep MCTS rollout can be 10–50×. For a coding agent loop
where you make hundreds of model calls per session, the cost
multiplies through. The right deployment is *selective*: route only
the hard reasoning step (architectural plan, root-cause analysis,
math step) through `optillm`, not every chat turn. Some techniques
need careful per-task tuning (BoN sample count, MCTS depth/branch,
CePO planner depth) and the published numbers assume well-chosen
hyper-params. Documentation for individual techniques is uneven
(some have a paper + an example; some are one-paragraph notes). No
production-grade observability built in — you'll want
[`promptfoo`](../promptfoo/) or [`deepeval`](../deepeval/) on the
side to confirm the technique is actually winning on *your* data.

**When to choose.** You have an existing OpenAI-compatible
application (chatbot, RAG pipeline, agent framework, eval harness)
and you want to lift accuracy on a specific reasoning-heavy step
without rewriting the integration or adopting a new agent
framework. Drop `optillm` in front of your provider, prefix the
target model name on just that call, A/B with [`promptfoo`](../promptfoo/)
or [`deepeval`](../deepeval/) to confirm the win is real on your
distribution. Pair with [`mlx-lm`](../mlx-lm/) or
[`openllm`](../openllm/) on the upstream side for a fully self-
hosted "cheap small model + inference-time compute = strong
reasoning" stack. Skip in favour of an explicitly reasoning-tuned
model (DeepSeek-R1, OpenAI o-series, Claude with extended thinking)
if you'd rather pay per-token for native long thinking than pay
per-N-samples for ensemble inference, or skip if your bottleneck is
*tool use* rather than *reasoning quality* — that's an agent-loop
problem, not an inference-proxy problem.
