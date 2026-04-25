# llm-guard

> Snapshot date: 2026-04. Upstream: <https://github.com/protectai/llm-guard>

"**The Security Toolkit for LLM Interactions.**" `llm-guard` is a
Python library + optional FastAPI service from Protect AI that
sits *between* your application and an LLM provider, running a
configurable pipeline of input scanners (run on the user prompt
before the model sees it) and output scanners (run on the model
completion before your app sees it) that detect, sanitize, or
reject prompts and completions failing security or compliance
policy. The 20+ input scanners cover prompt injection, jailbreak
patterns, PII, secrets, ban-substrings, ban-topics, ban-code,
language detection, token-limit, toxicity, regex, anonymize-with-
restore, and gibberish; the 15+ output scanners cover bias,
toxicity, malicious-URL, sensitive-info, no-refusal, factual-
consistency, JSON-schema, regex, language, code-detection,
relevance, and reading-time. Each scanner returns a tuple `(text,
is_valid, risk_score)` so you can choose to *block* (`is_valid =
False`), *sanitize* (modified `text` returned), or *score-and-pass*
(use `risk_score` for downstream alerting). The whole pipeline runs
local — the underlying detectors are HF transformer models that
download once and run on CPU or GPU.

## 1. Install footprint

- Library: `pip install llm-guard` (Python 3.9+). Pulls
  `transformers`, `torch`, `presidio-analyzer`, `presidio-anonymizer`,
  `faker`, `tiktoken`, `nltk`, `optimum`. First-call downloads
  per-scanner model weights from HF (a few hundred MB total for the
  default scanner set; ONNX-quantized variants available for ~3-4×
  faster CPU inference).
- API server: separate `llm-guard-api` package — `pip install
  llm-guard-api` boots a FastAPI service exposing `/scan/prompt`
  and `/scan/output` endpoints with per-deployment YAML scanner
  config; lets a polyglot stack (Go / TS / Ruby) call the same
  guardrail policy via HTTP.
- Docker: `laiyer/llm-guard-api:latest` for the API service; the
  library install is what most users ship inside their Python app
  process.
- Optional GPU: scanners run on CPU by default; `device="cuda"` per-
  scanner switches inference to GPU when the workload (and the
  model size) justifies it. Most policies stay CPU-bound.

## 2. Repo + version + license

- Repo: <https://github.com/protectai/llm-guard>
- Latest tag: **v0.3.16** (no GitHub Releases entry; tags drive PyPI
  publishes — `pip install llm-guard==0.3.16`)
- HEAD SHA: `9e007675b90796fe0382d9c321e275425aa1598d`
- License: **MIT** —
  <https://github.com/protectai/llm-guard/blob/main/LICENSE>
- Default branch: `main`
- Language: Python

## 3. Models supported

Provider-agnostic — `llm-guard` does not call LLMs itself, it
inspects the strings flowing in and out of *your* LLM call. Works
identically against OpenAI / Anthropic / Gemini / Bedrock / Vertex
/ Mistral / Cohere / Groq / Ollama / vLLM / any provider, because
the integration point is `prompt: str` in and `completion: str`
out. The detector models *behind* the scanners are local HF
transformers: `deepset/deberta-v3-base-injection` (prompt
injection), `unitary/unbiased-toxic-roberta` (toxicity), Presidio +
`StanfordAIMI/stanford-deidentifier-base` (PII / anonymize),
`mrm8488/bert-tiny-finetuned-sms-spam-detection` (ban-topics base),
plus task-specific small models for bias, factual-consistency,
relevance, gibberish, and language detection. Swap any per-scanner
model via the `model=...` constructor arg.

## 4. MCP support

No first-party MCP integration. Position is one layer below MCP — a
guardrail you run *inside* an MCP server's request handler, or a
sidecar your MCP client calls before forwarding to the LLM. The
FastAPI mode makes "wrap llm-guard as an MCP tool" a one-day glue
job for any MCP runtime that needs guardrails on the data flowing
through it.

## 5. Sub-agent model

None. Each `evaluate_prompt(scanners, prompt)` /
`evaluate_output(scanners, prompt, output)` call runs the configured
scanners sequentially (or in parallel via the FastAPI worker pool)
and returns the aggregate verdict. There is no agent loop.

## 6. Telemetry stance

Off. Library and API server both run fully local; the only network
egress is the one-time HF model download per scanner you enable
(plus your own HTTP clients hitting the FastAPI service if you run
that mode). No analytics, no phone-home, no opt-in usage stream.
Protect AI sells a separate commercial Guardian + Layer product line
that the OSS library is intentionally decoupled from.

## 7. Killer feature, weakness, when to choose

**Killer feature.** **A pre-built catalog of LLM-specific scanners
that runs entirely local with one decision-tuple shape.** Wiring
prompt-injection detection + PII anonymization + toxicity scoring
+ ban-substrings + token-limit on the input side, and bias +
factual-consistency + sensitive-info + JSON-schema + no-refusal +
malicious-URL on the output side is `from llm_guard.input_scanners
import ...; from llm_guard.output_scanners import ...; sanitized,
results, scores = scan_prompt(input_scanners, prompt)` — half a
day from `pip install` to a defensible guardrail policy in front of
your existing LLM call. The `Anonymize` → call LLM → `Deanonymize`
pair restores PII placeholders into the model's response *after*
the fact, so the LLM provider never sees the underlying SSN /
credit-card / email but the user still gets a coherent answer
mentioning "John". Local execution means no third-party guardrail
service is in your data path; same library deploys air-gapped.

**Weakness.** **Detector quality is bounded by the underlying small
models.** Prompt-injection detection on
`deberta-v3-base-injection` catches the common patterns but adversarial
attackers writing novel jailbreaks (multi-turn crescendo, ROT-13 /
Base64 obfuscation, tree-of-attacks-with-pruning) routinely slip
past — pair with [`deepteam`](../deepteam/) to *test* your llm-guard
config under adversarial pressure rather than assuming the default
scanners are sufficient. Sequential scanner execution adds latency
(per-scanner inference is 5–50 ms on CPU; a 6-scanner input
pipeline is a noticeable hit on a 200 ms LLM call); use the ONNX-
quantized variants and the FastAPI mode with a worker pool when
this matters. The `Toxicity` / `Bias` scanners encode
American-English social norms (the underlying training data) and
will produce culturally-specific false positives in non-English
contexts. Some scanners (factual-consistency, relevance) use small
NLI models that are unreliable on technical / domain-specific
content.

**When to choose.** You're putting an LLM in front of users (chat
assistant, RAG over internal docs, customer-support bot) and need
a defensible guardrail layer between the model and the world that
catches the obvious failure modes (prompt injection, PII leakage,
toxic output, schema-broken JSON, off-topic responses) without
adopting a SaaS guardrail vendor in your data path. Pair with
[`deepteam`](../deepteam/) for adversarial testing of your scanner
config, with [`deepeval`](../deepeval/) for correctness regression,
with an LLM gateway like [`portkey-gateway`](../portkey-gateway/)
or [`litellm`](../litellm/) if you also need routing and rate
limiting at the same choke point. Skip if your LLM call sits
entirely behind a single trusted internal user (no untrusted prompt
text reaches it), if a managed guardrail product is acceptable
(NeMo Guardrails, Guardrails AI hosted, AWS Bedrock Guardrails),
or if your concern is *capability* evaluation rather than
*adversarial resistance* — that's [`deepeval`](../deepeval/) /
[`promptfoo`](../promptfoo/), not llm-guard.
