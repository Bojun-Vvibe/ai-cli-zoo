# trufflehog

> **A secrets scanner that not only finds high-entropy or
> regex-matched strings in code, git history, S3 buckets, Docker
> images, GitHub orgs, GitLab repos, Postman workspaces, Jira,
> Confluence, Slack, etc. — but actively *verifies* each candidate
> against the corresponding live API to eliminate false positives.**
> Pinned to **v3.95.2**,
> [LICENSE](https://github.com/trufflesecurity/trufflehog/blob/main/LICENSE),
> AGPL-3.0.

Source: <https://github.com/trufflesecurity/trufflehog>

## TL;DR

`trufflehog` is what you reach for when `gitleaks` / `detect-secrets`
/ `git-secrets` give you a wall of "looks-like-an-AWS-key" hits and
you need to know *which ones are actually live*. The killer feature
is **verification**: for ~800 detector types (AWS, GCP, GitHub,
Slack, Stripe, Twilio, Datadog, Postman, Notion, OpenAI,
Anthropic…), the matched string is sent to the corresponding API in
a read-only probe (e.g. `aws sts get-caller-identity` for AWS,
`/user` for GitHub) and only credentials that come back as
*currently valid* are flagged as `--results=verified`. The result
is a triageable list ("8 verified live secrets in this repo's git
history") instead of a pile of regex matches with three weeks of
manual audit ahead of you. Same Go binary scans local filesystems,
git repos (with full history), GitHub orgs, GitLab groups, S3
buckets, GCS buckets, Docker images, container registries, Postman
workspaces, Jira, Confluence, Slack workspaces, and Elasticsearch
indices.

## Install

```bash
# Homebrew (macOS / Linux)
brew install trufflehog

# Curl installer (Linux / macOS)
curl -sSfL https://raw.githubusercontent.com/trufflesecurity/trufflehog/main/scripts/install.sh \
  | sh -s -- -b /usr/local/bin

# Go (any platform with a Go toolchain)
go install github.com/trufflesecurity/trufflehog/v3@latest

# Docker
docker run --rm -v "$PWD:/pwd" trufflesecurity/trufflehog:latest \
  filesystem /pwd

# Pre-built binary
curl -L "https://github.com/trufflesecurity/trufflehog/releases/download/v3.95.2/trufflehog_3.95.2_$(uname -s)_$(uname -m).tar.gz" \
  | tar xz trufflehog && sudo mv trufflehog /usr/local/bin/

# Linux package managers
# Arch (AUR): yay -S trufflehog-bin
# Nix:        nix-env -iA nixpkgs.trufflehog

# verify
trufflehog --version    # trufflehog 3.95.2
```

## License

**AGPL-3.0** — see
[LICENSE](https://github.com/trufflesecurity/trufflehog/blob/main/LICENSE).
Running the binary against your own repos / cloud accounts is
unrestricted; the obligation triggers if you embed the source into
a hosted service you offer to third parties (read the licence).
Most teams never touch this.

## One Concrete Example

```bash
# 1. scan the local working tree (current directory) for verified secrets
trufflehog filesystem . --only-verified

# 2. scan an entire git repo's history for verified secrets
trufflehog git https://github.com/some-org/some-repo --only-verified

# 3. scan a local git repo from a specific commit forward
trufflehog git file://. --since-commit=HEAD~50 --only-verified

# 4. scan an entire GitHub organisation, including all repos, issues, PRs, gists
GITHUB_TOKEN=ghp_... trufflehog github --org=some-org --only-verified

# 5. scan a Docker image (running container or registry tag)
trufflehog docker --image=postgres:16 --only-verified
trufflehog docker --image=registry.example.com/team/app:v1.2.3 --only-verified

# 6. scan an S3 bucket
AWS_PROFILE=ro trufflehog s3 --bucket=my-bucket --only-verified

# 7. machine-readable output for CI / SIEM
trufflehog filesystem . --json --only-verified > findings.jsonl

# 8. CI gate (non-zero exit on any verified finding)
trufflehog filesystem . --only-verified --fail

# 9. install as a pre-commit hook (scans the diff)
cat >> .pre-commit-config.yaml <<'YAML'
- repo: https://github.com/trufflesecurity/trufflehog
  rev: v3.95.2
  hooks:
    - id: trufflehog
      entry: trufflehog git file://. --since-commit HEAD --only-verified --fail
YAML

# 10. exclude false-positive files (test fixtures, example configs)
cat > trufflehog-excludes.txt <<'EOF'
tests/fixtures/.*
docs/examples/.*
\.snap$
EOF
trufflehog filesystem . --exclude-paths=trufflehog-excludes.txt --only-verified
```

## Niche It Fills

**Verified-secret scanning across git history + cloud surfaces +
SaaS workspaces in one binary.** The space is split: `gitleaks` and
`git-secrets` are git-only and regex-only (fast but high
false-positive rate); cloud-secret scanners (AWS Macie, GCP DLP)
cover one cloud each and are not source-code-aware; SaaS-specific
scanners (Slack DLP, Postman secret detection) cover one SaaS each.
`trufflehog` consolidates all of them into one Go binary with
verification across ~800 detectors, so a single CI step can scan
the repo + its docker image + the org's S3 buckets + the team's
Postman workspace + Slack channels and emit one machine-readable
findings file.

## Why use it

Three things `trufflehog` does that regex-only secret scanners do
not:

1. **Live verification eliminates false positives.** Each detector
   is a `(regex, verifier)` pair. The regex catches the candidate;
   the verifier makes a read-only API probe to confirm the
   credential is currently valid (`aws sts get-caller-identity`,
   GitHub `/user`, Slack `auth.test`, Stripe `/v1/charges?limit=1`,
   etc.). `--only-verified` drops everything else, so the output
   is "8 live secrets" not "800 things that look like AWS keys".
   On a typical legacy repo, verified-only filtering removes
   ~90–99 % of regex matches, making the remainder actionable.
2. **Source-agnostic engine, ~25 source types.** The same scan
   pipeline runs against local filesystem, git (with full history
   walk + per-commit detection), GitHub (repo + org + issues + PRs
   + gists + wikis), GitLab, Bitbucket, S3, GCS, Azure Blob,
   Docker images, container registries, Postman, Jira, Confluence,
   Slack workspaces, Elasticsearch indices, Jenkins, Travis,
   CircleCI logs, and SQL dumps. Output schema is identical
   across sources, so a SIEM ingester is one shape not 25.
3. **Detector catalog is plugin-shaped and large.** ~800 detectors
   covering most major SaaS (and growing); contributing a new
   detector is a Go interface implementation (`Type()`, `Keywords()`,
   `FromData()`, `Verify()`). Lets the catalog keep up with new
   SaaS vendors at community speed instead of waiting for the
   next CLI release of a regex-only scanner.

For an LLM-CLI workflow, `trufflehog` is the right pre-publish
gate to run *after* the model emits a generated PR: scan the
working tree for any keys / tokens / connection strings the model
may have hallucinated from training data or pasted from a previous
context, and fail the gate before the PR opens.

## Vs Already Cataloged

- **Vs `gitleaks` (if cataloged):** different sweet spots.
  `gitleaks` is faster on pure-regex scanning of git history, has
  a lighter binary, and is the default in many `pre-commit`
  setups. `trufflehog` is the right answer when **verified** vs
  unverified matters (post-incident audit, SaaS-key sweep, cloud
  bucket scan) — the live API probes are what `gitleaks` does
  not do. Pair: `gitleaks` on every commit (cheap, fast),
  `trufflehog --only-verified` on a nightly schedule (broader
  surface, more accurate signal).
- **Vs [`lefthook`](../lefthook/), [`cocogitto`](../cocogitto/):**
  orthogonal — they are git workflow tools; `trufflehog` is the
  scanner you would *run from* a `lefthook.yml` `pre-push` hook.
- **Vs [`age`](../age/), [`sops`](../sops/):** complementary.
  `age` / `sops` are encryption-at-rest for secrets you keep on
  purpose; `trufflehog` finds the secrets you did *not* mean to
  commit. A typical workflow uses both: `sops` to manage real
  config, `trufflehog` in CI to catch the ones that slipped past.
- **Vs cloud-native scanners (AWS Macie, GCP DLP, Azure Purview):**
  `trufflehog` is multi-cloud and source-code-aware in one binary,
  cheaper to run on dev infrastructure, and actionable from a
  laptop. Cloud-native scanners are richer for at-rest data
  classification across managed services where IAM-driven scope
  matters more than CLI ergonomics.

## Caveats

- **AGPL-3.0 licence.** Fine for using the binary against your
  own repos / cloud accounts from CI; has obligations if you
  embed `trufflehog` source into a SaaS product you offer to
  third parties. The standalone binary on a CI runner does not
  trigger the network-copyleft clause; modifying the binary and
  exposing it as a hosted scanner does.
- **Verification makes outbound API calls from wherever you
  run.** `--only-verified` *will* call AWS / GitHub / Slack /
  Stripe / etc. with the candidate credential. Run from a
  network that allows that egress (the candidate creds need
  reachability to their issuer). For air-gapped scans, drop
  `--only-verified` and accept the higher false-positive rate.
- **Scanning rate-limits matter.** Verifying thousands of
  candidate GitHub tokens in one scan can hit GitHub's
  unauthenticated rate-limit; pass `--concurrency=8` or smaller
  in CI environments where you do not own the upstream
  rate-limit budget.
- **Detector recall has gaps.** ~800 detectors is a lot but not
  every SaaS is covered; for proprietary internal credential
  formats (custom JWT shapes, internal API tokens), write a
  `customDetector` regex (no verification) and accept the
  false-positive rate for those, or contribute a new detector
  upstream.
- **Verification is a side-effect.** A "verified GitHub PAT"
  finding means trufflehog *just used* that token to call
  `/user`. Some compliance regimes (PCI, HIPAA on certain
  surfaces) require that secret-scanning be non-touching; if
  that is your environment, scan with `--no-verification` and
  treat all findings as candidates.
