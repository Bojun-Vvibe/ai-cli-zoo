# sake

> **A task runner for remote hosts over SSH** — a Go CLI that
> reads a declarative `sake.yaml` defining hosts (or host
> groups, with tags and inventory imports), tasks (named shell
> snippets with optional templating, env, and per-host TTY),
> and lets you run any task across any subset of hosts in
> parallel with a single command, streaming output prefixed by
> hostname or aggregated as a final table. Pinned to **v0.15.1**
> (commit `72850d13cf296bf4bdcdd3e827b23115946cf5e0`,
> [LICENSE](https://github.com/alajmo/sake/blob/main/LICENSE),
> MIT).

Source: <https://github.com/alajmo/sake>

## TL;DR

`sake` is what you reach for when you have 3–300 hosts you ssh
into regularly (homelab, edge boxes, CI runners, a fleet of
small VPS), and "for h in hosts; do ssh $h uptime; done" has
finally cost you one too many evenings. You write one
`sake.yaml` listing the hosts and a handful of tasks
(`uptime`, `disk-free`, `apt-upgrade`, `pull-latest`,
`restart-app`), then `sake run uptime --all` runs in parallel,
prefixed and colour-coded by host, with a `--output table`
mode that turns it into a sortable matrix. No agent on the
target, no Python on the target, no inventory format invention
— just SSH and YAML.

## Install

```bash
# Homebrew (macOS / Linux)
brew install sake

# Go (any OS with a Go toolchain)
go install github.com/alajmo/sake@latest

# from a release binary (Linux x86_64 example)
curl -sfL https://raw.githubusercontent.com/alajmo/sake/main/install.sh | sh

# generate a starter sake.yaml
sake init

# verify
sake --version    # 0.15.1
```

`sake` shells out to your system `ssh` (so `~/.ssh/config`,
agent forwarding, jump hosts, ProxyCommand, and ControlMaster
all just work).

## License

MIT — see [LICENSE](https://github.com/alajmo/sake/blob/main/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```yaml
# sake.yaml
servers:
  web-1:
    host: web-1.prod.example.net
    user: deploy
    tags: [web, prod]
  web-2:
    host: web-2.prod.example.net
    user: deploy
    tags: [web, prod]
  edge-tokyo:
    host: 198.51.100.7
    user: ubuntu
    tags: [edge]

tasks:
  uptime:
    cmd: uptime
  disk:
    cmd: df -h /
  pull:
    desc: git pull on the deployed app
    cmd: cd /srv/app && git pull --ff-only
  restart-nginx:
    cmd: sudo systemctl restart nginx
    tty: true   # needs sudo password? give it a TTY
```

```bash
# 1. list the inventory
sake list servers

# 2. list available tasks
sake list tasks

# 3. run a task on every host tagged 'web'
sake run uptime --tags web

# 4. run on everything, with output collapsed to a table
sake run disk --all --output table

# 5. ad-hoc one-shot command (no task definition needed)
sake exec --tags prod -- 'cat /etc/os-release | grep VERSION='

# 6. run multiple tasks in sequence per host
sake run pull restart-nginx --tags web --output prefixed

# 7. open an SSH session into a host by name (uses ~/.ssh/config + sake.yaml)
sake ssh web-1

# 8. inventory from a script (anything that prints YAML on stdout)
# servers_from: { cmd: ./inventory.sh }   # in sake.yaml
```

## Niche It Fills

**Lightweight parallel-SSH task runner.** The space splits
into: single-shot parallel-ssh (`pssh`, `parallel-ssh`,
`mussh` — fast, but no inventory model, no task library, no
output framing), full configuration management (Ansible,
[`pyinfra`](../pyinfra/), Salt, Chef — powerful idempotent
operations DSL, but heavyweight for "just run uptime
everywhere"), and lightweight ssh-task runners (`sake`,
`mani`, [`pueue`](../pueue/) over ssh wrappers). `sake` lives
in the third group, and is the most ergonomic of them: real
inventory + tags + groups, task definitions reusable across
runs, prefixed/aggregated output, all in one Go binary with
zero target-side install.

## Vs Already Cataloged

- **Vs [`pyinfra`](../pyinfra/):** different layer. `pyinfra`
  is *configuration management* — declarative idempotent
  operations (`apt.packages`, `files.template`,
  `systemd.service`) that compute diffs and converge state.
  `sake` is *parallel command execution* — you write the
  shell, sake distributes it. Pick `pyinfra` for "ensure these
  hosts have nginx 1.26 with this config"; pick `sake` for
  "run `git pull && systemctl restart` on the web tier right
  now". Many fleets run both.
- **Vs [`sshs`](../sshs/) / [`sshx`](../sshx/):** orthogonal —
  `sshs` is an interactive picker over `~/.ssh/config` for
  *one* host; `sshx` is collaborative shell sharing. `sake` is
  fan-out execution. Pair them: `sshs` to log into one host
  for ad-hoc work, `sake` to do the same thing on twenty.
- **Vs [`mprocs`](../mprocs/) / [`overmind`](../overmind/) /
  [`process-compose`](../process-compose/):** those are
  multi-process *local* runners (foreman-style: a
  `Procfile` defining several long-running processes on one
  machine). `sake` is multi-host *remote* execution. Different
  axis entirely.
- **Vs [`ansible`](https://docs.ansible.com/) ad-hoc mode
  (`ansible all -m shell -a 'uptime'`):** ansible's ad-hoc
  works, but you pay the inventory + Python-on-target +
  module-bootstrap cost just to run `uptime`. `sake` skips all
  of that — it's `ssh` with a fan-out wrapper. Pick ansible
  when you also need its module library; pick sake when shell
  is enough.

## Caveats

- **Not idempotent.** `sake run apt-upgrade` runs
  `apt-get upgrade` every time you call it; there is no "skip
  if already up to date" semantics. That is the whole point of
  the design (it is a parallel-ssh, not a config manager), but
  it means you must not confuse it with Ansible / pyinfra
  for state convergence.
- **Inventory is YAML, not a service-discovery integration.**
  No native Consul / Tailscale / Cloud-API inventory plugin
  layer the way Ansible has. The escape hatch is
  `servers_from: { cmd: ./inventory.sh }`, which lets you
  shell out to anything that prints the right YAML — workable,
  but a build-step you own.
- **Output framing has tradeoffs.** `--output prefixed`
  interleaves lines from many hosts (great for live tail,
  awful for grep); `--output table` waits for every host to
  finish before printing (great for diffing answers across
  hosts, awful when one host is hung). Pick per command, no
  one mode is universally right.
- **No built-in secret store / vault.** `sake` reads env vars
  and shell expansions in `sake.yaml`, but has no
  `ansible-vault` equivalent. Pair it with
  [`sops`](../sops/) / [`gopass`](../gopass/) /
  [`teller`](../teller/) and source secrets at invocation
  time.
- **YAML schema is sake's, not Ansible's or Salt's.** If you
  already have an Ansible inventory you want to keep, the
  `servers_from` shell-out is the bridge — there is no native
  importer.
