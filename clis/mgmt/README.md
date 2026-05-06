# mgmt

> Snapshot date: 2026-05. Upstream: <https://github.com/purpleidea/mgmt>

**A reactive, parallel, distributed configuration-management engine
— graph-of-resources DSL where the engine watches every resource
for drift and converges in real time, instead of running on a cron.**
mgmt (`mgmt run`, `mgmt deploy`) is a single Go binary that takes a
declarative `.mcl` program describing system resources (files,
packages, services, users, network interfaces, virtual machines,
DNS records, container images, …), builds a DAG of those resources,
and runs an event-driven engine that *subscribes* to filesystem
events, package-manager state, systemd unit transitions, etcd /
DNS / cloud API changes, and reacts the moment one of them drifts
— so the system converges in milliseconds instead of waiting for
the next 30-minute Puppet / Ansible / Chef sweep.

## Repo + version + license

- Repo: <https://github.com/purpleidea/mgmt>
- Latest release: **`1.0.2`** (2026-02-22)
- License: **GPL-3.0** —
  <https://github.com/purpleidea/mgmt/blob/master/LICENSE>
- License path in repo: `LICENSE`
- Default branch: `master`
- Language: Go

## Install

```bash
# Distro packages exist for Fedora / Arch / Debian-derivatives
sudo dnf install mgmt
yay -S mgmt

# Or build from source (Go 1.22+)
git clone https://github.com/purpleidea/mgmt
cd mgmt && make build

# Run a one-file mcl program against the local host
mgmt run lang examples/lang/hello0.mcl

# Run against an etcd-backed cluster (multi-host distributed mode)
mgmt run --tmp-prefix lang examples/lang/cluster.mcl

# Deploy a program to a running cluster (hot-swap the graph)
mgmt deploy lang examples/lang/v2.mcl

# Validate / lint without applying
mgmt run lang --noop examples/lang/foo.mcl

# Inspect the generated resource graph as DOT for graphviz
mgmt run lang --graphviz=/tmp/g.dot examples/lang/foo.mcl
```

## Niche

The "**reactive, event-driven config management**" slot.

Puppet, Ansible, Chef and Salt are *poll-based*: a daemon (or a
cron / Tower scheduler) wakes up every N minutes, evaluates the
catalog, and applies whatever has drifted since the last run.
This works, but the convergence window is the polling interval
— if a service unit gets disabled at 10:01 and the agent runs
every 30 minutes, the system is wrong from 10:01 to 10:30.
Scaling the polling rate down doesn't work either: every host
hammers the catalog server, and a misconfigured `notify` chain
re-applies expensive resources on every tick.

mgmt inverts the model. Each resource type knows how to *watch*
its real-world state — `inotify` on files, `dbus` events on
systemd units, `apt` / `dnf` / `pacman` callbacks on package
state, `etcd` watches on cluster shared state, cloud-API event
streams — and the engine reacts the moment a watch fires,
re-running only the affected subgraph. Convergence is
sub-second; quiescent hosts use almost no CPU.

Useful for:

- **Drift-sensitive infrastructure** — kiosks, embedded
  appliances, edge nodes — where a 30-minute drift window is
  unacceptable but a poll-every-second loop melts the host.
- **Coordinated multi-host changes** — etcd-backed clusters
  where a config change on one host needs to ripple
  predictably to N peers (load-balancer pool members,
  consensus quorums, blue-green flips).
- **Greenfield Linux fleets** that don't already have a Puppet
  master / Ansible Tower / Chef server investment to amortise
  — mgmt has no central server requirement; a single binary
  per host plus etcd for cluster mode is the whole stack.
- **Replacing a hand-rolled `inotify` + bash + cron pile**
  with a typed DSL and a real DAG.

## Why it matters

- **Reactive engine, not a polling loop** — every resource
  subscribes to OS / cloud / cluster events; the engine
  re-runs only the affected subgraph, in milliseconds, when an
  event fires. Idle hosts use near-zero CPU.
- **Parallel by default** — independent branches of the DAG
  execute concurrently; the dependency graph is real (not a
  best-effort `notify` chain), so resource ordering is
  enforced and parallelism is maximised.
- **`mcl` DSL** — a typed functional configuration language
  with first-class functions, polymorphic types, and a module
  system; not a YAML / JSON dialect with templating bolted on.
  The engine type-checks the program before applying.
- **Distributed mode via etcd** — `mgmt run --seeds=<peer>`
  joins a cluster; shared state lives in etcd; the same
  `.mcl` program runs identically on every node, with cluster
  primitives (`world.kvlookup`, leader election) exposed in
  the language for multi-host coordination.
- **Hot-swap deploys** — `mgmt deploy` ships a new graph to a
  running cluster, the engine diffs against the live graph,
  and only the changed subgraph re-runs; no full re-evaluation
  pass.
- **Graphviz output** — `--graphviz` emits the resource DAG as
  DOT for visual review of what depends on what (worth running
  on every PR; "did this change add an unexpected edge" is
  obvious in the rendered diagram).
- **Honest scope** — mgmt is a config-management *engine* and
  language; it does not bundle a curated module library the
  size of Ansible Galaxy or the Puppet Forge, and it does not
  do agentless SSH push (it's an agent on each host, like
  Puppet / Chef). For provisioning the host in the first
  place (PXE / cloud-init / Terraform), pair with the relevant
  tool — mgmt picks up at first boot.
- **Active in 2026** — `1.0.2` (2026-02-22) is the most recent
  release at snapshot time; the 1.0 milestone (2025) marked
  the language as stable. Cadence is irregular but the
  project has been driven by the same primary author since
  2013.
- **Pre-1.0 friction is largely behind it** — the `mcl`
  language and resource API stabilised at 1.0; modules from
  earlier 0.x releases may need light updating.
- **GPL-3.0** — affects redistribution of the binary; using
  mgmt to manage your own infrastructure has no licensing
  surface. Check the license carefully if you intend to embed
  the engine inside a commercial product.
