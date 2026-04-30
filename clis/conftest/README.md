# conftest

- **Repo:** https://github.com/open-policy-agent/conftest
- **Version:** v0.68.2
- **License:** [LICENSE](https://github.com/open-policy-agent/conftest/blob/master/LICENSE) (Apache-2.0)
- **Category:** Policy-as-code test runner for structured config

## What it is

`conftest` is a CLI that runs [OPA Rego](https://www.openpolicyagent.org/) policies
against arbitrary structured configuration — Kubernetes manifests, Terraform
plans, Dockerfiles, Helm output, `tekton.yaml`, `cloudformation.json`,
`serverless.yml`, plain JSON / YAML / TOML / HCL / INI / EDN. It is **not** an
admission controller and not a runtime enforcer; it is the unit-test harness
for "this YAML must satisfy these rules" that drops into a `Makefile` or a CI
job and exits non-zero when a `deny` / `violation` / `warn` rule fires.
Policies live in a `policy/` directory as `.rego` files, are versioned with the
code they govern, and can be packaged as OCI artifacts for sharing across
teams.

## Install

```
brew install conftest                                                  # macOS / Linuxbrew
# or grab the binary from https://github.com/open-policy-agent/conftest/releases
conftest --version
```

## Basic usage

```
conftest test deployment.yaml                  # run policy/*.rego against one file
conftest test --policy ./policy/k8s manifests/ # explicit policy dir, recursive input
conftest test -p policy/ --combine *.yaml      # evaluate all docs as one input
conftest verify                                # run *_test.rego against fixtures
conftest parse main.tf                         # show how conftest unmarshals input
conftest pull oci://ghcr.io/example/policies   # fetch policy bundle from a registry
conftest push oci://ghcr.io/me/k8s-policies    # publish your policy bundle
```

## When to use it

- You have **structured config that needs hard rules** (no `:latest` tags, no
  privileged pods, every Service has a `team:` label, every S3 bucket has
  encryption) and you want those rules expressed once in Rego rather than
  re-implemented in five YAML linters.
- You want **the same policies in CI and locally** — `conftest test` in
  pre-commit / GitHub Actions / GitLab CI gives the developer the same
  red / green answer the pipeline will give, before they push.
- You want **OPA's policy language without an OPA server** — Rego compiles
  in-process, no sidecar, no daemon, one Go binary.
- Skip it for **runtime Kubernetes admission** (use [Gatekeeper](https://github.com/open-policy-agent/gatekeeper)
  or [`kyverno`](../kyverno/) — conftest is a CI gate, not an admission webhook),
  and skip it when your only need is **schema validation** (use
  [`kubeconform`](../kubeconform/) — conftest is for semantic policy on top of
  valid schemas, not schema validation itself).
