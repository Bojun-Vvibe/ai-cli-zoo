# ragas

> Snapshot date: 2026-04. Upstream: <https://github.com/explodinggradients/ragas>
> License file: <https://github.com/explodinggradients/ragas/blob/main/LICENSE>
> Pinned: **v0.4.3** (2026-01-13, PyPI: `ragas`).

Ragas is the eval framework that *the* RAG-eval metrics
(`faithfulness`, `answer_relevancy`, `context_precision`,
`context_recall`, `context_entity_recall`, `noise_sensitivity`) were
first published under, plus a generator that can synthesize an
evaluation set from your own document corpus when you don't have one.
The premise: traditional NLP metrics (BLEU / ROUGE) are useless for
RAG output because there's no single ground-truth string, and human
eval doesn't scale; instead, an LLM judge scores each
(question, retrieved-context, generated-answer, optional reference)
quad against a published, peer-reviewed rubric. Ships a `ragas` CLI
for experiment tracking and dataset management on top of the
library, plus pluggable backends for storing eval results (local
CSV / JSONL by default).

## 1. Install footprint

- `pip install ragas` (Python ≥ 3.9). Core pulls `langchain-core`,
  `pydantic`, `datasets`, `numpy`, `pandas`, `nest_asyncio`,
  `appdirs`, `tiktoken`, `diskcache`. Provider extras:
  `ragas[openai]`, `ragas[anthropic]`, `ragas[google]`,
  `ragas[bedrock]`, `ragas[azure]`, plus `ragas[all]` for every
  judge backend.
- CLI: `ragas` (Typer app, entry point `ragas.cli:app`).
  Subcommands include `ragas hello-world`, `ragas evals run`,
  `ragas datasets list`, `ragas projects create`. The CLI is mostly
  experiment scaffolding; the eval itself is library-driven via
  `evaluate(dataset, metrics=[...])`.
- Pluggable backends declared via Python entry points
  (`ragas.backends`): `local/csv`, `local/jsonl`, plus user-defined
  for Postgres / S3 / a SaaS endpoint.
- The legacy `ragas` v0.1–v0.2 API (`from ragas.metrics import
  faithfulness`) is still supported in v0.4 alongside the new
  experiment-tracking surface, so existing eval scripts don't break.

## 2. Repo + version + license

- Repo: <https://github.com/explodinggradients/ragas>
- Latest release: **v0.4.3** (2026-01-13)
- License: **Apache-2.0** —
  <https://github.com/explodinggradients/ragas/blob/main/LICENSE>
- Default branch: `main`
- Language: Python

## 3. Models supported

The system-under-test is whatever produces your RAG answers — Ragas
doesn't call it directly, you pass the (question, context, answer)
tuples in. The *judge* model is what Ragas calls; supported via
LangChain's chat-model adapters: OpenAI (`gpt-4o` / `gpt-4o-mini` /
`o1` / `o3-mini`, the typical default), Anthropic (Claude Sonnet /
Opus / Haiku), Google Gemini (AI Studio + Vertex), AWS Bedrock, Azure
OpenAI, plus any OpenAI-compatible endpoint (Ollama, vLLM, LM Studio,
LiteLLM proxy) for fully-local judge runs. Embedding models for
context-similarity metrics: OpenAI / Cohere / HuggingFace
sentence-transformers / FastEmbed (CPU, no network after first pull).
Pick a strong judge — judge quality is the floor on metric quality,
and a `gpt-4o-mini` judge will not catch hallucinations a `gpt-4o`
judge would.

## 4. MCP support

No (not the protocol). Ragas is a metrics + dataset library; the
LLM calls it makes are judge calls, not tool calls, so MCP doesn't
fit. If your *system-under-test* is an MCP-using agent, run it
yourself, capture the (question, retrieved-context, answer) trace,
and feed those into Ragas — that's the integration shape.

## 5. Sub-agent model

None. Each metric is a one-shot judge call (or a small fixed pipeline
of two-three calls — `faithfulness` decomposes the answer into
claims, then verifies each claim against the context, which is two
sequential calls). `evaluate()` parallelises across rows with
`asyncio.gather`, and the per-row concurrency is controlled by
`max_workers` and `RunConfig.max_retries`. No agent loop, no memory,
no scheduler — that's intentional; the project's stance is
"reproducible numerical scores, not autonomous behaviour."

## 6. Telemetry stance

