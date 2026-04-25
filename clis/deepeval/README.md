# deepeval

> Snapshot date: 2026-04. Upstream: <https://github.com/confident-ai/deepeval>
> License file: <https://github.com/confident-ai/deepeval/blob/main/LICENSE.md>
> Pinned: `v3.9.7` (2025-12-01). Active development on `main`; pin the
> exact PyPI version in `requirements.txt` if you put deepeval in CI,
> because the metric implementations evolve and a re-eval against a
> moving version will silently change scores.

A **pytest-shaped LLM evaluation framework**. You write `test_*.py`
files that build a `LLMTestCase` (input, actual_output, optionally
retrieval_context, expected_output, tools_called) and assert against
**metrics** — G-Eval, faithfulness, answer-relevancy, hallucination,
tool-correctness, task-completion, ~40 more. Each metric is itself an
LLM-as-a-judge (or a classical NLP model) call, so a deepeval suite
is "use one LLM to test another LLM," wrapped in an interface that
ordinary Python developers already know.

The CLI is `deepeval` (entry point of the `deepeval` PyPI package);
the everyday command is `deepeval test run path/to/test_*.py`, which
reuses pytest collection but renders its own results table.

## 1. Install footprint

- `pip install -U deepeval` (or `uv pip install deepeval`). Pulls
  pytest, openai, anthropic, several HF libraries, and (lazily on
  first use) sentence-transformers — first run after install can
  download a few hundred MB of judge / embedding models.
- Python 3.9+. No daemon; no system services. Optional cloud sync to
  Confident AI requires `deepeval login` and is purely opt-in.
- Workspace footprint: a `.deepeval/` directory at the cwd holding
  cached judge prompts, downloaded NLP models, and (if used) a local
  results SQLite. Add it to `.gitignore`.
- Test files live wherever you put them (typical layout:
  `tests/llm/test_rag.py`); deepeval discovers them via pytest.

## 2. License

Apache-2.0 (file is `LICENSE.md` at the repo root, not `LICENSE`).

## 3. Models supported

Two distinct axes:

- **Models you are evaluating** — anything. deepeval doesn't care; you
  call your LLM in your test, attach the response to a `LLMTestCase`,
  and hand the case to the metric. The framework never speaks to your
  system-under-test.
- **Models the metrics use as judges** — configurable per metric:
  - **OpenAI** — default (`gpt-4o`, `gpt-4o-mini`, `o1`); set
    `OPENAI_API_KEY`.
  - **Anthropic** — Claude Sonnet / Opus / Haiku via
    `AnthropicModel`.
  - **Gemini, Bedrock, Vertex, Azure OpenAI, Ollama, vLLM, LiteLLM
    proxy** — pick via `deepeval set-<provider>` CLI helpers or by
    instantiating the matching `DeepEvalBaseLLM` subclass.
  - **Local NLP models** — several metrics (e.g. bias, toxicity, some
    contextual ones) ship a local sentence-transformers / HF pipeline
    fallback, so a subset of the suite is runnable fully offline.

If you swap the judge model, rerun your suite — judge choice changes
absolute scores meaningfully. Pin it.

## 4. MCP support

**Yes — as a thing being measured, not a thing exposed.** deepeval
ships dedicated MCP metrics (`MCP Task Completion`, `MCP Use`,
`Multi-Turn MCP Use`) that score how well an agent uses a set of MCP
servers. deepeval itself does not run as an MCP server; you call it
from pytest, not from another agent. To regression-test an MCP-enabled
agent ([`opencode`](../opencode/), [`claude-code`](../claude-code/),
[`goose`](../goose/), etc.), drive the agent in your test, capture
which MCP tools it called and with what arguments, then assert with
`MCPUseMetric` / `MCPTaskCompletionMetric`.

## 5. Sub-agent model

None at the framework level — each metric is a one-shot judge call.
But the **DAG metric** lets you compose deterministic LLM-as-a-judge
graphs (a tree of yes/no questions), which is effectively a fixed
mini-agent for a single rubric. And `EvaluationDataset.evaluate(...)`
parallelises across test cases (configurable concurrency), so a
1 000-case suite finishes in minutes, not hours.

## 6. Telemetry stance

**On by default, opt-out.** First-run banner says so. Two flavours:

- **Anonymous usage telemetry** (which CLI commands you ran, which
  metrics you used) — disable with `DEEPEVAL_TELEMETRY_OPT_OUT=YES`.
- **Confident AI cloud sync** (your full test cases + scores) — only
  active after explicit `deepeval login`. Without login, results stay
  local.

