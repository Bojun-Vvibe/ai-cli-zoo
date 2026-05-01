# parallel

> **Ole Tange's `xargs` on steroids: a perl one-binary `parallel` that
> does grouped output, `--joblog` resumable runs, remote SSH fan-out
> with `--sshlogin`, and `{1}/{2}/{=1 perl =}` template substitution
> — the workhorse 2010s tool that still beats every "modern parallel
> runner" on resumability and remote dispatch.** Pinned to
> **20260422**, GPL-3.0
> ([COPYING](https://git.savannah.gnu.org/cgit/parallel.git/tree/COPYING)).

- **Repo:** https://git.savannah.gnu.org/cgit/parallel.git/
  (mirror: https://github.com/martinda/gnu-parallel)
- **Latest version:** 20260422 (monthly snapshot release cadence)
- **License:** GPL-3.0 (`COPYING` at repo root, SPDX `GPL-3.0-or-later`)
- **Category:** `shell-utilities` / `job-runner`
- **Language:** Perl

## What it does

`parallel` reads jobs from stdin (one per line) or from `:::` arg
lists and runs them in parallel up to `--jobs N` (default = number of
cores), with output captured per-job and printed in input order via
`--keep-order` so a 1000-job run looks indistinguishable from a serial
run except in wall time. The killer features the modern Rust
re-imaginings still don't ship together are: (a) `--joblog jobs.log`
records every job's exit code + runtime + host, and `--resume` /
`--resume-failed` re-reads that log to skip what already succeeded —
turning a 4-hour batch into a "rerun until green" loop; (b)
`--sshlogin host1,host2,host3` (or `--sshloginfile hosts.txt`) fans
the same job stream out to remote machines with automatic load
balancing, and `--transferfile` / `--return` stage inputs and outputs
across the SSH boundary so you don't write the rsync glue yourself;
(c) `{1}` / `{2}` / `{1.}` (no extension) / `{1/}` (basename) /
`{1//}` (dirname) / `{= perl-expr =}` template substitution makes
"for each pair in this CSV, on three remote hosts, run this command"
a one-liner.

## Install

```sh
# macOS
brew install parallel

# Debian / Ubuntu
sudo apt install parallel

# From source (matches the snapshot pinning)
wget https://ftp.gnu.org/gnu/parallel/parallel-20260422.tar.bz2
tar xjf parallel-20260422.tar.bz2 && cd parallel-20260422
./configure && make && sudo make install
```

(One-time `parallel --citation` ack silences the citation notice.)

## Usage

```sh
# Convert every PNG to WebP, 8 jobs in parallel, keep stdout order
ls *.png | parallel -j8 --keep-order \
  'cwebp -q 85 {} -o {.}.webp'

# Resume a long batch after a crash; only failed/missing jobs rerun
parallel --joblog encode.log --resume-failed -j16 \
  'ffmpeg -i {} -c:v libx264 out/{/.}.mp4' ::: src/*.mov
```

## Use when

- You need **resumable** batch jobs — `--joblog` + `--resume-failed`
  is the feature most "modern" alternatives skip.
- You're fanning out across **multiple SSH hosts** without standing
  up Slurm / Nomad / Ray.
- You want template substitution richer than `xargs -I{}` (basename,
  dirname, no-ext, embedded perl).

## Skip when

- A simple `xargs -P` covers the case (no resume, no remote, no
  templates) — fewer moving parts, no citation banner.
- You're already on a real scheduler (Slurm, Nomad, k8s Jobs, GitHub
  Actions matrix) — those own retry / placement / quota.
- You need structured progress output — `parallel` is line-oriented;
  prefer [`pueue`](../pueue/) or [`task-spooler`](../task-spooler/)
  for queue-shaped TUIs.

## Comparison to nearest neighbours

- **vs [`pueue`](../pueue/):** `pueue` is a long-lived task daemon
  with TUI and per-job state; `parallel` is a one-shot launcher with
  a job log. Pick `parallel` for "run 10k jobs across 5 hosts then
  exit," pick `pueue` for "manage a personal queue all day."
- **vs [`task-spooler`](../task-spooler/):** `tsp` is single-host
  FIFO with finalize hooks; `parallel` does multi-host SSH fan-out
  and template substitution `tsp` lacks.
- **vs `xargs -P`:** xargs is in coreutils and Just Works for the
  trivial case; `parallel` adds resumability, ordered output, remote
  hosts, and `{= perl =}` templating at the cost of being Perl.
