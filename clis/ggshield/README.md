# ggshield

## What it does
The official GitGuardian command-line client: a Python CLI that scans local
files, staged diffs, full git history, Docker images, PyPI packages, IaC
templates, and SCA dependency manifests for **leaked secrets** (API keys,
tokens, private keys, database URIs, cloud credentials — ~400+ detector
families) by sending normalized blobs to the GitGuardian Public Monitoring
API and rendering the matches as line-anchored findings. The intended wire-up
is `ggshield install -m local` (writes a `pre-commit` hook into the current
repo) or `ggshield install -m global` (writes a `~/.git/hooks/pre-commit`
template) so every commit and every `git push` is scanned before it ever
leaves the laptop, plus `ggshield secret scan ci` for GitLab CI / GitHub
Actions / Azure Pipelines / CircleCI / Jenkins to scan the full diff of
incoming MRs/PRs server-side.

## Why it's interesting
Different shape from `trufflehog` (entropy-and-regex scanner you run yourself
with no managed detector catalog), `gitleaks` (offline TOML-rule engine —
fast, but you maintain the rules), `ripsecrets` (Rust, zero-config, single
binary, ~100 detectors — choose for low-friction local scans without a SaaS
account), and `talisman` (commit-time only, regex-only, no live detector
updates). `ggshield` is the *managed-detector-catalog* option: you trade a
network call and a free-tier account for a continuously updated set of
high-signal detectors with low false-positive rates and per-secret
remediation guidance from the GitGuardian dashboard. Choose it when you have
a team that needs a single source of truth for "what secrets did we leak,
ever, anywhere" with assignable incidents; do **not** choose it for fully
offline / air-gapped scanning (use `gitleaks` or `ripsecrets`) or when you
cannot send file contents to a third-party API at all.

## Niche category
Pre-commit / CI secrets scanner — managed detector catalog over the
GitGuardian API.

## Repo
https://github.com/GitGuardian/ggshield

## Version pinned
`v1.46.0` (latest tagged release)

## License
- SPDX: `MIT`
- License file in upstream repo: `LICENSE`

## Install
```sh
brew install gitguardian/tap/ggshield
# or via pipx (cross-platform)
pipx install ggshield
# or pip
pip install ggshield

# One-time auth (opens browser to create a personal access token)
ggshield auth login

# Wire into the current repo as a pre-commit hook
ggshield install -m local
```

## Usage examples
```sh
# Scan everything currently staged (this is what the pre-commit hook runs)
ggshield secret scan pre-commit

# Scan a path on disk (no git involved)
ggshield secret scan path -r ./services

# Scan the full git history of the current branch
ggshield secret scan repo .

# Scan a specific commit range (e.g. a feature branch vs main)
ggshield secret scan commit-range origin/main..HEAD

# Scan a built Docker image for baked-in secrets
ggshield secret scan docker myorg/api:latest

# Scan a PyPI package before installing it
ggshield secret scan pypi requests==2.32.3

# CI mode — auto-detects the platform (GH Actions / GitLab / etc.)
ggshield secret scan ci

# IaC scan (Terraform / CloudFormation / Kubernetes / Helm)
ggshield iac scan all ./infra

# SCA scan (vulnerable dependencies in lockfiles)
ggshield sca scan all .
```
