# troubleshoot

> **Preflight checks and support-bundle collection for Kubernetes
> applications** — ships two CLIs (`preflight` and `support-bundle`,
> both also available as `kubectl` plugins via krew) that read a
> declarative YAML spec, query a live cluster (or a host, or a
> running pod), evaluate cel-like analyzer rules, and emit a pass/
> fail report or a tar.gz of cluster state for offline triage.
> Pinned to **v0.128.1** (released 2026-05-01,
> [`gh api repos/replicatedhq/troubleshoot/releases/latest`](https://github.com/replicatedhq/troubleshoot/releases/latest),
> [LICENSE](https://github.com/replicatedhq/troubleshoot/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/replicatedhq/troubleshoot>

## TL;DR

The Kubernetes diagnostic space splits cleanly into two jobs:
**"is this cluster *ready* to install my app?"** (preflight) and
**"my app is broken — capture everything an SRE needs to debug it
offline."** (support-bundle). `troubleshoot` is the only widely-
adopted tool that solves both with the same spec language and the
same analyzer engine, which is why it's the de-facto framework for
ISVs shipping software *into* customer-managed clusters (Replicated,
the maintainer, built it for that exact use case but the project is
upstream Apache-2.0 and used outside Replicated's commercial product).
The spec is a Kubernetes CRD-shaped YAML: `collectors:` describes
what to gather (cluster resources, pod logs, host CPU/memory, a
SQL query against an in-cluster Postgres, an HTTP probe), and
`analyzers:` evaluates collected facts against thresholds ("at
least one node has 8GiB free", "the storage class `standard` exists",
"this Postgres returns rows for `SELECT 1`"). The output is either
a CLI exit code + colored table (preflight) or a tarball you attach
to a support ticket (support-bundle). Compared to running your own
shell scripts: it's declarative, version-controlled, redaction-aware
(strips secrets, tokens, IPs from the bundle by default), and the
same spec runs in a customer's air-gapped cluster as in your local
kind cluster.

## Install

```bash
# kubectl plugins via krew (recommended — gets both)
kubectl krew install preflight support-bundle

# Standalone binaries via the install scripts
curl https://krew.sh/preflight | bash
curl https://krew.sh/support-bundle | bash

# Homebrew tap
brew install replicatedhq/replicated/preflight
brew install replicatedhq/replicated/support-bundle

# From a release tarball
curl -LO https://github.com/replicatedhq/troubleshoot/releases/download/v0.128.1/support-bundle_linux_amd64.tar.gz
tar xzf support-bundle_linux_amd64.tar.gz && sudo mv support-bundle /usr/local/bin/

# verify
preflight version
support-bundle version
```

## Representative examples

```bash
# 1. Run a preflight against the current kubeconfig context
#    (spec can be a local file, an http(s) URL, or an OCI artifact)
kubectl preflight https://example.com/specs/preflight.yaml
kubectl preflight ./preflight.yaml --interactive=false --format=json

# 2. Minimal preflight spec — "cluster has >=3 nodes and a default
#    StorageClass"
#    preflight.yaml
#    apiVersion: troubleshoot.sh/v1beta2
#    kind: Preflight
#    spec:
#      analyzers:
#        - nodeResources:
#            checkName: at-least-3-nodes
#            outcomes:
#              - fail: { when: "count() < 3", message: "need >=3 nodes" }
#              - pass: { message: "ok" }
#        - storageClass:
#            checkName: default-storageclass
#            storageClassName: standard

# 3. Collect a support bundle (cluster + host + redaction defaults)
kubectl support-bundle ./support-bundle.yaml \
  --output bundle-$(date +%s).tar.gz

# 4. Collect against a *broken* cluster where the API server is
#    unreachable but the host is fine — host collectors only
support-bundle --load-cluster-specs=false ./host-collectors.yaml

# 5. Analyze an existing bundle offline (no cluster needed)
support-bundle analyze ./bundle.tar.gz --analyzers ./analyzers.yaml

# 6. Redact custom patterns out of the bundle (PII, internal hostnames)
#    redactor.yaml
#    apiVersion: troubleshoot.sh/v1beta2
#    kind: Redactor
#    spec:
#      redactors:
#        - name: customer-id
#          removals:
#            regex: [{ redactor: "(customer-)[0-9]+" }]
support-bundle ./bundle.yaml --redactors ./redactor.yaml

# 7. Embed in a Helm chart's CRD-shaped resource so `helm install`
#    can run preflights as a hook
#    templates/preflight.yaml
#    apiVersion: troubleshoot.sh/v1beta2
#    kind: Preflight
#    metadata: { annotations: { "helm.sh/hook": "pre-install" } }
```

## When to use vs. alternatives

- Pick **troubleshoot** when you ship software into clusters you
  don't own (ISVs, on-prem installers, regulated customers) and
  need a *declarative, redaction-aware, customer-runnable* preflight
  + bundle pipeline. Also pick it when your support team's first
  ask on every ticket is "send us a support bundle" — replacing
  ad-hoc `kubectl get all -A -o yaml > dump.yaml` scripts with a
  versioned, redacted spec pays for itself in one P1.
- Pick [`k8sgpt`](../k8sgpt/) when the goal is *interactive triage*
  ("explain this CrashLoopBackOff with an LLM") rather than
  reproducible offline bundles. `k8sgpt` reads cluster state and
  hands a curated context to an LLM; `troubleshoot` produces the
  raw bundle that a human or `k8sgpt` can chew on later.
- Pick [`popeye`](../popeye/) for *opinionated cluster sanitation*
  (ConfigMap unused, requests/limits missing, RBAC drift) on a
  cluster you operate. `popeye` is a fixed ruleset; `troubleshoot`
  analyzers are user-defined per app.
- Pick [`kube-bench`](../kube-bench/) for CIS-benchmark conformance
  specifically — a different question ("is the *control plane*
  hardened?") from "is *my app* installable here?"
- Pick [`kube-linter`](../kube-linter/) /
  [`kubescape`](../kubescape/) for *static* manifest scanning
  pre-deploy — they don't talk to a live cluster.
- Pick a hand-rolled `kubectl get … -o yaml | tar` script when the
  scope is one engineer one cluster one debug session — the spec
  ceremony only pays back when the same collection has to be run
  by a customer over Zoom in three months.
- Caveats: the analyzer DSL is powerful but not Turing-complete
  (cel-style outcomes only — for arbitrary logic, write a custom
  collector that shells out and returns JSON the analyzer can
  read), bundle size grows fast on busy clusters (use
  `--since=1h` and namespace allow-lists), and the redactor is
  pattern-based — review the bundle before sending it externally,
  especially for app-specific tokens the default redactors don't
  know about.
