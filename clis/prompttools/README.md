# prompttools

> Snapshot date: 2026-04. Upstream: <https://github.com/hegelai/prompttools>
> License file: <https://github.com/hegelai/prompttools/blob/main/LICENSE>
> (`LICENSE`, sha `f49a4e16e68b128803cc2dcea614603632b04eac`, 11 356 bytes).
> Pinned: `v0.0.45` (latest tagged release, 2024-01-02). The repo is
> Apache-2.0; the maintained `main` branch (HEAD `63bedaa`) carries
> small fixes on top of the tag, so for production work pin the
> tag and not `main`.

A **prompt-and-RAG test harness** built around the same idea as
software unit testing: define a *grid* of prompts × models ×
hyperparameters × inputs, run it, score every cell with a
deterministic eval (regex match, JSON-schema match, semantic
similarity, latency, token cost, custom Python), and get a sortable
DataFrame back. The companion `prompttools.experiment.*` modules
cover both raw LLM calls (OpenAI / Anthropic / Replicate / HuggingFace
/ LlamaCpp / Mistral / Google PaLM) and **vector-DB recall**
experiments against [`chroma`](../chroma/), [`weaviate`](../weaviate/),
[`lancedb`](../lancedb/), Qdrant, Pinecone, and Milvus — so you can
A/B "embedder × chunk-size × top-K" with the same harness you use
for "model × temperature × system-prompt".

## 1. Install footprint

- **Python library**: `pip install prompttools` (or `uv pip install
  prompttools`). ~30 MB before provider extras. Requires Python 3.8+.
- **Provider extras** are opt-in — install only what you need
  (`pip install prompttools[chromadb]`, `[weaviate-client]`,
  `[lancedb]`, `[mistralai]`, `[google-generativeai]`,
  `[llama-cpp-python]`, etc.). The base install pulls only `openai`
  + `pandas` + `jinja2` so a minimal experiment is small.
- **Playground (Streamlit)**: `git clone
  https://github.com/hegelai/prompttools && cd prompttools/prompttools/
  playground && streamlit run playground.py` — a local web UI for
  building prompt grids by hand and seeing the result table without
  writing Python.
- **Notebook examples**: ~30 ready-to-run Jupyter notebooks under
  `examples/notebooks/` cover OpenAI vs Anthropic A/B, RAG recall
  benchmarking on [`chroma`](../chroma/) vs [`weaviate`](../weaviate/),
  semantic-similarity scoring, and CI integration via `pytest`.
- Workspace footprint: experiments serialize to a pandas DataFrame
  in memory; persist with `experiment.to_csv(...)` or
  `experiment.to_pandas_df().to_parquet(...)`. No daemon, no
  database, no background processes.

## 2. Repo, version, license

- Repo: <https://github.com/hegelai/prompttools>
- Default branch: `main` (HEAD `63bedaa` at snapshot).
- Latest tag: `v0.0.45` (2024-01-02). Active development continues
  on `main`, but for a pinnable build use the tag.
- License: **Apache-2.0**. License file: `LICENSE`
  (sha `f49a4e16e68b128803cc2dcea614603632b04eac`, 11 356 bytes).
  No CLA, no carve-out, no enterprise tier — the harness *is* the
  product. Hegel AI also offer a hosted experiment dashboard, but
  the OSS library is fully usable standalone.

## 3. What it actually does

Three orthogonal concepts:

1. **Experiment** — a Cartesian product over user-defined parameter
   axes. `OpenAIChatExperiment(model=["gpt-4", "gpt-3.5-turbo"],
   messages=[...], temperature=[0.0, 0.7])` is a 4-cell grid; calling
   `.run()` issues all 4 calls (in parallel where safe) and stores
   results as rows in a DataFrame.
2. **Eval function** — a `Callable[[row, ...], float]` that scores
   each row. Built-ins: `validate_json_response`,
   `semantic_similarity` (sentence-transformers), `autoeval_with_llm`
   (use a stronger model as judge), `measure_similarity_to_expected`,
   `latency`, regex matchers, exact-match.
3. **Result table** — `experiment.evaluate(name, eval_fn)` adds a
   column; `experiment.visualize()` renders an HTML / notebook table
   sorted by any column. Aggregations are pandas one-liners
   (`df.groupby("model")["score"].mean()`).

The vector-DB experiments work the same way: axes are
`(embedding_fn, chunk_size, top_k, distance_metric)`, the cell value
is "recall@k against a labelled query/doc pair set", and the eval
column is "fraction of expected docs in top_k". Same harness, same
DataFrame shape.

## 4. MCP support

**No native MCP server.** `prompttools` is a *test harness*, not a
runtime — agents do not call it during normal operation; CI does.
For an MCP-shaped wrapper, expose `experiment.run()` +
`experiment.evaluate()` behind [`fastmcp`](../fastmcp/) tools so an
agent can request a prompt-grid sweep on demand and read back the
top row. The repo ships no such adapter out of the box.

## 5. Sub-agent model

None — this is a library, not an agent. The closest concept is
`autoeval_with_llm`, which uses one model (e.g. `gpt-4o`) as a judge
to score the outputs of *another* (e.g. `gpt-3.5-turbo`). That is a
single evaluator call per row, not an autonomous loop.

