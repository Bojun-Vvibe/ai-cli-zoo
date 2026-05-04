# dozzle

> **Real-time web UI for Docker, Podman, and Swarm container
> logs** — a single static Go binary (or a 30 MB container)
> that mounts the Docker socket, auto-discovers every running
> container on every reachable host, and serves a browser
> dashboard with live multiplexed log streams, full-text
> search across containers, swarm-aware host grouping, and
> stats panels — pinned to **v10.5.1** (commit
> [`4de47308`](https://github.com/amir20/dozzle/commit/4de47308a6709cc797ac84194e5f206db3305469),
> [LICENSE](https://github.com/amir20/dozzle/blob/v10.5.1/LICENSE),
> MIT).

Source: <https://github.com/amir20/dozzle>

## TL;DR

`docker logs -f` for a fleet, in a browser tab. Point dozzle
at one or more Docker / Podman sockets (local socket, remote
TCP, swarm manager, or a list of hosts), open
`http://localhost:8080`, and every container on every host
shows up in a left-pane list with live-tailing log views,
shared search, and a stats overlay — without installing
Loki, Grafana, Promtail, or an agent on each box.

The killer property is **zero-config fleet log viewing**.
Drop the `amir20/dozzle:v10.5.1` image on a swarm manager
or run the binary on a host with `DOCKER_HOST` pointed at a
remote daemon, and you get a unified log dashboard for the
entire cluster in under a minute. No log pipeline, no
storage backend, no per-service config — dozzle reads the
sockets in real time and forgets the data when the browser
tab closes.

## Install

```bash
# container (recommended for production)
docker run -d --name dozzle \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -p 8080:8080 \
  amir20/dozzle:v10.5.1

# Homebrew (macOS / Linux)
brew install dozzle

# Go install
go install github.com/amir20/dozzle@v10.5.1

# prebuilt binary
curl -sSfL \
  https://github.com/amir20/dozzle/releases/download/v10.5.1/dozzle_linux_amd64.tar.gz \
  | tar -xz -C ~/.local/bin dozzle

# verify
dozzle --version    # 10.5.1
```

The binary expects either `DOCKER_HOST` set or
`/var/run/docker.sock` readable by the running user. For
multi-host setups, pass `--remote-host name=tcp://host:2376`
flags or mount a config file listing the hosts.

## Example usage

```bash
# single local daemon
dozzle

# remote daemon over TLS
DOCKER_HOST=tcp://prod-host-1:2376 \
DOCKER_TLS_VERIFY=1 \
DOCKER_CERT_PATH=/etc/docker/certs \
  dozzle

# multi-host fleet (one dozzle, many sockets)
dozzle \
  --remote-host name=app-1,host=tcp://10.0.0.1:2376 \
  --remote-host name=app-2,host=tcp://10.0.0.2:2376 \
  --remote-host name=db-1,host=tcp://10.0.0.3:2376

# swarm mode (runs on every manager, replicas auto-link)
docker service create --name dozzle \
  --constraint=node.role==manager \
  --mode global \
  --mount type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  -p 8080:8080 \
  amir20/dozzle:v10.5.1

# enable auth (htpasswd-style local users)
dozzle --auth-provider simple \
       --username admin --password "$(cat ./pw)"

# enable OIDC SSO (any standard provider)
dozzle --auth-provider forward-proxy \
       --auth-header-user X-Forwarded-User
```

The browser UI exposes:

- left-pane container list grouped by host, swarm service,
  or compose project
- live-tailing log view (`Ctrl+L` clear, `j`/`k` scroll,
  `/` search)
- multi-select containers to merge logs into one timeline
- per-container stats panel (CPU / mem / network)
- log download as `.txt` for the visible time window
- log level coloring (auto-detects `level=info` /
  `[ERROR]` / JSON `level` field)

## Why it matters

- **Zero-pipeline fleet log dashboard.** The full
  observability story — Promtail / Vector → Loki → Grafana
  with retention tiers — is the right answer for
  long-running services. dozzle is the right answer for
  the first hour of a new project, the staging cluster
  nobody wired up to the central stack, and the on-call
  laptop where the operator just wants to see what is
  printing right now across 12 containers.
- **Multi-host without an agent per host.** A single
  dozzle process can poll multiple remote Docker daemons
  via `--remote-host` and present them as one dashboard.
  Compare to a `docker logs -f` SSH-per-host workflow
  that does not survive a 2 a.m. fleet-wide failure.
- **Swarm-native.** Deployed `--mode global` on swarm
  managers, dozzle replicas auto-link via the cloud
  feature and present a unified view; failover does not
  drop log continuity for the active browser session.
  Compose projects are auto-grouped, swarm services are
  auto-grouped, hosts are auto-grouped — the operator
  picks the grouping that matches the question.
- **Search across containers.** The browser search is a
  literal substring or regex match over the live buffer
  of every visible container — `oom-killer` typed once
  highlights matches across all panes simultaneously.
- **Read-only by design.** dozzle cannot `docker restart`,
  `docker exec`, or `docker rm`. Safe to expose to a
  read-only audience (with auth) without granting them
  daemon control.
- **Static binary + tiny container.** ~30 MB image; no
  Java, no Node runtime, no separate database. Suitable
  for resource-constrained edge nodes, single-board
  deploys, and air-gapped environments.

## Vs Already Cataloged

- **Vs [`lazydocker`](../lazydocker/):** lazydocker is a
  TUI for one local daemon — fantastic for a developer
  laptop where the workflow is "pick one container, look
  at it, kill it, restart it." dozzle is a browser UI for
  a fleet — fantastic when there are 20 containers across
  3 hosts and the operator wants to merge logs from 4 of
  them in one pane while a teammate watches over their
  shoulder via a shared URL. Pick lazydocker for solo CLI
  workflows; pick dozzle for shared fleet visibility.
- **Vs [`ctop`](../ctop/):** ctop is a TUI for live
  container resource stats with a sparkline-driven layout.
  Orthogonal: ctop answers "which container is burning
  CPU"; dozzle answers "what is that container printing
  to stdout right now." Compose them — ctop on the
  terminal, dozzle in the browser.
- **Vs [`logdy`](../logdy/) / [`lazyjournal`](../lazyjournal/):**
  logdy is a generic log-stream-to-browser bridge for
  arbitrary stdout pipes; lazyjournal is a TUI that spans
  systemd / syslog / docker / kube. dozzle is
  Docker-native and fleet-shaped — it does not try to
  read journalctl or syslog, but it scales to a swarm and
  serves a multi-user web UI, which the other two do not.
- **Vs Grafana Loki + Promtail + Grafana stack:** Loki
  indexes logs across the fleet with label-based query,
  retention tiers, and historical search. dozzle has no
  storage and no historical search — the buffer is the
  in-flight tail. The two compose: Loki for the long
  story, dozzle for the right-now story.
- **Vs [`stern`](../stern/):** stern is the kube-pod log
  multiplexer — a CLI that streams logs from pods
  matching a regex. dozzle does not target kube directly
  (it speaks the Docker API). For a Docker / Podman /
  swarm fleet, dozzle wins on UX; for a Kubernetes
  cluster, stern + a real observability stack is the
  right pair.

## License

MIT — see
[LICENSE](https://github.com/amir20/dozzle/blob/v10.5.1/LICENSE).
Run inside private container images, ship as part of a
commercial appliance, embed in an internal admin portal —
all unrestricted under MIT terms. Optional `dozzle/cloud`
add-on is a separate paid hosted control plane; the binary
itself is fully self-contained and fully open source.

## Caveats

- **Not a log archive.** dozzle keeps an in-memory tail
  per container; closing the browser tab discards
  unobserved history. For "what happened yesterday at
  03:14," use Loki / Elasticsearch / a sidecar exporter.
- **Daemon-trust scope.** Mounting the Docker socket
  gives dozzle full daemon access. The web UI is
  read-only, but the *process* has root-equivalent power
  on that daemon — treat the dozzle host like any other
  daemon-control surface and put the UI behind auth /
  reverse proxy with TLS.
- **Auth is opt-in.** Out of the box dozzle binds to
  `0.0.0.0:8080` with no auth — fine on a laptop, not
  fine on a public IP. Always set `--auth-provider` or
  put it behind a reverse-proxy auth gate (Traefik
  forward auth, [`caddy`](../caddy/) basic auth, an OIDC
  proxy) before exposing.
- **Active 10.x line.** API surface (config flags,
  `--remote-host` form) is stable across the 10.x line
  but the swarm / cloud features are evolving — pin to
  `v10.5.1` in production manifests rather than tracking
  `latest`.
- **No Kubernetes.** dozzle reads the Docker / Podman
  API; it does not speak the kubelet API. For pure-kube
  workflows pair with [`k9s`](../k9s/) (interactive
  cluster TUI) and `stern` (log multiplexer).

## As of

2026-05-04. Upstream tag `v10.5.1` (2026-04-30). Active
release line with weekly minor releases; the
multi-host / swarm / cloud architecture stabilized in the
10.x line and is the right pin point for a long-lived
internal deployment. Re-verify on 11.x for any auth /
config-flag changes.
