# pulumi

> **Infrastructure as code in real programming languages.** Define
> AWS / Azure / GCP / Kubernetes / Cloudflare resources in TypeScript,
> Python, Go, .NET, Java, or YAML against a stateful resource graph
> with diff / preview / up / destroy / refresh / import lifecycle
> verbs. Pinned to **v3.232.0**
> ([LICENSE](https://github.com/pulumi/pulumi/blob/v3.232.0/LICENSE),
> Apache-2.0).

Source: <https://github.com/pulumi/pulumi>

## TL;DR

`pulumi` is the IaC engine that lets you skip the HCL DSL and write
infrastructure in a Turing-complete language with the same imports,
loops, abstractions, and unit-test stories your application code
already uses. A *program* (e.g. `index.ts`) registers `Resource`
instances; the engine builds a desired-state graph; `pulumi preview`
diffs against the recorded *state* (stored in Pulumi Cloud, S3, GCS,
Azure Blob, Postgres, or local file via `pulumi login --local`); and
`pulumi up` reconciles. *Stacks* are named state instances per
environment (`dev`, `staging`, `prod`) sharing one program.
*Resource providers* are versioned plugins for ~150 clouds / SaaS
APIs (terraform-bridged or natively written), and *Crosswalk for
AWS* / `awsx` ship higher-level "VPC with sane defaults" components
that collapse 30 raw resources to one constructor call. Secrets are
encrypted in state via a per-stack KMS / passphrase / cloud key, and
`pulumi import` adopts existing cloud resources into a program with
generated code.

## Install

```bash
# Homebrew (macOS / Linux)
brew install pulumi/tap/pulumi

# Direct binary install (Linux / macOS)
curl -fsSL https://get.pulumi.com | sh -s -- --version 3.232.0

# Windows (winget)
winget install Pulumi.Pulumi --version 3.232.0
```

## Example

```bash
# Scaffold a TypeScript program against AWS
mkdir infra && cd infra
pulumi new aws-typescript --name infra --stack dev

# Set the AWS region for this stack and a secret
pulumi config set aws:region us-west-2
pulumi config set --secret dbPassword "$DB_PASSWORD"

# Preview the diff, then apply
pulumi preview --diff
pulumi up --yes

# Adopt an existing S3 bucket into the program
pulumi import aws:s3/bucket:Bucket legacy my-existing-bucket

# Snapshot state, destroy the stack, remove it
pulumi stack export --file dev.checkpoint.json
pulumi destroy --yes && pulumi stack rm dev
```

## When to use

- You want loops, conditionals, real type-checking, and shared
  package abstractions across many similar resources without writing
  a Terraform module DSL on top of HCL.
- Your team already writes TypeScript / Python / Go and you want
  infra reviews to land in the same PR + CI as application code.
- You need to mix-and-match providers (AWS + Cloudflare + Datadog +
  Auth0) inside one resource graph with cross-references.

## When NOT to use

- Your team standardised on Terraform / OpenTofu and the existing
  module ecosystem (`terraform-aws-modules/*`) covers the surface —
  switching costs outweigh language ergonomics.
- You need a declarative-only artifact reviewers can read without a
  language runtime — plain `kustomize` / `helm` / Terraform HCL is
  the simpler answer.
- You want zero state management — pick CDK-for-Terraform-with-local-
  state or click-ops; Pulumi assumes a durable state backend.
