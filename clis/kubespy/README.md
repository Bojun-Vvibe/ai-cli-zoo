# kubespy

- **Repo:** https://github.com/pulumi/kubespy
- **Version:** v0.6.3 (latest release, 2024-04-09)
- **License:** Apache-2.0 ([LICENSE](https://github.com/pulumi/kubespy/blob/master/LICENSE))
- **Language:** Go
- **Install:** `brew install kubespy` · `go install github.com/pulumi/kubespy@latest` · prebuilt binaries on the GitHub release page · binary name is `kubespy`

## What it does

`kubespy` is a small focused tool for **watching a single Kubernetes
object change over time** and showing each mutation as a structured
diff. It opens a watch on the API server for a specified
`resource/name` and re-renders the object every time the API server
emits an `ADDED` / `MODIFIED` / `DELETED` event, so you can see the
exact sequence of `.status.conditions` flips, `.spec` patches by
controllers, replica counts walking up, finalizers landing and
clearing, and ownership references appearing — all the per-tick
state that `kubectl get -w` collapses into a single line.

Three subcommands cover the common shapes:

- `kubespy trace <resource> <name>` — emit a high-level human-
  readable timeline of state transitions (e.g. for a Deployment,
  the rollout's reconcile steps and ready-replica progression).
- `kubespy changes <resource> <name>` — emit a JSON Patch (RFC 6902)
  diff for every mutation, so you can read exactly what changed
  between ticks.
- `kubespy status <resource> <name>` — re-render only the `.status`
  subtree on every change, useful for watching a CR reach `Ready`.

It is read-only, uses the same `~/.kube/config` your `kubectl`
already does, and does not need any in-cluster install.

## When to pick it / when not to

Reach for `kubespy` when an object is *changing* and you need to
see the trajectory, not the final state — a Deployment that takes
20 seconds to roll out, a Custom Resource whose operator walks it
through six `Phase` values before reaching `Ready`, a PVC that
binds and then loses its binding, a Pod whose container restart
loop you want to see condition by condition. It is the right tool
when "describe shows it's stuck in `Pending` but I don't know
which controller paused" and you want to *watch* that controller's
next write to confirm.

Skip it when the question is about **many** objects at once
(use `k9s` or `stern` for cross-pod views), when you only need
the current snapshot (`kubectl get -o yaml` or `kubectl describe`
is one command), or when the controller never writes back to the
object (some webhooks mutate at admission time and never patch
again — `kubespy` will show one event and then nothing). It is
also dormant upstream — last release April 2024 — so treat it as
a battle-tested classic, not a roadmap.

## Why it matters in an AI-native workflow

Coding agents that apply a manifest and then poll for "is it
ready" routinely race the controller and either declare success
too early or time out without learning what changed. Piping
`kubespy changes deploy/api --output json` into the agent's
context for a bounded window after each apply gives a deterministic,
timestamped sequence of patches the agent can reason about
("the controller flipped `Available=False` because the readiness
probe failed at t+8s") instead of guessing from the final
`describe` output why a rollout never finished.

## Example invocations

```bash
# Human-readable rollout timeline for a Deployment
kubespy trace deploy api

# JSON Patch diff for every change to a CR
kubespy changes mycrd.example.com mycrd-instance

# Watch only the .status subtree of a PVC until it binds
kubespy status pvc data-postgres-0

# Bounded window: watch for 30s and exit (wrap with timeout(1))
timeout 30 kubespy changes pod api-7c9d-xyz
```

## Alternatives in this catalog

- [`k9s`](../k9s/) — full TUI; its `:events` and per-resource
  describe views give a similar live signal but interleave many
  objects rather than zooming on one.
- [`stern`](../stern/) — log-tail counterpart; once `kubespy`
  shows you *that* a Pod restarted, `stern` shows you *why*.
- [`kubectl-tree`](../kubectl-tree/) — owner-graph snapshot;
  `kubectl tree` finds which children a root spawned, then
  `kubespy` watches one of those children change over time.
- [`holmesgpt`](../holmesgpt/) — SRE-grade investigation agent;
  reaches for `kubectl describe` + Prometheus rather than per-tick
  watch streams (different layer).
