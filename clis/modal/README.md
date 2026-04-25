# modal

> Snapshot date: 2026-04. Upstream: <https://github.com/modal-labs/modal-client>

**Run any Python function in the cloud as if it were local — and
serve LLMs / agents on serverless GPUs from a `modal.com` account.**
The `modal` CLI + Python SDK lets you decorate a function with
`@app.function(gpu="A100", image=Image.debian_slim().pip_install(...))`
and `modal run script.py` ships the code to remote infrastructure,
provisions the GPU on demand, and streams logs back to your
terminal — useful as the *execution substrate* under agent
frameworks (sandboxed code execution, GPU-bound inference, batch
eval fan-out, scheduled crons) without managing Kubernetes.

## Repo + version + license

- Repo: <https://github.com/modal-labs/modal-client>
- Latest tag: **`v1.3.1`** (2026-04 — actively released, no GitHub
  "release" page, tag-driven shipping)
- HEAD on `main`: `71251fe`
- License: **Apache-2.0** —
  <https://github.com/modal-labs/modal-client/blob/main/LICENSE>
- License path in repo: `LICENSE`
- License SPDX: `Apache-2.0`
- Default branch: `main`
- Language: Python (client) against a hosted Rust control plane

## Install

```bash
pip install modal
modal setup                      # browser-based auth, writes ~/.modal.toml
modal run hello.py               # invoke a @modal.function locally → cloud
modal deploy app.py              # persistent deployment with HTTPS endpoint
modal serve app.py               # local-dev mode with hot reload
modal volume create my-data      # persistent storage primitives
modal nfs / dict / queue / secret  # shared-state primitives across runs
```

## Niche

The "**serverless GPU + sandboxed code-exec substrate for AI
workloads**" lane. Adjacent to [`e2b`](../e2b/) (which targets the
"sandboxed code interpreter as an agent tool" use case) and
[`container-use`](../container-use/) (which targets the "give each
agent its own ephemeral workspace" use case), but Modal's reach is
broader: GPU inference servers, batch jobs, web endpoints, cron
schedules, distributed map/fan-out, and persistent volumes — all
from one Python decorator and one CLI.

## Why it matters

- **`modal run` is `python script.py` with a GPU.** The DX
  collapses the "package, push to registry, write a deployment
  manifest, kubectl apply, check pod logs" loop into one command;
  cold-start is single-digit seconds because the platform reuses
  warm containers across invocations, and `gpu="A100"` /
  `gpu="H100"` / `gpu="L40S"` is a function-decorator argument,
  not a Terraform module.
- **Sandboxes as a first-class primitive.** `modal.Sandbox.create(...)`
  spins an ephemeral container an agent can `exec()` arbitrary
  shell into, with filesystem + network policy + a per-sandbox
  volume — the same pattern [`openhands`](../openhands/) and
  agent frameworks use for "let the LLM run code", but elastic
  to thousands of parallel sandboxes without you running the
  control plane.
- **Inference + batch in the same SDK.** A `@app.function(gpu=...)`
  loaded with `vllm` becomes an HTTP endpoint via `@modal.web_endpoint`
  *or* a fan-out batch job via `func.map(inputs)` — the same code
  serves prod traffic and runs your nightly eval sweep across
  10,000 prompts.
- **Shared-state primitives that fit agent workloads.**
  `modal.Dict` (cross-run KV), `modal.Queue` (fan-in/fan-out),
  `modal.Volume` / `modal.NetworkFileSystem` (persistent storage
  mounted into the container), `modal.Secret` (provider keys
  injected at function start) — the pieces an agent runtime
  actually needs to remember things between invocations.
- **Hosted control plane, your code.** The client is OSS
  Apache-2.0; the platform is a paid managed service (with a
  generous free tier). For air-gapped or fully self-hosted
  alternatives, see [`ramalama`](../ramalama/) /
  [`vllm`](../vllm/) for inference and [`harbor`](../harbor/) for
  the local-stack orchestrator equivalent.
