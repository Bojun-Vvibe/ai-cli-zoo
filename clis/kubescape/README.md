# kubescape

> **Kubernetes / IaC security posture scanner.** Single Go
> binary that runs OPA-backed control frameworks (NSA, MITRE,
> CIS, SOC2, ArmoBest) against live clusters, raw manifests,
> Helm charts, Kustomize overlays, and Terraform — emitting
> per-control pass/fail with remediation YAML, exception
> handling, and JUnit / SARIF / JSON / PDF output for CI gating.
> Pinned to **v4.0.5**
> ([LICENSE](https://github.com/kubescape/kubescape/blob/master/LICENSE),
> Apache-2.0).

Source: <https://github.com/kubescape/kubescape>

## TL;DR

`kubescape scan` walks a target (cluster / file / chart / repo)
through every rule in a chosen framework and prints a control
table with severity, failed-resource count, and a one-line
"how to fix". Unlike a generic linter, controls are
*compliance-mapped* — each finding cites the NSA / MITRE / CIS
section it violates, so the same scan produces both a
developer-friendly diff *and* an auditor-shaped report.
Exceptions are first-class (`exceptions.json`) so legacy
workloads do not poison the score, and the `--submit` flag
ships results to the optional Armo cloud only when the user
opts in — local `--format json` works fully offline.

## Install

```bash
# Homebrew (macOS / Linux)
brew install kubescape

# Install script (pins binary to ~/.kubescape/bin)
curl -s https://raw.githubusercontent.com/kubescape/kubescape/master/install.sh | /bin/bash

# Krew (kubectl plugin)
kubectl krew install kubescape

# Container
docker run --rm -v $(pwd):/work quay.io/kubescape/kubescape:v4.0.5 scan /work
```

## Example

```bash
# Scan the current kube-context against the NSA framework
kubescape scan framework nsa

# Scan a Helm chart pre-deploy and fail CI under 80% compliance
kubescape scan ./charts/api --compliance-threshold 80 --format junit > report.xml

# Scan a single manifest with a specific control set, JSON for downstream
kubescape scan control "C-0016,C-0017" deploy.yaml --format json -o findings.json

# Scan IaC (Helm + Kustomize + raw YAML) without a cluster
kubescape scan . --enable-host-scan=false --submit=false
```

## When to use

- You ship Kubernetes workloads and need a single tool that
  speaks both "developer linter" and "compliance auditor".
- CI must block PRs whose manifests regress against NSA / CIS /
  MITRE controls, with a stable JUnit / SARIF artifact.
- You want exception management (mute a control for one
  namespace) without forking the rule set.

## When NOT to use

- You only need image-layer CVE scanning — that is `trivy` /
  `grype`, which kubescape itself can call but is not the
  primary surface here.
- You want runtime threat detection (eBPF, syscall filtering)
  — look at `falco` or `tetragon`; kubescape is posture, not
  runtime.
- You need a generic OPA / Rego playground unrelated to
  Kubernetes — use `opa` directly.
