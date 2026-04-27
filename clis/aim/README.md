# aim

- **Repo:** https://github.com/aimhubio/aim
- **Version:** v3.29.1 (latest release; HEAD pinned 6e098e3)
- **License:** Apache-2.0 (`LICENSE`)
- **Category:** ML experiment tracking / observability

## What it does

A self-hosted, open-source experiment tracker for ML and LLM training runs.
Logs metrics, hyperparameters, system stats, and arbitrary objects (text,
images, audio, distributions) to a local on-disk store, then serves a
local web UI plus a Python SDK and `aim` CLI for querying runs with a
pandas-friendly query language.

## Install

```sh
pip install aim
aim init           # create .aim repo in cwd
aim up             # start the UI on :43800
```

## Why it's interesting

Unlike SaaS-first trackers, the entire data plane is a local directory you
own — no account, no egress, no per-seat pricing. The query layer (`AimQL`)
treats runs as first-class Python objects you filter with real expressions
(`run.hparams.lr < 1e-3 and metric.name == "loss"`), which makes it usable
both as an interactive notebook tool and as a CI artifact you check in.
Equally at home tracking a 5-line scikit-learn loop or a multi-week
LLM fine-tune across many GPUs.
