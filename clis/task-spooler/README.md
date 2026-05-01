# task-spooler

## What it does
A **personal Unix batch job queue** (binary name `tsp` on Debian /
`ts` on the upstream and on Homebrew): you start a long-lived
per-user spooler the first time you invoke it, then enqueue commands
with `tsp <cmd> <args>` and they run **one at a time, in submission
order**, with stdout / stderr captured to a per-job temp file you
can `tsp -c <id>` (cat finished output), `tsp -t <id>` (tail
running output), or `tsp -i <id>` (job info: exit code, runtime,
queue position). The queue lives entirely in a per-user Unix-domain
socket — no daemon to install, no `systemd` unit, no root, no
config file — so a fresh shell on a fresh box becomes a job server
with one `apt-get install task-spooler` and one `tsp ls`. The
default concurrency is **1 slot** (true serial queue), bumped with
`tsp -S <n>` for n parallel slots, and per-job `tsp -G <gpus>` /
`tsp -L <label>` / `tsp -d` (depend on previous) / `tsp -W <id>`
(wait until job done, exit with its exit code) / `tsp -m <addr>`
(email on completion) cover the "I want to chain a 6-hour fine-tune
after the current 4-hour eval finishes, then email me" workflow
without writing a shell script. Job state survives the controlling
terminal closing — `tsp` daemonises the spooler on first use — so
queueing a job over SSH and disconnecting Just Works, the way
`nohup` + `&` does not.

## Why it's interesting
Different shape from `at` / `batch` (system-wide, root-owned,
needs `atd` running, no live tail, no per-user queue), from
`cron` (time-based, not order-based, no concurrency control, no
output capture), from GNU `parallel` (one-shot fan-out across an
input list — finishes when the input is consumed; task-spooler is
a *long-lived* queue that accepts new jobs over its lifetime),
from `nohup cmd &` + `wait` (no ordering, no introspection, no
slot limit, output management is on you), from
[`pueue`](../pueue/) (Rust daemon with a `pueued` background
process and a YAML config — task-spooler is the strictly-simpler
"one C binary, one Unix socket, no daemon to manage" shape that
predates pueue by ~15 years and survives on systems where
installing a Rust toolchain is not on the table), and from a real
batch scheduler (Slurm / LSF / SGE — multi-node, multi-user, fair
share, accounting; task-spooler is **single-host, single-user**
and proud of it). Pick task-spooler when the actual ask is "queue
these 40 GPU training runs on this one box, run them serially with
one GPU each, let me reorder / cancel / re-tail mid-flight, and
don't make me write a Python wrapper" — the `--gpu` slot
abstraction is the reason ML researchers reach for it ahead of
`pueue` on shared lab machines. Do **not** pick it for
multi-machine fan-out (use Slurm / Ray / Nomad), for reproducible
declarative pipelines (use Snakemake / Nextflow), or when the
queue must outlive the user account (use a system-wide scheduler).

## Niche category
Single-user single-host serial / N-slot job queue — one C binary,
no daemon to manage, GPU-slot aware, survives terminal close.

## Repo
https://github.com/justanhduc/task-spooler

## Version pinned
`v2.0.0` (latest tagged release as of 2026-05-01)

## License
- SPDX: `GPL-2.0`
- License file in upstream repo: `COPYING`
  (https://github.com/justanhduc/task-spooler/blob/master/COPYING)

## Install
```sh
# Debian / Ubuntu (binary name: tsp)
apt-get install task-spooler

# Homebrew (binary name: ts)
brew install task-spooler

# Alpine
apk add task-spooler

# From source (any POSIX with a C compiler)
git clone https://github.com/justanhduc/task-spooler
cd task-spooler && make && sudo make install
```

## Usage examples
```sh
# Enqueue a job (returns the job id, runs when the queue reaches it)
tsp ./train.py --epochs 50 --out runs/exp01

# List the queue (running, queued, finished)
tsp

# Allow 4 jobs to run concurrently instead of the default 1
tsp -S 4

# Show finished job output
tsp -c 17

# Tail a currently-running job
tsp -t 17

# Block until job 17 finishes, then exit with its exit code
tsp -W 17

# Chain: only run after the previous job finishes successfully
tsp -d ./eval.py --checkpoint runs/exp01/last.pt

# GPU-slot aware: claim 1 GPU, the queue won't oversubscribe
TS_SLOTS=4 tsp -G 1 ./train.py --gpu 0

# Email me when the job finishes
tsp -m me@example.com -L "long-finetune" ./finetune.sh
```

## Date added
2026-05-01
