# noseyparker

> **A fast, signature-driven secret detector for source-code
> trees, raw git histories, GitHub orgs, S3 buckets, JIRA
> issues, and Slack archives** — a single Rust binary with
> Hyperscan / vectorscan-accelerated regex matching, ~170
> built-in rules, and a denormalized SQLite "datastore" so a
> 50 GB monorepo with 200k commits scans in minutes and
> reruns hit cache. Pinned to **v0.24.0**
> ([LICENSE](https://github.com/praetorian-inc/noseyparker/blob/main/LICENSE),
> Apache-2.0).

Source: <https://github.com/praetorian-inc/noseyparker>

## TL;DR

`noseyparker` is the secret scanner you reach for when the
target is bigger than a single working tree. Point it at a
local checkout *or a bare git repo* and it walks every blob
ever committed (including ones reachable only from deleted
branches and dangling commits) through a Hyperscan multi-
pattern engine — the same DFA library suricata / snort use
— so 170 rules run in roughly the time it takes plain
`ripgrep` to do one. Findings land in a `--datastore`
directory (SQLite + content-addressed blob store), so
`noseyparker scan` is incremental: a daily cron rescans the
deltas in seconds instead of re-walking the history. Inputs
are a feature, not an afterthought: `noseyparker scan
--github-org=foo` enumerates every public + accessible-
private repo in an org via the GitHub API and scans them
with one command, and v0.20+ added first-class scanners for
GitHub Issues / Discussions / PR bodies, JIRA tickets, S3
prefixes, and Slack export `.zip`s — the places where the
"git push --force-with-lease deleted it" credential actually
lives. Output ships as JSON, JSONL, SARIF (for code-
scanning UIs), and a human report; v0.24 added Hashicorp
Vault token rules + Sourcegraph / Auth0 / Postmark / Tavily
patterns.

## Install

```bash
# Homebrew (macOS / Linux)
brew install noseyparker

# Single-binary download (GitHub releases, ~25 MB)
curl -LO https://github.com/praetorian-inc/noseyparker/releases/download/v0.24.0/noseyparker-v0.24.0-aarch64-apple-darwin.tar.gz
tar xzf noseyparker-v0.24.0-aarch64-apple-darwin.tar.gz
sudo mv bin/noseyparker /usr/local/bin/

# Docker (multi-arch)
docker pull ghcr.io/praetorian-inc/noseyparker:v0.24.0

# Build from source (Rust >= 1.74, requires Hyperscan/vectorscan)
git clone --depth 1 --branch v0.24.0 \
  https://github.com/praetorian-inc/noseyparker.git
cd noseyparker && cargo build --release
```

## Usage

```bash
# Scan a working tree + the .git history backing it
noseyparker scan --datastore ./np.ds .

# Rescan tomorrow — only new commits + changed blobs are walked
noseyparker scan --datastore ./np.ds .

# Human-readable report
noseyparker report --datastore ./np.ds

# Machine-readable for CI / dashboards
noseyparker report --datastore ./np.ds --format sarif > findings.sarif
noseyparker report --datastore ./np.ds --format jsonl | jq .

# Enumerate + scan an entire GitHub org
GITHUB_TOKEN=ghp_… \
  noseyparker scan --datastore ./np.ds --github-org example-org

# Scan a Slack workspace export (zip from the admin console)
noseyparker scan --datastore ./np.ds slack-export-2025-04-01.zip
```

## Why it's interesting

The crowded space of secret scanners — [`gitleaks`](../gitleaks/),
[`trufflehog`](../trufflehog/), [`ripsecrets`](../ripsecrets/),
[`talisman`](../talisman/), [`ggshield`](../ggshield/) — almost
all model the problem as "scan a working tree, optionally
walk recent commits". `noseyparker` is the one designed
around the assumption that the secret you care about was
**force-pushed away years ago and only survives in a dangling
blob in someone's fork**, and that you want to rescan the
whole world weekly without paying full price each time. Three
things make it different in practice: (1) Hyperscan's multi-
pattern matcher means rule count doesn't add linear cost, so
the default 170-rule corpus is fast enough to leave on; (2)
the content-addressed SQLite datastore makes rescans
incremental and findings deduplicated across history /
forks / mirrors; (3) the input adapters (`--github-org`,
JIRA, Slack export, S3 prefix, generic enumerator) move
this from "developer pre-commit hook" into "security-team
fleet inventory tool". Pair it with [`gitleaks`](../gitleaks/)
or [`talisman`](../talisman/) on the developer boundary;
reach for `noseyparker` when the question is "what's
already leaked across all 4,000 repos in our org and every
fork of them?". Not a replacement for runtime credential
rotation — finding a key is step one, revoking it is step
two — but the missing inventory tool that tells you which
keys to revoke first.
