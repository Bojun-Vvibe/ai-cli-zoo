# forgejo

- **Repo:** https://codeberg.org/forgejo/forgejo
- **Version:** v15.0.1 (latest stable, 2026-04-29)
- **License:** GPL-3.0-or-later ([LICENSE](https://codeberg.org/forgejo/forgejo/src/branch/forgejo/LICENSE))
- **Language:** Go (server) + TypeScript (web UI)
- **Install:** `brew install forgejo` · static binary releases on the Codeberg release page (`forgejo-15.0.1-linux-amd64`, `forgejo-15.0.1-darwin-arm64`, etc., plus signed `.asc` and `.sha256`) · official Docker image `codeberg.org/forgejo/forgejo:15` · Helm chart from `https://code.forgejo.org/forgejo-helm/forgejo` · Debian `.deb` from the Forgejo APT repo

## What it does

`forgejo` is a self-hosted git forge — repositories, pull requests
(called "pull requests" here, not "merge requests"), issues, wiki,
releases, packages registry (npm / Maven / PyPI / Container / Helm /
Composer / Cargo / Conan / Generic / RPM / Debian), CI (`forgejo
actions`, GitHub Actions–compatible runners), code review with inline
comments, fork-and-PR workflow, organizations and teams with RBAC,
SSH + HTTP(S) git transports, signed-commit verification (GPG and SSH
keys), webhook fan-out, OAuth2 provider mode, OIDC / LDAP / SAML SSO,
two-factor auth (TOTP + WebAuthn), federation experiments (ActivityPub,
work-in-progress), and a single-binary install that boots in seconds
on a $5 VPS or a Raspberry Pi. Forgejo is the soft-fork of Gitea that
the Gitea community spun up in late 2022 after Gitea's main repo was
transferred to a for-profit company; the Forgejo project is now a
project of Codeberg e.V., a German non-profit, and is the upstream
the Codeberg public hosting service runs on. v15 (April 2026) is the
current stable line, with hard fork from Gitea now complete (the
codebases have diverged enough that "Forgejo is just Gitea with a new
logo" stopped being true around v9 / v10 — v15 ships features Gitea
does not have, including Forgejo Actions runners, the federation
research, code-search built in, and a redesigned package registry).
The whole forge runs from a single ~150 MB Go binary plus a SQLite,
PostgreSQL, MySQL, or TiDB database; storage is the database plus a
data directory of bare git repos and uploaded artifacts.

## When to pick it / when not to

Pick `forgejo` when you want to self-host a real git forge — issues,
PRs, code review, CI, packages — under your own roof, on a budget that
fits a small VPS, with a non-profit governance model that makes
"vendor pivots into pricing changes" structurally impossible. The
single-binary install means a fresh forge takes 10 minutes from `wget`
to the first repo push, and the Forgejo Actions runner means CI for
that forge is one more binary on the same box (or a worker pool on
others). Pair with [`gitea-mirror`] / built-in mirror feature for
two-way mirroring with GitHub, with [`drone`] (or Forgejo Actions
itself) for CI, with [`gh`](../gh/)-style ergonomics via the `tea` /
`forgejo-cli` companion for terminal git workflows, and with
[`woodpecker-ci`] when you want a more powerful CI engine bolted on
to Forgejo's webhook surface. The community is small but engaged and
ships a release every ~6 weeks; the public Codeberg instance is the
proof point that the same software runs at the scale of the entire
free-software hobbyist world.

Skip it if you need GitHub-grade ecosystem integration — Marketplace
apps, Copilot-style AI features, deep CodeQL, Dependabot at the
GitHub-scale ML quality, third-party SaaS integrations that ship a
"Connect to GitHub" button — none of that exists outside GitHub
itself. Skip it for very large teams (hundreds of devs, terabytes
of LFS, thousands of concurrent CI jobs) where GitLab Enterprise or
GitHub Enterprise are the realistic benchmarks; Forgejo scales to a
mid-sized engineering org but does not pretend to compete on the
biggest-customer axis. Skip the upstream Gitea fork instead if your
infra team already standardised on Gitea and you have not hit a
problem the Forgejo divergence solves; the two forges are still
import-compatible at the database level, but each release widens the
gap. Skip the federation features for now — ActivityPub support is
explicitly research, not a recommended production setup.

## Example invocations

```bash
# Boot a single-node forge from a downloaded binary
wget https://codeberg.org/forgejo/forgejo/releases/download/v15.0.1/forgejo-15.0.1-linux-amd64
chmod +x forgejo-15.0.1-linux-amd64
./forgejo-15.0.1-linux-amd64 web --config /etc/forgejo/app.ini
# Then open http://<host>:3000 and complete the setup wizard

# Boot via Docker for evaluation (data persists in ./forgejo-data)
docker run -d --name forgejo \
  -p 3000:3000 -p 222:22 \
  -v $(pwd)/forgejo-data:/var/lib/forgejo \
  codeberg.org/forgejo/forgejo:15

# Create the initial admin user from the CLI (no signup / no email needed)
./forgejo admin user create \
  --admin --username root --password 'CHANGE-ME' --email root@example.com

# Run a built-in maintenance task: re-index all repos for the search engine
./forgejo admin reindex code

# Dump everything (db + repos + attachments) for backup
./forgejo dump --type tar.zst -f forgejo-backup-$(date +%F).tar.zst

# Run a Forgejo Actions runner (separate binary, register against the forge)
forgejo-runner register --instance https://forge.example.com \
  --token <runner-token> --name builder-1 --labels ubuntu-latest:docker://node:22
forgejo-runner daemon

# Use the official tea client to open a PR from the terminal
tea login add --url https://forge.example.com --token <user-token>
tea pr create --base main --head feature/x --title "feat: add x" --description "..."
```
