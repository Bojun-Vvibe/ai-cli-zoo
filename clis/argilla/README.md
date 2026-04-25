# argilla

> Snapshot date: 2026-04. Upstream: <https://github.com/argilla-io/argilla>

"**Build high-quality datasets for AI**." Argilla is a Python SDK +
self-hostable web UI for **collaborative dataset curation** — domain
experts and AI engineers annotate, validate, score, and version the
records that feed your fine-tunes, RAG eval sets, preference data, and
LLM-as-judge training. Records are typed (text, chat, rating, ranking,
span / token labels), every annotation is attributed, and the same
`Dataset` object is the unit of work in the UI, in pandas, and on the
hub. The runtime ships as a FastAPI server + Postgres + Elasticsearch /
OpenSearch stack you point your `argilla.Argilla(api_url=..., api_key=...)`
client at; an OSS docker-compose brings the whole thing up locally.

## 1. Install footprint

- `pip install argilla` (client SDK; ~25 MB with deps).
- Server: `docker run -d --name argilla -p 6900:6900 argilla/argilla-quickstart:latest`
  for the bundled Postgres + ES + FastAPI stack, or `docker compose up`
  from the `argilla-io/argilla` repo's `docker/` for a production split.
- Hugging Face Spaces one-click template (`argilla-io/argilla-template-space`)
  brings up a hosted instance in ~3 minutes for prototypes.
- Python ≥ 3.9. Pulls `httpx`, `pydantic` v2, `datasets` (HF), `pandas`.

## 2. Repo + version + license

- Repo: <https://github.com/argilla-io/argilla>
- Latest release: **v2.8.0** (2025-12, default branch is `develop`)
- License: **Apache-2.0** —
  <https://github.com/argilla-io/argilla/blob/develop/LICENSE>
- Language: Python (server) + TypeScript (Vue 3 web UI)

## 3. What it integrates with

- **HF `datasets`**: `rg.Dataset.from_hub(...)` and `.to_hub(...)` round-trip
  with the Hugging Face Hub; `from_datasets` / `to_datasets` for in-memory
  conversion to feed `transformers` / `trl` / `peft` fine-tuning loops.
- **Distilabel** (sister project from the same team) writes synthetic
  preference pairs / SFT / DPO records straight into an Argilla `Dataset`
  for human refinement before training.
- **LangChain / LlamaIndex / Haystack callback handlers** push every
  request + response + retrieval into a `chat` dataset for review.
- **`spacy-llm` + `spacy.training`** read span annotations directly.

## 4. Sub-agent / multi-user model

Not an agent framework — Argilla is the **human-in-the-loop substrate**
agent frameworks land their outputs in. Multiple annotators work the
same `Dataset` concurrently with role-based workspaces (`owner` / `admin`
/ `annotator`), per-user assignment policies (round-robin, overlap-N for
inter-annotator agreement, custom Python rules), and `responses` are
attributed per annotator so disagreement is queryable instead of
collapsed.

## 5. Telemetry stance

Anonymous Posthog usage telemetry is **on by default in the server** —
opt-out by setting `ARGILLA_ENABLE_TELEMETRY=false` before first
container start. The Python SDK does not phone home; egress is exactly
the API URL you constructed `Argilla()` with.

## 6. Killer feature, weakness, when to choose

**Killer feature.** **The same typed `Dataset` is the unit of work for
every consumer** — record schemas (`fields=[rg.TextField(...),
rg.ChatField(...)]`, `questions=[rg.RatingQuestion(...),
rg.RankingQuestion(...), rg.SpanQuestion(...)]`, `metadata=[...]`,
`vectors=[...]`) declare *once* what an annotator will see and *also*
what `to_datasets()` will return, so the moment a domain expert finishes
labelling, your trainer / evaluator can `from_argilla(...).filter(...)`
on the same fields without a glue script. Suggestions (`record.suggestions
= [{"question_name": "rating", "value": 4}]`) let you pre-fill
LLM-judge or model-output scores so annotators only correct the
disagreements — the practical lever that turns "hire 5 annotators for
a month" into "have 1 senior annotator review 1000 LLM judgments in a
day". Vector search (`record.vectors = {"embedding": [...]}`) over the
same dataset gives the curator "find more records like this one" as a
button, not a notebook.

**Weakness.** **Operational footprint is real** — Postgres + ES /
OpenSearch + FastAPI + Redis is the floor; there is no SQLite mode for
"just give me a UI on a laptop", so for a one-person prototype the
quickstart Docker image is the answer (and the inverse: enterprise
deployments need a real Elasticsearch cluster, not the bundled one). The
2.x rewrite changed the SDK surface significantly from 1.x — pre-2.0
recipes will not run, and the migration touches every `from_dict` /
`from_pandas` / `init` call site. Default branch is `develop` (not
`main`) — minor papercut for drive-by contributors and CI scripts that
assume `main`.

**When to choose.** Your bottleneck is **dataset quality**, not model
choice or agent topology — you are gating a fine-tune / DPO run / RAG
eval / safety-classifier on having the *right* records reviewed by the
*right* humans. Pair with [`distilabel`](../distilabel/) (synthesise
the candidate pool) → Argilla (humans curate) → `trl` / `peft` (train).
For "I just want to label spans for an NER model" Argilla is overkill —
use Label Studio. For "I want to *generate* LLM-judge scores
programmatically" use [`ragas`](../ragas/) / [`deepeval`](../deepeval/)
/ [`opik`](../opik/); Argilla is where their scores go to be
*reviewed*, not where they are produced.
