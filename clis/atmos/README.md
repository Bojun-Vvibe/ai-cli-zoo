# atmos

- **Repo:** https://github.com/cloudposse/atmos
- **Version:** v1.216.0
- **License:** [LICENSE](https://github.com/cloudposse/atmos/blob/main/LICENSE) (Apache-2.0)
- **Category:** Infrastructure as Code (IaC) / Environment composition

## What it is

Atmos is Cloud Posse's framework for composing Terraform/OpenTofu and Helmfile
configurations across many environments (stacks) using YAML inheritance. Instead
of duplicating `tfvars` per region/account/tenant, you describe stacks
hierarchically (`orgs/acme/plat/prod/us-east-1.yaml`), import shared
mixins, and let Atmos render the merged variables for each component before
calling the underlying tool.

## Why it's interesting

- **YAML stack inheritance with deep-merge semantics** — change a default in one
  base file and every inheriting stack picks it up; override per-environment
  without forking modules.
- **Wraps Terraform/OpenTofu and Helmfile uniformly** — `atmos terraform plan
  vpc -s plat-prod-use1` and `atmos helmfile sync app -s ...` share one
  vocabulary for components and stacks, plus dependency ordering and workflow
  pipelines.
- **First-class custom commands and validation** (OPA/JSON Schema) so you can
  bake org-specific guardrails (allowed regions, tagging policies) into the CLI
  rather than ad-hoc CI scripts.
