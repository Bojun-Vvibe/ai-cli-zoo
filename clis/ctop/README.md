# ctop

- **Repo:** https://github.com/bcicen/ctop
- **Version:** v0.7.7 (latest stable, 2022)
- **License:** MIT ([LICENSE](https://github.com/bcicen/ctop/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install ctop` · `docker run --rm -ti --name=ctop -v /var/run/docker.sock:/var/run/docker.sock quay.io/vektorlab/ctop:latest` · static binaries on the GitHub release page

## What it does

`ctop` is `top` for containers. Run it on any host with a Docker socket (or
containerd / runc via the appropriate connector) and you get a live, sortable
TUI table of every running container with CPU%, memory used / limit, net
RX/TX rates, block IO rates, PIDs, and uptime — refreshed once per second
without polling `docker stats` in a loop. Press `enter` on a row to drop
into a per-container detail view with sparkline graphs of the same four
metrics over the last minute, plus shortcuts to view logs (`l`), exec a
shell (`e`), restart (`r`), stop/kill (`s` / `k`), or pause (`p`) — all
without leaving the terminal. Sort hotkeys cycle through CPU, mem, net, IO,
name; filter mode (`f`) narrows the list by substring; `a` toggles the
"show all containers including stopped" view that `docker ps -a` gives you.
The single static Go binary is ~10 MB, talks to the Docker socket directly
(no daemon, no agent, no telemetry), and works equally well over SSH on a
headless server. Since v0.7 it also supports `runc` and `containerd`
sockets, so it is not strictly Docker-tied — anything OCI-compatible with a
reachable socket will populate the table.

## When to pick it / when not to

Pick `ctop` when you SSH into a host running 5–50 containers and need to
answer "which one is on fire right now" in under five seconds. `docker
stats` shows the same numbers but as a scrolling text dump that you cannot
sort, cannot filter, and cannot drill into; `ctop` is the same data wrapped
in a real TUI. It is the right tool for a 2 AM page where prod is degraded
and you do not yet know which container is eating CPU or leaking memory —
the sortable view tells you instantly, and the inline `l` / `e` shortcuts
let you grab logs or shell in without remembering the container ID. Pair
with [`lazydocker`](../lazydocker/) when you want a heavier multi-pane TUI
that also covers images, volumes, networks, and compose stacks; pair with
[`dive`](../dive/) when the question shifts from "which container is hot"
to "why is this image 2 GB"; pair with [`bottom`](../bottom/) or
[`btop`](../btop/) when you also need host-level (not container-level)
metrics in the same session.

Skip it on Kubernetes — `ctop` only sees containers visible to the local
container runtime socket on one node, which on a kubelet host is a
fragmented, kube-managed view that does not map cleanly to pods or
deployments. Use [`k9s`](../k9s/) for pod-level TUI on Kubernetes and
[`kdash`](../kdash/) for cluster-wide. Skip it for production observability
— it is a live debugging tool, not a metrics pipeline; for that you want
cAdvisor + Prometheus + Grafana, or at minimum a `docker stats --format`
line piped into your existing logging stack. Skip it if you need
per-process CPU / memory inside the container — `ctop` aggregates at the
container boundary; for in-container drill-down, `docker exec` and run
[`bottom`](../bottom/) or `htop` inside. And note that upstream development
has been quiet since 2022 — the tool still works, but do not expect new
features. For a more actively developed alternative with similar shape see
`oxker` (Rust).

## Example invocations

```bash
# Default: live sortable TUI of all running containers on the local Docker socket
ctop

# Include stopped containers in the list
ctop -a

# Sort by memory usage on launch (then 'm', 'c', 'n', 'i' to re-sort live)
ctop -s mem

# Connect to a remote Docker daemon
DOCKER_HOST=ssh://user@prod-host-1 ctop

# Use a containerd socket instead of Docker
ctop -connector containerd

# Run ctop itself as a container (read-only mount of the host socket)
docker run --rm -ti \
  --name=ctop \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  quay.io/vektorlab/ctop:latest

# Filter to containers whose name matches a substring
ctop -f api-

# Inside the TUI: 'enter' to drill in, 'l' for logs, 'e' for exec, 's' to stop
```
