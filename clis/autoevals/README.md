# autoevals

> Snapshot date: 2026-04. Upstream: <https://github.com/braintrustdata/autoevals>
> Pinned release: `py-0.2.0` (HEAD `f86168887c610bcb45b6cbaa377562992f7ba709`).
> License file: [`LICENSE`](https://github.com/braintrustdata/autoevals/blob/main/LICENSE)
> (MIT).

`autoevals` is a small MIT scoring library from Braintrust that
packages the **standard set of LLM-output scoring functions** —
exact-match / numeric / JSON-validity / Levenshtein / embedding
similarity classics, plus model-graded scorers (`Factuality`,
`AnswerRelevancy`, `AnswerCorrectness`, `Battle`, `ClosedQA`,
`Humor`, `Possible`, `Security`, `Sql`, `Translation`, plus a
generic `LLMClassifier` builder over a Chevron-templated
prompt) — as plain Python / TypeScript callables that return a
`Score` object with a `0..1` value, optional `metadata`, and an
explanation. No daemon, no DSL, no project structure: import a
scorer, call it on `(input, output, expected)`, get a number.
The catalog's reference for **"I need a battle-tested model-graded
scorer I can drop into pytest, a notebook, a custom eval harness,
or a CI step without adopting a whole eval platform"**, sibling to
the gating-shaped harnesses ([`promptfoo`](../promptfoo/),
[`deepeval`](../deepeval/)) — narrower because it is just the
scoring layer, sharper because every other harness wraps a copy
of these scorers anyway.

## 1. Install footprint

- Python: `pip install autoevals` (Python 3.9–3.13). One package,
  ~10 deps (`openai`, `chevron`, `pyyaml`, `levenshtein`,
  `polyleven`, `braintrust-core`, `pydantic`).
- TypeScript: `npm install autoevals` — same scorer surface,
  same prompts, same return shape.
- Zero hosted-service requirement — model-graded scorers issue
  one OpenAI-compatible chat call against whatever client you
  pass (`openai`, `litellm`, `anthropic`, an Azure / Bedrock
  client, or a local `vllm` / `ollama` endpoint via the OpenAI
  shim).

## 2. Repo, version, license

- Repo: <https://github.com/braintrustdata/autoevals>
- Latest release: `py-0.2.0` (2026-04-02; per-language tag).
- HEAD pinned at this snapshot:
  `f86168887c610bcb45b6cbaa377562992f7ba709`.
- License: MIT. License file at
  [`LICENSE`](https://github.com/braintrustdata/autoevals/blob/main/LICENSE).
- Maintainer: Braintrust Data, Inc.

## 3. What it actually does

```python
from autoevals import Factuality, Levenshtein, JSONDiff
from autoevals.ragas import AnswerRelevancy, ContextRelevancy

# Heuristic — no LLM call
score = Levenshtein()(output="hello world", expected="hello, world!")
# Score(name='Levenshtein', score=0.846..., metadata={})

# Model-graded — one chat call
score = Factuality()(
    input="What is the capital of France?",
    output="Paris is the capital of France.",
    expected="Paris",
)
# Score(name='Factuality', score=1.0,
#       metadata={'rationale': '...', 'choice': 'A'})

# Structured-output check
score = JSONDiff()(
    output='{"name": "alice", "age": 30}',
    expected='{"name": "alice", "age": 31}',
)
# Score(name='JSONDiff', score=0.83, metadata={...})

# RAG-shaped scorers
score = AnswerRelevancy(strictness=3)(
    input="What is the capital of France?",
    output="Paris is the capital of France.",
)
```

Every scorer is a callable class with the same signature
(`(input?, output, expected?, **kwargs) -> Score`), so swapping
heuristic for model-graded — or vice versa — is a one-line
change, and a custom scorer is one Pydantic class away. The
`LLMClassifier` builder takes a Chevron prompt template + a
`{choice: score}` mapping and produces a fully-typed scorer
class for project-specific rubrics.

## 4. MCP support

None first-party. `autoevals` is the scoring primitive; an MCP
server wrapping these scorers as tools (`score_factuality`,
`score_answer_relevancy`) is a small adapter.

## 5. Sub-agent model

None — pure scoring functions. Model-graded scorers issue a
single LLM call per `(input, output, expected)` triple; batch
parallelism is the caller's responsibility (use
`concurrent.futures.ThreadPoolExecutor` or `asyncio.gather` with
the async client).

## 6. Telemetry stance

Off. `autoevals` makes **only** the LLM call you configure
(model-graded scorers) and the heuristic CPU work for the rest;
no analytics, no Sentry, no hosted service phone-home. The
sibling Braintrust eval platform (`braintrust` SDK) is opt-in
via separate signup and a different package.

## 7. When it is the right answer

- You have an existing test harness (pytest, a notebook, a
  Jupyter cell, a CI script, a custom eval loop) and want
  battle-tested scoring functions you can call directly without
  adopting a YAML / project-shape opinion.
- You want **the same scorer in Python and TypeScript** — the
  TS port has parity, so a JS playground and a Python regression
  suite produce identical scores on the same `(output, expected)`
  pair.
- You need to author a custom model-graded scorer in 10 lines —
  `LLMClassifier(name="...", prompt_template="...", choice_scores={"A": 1, "B": 0})`
  is the entire shape.

## 8. When to reach for something else

- You want a **gating harness** — declarative test matrices, HTML
  diff viewer, CI failure on regression, replayable test rows —
  that's [`promptfoo`](../promptfoo/) and [`deepeval`](../deepeval/);
  both *use* `autoevals`-shaped scorers internally but add the
  harness around them.
- You want a hosted **trace + eval + dataset** platform with a
  UI for non-engineer reviewers — that's the Braintrust hosted
  product (or [`langfuse`](../langfuse/) / [`opik`](../opik/) /
  [`helicone`](../helicone/) for the OSS-self-hostable equivalents).
- Your eval is mostly RAG-recall A/Bs across vector configs —
  `prompttools` gives you a Cartesian DataFrame in one harness;
  pair with `autoevals` only if you want one of its specific
  model-graded scorers as a column.

## 9. Cross-references

- [`promptfoo`](../promptfoo/) — declarative YAML eval harness
  with model-graded assertions; `autoevals` lives one tier below.
- [`deepeval`](../deepeval/) — pytest-shaped eval harness with
  agentic / MCP metrics; same scoring shape.
- [`langfuse`](../langfuse/), [`opik`](../opik/),
  [`helicone`](../helicone/) — hosted/self-hosted trace + eval
  platforms that consume scorer outputs as run-level metrics.
- [`instructor`](../instructor/) — pair with `JSONDiff` /
  `LLMClassifier` to score structured-output extractions against
  a typed schema.
