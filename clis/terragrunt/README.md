# terragrunt

> **Thin wrapper around OpenTofu / Terraform that fills in the
> gaps for production multi-environment IaC.** Keeps `backend`
> blocks, provider configs, and module input wiring DRY across
> dozens of stacks via `terragrunt.hcl` units, and runs N stacks
> in parallel with explicit dependency edges. Pinned to **v1.0.3**
> ([LICENSE.txt](https://github.com/gruntwork-io/terragrunt/blob/v1.0.3/LICENSE.txt),
> MIT).

Source: <https://github.com/gruntwork-io/terragrunt>

## TL;DR

`terragrunt` exists because plain Terraform / OpenTofu repos turn
into copy-paste deserts the moment you have more than one
environment. Each *unit* is a directory with a `terragrunt.hcl`
that declares `terraform { source = "..." }` (a Git ref or local
path to a reusable module), an `inputs = { ... }` block, and a
`dependency "vpc" { config_path = "../vpc" }` edge so the unit
can read another unit's outputs at plan time. A top-level
`root.hcl` (formerly `terragrunt.hcl`) defines the `remote_state`
backend once and every child unit `include "root" { path =
find_in_parent_folders("root.hcl") }`s it — one place to change
the S3 bucket / encryption key / lock table for the whole tree.
`terragrunt run --all plan` walks the dependency graph, runs
`tofu plan` in each leaf concurrently, and surfaces a single
exit code; `--queue-include-dir` / `--queue-exclude-dir` scope
which units a CI job touches. Pairs with OpenTofu (default since
the v1.0 cutover) or Terraform; the underlying engine is selected
via `--tf-path` or autodetection.

## Install

```bash
# Homebrew (macOS / Linux)
brew install terragrunt

# Direct binary (Linux amd64)
curl -fsSL -o /usr/local/bin/terragrunt \
  https://github.com/gruntwork-io/terragrunt/releases/download/v1.0.3/terragrunt_linux_amd64
chmod +x /usr/local/bin/terragrunt

# mise / asdf
mise use -g terragrunt@1.0.3
```

## Example

```bash
# Tree layout
# .
# ├── root.hcl                  # one backend + provider config
# └── prod/
#     ├── vpc/terragrunt.hcl    # source = git::...//modules/vpc
#     └── eks/terragrunt.hcl    # depends on ../vpc

# Plan a single unit
cd prod/eks && terragrunt plan

# Plan every unit under prod/, respecting dependency order
terragrunt run --all plan --queue-include-dir prod

# Apply with auto-approve in CI; fail the job on any unit error
terragrunt run --all apply --non-interactive --queue-include-dir prod

# Inspect the resolved dependency DAG
terragrunt dag graph | dot -Tsvg > dag.svg
```

## When to use

- You manage 10+ Terraform/OpenTofu stacks across `dev` / `staging`
  / `prod` (or per-region, per-tenant) and the backend / provider
  boilerplate has become the source of every merge conflict.
- You want one `apply --all` in CI that respects cross-stack
  dependencies (VPC before EKS before app) without hand-rolling
  a wrapper script.
- You need to pin every unit to an exact module version
  (`source = "git::...?ref=v1.4.2"`) and bump them in PRs that
  reviewers can grep.

## When NOT to use

- You have one stack and one environment — plain `tofu` /
  `terraform` is simpler; terragrunt is overhead until the second
  copy of `backend.tf` shows up.
- Your team has standardised on Terragrunt-free patterns
  (workspaces, [Atlantis](https://www.runatlantis.io/) per-dir
  apply, or CDK-for-Terraform) and the migration cost outweighs
  DRY wins.
- You want a controller-based GitOps loop — pair a Kubernetes-
  native operator ([`crossplane`](https://crossplane.io), the
  Terraform Operator, or [Argo CD](../argocd/) + a plain TF
  module renderer) with the cluster instead.
