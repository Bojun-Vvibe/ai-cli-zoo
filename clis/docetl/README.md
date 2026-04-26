# docetl

- **Repo:** https://github.com/ucbepic/docetl
- **Version:** `0.2.6` (latest release, 2025-12-28)
- **License:** MIT (`LICENSE`)
- **Language:** Python
- **Install:** `pip install docetl` then `docetl run pipeline.yaml`,
  or use the **DocWrangler** web UI (`docetl-ui`) for interactive
  pipeline authoring.

## One-line summary

A **declarative LLM-powered ETL system** from UC Berkeley's EPIC
data lab — you describe a document-processing pipeline as a YAML
DAG of map / filter / reduce / resolve / equijoin / unnest /
gather operators backed by LLM prompts, and the engine handles
batching, caching, document chunking with semantic gleaning,
operator decomposition, and an "agent that rewrites your pipeline
for accuracy" optimization pass.

## What it does

DocETL treats unstructured-document processing as a SQL-shaped
query plan, but every operator is backed by an LLM call instead of
a relational expression:

- **Operators**: `map` (LLM transforms each doc), `filter` (LLM
  yes/no), `reduce` (LLM rolls up a group), `resolve` (LLM-based
  entity resolution / deduplication), `equijoin` (LLM-judged join
  predicate), `split` + `gather` (chunk a long doc, process chunks,
  reassemble with overlapping context), `unnest` (explode nested
  output), `cluster` (k-means or LLM-themed grouping).
- **Pipeline-as-YAML**: each step names its operator, prompt
  template, output schema (Pydantic-style JSON), model, and an
  optional `optimize: true` flag that asks the agent-optimizer to
  rewrite this step.
- **Agent optimizer**: the headline feature. You mark expensive /
  accuracy-critical operators with `optimize: true`; the optimizer
  runs sample data, evaluates output quality with judge prompts,
  and **rewrites your pipeline** — e.g. decomposing a single
  hard `map` into a `split → map → reduce` with semantic gleaning,
  or replacing one giant prompt with two specialist prompts with
  a verifier. The rewritten plan is emitted as a new YAML.
- **DocWrangler IDE**: a web UI (`pip install docetl-ui`) where
  you author the pipeline visually, see operator outputs row-by-row
  on a sample, and rerun individual steps without re-executing the
  whole DAG. Built specifically for the iterative
  "tweak prompt → see what changed → tweak again" loop.
- **Caching**: every LLM call is keyed by `(prompt, model,
  input-hash)` and stored in a local SQLite cache, so reruns of
  unchanged steps are free.

```yaml
# pipeline.yaml
default_model: gpt-4o-mini
operations:
  - name: extract_themes
    type: map
    prompt: "Extract 3 themes from this transcript: {{ input.text }}"
    output:
      schema: { themes: list[str] }
  - name: cluster_themes
    type: reduce
    reduce_key: source
    prompt: "Group these themes: {{ inputs }}"
    output:
      schema: { clusters: list[dict] }
pipeline:
  steps:
    - name: theme_extraction
      input: transcripts
      operations: [extract_themes, cluster_themes]
  output:
    type: file
    path: out.json
```

```bash
pip install docetl
docetl run pipeline.yaml          # execute
docetl build pipeline.yaml        # run the agent optimizer + emit rewritten yaml
```

## When to choose it

- You're processing a corpus of **long, messy documents**
  (transcripts, contracts, research papers, support tickets) where
  the work is naturally a query plan: extract → filter → group →
  summarize.
- You want **the optimizer-rewrites-your-pipeline** behavior — i.e.
  you don't want to manually figure out whether to chunk, how to
  chunk, when to add a verifier prompt.
- You want **declarative reproducibility** (YAML in git, cache on
  disk) instead of an imperative agent loop you have to babysit.

## When NOT to choose it

- Your task is a single-shot chat, an interactive coding agent, or
  a long-horizon planning agent — DocETL is a batch ETL engine,
  not a conversational agent.
- You need streaming partial results to a user — the model is
  "submit a pipeline, wait for the output file".
- You're already happy in LangChain / LlamaIndex pipelines and
  don't want a second declarative layer; DocETL replaces that
  layer rather than composing with it.
