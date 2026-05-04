# opencost

> **Vendor-neutral, Kubernetes-native cost monitoring for any cluster on any
> cloud.** A Go workload that runs in-cluster, joins Prometheus-collected
> resource usage (`container_cpu_usage_seconds_total`,
> `container_memory_working_set_bytes`, `kube_persistentvolumeclaim_*`,
> `node_*`) with cloud-provider pricing data (AWS / GCP / Azure / on-prem
> custom), and emits per-namespace / per-deployment / per-pod / per-label
> cost in dollars over arbitrary time windows via REST API, Prometheus
> metrics, CLI (`kubectl-cost`), and a UI. Pinned to **v1.120.1**
> (released 2026-04-28, SPDX: `Apache-2.0`,
> [LICENSE](https://github.com/opencost/opencost/blob/develop/LICENSE)).

Source: <https://github.com/opencost/opencost>

## Repo

- URL: <https://github.com/opencost/opencost>
- Owner/org: opencost (CNCF Incubating project, originally donated by Kubecost)
- License file: [LICENSE](https://github.com/opencost/opencost/blob/develop/LICENSE)

## Version

`v1.120.1` — released 2026-04-28. Verify with `opencost version` inside the
container or query `/about` on the API. The CNCF-donated open core has been
on a fast 1.x cadence since the 2024 incubation acceptance; check the
release notes for the relevant cloud provider's pricing-API field changes
each release.

## License

**Apache-2.0** — OSI-approved, permissive. Safe to redistribute, embed in a
platform image, fork for private extensions. The Kubecost commercial product
sits on top of opencost as the open core; opencost itself is the Apache-2.0
data plane and is the right pick when "we want the numbers, not a vendor
relationship" is the requirement.

## What it does

Deployed as a single Helm chart (or kustomize tree, or one Deployment + one
Service), opencost scrapes:

- **Cluster usage** from an in-cluster Prometheus (kube-state-metrics +
  cAdvisor + node-exporter — the same stack any production cluster already
  runs).
- **Cloud pricing** from AWS Cost & Usage Report / GCP Billing Export /
  Azure Rate Card / a user-supplied JSON for on-prem and bare metal.

It then computes per-resource hourly cost (CPU-core-hours × price,
GiB-hours × price, GiB-month × PV price, GB egress × egress price, GPU-hours
× GPU price) and aggregates over any group-by the API supports
(`namespace`, `deployment`, `controllerKind`, `pod`, `label:team`,
`annotation:cost-center`, `node`, `cluster`, custom). The same numbers are
exposed as Prometheus metrics for Grafana dashboards / alerting on cost
spikes / FinOps showback reports.

`kubectl-cost` is the CLI front-end — `kubectl cost namespace --window 7d`
prints a sorted table of namespace spend without leaving the terminal.

## When to use

- **Multi-tenant cluster, need per-team showback or chargeback.** Cloud
  provider bills give you "this cluster cost $14k last month"; opencost
  gives you "the `data-eng` namespace was $4.2k of that, of which the
  `airflow-workers` deployment was $2.8k, of which the `etl-large` pod
  spec accounted for 71%".
- **You want the numbers in Prometheus / Grafana** alongside the rest of
  your reliability metrics, not in a separate SaaS dashboard.
- **You run on-prem or hybrid** and need a tool that accepts a custom
  pricing sheet for bare-metal nodes alongside cloud pricing for the
  cloud-bursted nodes.
- **FinOps is becoming a CI gate.** opencost's API + the predicted-cost
  endpoint can answer "this PR adds 300m CPU × 4 replicas — projected
  monthly cost $X" as a check.

## When NOT to use

- **Single-tenant cluster, single team.** The cloud provider's bill is
  enough; the in-cluster stack is unjustified ops surface.
- **Non-Kubernetes workloads dominate** (VMs, serverless, managed DBs).
  opencost only sees what runs as pods. Use the cloud's native cost
  explorer or a SaaS like Vantage / CloudZero / Finout.
- **You need invoice-grade reconciliation.** opencost is *near*-real-time
  estimation against published rates and works for showback / engineering
  decisions; it is not a billing system. Reconcile against the cloud's
  authoritative bill monthly.

## Alternatives in this catalog

- [`k9s`](../k9s/) / [`kubectl-tree`](../kubectl-tree/) — operator UX for
  *what is running*, not *what it costs*. Orthogonal; pair them.
- [`popeye`](../popeye/) / [`kube-score`](../kube-score/) /
  [`kube-linter`](../kube-linter/) — cluster *health* and *config quality*
  rather than cost. Run alongside.
- [`pluto`](../pluto/) / [`kubescape`](../kubescape/) — API deprecation and
  security posture; orthogonal axes.
- [`steampipe`](../steampipe/) — query cloud APIs (including billing) in
  SQL; pick when you want ad-hoc cost queries across *all* cloud resources
  not just the Kubernetes-shaped ones.
- [`infracost`](../infracost/) — pre-deploy IaC cost estimation from
  Terraform / OpenTofu plans; complementary (predict before apply,
  measure after deploy).
- [`grafana`](https://grafana.com/) dashboards consuming the opencost
  Prometheus metrics are how most teams operationalise the data.