For agent-level evals (multi-turn rollouts, tool-use traces),
[`deepeval`](../deepeval/), [`inspect-ai`](../inspect-ai/), or
[`ragas`](../ragas/) are the better picks; `prompttools` stops at
"single prompt → single response → score".

## 6. Telemetry stance

**No telemetry in the OSS library.** `pip install prompttools` does
not phone home; experiments stay local in a pandas DataFrame.
Hegel AI offer an optional hosted experiment dashboard (sign-up
required, separate URL); using it is opt-in and the auth flow is
explicit. For air-gapped CI, the library has no network egress
beyond the LLM / vector-DB calls *you* configure inside an
experiment.

Outbound traffic is exactly the model + vector-DB calls in your
experiment definition — same shape as if you had written the calls
by hand.

## 7. Token / context strategy

Every row records `prompt_tokens`, `completion_tokens`, `latency`,
and `cost` (using OpenAI / Anthropic published per-token prices)
where the provider returns them. This makes the everyday workflow
"sweep models × temperature × system-prompt, then `df.sort_values(
['score', 'cost'])` to find the cheapest cell that meets a quality
bar" — one DataFrame query, no spreadsheet copy-paste.

For long-context experiments, the harness streams responses where
the provider supports it but stores only the final concatenated
text; the `messages=` axis is a list of full message-arrays, so you
can A/B "5-shot vs 0-shot system prompt" by passing two complete
message lists and reading off the score delta.

## 8. Hot keybinds

No TUI. The everyday surface is Python:

```python
from prompttools.experiment import OpenAIChatExperiment

messages_a = [{"role": "user", "content": "Summarise: {{doc}}"}]
messages_b = [{"role": "system", "content": "You are terse."},
              {"role": "user",   "content": "Summarise: {{doc}}"}]

exp = OpenAIChatExperiment(
    model=["gpt-4o", "gpt-4o-mini"],
    messages=[messages_a, messages_b],
    temperature=[0.0, 0.7],
    user_inputs={"doc": ["<text 1>", "<text 2>", "<text 3>"]},
)
exp.run()

# Eval — semantic similarity to a reference summary
from prompttools.utils import semantic_similarity
exp.evaluate("similarity", semantic_similarity,
             expected=["<ref 1>", "<ref 2>", "<ref 3>"])

# Inspect — full grid as a sortable table
exp.visualize()
df = exp.to_pandas_df()
print(df.sort_values(["similarity", "latency"], ascending=[False, True]).head())
```

The Streamlit playground (`streamlit run prompttools/playground/
playground.py`) is the same primitives behind a web form — useful
for non-Python collaborators to propose prompt variants.

## 9. Killer feature, weakness, when to choose

**Killer feature.** **One harness for prompts *and* RAG.** The same
DataFrame shape — one row per parameter cell, one column per eval —
covers "model × temperature × system-prompt" *and* "embedder ×
chunk-size × top-K against [`chroma`](../chroma/) /
[`weaviate`](../weaviate/) / [`lancedb`](../lancedb/)". You do not
have to learn a second framework when your A/B graduates from
"which prompt" to "which retrieval config". The result is a
`pandas.DataFrame`, so sorting / filtering / plotting / `pytest`
assertions are zero-effort.

**Weakness.** **Release cadence has slowed** — `v0.0.45` is the
last tag (2024-01-02); `main` keeps moving but tagged releases are
infrequent. Pin the tag for reproducible CI, and read `main`'s
changelog yourself before bumping. **No agent / multi-turn eval** —
single prompt in, single response out; for tool-use rollouts pick
[`inspect-ai`](../inspect-ai/) or [`deepeval`](../deepeval/). **No
hosted artifact storage in OSS** — persisting an experiment across
runs means you `to_parquet()` and check the file in or push to S3
yourself.

**When to choose.**

- You want **a single test harness for both prompt-A/B and RAG-
  recall sweeps**, returning a pandas DataFrame you can already
  reason about.
- You want **Apache-2.0 with no enterprise carve-out** and **no
  telemetry** in the OSS path.
- You want a **Streamlit playground** so non-coding collaborators
  can propose prompt variants without touching Python.
- You want **`pytest`-friendly assertions** ("score on row X must
  be ≥ 0.8") that fail CI on prompt regressions.
- You want **vector-DB recall benchmarking** as a first-class
  primitive and not bolted on.

**When not to choose.** You need **agent-level rollouts** with
multi-turn tool use and trajectory scoring → pick
[`inspect-ai`](../inspect-ai/) or [`deepeval`](../deepeval/). You
want **LLM-as-judge with a curated rubric library** and dataset
versioning → [`ragas`](../ragas/) or
[`arize-phoenix`](../arize-phoenix/). You want **CI-grade prompt
regression with a managed dashboard, GitHub annotations, and
team-shared datasets** → [`promptfoo`](../promptfoo/) is the more
actively maintained pick. You want **adversarial / red-team probing
of an LLM** → [`garak`](../garak/) is purpose-built for that and
`prompttools` is not.
