# tenv

- **Repo:** https://github.com/tofuutils/tenv
- **Version:** v4.10.0 (latest stable)
- **License:** Apache-2.0 ([LICENSE](https://github.com/tofuutils/tenv/blob/main/LICENSE))
- **Language:** Go
- **Install:** `brew install tenv` · `go install github.com/tofuutils/tenv/v4/cmd/tenv@latest` · static binaries on the GitHub release page (`tenv_v4.10.0_Darwin_arm64.tar.gz`, `tenv_v4.10.0_Linux_amd64.tar.gz`, `.deb` and `.rpm` packages, `tenv_v4.10.0_Windows_amd64.zip`) · `aur/tenv-bin` on Arch · official container `ghcr.io/tofuutils/tenv:v4.10.0`

## What it does

`tenv` is a single-binary version manager that owns the *whole* HashiCorp / OpenTofu Infrastructure-as-Code toolchain on one machine: **OpenTofu, Terraform, Terragrunt, Terramate, and Atmos**. It is the explicit successor to the four-separate-tools world of `tfenv` (Terraform), `tofuenv` (OpenTofu), `tgenv` (Terragrunt), and a hand-rolled wrapper for Terramate / Atmos — same `.tool-versions`-shaped UX, but one Go binary, one config schema, one shim layer, and one upgrade path. Versions are pinned per project via any of the conventions tenv understands: a `terraform` / `tofu` / `terragrunt` / `terramate` / `atmos` line in `.tool-versions`, a `required_version` constraint inside the `terraform { }` block of the root module (tenv parses the HCL itself and resolves the highest version that satisfies it), a `.terraform-version` / `.opentofu-version` file, an env var (`TENV_AUTO_INSTALL=true`, `TFENV_TERRAFORM_VERSION=1.13.4`), or an explicit `tenv tf use 1.13.4` written to the local config. The shim binaries `terraform`, `tofu`, `terragrunt`, `terramate`, `atmos` live in `~/.tenv/bin/` and at exec time walk the cwd upward to find the controlling pin, then dispatch to the matching installed version under `~/.tenv/Terraform/<v>/`. Downloads are signature-verified end-to-end: HashiCorp releases against their PGP key, OpenTofu / Terragrunt / Terramate against cosign keyless attestations, and tenv refuses to install unsigned or signature-mismatched binaries unless you explicitly opt out with `--skip-signature`. The `tenv tf list-remote` / `tenv tofu list-remote` commands query upstream release indexes directly so you see real availability, not a cached snapshot. tenv is the tool the OpenTofu community itself recommends in its onboarding docs (the `tofuutils` GitHub org is run by OpenTofu maintainers), which is part of why it has displaced `tfenv` as the default in 2025–2026.

## When to pick it / when not to

Pick `tenv` whenever a repo or a laptop touches more than one of {Terraform, OpenTofu, Terragrunt, Terramate, Atmos} and you want one version-management story across the whole stack — especially relevant now that the Terraform → OpenTofu fork has split many shops into "some modules still on Terraform 1.5.x under BSL, some modules on OpenTofu 1.8+ under MPL", and the same engineer needs both binaries on PATH the same afternoon. Pick it on CI runners: `tenv tf install` (with no version argument) reads the module's `required_version` block and installs exactly the version the code asks for, no `setup-terraform@v3` action with a hard-coded version drifting against the repo. Pick it for security-conscious teams that already enforce signed-binary install policies — the default-on signature verification is meaningful (the alternative tools historically did `curl | tar`). Pair with [`asdf`](../asdf/) or [`mise`](../mise/) only if you already standardize on one of those across many languages and are willing to use the `asdf-tenv` / mise plugin instead — but in practice tenv standalone is lighter for IaC-only repos. Pair with [`tflint`](../tflint/), [`tfsec`](../tfsec/), and [`infracost`](../infracost/) downstream of the version-pin step. Skip tenv for shops on exactly one Terraform version forever (the official `terraform` binary plus a CI lock is simpler), for environments where a cluster operator already publishes a curated container image with the IaC binaries baked in (use the image, do not version-manage at runtime), and for teams that have committed to `mise` or `asdf` as the universal version manager for *all* languages (use the corresponding plugin there instead of layering two version managers).

## Vs already cataloged

- **Vs [`asdf`](../asdf/) / [`mise`](../mise/):** broader-vs-deeper. asdf and mise are universal version managers across hundreds of plugins (Node, Python, Ruby, Go, Java, plus the IaC tools). tenv covers only the IaC five but does it more carefully — first-class HCL `required_version` parsing, signature verification on by default, no plugin shim layer to misbehave. If a repo is purely IaC, tenv is the lighter answer; if the same laptop juggles 12 languages, asdf/mise wins.
- **Vs [`pkgx`](../pkgx/):** different model. pkgx is invocation-scoped and does not write a global pin; tenv writes a deterministic per-repo pin and a global default. For IaC, you almost always want the pin (a `terraform apply` against the wrong minor version will rewrite your state), so tenv's model fits better.
- **Vs [`pulumi`](../pulumi/):** orthogonal — Pulumi is a different IaC engine, not a version-managed Terraform. tenv has nothing to say about Pulumi; if you adopt Pulumi you do not need tenv (you need [`pulumi`](../pulumi/) plus a Node / Python / Go version manager).
- **Vs [`tflint`](../tflint/) / [`tfsec`](../tfsec/) / [`infracost`](../infracost/):** stack-mates. Those run on top of whichever Terraform / OpenTofu binary tenv resolves; they are complements, not substitutes.
- **Vs the legacy `tfenv` / `tofuenv` / `tgenv`:** tenv is the consolidated successor — same `.tool-versions` and `.terraform-version` conventions, but one binary instead of three Bash shim trees, with first-party signature verification, OpenTofu support not bolted on, and active maintenance from the OpenTofu maintainer org.

## Caveats

- **The default install of `tenv` does not auto-install missing versions on first invocation.** Set `TENV_AUTO_INSTALL=true` (or pass `--auto-install`) so that the first `terraform plan` in a fresh checkout fetches the pinned version instead of erroring with `version 1.13.4 not installed`. Most teams put this in CI and in the `.envrc` for local dev.
- **Signature verification requires network access to the upstream PGP / cosign attestation servers.** On air-gapped CI runners, mirror the keys / attestations alongside the binaries, or run `--skip-signature` only inside a controlled mirror that you have already signature-verified at fetch time.
- **The shim layer in `~/.tenv/bin` must come before the system `terraform` on PATH.** A leftover `/usr/local/bin/terraform` from a prior `brew install terraform` will silently win otherwise. After install, verify with `which terraform` → `~/.tenv/bin/terraform`.
- **HCL `required_version` resolution is strict.** A constraint like `>= 1.5, < 1.6` will resolve to the highest 1.5.x available; `~> 1.5.0` resolves to highest 1.5.x; if the constraint cannot be satisfied tenv errors out instead of silently picking something close. This is correct behavior but can surprise teams migrating from `tfenv`'s looser matching.
- **The Terraform → OpenTofu fork means license boundaries matter.** Terraform ≥ 1.6 is BSL-1.1, OpenTofu is MPL-2.0. tenv installs both happily; what you do with each binary is governed by the upstream license, not by tenv. Pick the one your legal posture permits and pin it explicitly in `.tool-versions`.
- Apache-2.0 ([LICENSE](https://github.com/tofuutils/tenv/blob/main/LICENSE)) — permissive; safe to redistribute the `tenv` binary inside internal developer-platform images and inside customer-shipped IaC bundles.

## Example invocations

```bash
# Install tenv itself
brew install tenv

# Turn on auto-install so first use of any tool fetches the pinned version
export TENV_AUTO_INSTALL=true

# Install a specific OpenTofu, Terraform, Terragrunt
tenv tofu install 1.8.5
tenv tf install 1.13.4
tenv tg install 0.66.9

# Pin per-project (writes .opentofu-version / .terraform-version)
cd ~/code/infra
tenv tofu use 1.8.5
tenv tf use 1.13.4

# Or pin via .tool-versions for asdf-style cross-tool config
cat > .tool-versions <<EOF
opentofu 1.8.5
terraform 1.13.4
terragrunt 0.66.9
EOF

# Let tenv resolve the version from the module's required_version block
cd ~/code/infra/prod
tenv tf detect          # prints which version satisfies required_version
tofu plan               # shim dispatches to the resolved version

# List what is installed and what is available upstream
tenv tofu list
tenv tofu list-remote | tail -20

# Update tenv itself (or `brew upgrade tenv`)
tenv update
```
