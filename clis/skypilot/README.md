# skypilot

- **Repo:** https://github.com/skypilot-org/skypilot
- **Version:** `v0.12.0` (latest release)
- **License:** Apache-2.0 (`LICENSE`)
- **Language:** Python
- **Install:** `pip install "skypilot[aws,gcp,kubernetes]"` then `sky check`

## One-line summary

A CLI that takes a single YAML task spec and runs it on the **cheapest
available GPU across any cloud or Kubernetes cluster you have credentials
for**, with autostop, spot recovery, and managed multi-node jobs built in.

## What it does

SkyPilot is not an LLM agent — it's the layer underneath one when you need
real GPUs. The CLI is the entire surface:

- `sky launch task.yaml` — provisions the cheapest VM that satisfies the
  resource block (`accelerators: A100:8`, `cpus: 16+`, `disk_size: 200`),
  syncs your working dir, runs `setup` then `run`, and streams logs.
- `sky exec` / `sky logs` / `sky queue` — re-run on an existing cluster,
  tail logs, see the job queue.
- `sky jobs launch --use-spot` — managed spot jobs with automatic
  preemption recovery and checkpoint resumption.
- `sky serve up service.yaml` — autoscaling inference endpoints with a
  load balancer, health checks, and rolling upgrades across regions.
- `sky down` / `sky autostop` — cost discipline; the default is "kill
  idle clusters after N minutes" so you don't wake up to a $400 bill.

The killer trick is the optimizer: it queries live spot/on-demand prices
across AWS, GCP, Azure, OCI, Lambda, RunPod, Fluidstack, Cudo, Paperspace,
and any Kubernetes context, then picks the cheapest region that has the
hardware in stock right now.

```yaml
# train.yaml
resources:
  accelerators: A100:8
  use_spot: true
setup: |
  pip install -r requirements.txt
run: |
  torchrun --nproc_per_node=8 train.py
```

```bash
sky launch -c train train.yaml
sky logs train
sky down train
```

## When to choose it

- You need GPUs that don't exist in your default region/cloud and don't
  want to learn five different web consoles.
- You're running **fine-tuning, eval sweeps, or batch inference** and the
  hourly cost difference between A100 spot in one region vs on-demand in
  another is real money.
- You want **infra-as-YAML** for ML jobs without standing up Kubeflow,
  Airflow, or a custom Slurm cluster.

## When NOT to choose it

- You only ever use one cloud and your team already has solid
  Terraform / Kubernetes pipelines — SkyPilot's portability premium is
  wasted.
- You need long-lived stateful services with strict SLAs; SkyPilot
  optimizes for throwaway compute, not five-nines uptime.
- You can't grant the CLI broad cloud credentials (provisioning,
  networking, IAM); the convenience evaporates without them.