**Anonymous usage telemetry on by default**, opt out by setting
`RAGAS_DO_NOT_TRACK=true`. Pings include CLI command name, library
version, Python version, OS — no prompt content, no document
content, no scores. The eval calls themselves go to whichever judge
backend you configured (OpenAI / Anthropic / local) — that egress is
your judge provider seeing your test data, which is the load-bearing
privacy decision; route to a local Ollama / vLLM judge if your
test set is sensitive. No first-party cloud control plane in OSS.

## 7. Prompt-cache strategy

The judge prompts are templated and stable per metric, so any
provider-side prefix cache (OpenAI auto, Anthropic ephemeral) hits
on the metric's instruction block; Ragas itself does not configure
Anthropic `cache_control` markers as of v0.4.3, so you get whatever
the provider-side automatic prefix cache gives you. The library
caches synthetic-data-generation outputs to local disk via
`diskcache` (`~/.cache/ragas/`) so a re-run of `TestsetGenerator`
against the same corpus skips repeat generations.

## 8. Hot keybinds (CLI)

Ragas is not a TUI — the CLI is one-shot subcommands. Useful entry
points:

| Command | Action |
|---------|--------|
| `ragas hello-world` | Smoke test: runs a 5-row eval against a built-in dataset to verify the install + judge model |
| `ragas projects create <name>` | Initialise an eval project (local backend) |
| `ragas datasets list` | List datasets registered to the active project |
| `ragas evals run <experiment.py>` | Run an experiment script and persist results to the configured backend |
| `ragas --version` | Print library version |

The library API is what most users invoke:

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision

result = evaluate(
    dataset=ragas_dataset,
    metrics=[faithfulness, answer_relevancy, context_precision],
    llm=judge_llm,
    embeddings=judge_emb,
)
result.to_pandas().to_csv("eval_run_2026_04.csv")
```

## 9. Killer feature, weakness, when to choose

- **Killer:** **The peer-reviewed RAG metrics that everyone else
  cites are *here*, in the canonical implementation.** When a paper
  reports `faithfulness` or `context_recall` numbers on a RAG
  benchmark, this is the library it's measured with. The metric
  catalog covers every common RAG failure mode — hallucination
  (`faithfulness` decomposes into per-claim checks),
  retrieval-quality (`context_precision` / `context_recall` /
  `context_entity_recall`), answer-quality (`answer_relevancy`,
  `answer_correctness`, `answer_similarity`), robustness
  (`noise_sensitivity` against irrelevant context injection) — plus
  a `TestsetGenerator` that reads your own docs and synthesises
  question / answer / context triples across difficulty levels
  (simple, multi-context, reasoning, conversational), so you can
  start evaluating even when you have no labelled set. Numbers are
  on a 0–1 scale, comparable run-to-run, and the per-row breakdown
  is in a pandas DataFrame the moment `evaluate()` returns.
- **Weakness:** **You're trusting an LLM-judge.** Every score is the
  output of another LLM call, and judge bias is real — a `gpt-4o-mini`
  judge will rate `gpt-4o-mini` answers higher than a `claude-sonnet`
  judge will, the verbosity bias rewards longer answers, and certain
  metrics (`context_precision`) collapse when the judge gets the
  domain wrong. Cost adds up: a 1000-row eval with five metrics is
  5000+ LLM calls (some metrics decompose into N sub-calls per row),
  which on `gpt-4o` is real money per run; budget for `gpt-4o-mini`
  or a local judge for development and reserve `gpt-4o`/Sonnet for
  release-gate runs. The LangChain dependency surface drifts faster
  than Ragas pins, so an unlucky upgrade order breaks imports.
- **Choose it when:** you're building or shipping a RAG system and
  need a metric harness that lives in CI, gates releases on
  faithfulness / recall thresholds, and produces numbers a paper or
  a blog post can defend. Pair with [`deepeval`](../deepeval/) when
  you also need pytest-shaped agentic-task / tool-correctness
  metrics on top of the RAG layer (the two libraries' metric
  catalogs overlap on RAG basics but are complementary on agents),
  and with [`promptfoo`](../promptfoo/) for the YAML-config /
  CI-runner side of eval. Skip if your system isn't RAG-shaped — for
  pure agent / chatbot eval, deepeval / promptfoo will be a closer
  fit. The catalog's canonical RAG-eval entry: when a stranger
  shows up with "is my retriever actually helping," this is what
  you reach for first.