The judge model provider obviously sees the test cases — including
`actual_output` from your system-under-test, which often contains
production payloads. For sensitive evals, configure a local judge
(Ollama / vLLM) so nothing leaves the box.

## 7. Prompt-cache strategy

**None at the deepeval layer.** Each metric call is a fresh judge
prompt. Two consequences:

- A re-run of the same suite costs the same money as the first run.
  deepeval does not memoise (input, output, metric) → score.
- Anthropic's prefix caching and OpenAI's automatic prompt caching
  *do* fire if your metric prompts are stable across cases, so the
  amortised cost is lower than naive token counts suggest, but you
  rely on the provider for it.

For expensive eval suites, batch via `EvaluationDataset` (one big
parallel run is cheaper than many small ones because of provider-side
cache warmth) and snapshot results to a JSON file you re-load instead
of re-running.

## 8. Hot keybinds

No TUI. CLI surface:

- `deepeval test run tests/` — run a directory of pytest-shaped eval
  files, render the results table.
- `deepeval test run tests/test_rag.py -n 4` — parallel workers.
- `deepeval test run -i` — interactive mode that prints metric
  reasoning per failure.
- `deepeval set-openai --model gpt-4o-mini` — globally pin the judge
  model + key.
- `deepeval set-ollama --model llama3.1` — point all judges at a
  local Ollama server.
- `deepeval login` — opt into Confident AI cloud sync (skip this and
  everything stays local).
- `deepeval view` — open the last run's HTML report locally.

Programmatic:

```python
from deepeval import assert_test
from deepeval.metrics import GEval, AnswerRelevancyMetric
from deepeval.test_case import LLMTestCase, LLMTestCaseParams

def test_factuality():
    case = LLMTestCase(
        input="Who wrote Pride and Prejudice?",
        actual_output=my_rag_chain.invoke("Who wrote Pride and Prejudice?"),
        expected_output="Jane Austen",
    )
    assert_test(case, [
        AnswerRelevancyMetric(threshold=0.7),
        GEval(name="Correctness", criteria="...", evaluation_params=[
            LLMTestCaseParams.ACTUAL_OUTPUT,
            LLMTestCaseParams.EXPECTED_OUTPUT,
        ]),
    ])
```

## 9. Killer feature, weakness, when to choose

**Killer feature.** **Pytest is the interface**. Every Python team
already knows how to write a pytest test, run it locally, run it in CI,
mark slow ones with `@pytest.mark.slow`, parametrise via fixtures,
and post the JUnit XML to GitHub Actions. deepeval inherits all of
that for free. A 50-case eval suite that runs on every PR is a
half-day of work, not a project. The metric catalog is also genuinely
broad — agentic (task completion, tool correctness, plan adherence),
RAG (faithfulness, contextual recall), multi-turn (knowledge
retention, role adherence), MCP-specific, plus G-Eval / DAG for
custom criteria — so you rarely need to write a metric from scratch.

**Weakness.** **Cost and non-determinism.** Every metric is an LLM
call against a frontier judge model; a suite of 200 cases × 4 metrics
is 800 paid round-trips per run. Scores also vary between runs even
with `temperature=0` — judge outputs are stable enough for trend lines
but noisy enough that a single-point regression check is risky.
Practical mitigation: pin judge model + version, run each case 3×
and average, treat thresholds as ranges not equalities. Telemetry
default-on is also a gotcha for regulated environments — set the
opt-out env var in your CI image.

**When to choose.**

- You have an LLM app (RAG, chatbot, agent) and you want **regression
  tests in CI** without standing up an eval platform.
- You need **agentic metrics** (tool correctness, task completion,
  step efficiency, MCP usage) — this is one of the few open-source
  catalogs that ships them.
- Your team already lives in pytest and you want **one test command**
  for unit, integration, and eval.

**When not to choose.** You want a **full observability platform** with
trace exploration, prompt versioning, and a UI for non-engineers — pick
[Langfuse](https://github.com/langfuse/langfuse),
[Phoenix](https://github.com/Arize-ai/phoenix), or
[Helicone](https://github.com/Helicone/helicone) (deepeval's
optional Confident AI SaaS overlaps but the OSS core is test-runner
only). You only need **prompt comparison across providers** without
metrics — [`promptfoo`](../promptfoo/) is closer to that shape (YAML
matrix, HTML diff viewer, no pytest dependency). You need
**deterministic, auditable scores** for compliance — LLM-as-a-judge
is not the right tool; use rule-based assertions.
