# tflint

> **Pluggable linter for Terraform.** Catches what `terraform
> validate` misses — deprecated syntax, unused declarations,
> bad naming, and **provider-specific** errors like invalid AWS
> instance types, nonexistent GCP machine families, or Azure
> region typos that would only surface at `apply` time. Plugin
> architecture ships first-party rulesets for AWS, GCP, Azure,
> plus community plugins. Pinned to **v0.55.1**
> ([LICENSE](https://github.com/terraform-linters/tflint/blob/master/LICENSE),
> MPL-2.0).

Source: <https://github.com/terraform-linters/tflint>

## TL;DR

`terraform validate` only checks HCL syntax and config-time
references; it will happily accept `instance_type =
"t2.notreal"`. `tflint --init && tflint` loads the AWS plugin,
walks the module, and rejects bad enums, deprecated arguments,
and missing required attributes — including ones that depend on
the provider's API shape, not the Terraform schema. Config in
`.tflint.hcl` selects rulesets and per-rule severity. Returns
non-zero on findings so it drops straight into CI.

## Install

```bash
# Homebrew (macOS / Linux)
brew install tflint

# Install script (any platform)
curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash

# Docker
docker run --rm -v $(pwd):/data -t ghcr.io/terraform-linters/tflint:v0.55.1

# Pre-built binary
curl -L https://github.com/terraform-linters/tflint/releases/download/v0.55.1/tflint_linux_amd64.zip -o tflint.zip
```

## Example

```bash
# One-time: install plugins declared in .tflint.hcl
tflint --init

# Lint current module with default config
tflint

# Recurse into every module under terraform/
tflint --recursive --chdir terraform/

# JSON output for CI annotation
tflint --format=json
```

`.tflint.hcl` minimal:

```hcl
plugin "aws" {
  enabled = true
  version = "0.40.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}
```

## When to use

- You want fail-fast detection of bad provider enums (instance
  types, regions, image IDs) before `terraform plan` even runs.
- You want naming / style rules enforced uniformly across many
  modules and authors.
- You want one CI step that covers AWS + GCP + Azure modules
  through a single plugin host.

## When NOT to use

- You need policy-as-code for what infra is *allowed* (cost
  caps, tag enforcement against external systems, drift) — that
  is `checkov` / `conftest` / Sentinel territory.
- You only write Terragrunt scaffolding without raw HCL — tflint
  reads HCL directly, you may need `terragrunt run-all` wiring.
- You want security-posture scans (e.g., "S3 bucket is public");
  tflint will catch *invalid* config, not *unsafe* config — pair
  with `checkov` or `tfsec`.
