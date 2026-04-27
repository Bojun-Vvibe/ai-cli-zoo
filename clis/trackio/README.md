# trackio

- **Repo:** https://github.com/gradio-app/trackio
- **Version:** trackio@0.25.1 (released 2026-04-26; HEAD pinned 0b9d4d0)
- **License:** MIT (`LICENSE`, blob SHA `b437526c08ea63d41b7dfb177a13d44d49869815`)
- **Category:** Lightweight local-first experiment tracker

## What it does

A drop-in `wandb`-API-compatible experiment tracker from Hugging Face
that runs **fully local by default**: `import trackio as wandb`, call
`wandb.init(project=..., config=...)` and `wandb.log({...})` exactly as
you already do, and runs land in a local SQLite store with a Gradio
dashboard at `trackio show`. Optional `space_id="user/space"` argument
mirrors runs to a free Hugging Face Space, giving a shareable URL
without standing up a server or paying a SaaS bill.

## Install

```sh
pip install trackio
python my_train.py             # uses trackio.init / trackio.log
trackio show                   # opens local dashboard at :7860
trackio show --project my-run  # focus on one project
```

## Why it's interesting

The "I just want to plot a loss curve and not get billed" tier of the
experiment-tracking stack: zero account, zero hosted egress, zero new
API to learn — the surface is intentionally a subset of the existing
`wandb` Python API so swapping `import wandb` → `import trackio as wandb`
in a fine-tuning script (Hugging Face `Trainer`, `accelerate`, plain
`torch`) keeps every existing `wandb.log` call working. The dashboard
is a Gradio app, so the same `mcp_server=True` upgrade path that
[`gradio`](../gradio/) entries get applies — a coding agent can read
metrics over MCP without scraping HTML. SQLite-as-the-store means
`scp run.db` is the entire "share my results" workflow when the HF
Space mirror is not desired, and `trackio.import_csv` / `import_tfevents`
backfill from existing logs.
