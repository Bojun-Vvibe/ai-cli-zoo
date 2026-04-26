# adala

> Snapshot date: 2026-04. Upstream: <https://github.com/HumanSignal/Adala>

**A**utonomous **DA**ta **(L)**abeling **A**gent framework — a
Python library that frames data labeling, classification, and
extraction as an *iteratively-improving agent* rather than a
static prompt. You hand Adala a dataset (DataFrame), a skill
(e.g. "classify each row as one of {positive, negative,
neutral}"), and ground-truth feedback for a few rows; the agent
predicts, scores itself against the feedback, mutates its own
instruction prompt, re-predicts, and converges on an instruction
that performs well on the held-out feedback before being
unleashed on the full dataset.

It is the catalog's reference for **prompt-as-learnable-parameter
applied to bulk dataset labeling**, from the team behind Label
Studio (HumanSignal).

## 1. Install footprint

- `pip install adala` (Python 3.9+).
- Pulls `pandas`, `pydantic`, `litellm`, `openai`, `rich`. ~80 MB
  venv.
- Defaults to OpenAI (`gpt-3.5-turbo` / `gpt-4`); any LiteLLM
  provider works via `OpenAIChatRuntime(model='...', base_url=...)`
  — Anthropic, Gemini, Ollama, vLLM, OpenRouter, Groq, DeepSeek,
  Bedrock, Vertex.
- API keys via env (`OPENAI_API_KEY`, etc.) or passed to the
  runtime constructor.
- Optional ML server mode: `docker compose up` from the repo
  spins up the FastAPI-backed `adala-server` for HTTP labeling
  jobs (used by Label Studio integration).

## 2. Repo, version, license

- Repo: <https://github.com/HumanSignal/Adala>
- Version checked: **0.0.4** (latest tagged release on PyPI and
  GitHub as of 2026-04). The project is under active development
  on `master` past that tag — pin commits, not the tag, for
  production use.
- HEAD pinned at this snapshot:
  `6d162b855b2ee73e5d8eff91bfa7f8c3ce89d9d5`.
