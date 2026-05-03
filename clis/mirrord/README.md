# mirrord

- **Repo:** https://github.com/metalbear-co/mirrord
- **Version:** 3.155.0 (latest stable)
- **License:** MIT ([LICENSE](https://github.com/metalbear-co/mirrord/blob/main/LICENSE))
- **Language:** Rust
- **Install:** `brew install metalbear-co/mirrord/mirrord` · `curl -fsSL https://raw.githubusercontent.com/metalbear-co/mirrord/main/scripts/install.sh | bash` · static binaries on the GitHub release page (`mirrord_mac_universal`, `mirrord_linux_x86_64`, `mirrord_linux_aarch64`) · IDE plugins for VS Code, IntelliJ, PyCharm, GoLand, WebStorm

## What it does

`mirrord` is the "run your laptop process *as if it were* a pod in your remote Kubernetes cluster" tool. You run your service locally — `python app.py`, `go run .`, `node server.js`, `cargo run`, whatever — under `mirrord exec --target pod/checkout-7d9f -- python app.py`, and mirrord injects a small dynamic library (`libmirrord_layer`) into the local process via `LD_PRELOAD` (Linux) / `DYLD_INSERT_LIBRARIES` (macOS) that intercepts every `connect`, `bind`, `accept`, `read`, `write`, `open`, `getaddrinfo`, and env-var lookup. The intercepted calls are routed over a long-lived gRPC tunnel to a tiny `mirrord-agent` pod that mirrord schedules into the target's namespace as an ephemeral container, which then performs the syscall *from inside the cluster network namespace, with the target pod's service account, and against the target pod's mounted secrets*. Net effect: your local debugger-attached binary sees Kubernetes service DNS (`postgres.checkout.svc.cluster.local`) resolve, sees in-cluster DNS, sees the same env vars and secret files as the real pod, can hit the same internal Kafka and Redis the pod can hit, and can receive *mirrored* copies of every request the real pod is currently serving (the default mode — production traffic is replayed to your laptop without disrupting the real pod) or *steal* the traffic outright (`--steal`, where requests are routed to your laptop instead of the pod). File reads, by default, are read from the cluster pod's filesystem; writes can be local-only (`--fs=read`), tunneled, or mixed via per-path policy. The model collapses the entire "build → push → deploy → wait → curl → repeat" inner loop down to "save → re-run locally → see request hit your breakpoint", with the cluster's actual networking and config in scope. mirrord runs against any Kubernetes target — Deployment, StatefulSet, Pod, Job, even a Service in steal-by-HTTP-header mode — and ships an open-source CLI plus a hosted "mirrord for Teams" product that adds RBAC and policy controls; the CLI alone is fully usable without the SaaS.

## When to pick it / when not to

Pick `mirrord` whenever the inner loop on a microservice is dominated by "I need the cluster's network and secrets to test this code path" and the round-trip of building an image, pushing it to the registry, kicking the deployment, and waiting for the pod to roll is destroying the day. Especially valuable for: microservices that depend on a rats-nest of in-cluster services (you cannot easily port-forward all 14 of them); secrets and IAM/IRSA bindings that only exist on the pod's service account (mirrord inherits them transparently — the local process gets the cluster identity); debugging a flaky test that only fails against real cluster state (attach a debugger to the local process under mirrord, hit the failing endpoint, see the breakpoint fire with the real upstream payload); staged migrations where you want to A/B a new version against live traffic without deploying it (`--steal --steal-filter 'header X-Canary=me'` routes only your-flagged requests to your laptop). Pair with [`telepresence`](../telepresence/) only as comparison (see below). Pair with [`k9s`](../k9s/) / [`stern`](../stern/) for the cluster-side observation. Pair with [`tilt`](../tilt/) / [`skaffold`](../skaffold/) when the inner loop you are optimizing is multi-service and you want fast image build + deploy *plus* mirrord's runtime overlay on the one service you are actively editing.

Skip mirrord for production clusters where injecting an ephemeral agent is policy-prohibited (use a dedicated dev / staging cluster); for environments without Kubernetes (it is K8s-specific); for purely local stacks where Docker Compose or Tilt's local-runtime mode already gives you fast iteration without involving any cluster; for cases where the service has no upstream dependencies (just run it locally with mocks); and for security-sensitive paths where steal mode would let your laptop receive real customer PII — the policy story for that lives in mirrord-for-Teams or in cluster-side admission rules, not in the OSS CLI alone.

## Vs already cataloged

- **Vs [`telepresence`](../telepresence/):** the closest comparison. Telepresence is the older incumbent (CNCF Sandbox), uses a VPN-style traffic agent + intercepts model, and rewrites cluster DNS at the host level. mirrord is newer, ships per-process via `LD_PRELOAD` (so two services on the same laptop can target two different clusters with no host-level state), needs no DNS / route changes, and has finer-grained controls over what is mirrored vs stolen vs read locally. Pick telepresence if your team is already on the CNCF stack and you want the long-running session model; pick mirrord if you want per-invocation, no-host-state, debugger-friendly inner loops.
- **Vs [`tilt`](../tilt/) / [`skaffold`](../skaffold/):** complementary. Tilt and Skaffold optimize the *deploy* step (fast incremental image build + apply); mirrord eliminates the deploy step entirely for the one service you are editing. Many teams run Tilt for the supporting services and mirrord for the service under active development.
- **Vs [`kubefwd`](../kubefwd/) / `kubectl port-forward`:** much narrower. Port-forwarding gives the local process *outbound* access to one cluster service per forward. mirrord gives the local process the cluster's full network, DNS, env, secrets, and (optionally) inbound traffic. Port-forwarding is fine when one TCP connection is enough; mirrord is for when "the local process should think it is the pod".
- **Vs [`k3d`](../k3d/) / [`kind`](../kind/):** different axis. Those give you a *local* cluster for offline iteration; mirrord targets your *real* cluster so the local process sees real upstream services and real config. Use kind/k3d when you want a sandbox; use mirrord when you want the actual environment in scope.
- **Vs [`devbox`](../devbox/) / [`colima`](../colima/):** orthogonal. Those are dev-shell and container-runtime layers; mirrord is the cluster-overlay on top. They compose.

## Caveats

- **mirrord injects a binary into the target namespace.** The `mirrord-agent` ephemeral container needs RBAC to attach an ephemeral container (`pods/ephemeralcontainers`, K8s 1.23+) or fall back to a transient agent pod (`pods` create + delete). On locked-down clusters, an admin needs to grant that — without it, `mirrord exec` errors immediately. The mirrord-for-Teams operator centralizes the RBAC instead of granting it per-developer.
- **Steal mode redirects real traffic.** `mirrord exec --steal` against a production-adjacent target will send live requests to your laptop. Reach for `--steal --steal-filter '...'` (HTTP-header / path filtering) so only the requests you mark are stolen, and never run unfiltered steal against a target that serves real users.
- **`LD_PRELOAD` / `DYLD_INSERT_LIBRARIES` injection has known compatibility edges.** Statically linked Go binaries built with `CGO_ENABLED=0` cannot be intercepted by the layer (no dynamic linker hook); mirrord handles Go specifically via a runtime patcher, but exotic toolchains (musl-static Rust, `cosmopolitan` libc, Bazel hermetic builds) may need `--copy-target` mode where mirrord runs your binary inside the agent container instead of locally. macOS SIP also blocks injection into Apple-signed binaries — wrap with a script you own.
- **Local file writes default to local-only.** This is usually correct (you do not want a local debugger run to mutate the cluster pod's filesystem) but it means a process that depends on writing then re-reading the same path through the *cluster* file API needs `--fs=write` or a per-path mapping. Read the FS-mode docs before debugging anything stateful.
- **The agent pod has CPU and memory cost in the target cluster** (small but non-zero) and lives for the duration of the session. On clusters with strict resource quotas, leftover agents from killed sessions can accumulate; the mirrord controller usually GCs them within minutes, but `kubectl get pods -A -l app=mirrord-agent` is the cleanup query.
- MIT ([LICENSE](https://github.com/metalbear-co/mirrord/blob/main/LICENSE)) — permissive; safe to redistribute the `mirrord` binary inside internal developer-platform images. The hosted mirrord-for-Teams operator is a separate commercial product; the CLI is open and standalone.

## Example invocations

```bash
# Install
brew install metalbear-co/mirrord/mirrord

# Mirror traffic of a target Deployment to a local process
# (default mode: target keeps serving; you get a copy of every request)
mirrord exec --target deployment/checkout -- python app.py

# Steal traffic instead of mirroring (requests go ONLY to your laptop)
mirrord exec --target deployment/checkout --steal -- python app.py

# Steal only requests that carry a specific header (safe for staging)
mirrord exec --target deployment/checkout \
  --steal --steal-filter 'header: X-Canary=me' -- python app.py

# Use a config file for repeatable runs (commit this to the repo)
cat > .mirrord/mirrord.json <<EOF
{
  "target": "deployment/checkout",
  "feature": {
    "network": { "incoming": "mirror" },
    "fs":      "read",
    "env":     true
  },
  "agent": { "namespace": "checkout" }
}
EOF
mirrord exec -- python app.py

# Attach to an already-running local process (port-mode)
mirrord port-forward --target deployment/checkout --ports 8080:8080

# Drop into a shell inside the cluster network namespace via mirrord
mirrord exec --target deployment/checkout -- bash

# Inspect what mirrord would route, without actually launching
mirrord verify-config .mirrord/mirrord.json
```
