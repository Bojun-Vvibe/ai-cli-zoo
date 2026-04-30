# tofu (OpenTofu)

- **Repo:** https://github.com/opentofu/opentofu
- **Version:** v1.11.6
- **License:** [LICENSE](https://github.com/opentofu/opentofu/blob/main/LICENSE) (MPL-2.0)
- **Category:** Infrastructure as Code (IaC)

## What it is

OpenTofu is a community-driven, Linux Foundation–hosted fork of Terraform created
after HashiCorp relicensed Terraform to the BSL. It speaks the same HCL
configuration language, consumes the same provider protocol, and reads existing
Terraform state files, so most modules transfer over by swapping the `terraform`
binary for `tofu`.

## Why it's interesting

- **Drop-in replacement** for the last MPL-licensed Terraform release with full
  HCL + provider compatibility, removing BSL exposure for downstream tooling.
- **State encryption built in** (client-side AES-GCM with KMS / PBKDF2 key
  providers) — a feature HashiCorp keeps gated behind Terraform Enterprise.
- **Provider-defined functions, early variable evaluation, and `for_each` in
  module sources** — language features that ship faster than upstream now that
  governance is foundation-led rather than vendor-led.
