# dvc

> **Git-shaped version control for datasets, models, and ML
> experiments: stores large binary artifacts in a content-addressed
> remote (S3 / GCS / Azure / SSH / local), checks in tiny pointer
> `.dvc` files alongside code in git, and re-runs declarative
> pipelines (`dvc.yaml`) when their declared inputs change so
> training is reproducible across machines and across teammates.**
> Pinned to **3.67.1**, Apache-2.0
> ([LICENSE](https://github.com/iterative/dvc/blob/main/LICENSE)).

- **Repo:** https://github.com/iterative/dvc
- **Latest version:** 3.67.1
- **License:** Apache-2.0 (`LICENSE` at repo root)
- **Category:** `mlops` / `data-versioning` / `experiment-tracking`
- **Language:** Python

## What it does

`dvc` (Data Version Control) solves the problem that git is bad at
binaries — it does not store deltas of multi-GB checkpoint files,
LFS is awkward across forks, and git history balloons. `dvc`
inverts the model: the actual binary lives in a content-addressed
remote (S3 / GCS / Azure Blob / SSH host / local NAS), git stores
only a small `.dvc` text file containing the SHA-256 of the
binary plus its size and path. `dvc pull` materialises the
binaries into the working tree on demand; `dvc push` uploads new
ones; `git checkout <branch>` followed by `dvc checkout` swaps
the working tree's binaries to match the branch's pointers. On
top of that, `dvc.yaml` declares a DAG of stages — each stage has
declared `deps:` (input files / params), `outs:` (produced
files), and a `cmd:` to run; `dvc repro` re-executes only the
stages whose declared inputs have changed since the last run,
caching outputs by input hash so two engineers running the same
pipeline on the same data hit cache, not GPU. `dvc exp run`
extends this to experiment management: each run is a
content-addressed snapshot of code + data + params + metrics that
can be diffed (`dvc exp diff`), shared (`dvc exp push`), and
rolled back without polluting git history.

## Install

```bash
# macOS
brew install dvc

# Cross-platform (recommended for ML projects — pin in requirements)
pip install 'dvc[s3]'           # with S3 remote support
pip install 'dvc[gs,azure,ssh]' # multi-remote
```

## Examples

```bash
# Initialise dvc inside an existing git repo + add a remote
dvc init
dvc remote add -d storage s3://my-bucket/dvc-cache
git add .dvc/config && git commit -m "chore(dvc): wire S3 remote"

# Track a dataset — produces data/raw.csv.dvc (commit this), pushes
# the actual file to S3
dvc add data/raw.csv
git add data/raw.csv.dvc data/.gitignore
dvc push

# Declare + run a reproducible training pipeline
dvc stage add -n train \
  -d src/train.py -d data/raw.csv -p train.lr,train.epochs \
  -o models/model.pkl -M metrics/eval.json \
  python src/train.py
dvc repro                       # runs only what's stale
dvc exp run -S train.lr=3e-4    # parameter sweep, cached by hash
dvc exp show                    # tabulate runs by metric
```

## Why it matters in an AI-native workflow

Most agent-driven ML / LLM workflows run aground on data
provenance: an agent fine-tunes a model on `data/v3/`, the
checkpoint hits 0.92 F1, and three weeks later nobody can
reconstruct what `v3` actually contained because the dataset is a
mutable directory on someone's laptop. `dvc` turns the dataset
into a content-addressed, branch-pinned object — `git log` of the
`.dvc` file is the dataset history, `dvc.lock` is the
machine-checkable proof of which input bytes produced which model
bytes. For agents specifically, `dvc repro` is the right
re-execution primitive: the agent edits `src/train.py`, runs
`dvc repro`, and the framework figures out automatically which
upstream stages need to re-run and which can be served from
cache, so the agent does not have to reason about pipeline
topology. Pairs with [`mlflow`](../mlflow/) (mlflow tracks
metrics + UI; dvc tracks artifacts + lineage — the two compose),
[`aim`](../aim/) (lightweight experiment dashboard, complements
dvc's CLI-first metric diff), and [`lakefs`](../) /
[`pachyderm`](../) for teams that need git-shaped semantics on
the *storage* layer rather than on top of it. Orthogonal to
[`metaflow`](../metaflow/) (metaflow is a workflow scheduler with
opinionated cloud execution; dvc is a versioning + repro layer
that runs anywhere git runs).
