# falco

> **Runtime security & observability engine — kernel-level
> syscall and Kubernetes audit-log monitoring with a YAML rules
> DSL.** CNCF graduated project. Reads syscalls via a kernel
> module / eBPF probe (or modern_bpf) and Kubernetes audit
> events, evaluates each event against a rule set, and emits
> alerts for behaviors like "shell spawned in a container",
> "package manager run in production", "sensitive file read by
> unexpected process", "privilege escalation". Defensive only —
> a detection engine, not an exploit toolkit. Pinned to
> **0.43.1**
> ([COPYING](https://github.com/falcosecurity/falco/blob/master/COPYING),
> Apache-2.0).

Source: <https://github.com/falcosecurity/falco>

## TL;DR

`falco` runs as a daemon on every node (host, VM, k8s node).
It hooks into the kernel via the `falco` module / eBPF probe /
modern_bpf, streams syscall events into userspace, and matches
them against rules in `/etc/falco/falco_rules.yaml` plus any
custom `*.yaml` you drop into `/etc/falco/rules.d/`. Matching
events are emitted to stdout, syslog, an HTTP webhook,
gRPC outputs, or `falcosidekick` for fan-out to Slack /
PagerDuty / Loki / Elasticsearch / Prometheus. Rules are
declarative (`condition:`, `output:`, `priority:`) and ship
with a curated default set covering container escape, crypto-
miners, suspicious network IO, secret access, and CIS-style
hardening drift.

## Install

```bash
# Container (the recommended path on Kubernetes nodes)
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --namespace falco --create-namespace \
  --set driver.kind=modern_ebpf

# Linux package (Debian / Ubuntu)
curl -fsSL https://falco.org/repo/falcosecurity-packages.asc | \
  sudo gpg --dearmor -o /usr/share/keyrings/falco-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/falco-archive-keyring.gpg] \
  https://download.falco.org/packages/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/falcosecurity.list
sudo apt update && sudo apt install -y falco

# Run with modern eBPF (no kernel module build required)
sudo falco --modern-bpf
```

## Example

```bash
# Run with a custom rules directory and JSON output
sudo falco -r /etc/falco/rules.d -o json_output=true

# Validate rule syntax without starting the engine
falco --validate /etc/falco/rules.d/my_rules.yaml

# Stream alerts via gRPC for a sidekick fan-out
sudo falco -o grpc.enabled=true -o grpc.bind_address=unix:///var/run/falco.sock
```

## When to use

- You need runtime detection on Kubernetes nodes / Linux hosts
  for behaviors that static scanners (`trivy`, `grype`) cannot
  see — they only catch known CVEs in images, not what the
  process actually does at runtime.
- You want a CNCF-graduated, vendor-neutral substrate to feed
  a SIEM with kernel-level signals.
- You need Kubernetes audit-log analysis with the same rule
  engine as your syscall rules.

## When NOT to use

- You only need image vulnerability scanning — use `trivy` /
  `grype` / `syft` instead; they answer a different question.
- You are on a managed serverless platform with no node access
  (Fargate, Cloud Run) — falco needs kernel-level visibility.
- You want offensive tooling — falco is detection only by
  design; it does not ship exploits.

## Niche / tags

`security` · `runtime-detection` · `observability` ·
`kubernetes` · `ebpf` · `cncf`
