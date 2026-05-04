# opentofu

> **Open-source, community-governed fork of Terraform shipping as a single Go
> binary `tofu`.** Drop-in for the Terraform CLI surface (`init` / `plan` /
> `apply` / `destroy` / `state` / `import` / `workspace`), reads the same
> HCL2 modules, talks to the same provider plugin protocol, and stores the
> same state file format — so an existing `*.tf` tree migrates with no
> rewrite. Pinned to **v1.11.6** (released 2026-04-08, SPDX: `MPL-2.0`,
> [LICENSE](https://github.com/opentofu/opentofu/blob/main/LICENSE)).

Source: <https://github.com/opentofu/opentofu>

## Repo

- URL: <https://github.com/opentofu/opentofu>
- Owner/org: opentofu (Linux Foundation project)
- License file: [LICENSE](https://github.com/opentofu/opentofu/blob/main/LICENSE)

## Version

`v1.11.6` — released 2026-04-08. Verify with `tofu version`. The 1.x line
maintains state-file compatibility with Terraform 1.5.x and continues to
diverge feature-wise (state encryption at rest, provider-defined functions,
`for_each` on providers, dynamic provider iteration, removed-block early
evaluation) on a separate roadmap from HashiCorp's BUSL-1.1 Terraform.

## License

**MPL-2.0** — OSI-approved, file-level copyleft. Safe to embed in CI runner
images, vendor into a build pipeline, distribute as part of an internal
platform binary. The fork was created precisely to keep the IaC tool under
an OSI license after Terraform moved to BUSL-1.1 in 1.6, so legal teams that
require OSI for foundational tooling can adopt opentofu without a license
review every release.

## What it does

`tofu init` resolves module sources + provider plugins from the OpenTofu
Registry (mirror of the Terraform Registry plus opentofu-specific
contributions) into `.terraform/`. `tofu plan` reads the HCL graph, queries
each provider for current state of the resources it manages, and prints the
diff between desired (the HCL) and actual (provider-reported). `tofu apply`
executes that diff, writes the new state to the configured backend (local
file, s3, gcs, azurerm, http, postgres, kubernetes, consul, …), and exits.
The state file may be encrypted at rest (1.7+) with a key from PBKDF2 / AWS
KMS / GCP KMS / OpenBao / Vault — a feature Terraform proper does not have.

The wire-compatible model means every existing provider in the Terraform
Registry (aws, google, azurerm, kubernetes, helm, github, datadog, cloudflare,
vault, …) works out of the box without a republish, and the opentofu
Registry mirrors them on first request.

## When to use

- **Your legal team requires OSI-approved licenses for foundational tooling.**
  BUSL-1.1 Terraform fails this gate; opentofu (MPL-2.0) passes.
- **You want state encryption at rest as a first-class feature** without
  wrapping the backend in an external KMS shim.
- **You want provider-defined functions** (`provider::aws::arn_parse(...)`)
  to push computation into the provider rather than into module HCL gymnastics.
- **You are starting a new IaC repo** and the choice between Terraform and
  opentofu is open. The community-governance posture (LF project, public
  RFCs, no single-vendor roadmap control) is the differentiator versus
  Terraform's vendor-controlled BUSL trajectory.
- **You want a drop-in for an existing Terraform 1.5.x tree.** State format
  compatibility plus identical CLI surface make migration mechanical.

## When NOT to use

- **You depend on a Terraform-only feature added after the fork point**
  (HCP Terraform / Terraform Cloud-specific workflows, certain newer
  Terraform-proper modules). Check feature parity for your specific stack
  before migrating.
- **The tool is the wrong layer.** For per-cluster Kubernetes state use
  GitOps controllers ([`flux`](../flux/) / [`argocd`](../argocd/)) reading
  rendered manifests; for cloud-resource-shaped IaC at programming-language
  expressiveness use [`pulumi`](../pulumi/).
- **You only have a handful of resources and a shell script works.** IaC
  pays back at scale; do not adopt for two S3 buckets.

## Alternatives in this catalog

- [`tenv`](../tenv/) — version manager that pins `tofu` (and `terraform` /
  `terragrunt` / `terramate` / `atmos`) per repo via `.opentofu-version` /
  HCL `required_version`. Use tenv to *manage* tofu the way nvm manages
  node — they compose, not compete.
- [`pulumi`](../pulumi/) — IaC in real programming languages (TS / Python /
  Go / .NET / Java) instead of HCL. Pick when the abstraction surface needs
  loops + classes + libraries that HCL strains under.
- [`tflint`](../tflint/), [`tfsec`](../tfsec/), [`checkov`](../checkov/),
  [`infracost`](../infracost/) — downstream of tofu in the IaC pipeline:
  lint HCL, scan for misconfig, estimate cost. All work on opentofu-flavored
  HCL since the language is the same.
- [`atmos`](../atmos/) — opinionated wrapper for managing many tofu/terraform
  stacks across many environments. Pick when the per-env values + stack
  composition story is the dominant cost.
- [`vault`](../vault/) / OpenBao — secrets backend tofu can pull provider
  credentials and resource inputs from. OpenBao is the same OSI-fork story
  one layer over (BUSL-1.1 Vault → MPL-2.0 OpenBao under LF governance).