- License: Apache-2.0. License file at
  [`LICENSE`](https://github.com/HumanSignal/Adala/blob/master/LICENSE).

## 3. What it actually does

The core loop:

1. You define an `Agent` with a `Skill` (or `LinearSkillSet` /
   `ParallelSkillSet` for multi-step labeling), a `Dataset`
   (Pandas DataFrame wrapped in `DataFrameDataset`), an
   `Environment` that knows the ground truth for a feedback
   subset, a `Runtime` (which LLM to call), and a `Memory`
   (vector store of past mistakes — optional).
2. Call `agent.learn(learning_iterations=N)`. For each
   iteration:
   a. Apply the current skill prompt to the dataset → predictions.
   b. Ask the environment for feedback on rows it knows the
      truth for (a small held-out subset).
   c. Compute per-row errors; ship error examples + current
      instruction back to the LLM with a "rewrite the
      instruction so these rows come out right" meta-prompt.
   d. Replace the skill's instruction with the rewrite.
3. Call `agent.run()` — apply the **converged** instruction to
   the full dataset and return labelled rows.

The labeled DataFrame is the deliverable. Skills compose: a
`LinearSkillSet` runs `extract_entities → classify_entity_type →
normalise_entity_name` in order, with each skill's output feeding
the next.

## 4. MCP support

None as of v0.0.4. Tooling for the agent is implicit (it calls a
single LLM per row); the surface is "Skill + Runtime", not a
generic tool/server protocol.

## 5. Sub-agent model

`SkillSet`s are the sub-agent surface:

- `LinearSkillSet([skill_a, skill_b])` — pipeline; output of A
  becomes input of B.
- `ParallelSkillSet([skill_a, skill_b])` — fan out the same row
  to multiple skills, merge predictions.
- The top-level `Agent` orchestrates the learning loop across the
  whole skill set; individual skills do not learn independently
  unless wired that way.

## 6. Telemetry stance

Off in the OSS library (no analytics in `adala` itself; egress is
the configured LLM endpoint plus, if you opt in, the Label Studio
HTTP endpoint for ground-truth fetching). The hosted Label Studio
SaaS is a separate product with its own telemetry — Adala the
library does not phone it home.

## 7. Token / context strategy

One row → one LLM call (default). For large datasets this is
*N rows × M iterations* requests during learning, then *N rows*
during the final `run()`. Adala does not batch rows by default;
you control parallelism via the `Runtime` (sync / async). The
`Memory` component lets the agent recall past errors across
iterations without bloating every prompt.

For datasets in the 10k+ row range this is expensive; the
intended workflow is "learn on a 100-row feedback subset with
expensive `gpt-4`, run on the full dataset with cheap
`gpt-3.5-turbo` or a local model".

## 8. Hot keybinds

None — `adala` is a Python library, not a TUI. Run from a
notebook or script.

## 9. Killer feature, weakness, when to choose

**Killer feature.** Adala treats the **prompt itself as a
learnable parameter**, optimised against ground-truth feedback
rather than hand-tuned. Most catalog entries that touch dataset
labeling ([`distilabel`](../distilabel/),
[`argilla`](../argilla/), [`docetl`](../docetl/),
[`marvin`](../marvin/)) either expect you to write the right
prompt yourself or focus on synthetic data generation. Adala
*closes the loop*: bad predictions on the feedback subset
literally rewrite the next iteration's instruction.

**Weakness.** Pre-1.0 (latest tag is `0.0.4`); the API is still
mutating on `master`. Pin a commit, not the tag. The skill /
runtime / environment / memory abstraction takes a few hours to
internalise — flatter than [`dspy`](../dspy/) but still a
non-trivial DSL. Single-agent learning loop only; no built-in
distributed mode (compose with Ray / Dask yourself).

**When to choose.**
- You have a **bulk labeling / classification / extraction job**
  (10k–10M rows) and a small ground-truth subset.
- You want the framework to **converge on the prompt for you**
  using that ground truth rather than guessing from one-shot
  examples.
- You already use Label Studio for human-in-the-loop annotation
  and want the LLM-side glue from the same vendor.

**When to skip.**
- Your job is "generate synthetic training data" rather than
  "label real data" → use [`distilabel`](../distilabel/).
- You want **structured-output extraction without learning** (no
  feedback loop, just call once) → use
  [`instructor`](../instructor/) /
  [`marvin`](../marvin/) /
  [`outlines`](../outlines/).
- You want to optimise prompts for an **arbitrary downstream
  metric** (not just classification accuracy) →
  [`dspy`](../dspy/) is the more general framework.
- You want a UI for human reviewers — Adala is library-only;
  pair it with Label Studio (or use Argilla).

## 10. Compared to neighbors in the catalog

| Tool | Primary job | Prompt is learnable? | Needs ground truth? | Ships UI |
|------|-------------|----------------------|---------------------|----------|
| adala | Bulk labeling / classification / extraction | **Yes** (instruction rewriting from feedback) | Yes (small subset) | No (pairs with Label Studio) |
| [distilabel](../distilabel/) | Synthetic dataset generation + AI feedback | No (you write pipelines) | No | No |
| [argilla](../argilla/) | Human-in-the-loop annotation platform | No | Produces it | Yes |
| [docetl](../docetl/) | Document-processing pipelines (map/reduce/resolve) | Optional optimiser | No | No |
| [dspy](../dspy/) | General prompt-program optimisation | **Yes** (any metric) | Yes (small train set) | No |
| [instructor](../instructor/) | Structured-output extraction | No | No | No |

Decision shortcut:

- "Label 100k rows; I have 200 hand-labelled" → `adala`.
- "Build a synthetic dataset from scratch" →
  [`distilabel`](../distilabel/).
- "Optimise an arbitrary LLM program against any metric" →
  [`dspy`](../dspy/).
- "Just extract a Pydantic schema reliably" →
  [`instructor`](../instructor/).
- "Run a human review queue" → [`argilla`](../argilla/) (pair
  with Adala for the LLM side).
