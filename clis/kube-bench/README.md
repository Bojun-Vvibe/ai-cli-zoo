# kube-bench

> **A CIS Kubernetes Benchmark scanner that audits a running
> cluster (or a single node) against the Center for Internet
> Security's published K8s hardening checks** — runs as a
> privileged Job / DaemonSet (or a host binary), enumerates
> `kubelet`, `kube-apiserver`, `etcd`, controller-manager, and
> scheduler config, and emits PASS / FAIL / WARN for every CIS
> control with the exact remediation command. Pinned to
> **v0.15.4** (SPDX: `Apache-2.0`,
> [LICENSE](https://github.com/aquasecurity/kube-bench/blob/main/LICENSE)).

Source: <https://github.com/aquasecurity/kube-bench>

## TL;DR

The CIS Kubernetes Benchmark is the de facto compliance
baseline (referenced by SOC 2, PCI, FedRAMP, HIPAA assessor
guides). Implementing it by hand means reading a 300-page PDF
and grepping `/etc/kubernetes/*.yaml` for two dozen flag
combinations on every node. `kube-bench` automates exactly
that:

1. **Detects platform** — kubeadm, EKS, AKS, GKE, OpenShift,
   k3s, RKE, MKE — and loads the matching CIS test set
   (`cfg/cis-1.24/`, `cfg/eks-1.5.0/`, etc.).
2. **Reads the live config** — `/etc/kubernetes/manifests/`,
   `/var/lib/kubelet/config.yaml`, `/etc/etcd/etcd.conf`, and
   the running process flags (`ps -ef | grep kube-apiserver`).
3. **Runs every check** — typically 100-200 controls per
   profile — and emits PASS / FAIL / WARN with the CIS
   control number, the rationale, and the *exact remediation
   command* (`Set --anonymous-auth=false in
   /etc/kubernetes/manifests/kube-apiserver.yaml`).
4. **Outputs JSON / JUnit / ASFF** so the result drops into
   GitHub Actions, GitLab CI, AWS Security Hub, or a
   compliance dashboard. Exit code is non-zero on any FAIL,
   so `kube-bench` itself is the CI gate.

## Install

```bash
# Run as a Job in-cluster (recommended for live audits)
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/v0.15.4/job.yaml
kubectl logs -f job/kube-bench

# Helm
helm repo add kube-bench https://aquasecurity.github.io/helm-charts/
helm install kube-bench kube-bench/kube-bench

# Host binary (audit a node from outside the cluster)
curl -Lo kube-bench.tar.gz "https://github.com/aquasecurity/kube-bench/releases/download/v0.15.4/kube-bench_0.15.4_linux_amd64.tar.gz"
tar xf kube-bench.tar.gz && sudo install kube-bench /usr/local/bin/

# Container (one-shot, mounts host paths)
docker run --rm --pid=host -v /etc:/etc:ro -v /var:/var:ro \
  aquasec/kube-bench:v0.15.4 run --targets master,node

# verify
kube-bench version    # 0.15.4
```

## License

Apache-2.0 — see
[LICENSE](https://github.com/aquasecurity/kube-bench/blob/main/LICENSE).
Permissive with patent grant; safe for commercial CI
integration and binary redistribution.

## One Concrete Example

```bash
# 1. run all default targets on the current node
sudo kube-bench

# 2. only the control-plane checks (run on a master)
sudo kube-bench run --targets master,etcd,policies

# 3. only the node checks (run on a worker)
sudo kube-bench run --targets node

# 4. pick a specific CIS profile (override autodetect)
sudo kube-bench run --benchmark cis-1.24

# 5. EKS-specific profile (managed control plane — node-only)
sudo kube-bench run --benchmark eks-1.5.0

# 6. JSON output for CI / dashboards
sudo kube-bench --json | jq '.Totals.total_fail'

# 7. JUnit XML for GitHub Actions / GitLab CI
sudo kube-bench --junit > kube-bench.xml

# 8. fail the CI pipeline only on FAILs (ignore WARNs)
sudo kube-bench --exit-code 1 || echo "CIS audit failed"

# 9. include only specific check IDs
sudo kube-bench --check 1.2.1,1.2.2,1.2.5

# 10. AWS Security Hub ASFF format (drops into Hub directly)
sudo kube-bench --asff
```

## Niche It Fills

**Automated CIS Kubernetes compliance auditing.** The CIS
Benchmark is the universal K8s hardening reference — every
SOC 2 / PCI / FedRAMP control map points at it. `kube-bench`
is the canonical, vendor-maintained implementation: tests are
versioned per Kubernetes release (CIS 1.20 through 1.31), per
distribution (kubeadm, EKS, AKS, GKE, OpenShift, k3s, RKE,
MKE), and per managed-vs-self-hosted split. No other open-source
tool ships this much CIS coverage as plain YAML test
definitions you can read and override.

## Why use it

1. **CIS coverage is exhaustive and versioned.** Drop in a
   new K8s minor release and `kube-bench` ships a matching
   profile within weeks. Custom controls live in
   `cfg/<profile>/master.yaml` as plain YAML — readable,
   diffable, overridable per-org.
2. **Auto-detects the platform.** EKS skips control-plane
   checks (you don't run the API server, AWS does). kubeadm
   includes them. k3s has its own profile because the
   manifest paths differ. You don't pick the right test set;
   `kube-bench` does.
3. **Remediation is in the output.** Every FAIL prints the
   exact `sed` / `kubectl edit` / kubelet-config-edit needed
   to flip the control to PASS. No round-trip to the CIS PDF.
4. **CI-gateable.** Non-zero exit on FAIL; JUnit XML and JSON
   are first-class. GitHub Actions, GitLab CI, Jenkins all
   parse the output natively.
5. **Vendor-maintained.** Aqua Security ships releases
   monthly, tracks new K8s versions, and accepts PRs from
   cloud providers (the EKS profile is co-maintained with AWS).

## Vs Already Cataloged

- **Vs [`trivy`](../trivy/):** trivy scans *images* and
  *config files* for CVEs and misconfigurations
  (`trivy config k8s-manifests/`). `kube-bench` audits a
  *running cluster's* runtime configuration against CIS. Use
  trivy at build time for images and at PR time for manifest
  YAML; use `kube-bench` at deploy time and on a recurring
  schedule for the live cluster's posture.
- **Vs [`gosec`](../gosec/):** completely orthogonal — gosec
  is a Go-source SAST scanner. No overlap.
- **Vs [`gitleaks`](../gitleaks/):** orthogonal — gitleaks
  hunts secrets in repos. `kube-bench` audits cluster config.
  Both belong in a defense-in-depth CI pipeline.

## Caveats

- **Reads live config — needs privileges.** When run as a Job
  it requires `hostPID: true` and host-path mounts of `/etc`,
  `/var`, and `/proc` to read kubelet / etcd files and process
  flags. Locked-down clusters may block this; the host-binary
  install is the alternative.
- **Managed control planes return "N/A" for control-plane
  checks.** EKS, AKS, GKE all hide the API server — controls
  1.x (control plane) skip with a "manual" status because
  there is nothing to audit. Coverage drops to the node
  profile (~30-50 checks) on managed K8s.
- **CIS PASS is necessary, not sufficient.** Passing every
  control means you're aligned with the published baseline; it
  does not mean the cluster is secure against application-layer
  attack, RBAC misuse, or supply-chain compromise. Pair with
  trivy (images), kyverno / OPA (admission), and a runtime
  detector (Falco, Tetragon).
- **Custom platforms need a profile.** Niche distros (Talos,
  k0s) may not have a maintained profile; fall back to the
  generic `cis-1.x` set and accept some false positives on
  paths.
