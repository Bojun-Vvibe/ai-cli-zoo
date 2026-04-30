# checkov

- **Repo:** https://github.com/bridgecrewio/checkov
- **Version:** 3.2.525
- **License:** [LICENSE](https://github.com/bridgecrewio/checkov/blob/main/LICENSE) (Apache-2.0)
- **Category:** Static analysis / IaC misconfiguration scanner

## What it is

`checkov` is a Python CLI that scans Infrastructure-as-Code and adjacent
artifacts — Terraform / Terragrunt / OpenTofu, CloudFormation, Kubernetes /
Helm / Kustomize, Dockerfiles, Serverless Framework, ARM / Bicep, Ansible,
GitHub Actions / GitLab CI workflows, OpenAPI specs, and the resulting
container images and packages — against ~1000 built-in policies covering
encryption-at-rest, public exposure, IAM least-privilege, logging, tagging,
and CIS / NIST / PCI / HIPAA mappings. Policies are written in Python or
declarative YAML; custom rules drop into a `--external-checks-dir` and
participate in the same suppression / skip-comment workflow as built-ins.
Output is plain text by default, with SARIF / JUnit / CycloneDX / SBOM /
GitHub-PR-comment formatters for CI integration.

## Install

```
pipx install checkov                                                   # recommended
# or: brew install checkov
# or: pip install checkov
checkov --version
```

## Basic usage

```
checkov -d .                                   # scan current dir, all frameworks
checkov -f main.tf                             # scan one file
checkov -d . --framework terraform,kubernetes  # restrict frameworks
checkov -d . --skip-check CKV_AWS_20           # suppress a noisy rule
checkov -d . --output sarif --output-file-path results
checkov -d . --check HIGH,CRITICAL             # severity gate
checkov -d . --baseline .checkov.baseline      # accept existing findings, gate new
```

## When to use it

- You want **broad multi-framework IaC coverage from one tool** — Terraform
  *and* Kubernetes *and* CloudFormation *and* Dockerfiles in a single CI
  step, with one suppression file and one SARIF feed into the
  code-scanning dashboard.
- You need **compliance-framework mapping out of the box** (CIS AWS / Azure /
  GCP / Kubernetes, NIST 800-53, PCI-DSS, HIPAA, SOC2) so audit asks turn
  into a `--check` filter rather than a research project.
- You want **a baseline workflow** (`--create-baseline` /
  `--baseline`) so adopting checkov on an existing repo does not flood the
  PR with hundreds of pre-existing findings — only *new* misconfigs gate.
- Skip it for **policy-as-code on already-valid IaC** when you want to write
  the rules yourself in Rego (use [`conftest`](../conftest/) — checkov is the
  batteries-included scanner, conftest is the bring-your-own-policy engine),
  and skip it for **runtime Kubernetes admission** (use [`kyverno`](../kyverno/)
  or Gatekeeper — checkov is a pre-deploy CI gate).
