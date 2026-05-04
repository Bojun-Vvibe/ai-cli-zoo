# cdebug

> **A swiss-army knife for container debugging — exec into distroless / scratch containers, port-forward into containers from the host, and capture traffic without a sidecar**
> — a single Go binary by Ivan Velichko (iximiuz) that lets you
> `cdebug exec` into a *running* container even when that container
> ships with no shell (distroless, scratch, FROM `gcr.io/distroless/static`),
> by sideloading a debug rootfs (`busybox`, `nixery`, custom) into
> a sibling container that shares the target's PID/network/mount
> namespaces. Pinned to **v0.0.18** ([LICENSE](https://github.com/iximiuz/cdebug/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/iximiuz/cdebug>

## TL;DR

`cdebug` solves the single most annoying class of container
problems: "production is on fire, the container is the one we
shipped, and we built it `FROM gcr.io/distroless/static` so there
is **no `sh`, no `ls`, no `curl`, no `nc`** to even confirm what
DNS resolves to." `kubectl debug --image=busybox` covers part of
this on Kubernetes, but on a plain Docker / nerdctl / containerd
host you are on your own — and even on Kubernetes the
ephemeral-container API is still gated, version-skewed, and
silently no-ops on managed clusters that haven't enabled it.

`cdebug exec` does the right thing in one command: it spawns a
sibling container from a debug image of your choice (default
`busybox`, but `nixery.dev/shell/curl/jq/strace` is a popular
recipe), joins the target's PID / NET / IPC / UTS / MOUNT
namespaces via the runtime API, and drops you into a real
interactive shell where `/proc/1/root` is the target's root
filesystem. The target container is never restarted, never
mutated, never gets a sidecar baked into its image. When you
exit, the sibling is removed.

`cdebug port-forward` is the second killer feature: it forwards a
host port into a container's network namespace **even when the
container published nothing** (`docker run` without `-p`). You
also get `cdebug exec --privileged` for `tcpdump` / `strace` /
`nsenter` workflows, and a `--platform` flag to pull a
debug image for the target's architecture (handy on Apple Silicon
talking to amd64 containers under emulation).

## Install

```bash
# Homebrew (macOS / Linux)
brew install cdebug

# Go
go install github.com/iximiuz/cdebug@latest

# Release binary (any OS)
curl -L https://github.com/iximiuz/cdebug/releases/download/v0.0.18/cdebug_linux_amd64.tar.gz \
  | tar -xz -C /usr/local/bin cdebug

# verify
cdebug --version
```

Runtime support: Docker, containerd (via nerdctl), Kubernetes
(via the kubelet API). No daemon, no config file. Needs the
runtime socket reachable (`/var/run/docker.sock` or equivalent)
and — for `port-forward` — `CAP_NET_ADMIN` on the host.

## Usage

```bash
# 1) Open a shell inside a distroless container that has no /bin/sh
cdebug exec -it --image nixery.dev/shell/curl/jq distroless-app
# you are now in a busybox+curl+jq shell that sees the target's
# filesystem at /proc/1/root and shares its network namespace, so
# `curl http://localhost:8080/healthz` hits the app directly.

# 2) Port-forward 5432 from the host into a Postgres container that
#    was started without -p (e.g. a docker-compose service on a
#    private network you can't reach from your laptop):
cdebug port-forward --address 127.0.0.1:5432 my-postgres:5432

# 3) tcpdump a running container without rebuilding the image
cdebug exec -it --privileged --image nicolaka/netshoot api-gateway \
  -- tcpdump -i any -w /tmp/api.pcap port 443
```

## Niche & tradeoffs

`cdebug` lives in the same conceptual slot as `kubectl debug`,
`docker exec`, and Nicola Kabar's `netshoot` image — but it is
the only one of those that works **uniformly** across Docker,
containerd, and Kubernetes, and the only one that turns
"distroless + no port published" from a debugging dead-end into
a one-liner. If your fleet is 100% Kubernetes on a recent
control plane with ephemeral containers enabled, `kubectl debug
-it pod/foo --image=busybox --target=app` covers the exec case
and you may not need `cdebug` for that workflow. The moment you
have a mixed fleet — local Docker Desktop for dev, containerd on
edge nodes, GKE / EKS for prod — `cdebug` becomes the one tool
your runbook can assume everywhere.

The tradeoffs are real and worth naming. (1) `cdebug` is still
pre-1.0 (`v0.0.x`) — the CLI surface and flag names have changed
between minor releases, so pin the version in your runbooks
rather than `@latest`. (2) The sibling-container trick depends
on the runtime exposing namespace-join semantics; locked-down
managed runtimes (some Fargate / Cloud Run / serverless
container hosts) will refuse the API call and `cdebug exec`
will fall back to a much weaker mode or fail outright. (3)
`port-forward` rewrites host iptables rules; in environments
with a managed CNI or a host firewall framework (firewalld,
nftables-with-policy), the rules can collide — verify with
`iptables -t nat -L` after a forward and tear down cleanly with
`Ctrl-C` rather than `kill -9`.

The right mental model is "**`kubectl debug` for everything that
isn't Kubernetes, plus the port-forward you wish `docker run`
gave you**." For deeper container *introspection* (filesystem
diff, layer inspection, image SBOMs) reach for [`dive`](../dive/),
[`syft`](../syft/), or [`grype`](../grype/) instead — `cdebug`
is firmly a *runtime* tool, not an *image* tool. For a more
opinionated TUI on top of Docker / containerd, see
[`lazydocker`](../lazydocker/) or [`oxker`](../oxker/); they
complement `cdebug` rather than replace it.
